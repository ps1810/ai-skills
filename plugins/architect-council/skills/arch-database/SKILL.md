---
name: arch-database
description: Architect-level database design — engine selection, schema and data modelling, indexing, query patterns, transactions and isolation levels, migrations at scale, partitioning and sharding, replication and read scaling. Use this when designing a feature that adds tables or changes a schema, when the user mentions SQL, Postgres, MySQL, MongoDB, DynamoDB, indexes, migrations, transactions, N+1 queries, or slow queries. The design-review skill loads this when it selects the database perspective.
---

# Database architecture

Schema decisions are the most expensive ones to reverse. A cache TTL is a config change; a column type on a 500-million-row table is a maintenance window and a migration plan. Spend the review's attention proportionally, and be explicit about which decisions are cheap to get wrong.

## Start from the query patterns

Model the access patterns before the entities. A schema designed from an entity diagram without knowing the queries produces tables that are correct and slow.

For the feature, list:
- Every read: its filter, sort, projection, and expected result size.
- Every write: single-row or multi-row, and what must commit atomically with it.
- Expected volume per pattern, and the growth rate of the largest table.
- Which reads are latency-critical and which can be async.

Then check each read has an index that serves it. A read with a filter on an unindexed column on a growing table is a future incident with a known date.

## Engine selection

Default to PostgreSQL unless something specific rules it out. It handles relational, JSON, full-text, geospatial, and time-series adequately, which means one operational surface instead of four. Introducing a second store is a real, permanent cost — backups, monitoring, failover, expertise, and consistency across the boundary — so require a specific reason.

| Engine | Choose when | Real cost |
|---|---|---|
| PostgreSQL | Default. Relational data, transactions, mixed workloads, JSONB for semi-structured | Vertical scaling limits on writes; connection overhead needs a pooler |
| MySQL / MariaDB | Existing expertise, tooling built around it, some managed-service edges | Weaker JSON, historically looser DDL semantics |
| SQLite | Embedded, single-writer, edge, tests | One writer; not for multi-instance services |
| DynamoDB / Cassandra | Known access patterns, need predictable latency at very high write volume | Queries you did not design for are impossible or expensive; no joins; eventual consistency by default |
| MongoDB | Genuinely document-shaped aggregates read and written whole | Cross-document transactions are awkward; teams often want a relational model and fight the engine |
| ClickHouse / BigQuery | Analytical scans over large volumes | Not for transactional writes or point updates |
| Elasticsearch / OpenSearch | Full-text relevance ranking, faceted search | A secondary index that can drift from the source of truth; not a system of record |
| Redis / Valkey | Ephemeral state, counters, rate limits, queues | Memory-bound; persistence is a durability compromise |
| Neo4j / graph | Traversal depth is the query, not a join | Narrow; a recursive CTE covers many "graph" cases |
| pgvector / dedicated vector store | Similarity search over embeddings | pgvector keeps one operational surface and is adequate to low millions of vectors with HNSW; a dedicated store (Pinecone, Qdrant, Weaviate) is a second system with its own consistency boundary, justified by scale or filtering needs pgvector cannot meet |

Two questions that settle most engine debates: **do you need a query you have not designed for yet** (points to relational) and **do you need write throughput beyond what one primary can take** (points to a partitioned store). Most systems asking for the second one have not yet measured the first.

## Schema design

**Normalize first, denormalize with evidence.** Normalization keeps invariants enforceable by the database. Denormalize when a measured read pattern requires it, and when you do, name what now maintains the duplicate and what happens when that fails.

**Get types right the first time.** These are the ones that hurt to change later:

- Money: never floating point. `NUMERIC`/`DECIMAL`, or integer minor units with the currency stored alongside.
- Timestamps: `TIMESTAMPTZ` (or UTC with an explicit convention). A naive local timestamp is a bug that surfaces at a DST boundary.
- Identifiers: bigint or UUID. UUIDv4 as a primary key fragments B-tree indexes and hurts insert locality — prefer UUIDv7 or ULID for time-ordered UUIDs, or a bigint surrogate with a separate external UUID.
- Enums: a lookup table or a constrained text column. Native enum types are awkward to alter.
- Text: no arbitrary `VARCHAR(255)` — either a real business constraint or unbounded `TEXT`.

**Constraints in the database, not only the application.** `NOT NULL`, foreign keys, `UNIQUE`, `CHECK`. Application-only validation means every path — migrations, admin scripts, the second service, the manual fix during an incident — is a way to violate the invariant. A design that omits constraints because "the ORM handles it" is a finding.

**Soft deletes need a decision, not a default.** `deleted_at` means every query needs the predicate and every unique index needs to account for it. It also conflicts with GDPR erasure — a soft-deleted row is retained data. Decide per table whether you need history, and if you do, consider an explicit archive rather than a flag.

**JSONB is for genuinely schemaless attributes**, not for avoiding migrations. Data that is queried and filtered belongs in columns; JSONB gives up type checking, constraints, and clear indexing. A JSONB column that every query digs into with `->>` is a schema that was not designed.

## Multi-tenancy in the schema

The security skill asks whether the tenant predicate can be omitted; this is where you make sure it cannot.

- **State the model**: shared tables with a `tenant_id` column (the default), schema per tenant (isolation, at the cost of N× migrations and a connection-routing layer), or database per tenant (only for contractual isolation or very large tenants).
- **`tenant_id` is the leading column of every composite index and every unique constraint.** `UNIQUE (email)` is a cross-tenant collision; `UNIQUE (tenant_id, email)` is the invariant you meant.
- **Row-level security** (Postgres `CREATE POLICY` with `current_setting('app.tenant_id')`) makes the predicate structural: a query without it returns nothing rather than everything. It costs a `SET` per connection checkout and some planner overhead; it is worth it whenever more than one code path builds queries.
- **Per-tenant limits** — row counts, storage, query cost — need a place to live and a job that enforces them, or one tenant's growth becomes everyone's incident.
- **Noisy-neighbour queries**: a hot tenant on a shared table degrades everyone. Partitioning by tenant (list or hash) bounds the blast radius and makes per-tenant export and deletion a partition operation.

## Retention and archival

Every table that grows needs a stated answer to "what removes old rows," decided at design time. The default answer, "nothing," is how a ten-million-row table becomes a billion-row table with the same indexes and a five-minute `DELETE`.

- **Time-partition anything append-heavy** (events, logs, audit, metrics) and drop partitions on schedule. `DROP PARTITION` is instant and generates no WAL; `DELETE ... WHERE created_at < ?` on the same data is a multi-hour lock-and-bloat exercise.
- **Archive before delete** when the data has value but no query path: copy to object storage in Parquet, then drop. Name what reads the archive and how.
- **Retention is a compliance number, not a capacity number**, for regulated data. The security skill sets the maximum; you implement it, and the implementation is a job with monitoring, not an intention.
- **Soft-deleted rows are retained data.** If the compliance answer is "delete," `deleted_at` is not deletion.

## Indexing

- Composite index column order follows the query: equality columns first, then range, then sort. `(tenant_id, status, created_at)` serves `WHERE tenant_id = ? AND status = ? ORDER BY created_at`.
- Every foreign key wants an index on the referencing side. Postgres does not create it automatically, and its absence makes deletes on the parent table scan the child.
- Covering indexes (`INCLUDE`) let an index-only scan avoid the heap for hot reads.
- Partial indexes (`WHERE deleted_at IS NULL`) are much smaller and often exactly what a soft-delete schema needs.
- Every index costs write throughput and storage. Unused indexes are a pure cost — check `pg_stat_user_indexes` before adding more.
- Read the plan, do not guess. `EXPLAIN (ANALYZE, BUFFERS)` on realistic data volumes. A plan on a thousand rows tells you nothing about a plan on ten million.

## Transactions and isolation

State the isolation level the design assumes. Postgres defaults to Read Committed, which permits non-repeatable reads and write skew — and most application code is written as if it were Serializable.

- **Read Committed**: each statement sees a fresh snapshot. A read-then-write sequence can be based on data that changed underneath it.
- **Repeatable Read** (Postgres: snapshot isolation): consistent snapshot for the transaction; prevents non-repeatable reads, still permits write skew.
- **Serializable**: full correctness, at the cost of serialization failures the application must retry. If you choose it, the retry loop is part of the design, not an afterthought.

Concrete checks:

- **Check-then-act is a race.** `SELECT` to see whether a row exists, then `INSERT`. Use `INSERT ... ON CONFLICT` or a unique constraint. This is the single most common concurrency bug in application database code.
- **Balance and quota checks** need `SELECT ... FOR UPDATE`, a constraint the database enforces, or Serializable with retries. Reading a balance and then writing a decrement is a double-spend under concurrency.
- **Keep transactions short and do no I/O inside them.** An HTTP call inside a transaction holds locks for the duration of a remote timeout.
- **Lock ordering.** Two code paths updating the same two rows in different orders deadlock. Impose an order and document it.
- **Idempotency for anything a client can retry.** An idempotency key with a uniqueness constraint, not a best-effort check.

## Migrations at scale

The rule that matters: **on a large table, a migration that takes a lock is an outage.**

- **Every migration sets `lock_timeout` (a few seconds) and `statement_timeout`.** A DDL statement that needs an `ACCESS EXCLUSIVE` lock queues behind any long-running transaction, and every query that arrives after it queues behind the DDL. With no `lock_timeout`, a migration waiting on one slow report blocks the entire table until the report finishes. With `lock_timeout = '5s'`, it fails fast and you retry. This is the single cheapest protection in this document and the one most often missing.
- Expand-migrate-contract for anything breaking: add the new column nullable, backfill in batches, dual-write, switch reads, then drop the old. Multiple deploys, deliberately.
- `CREATE INDEX CONCURRENTLY` on Postgres. A plain `CREATE INDEX` blocks writes for the duration.
- Adding a `NOT NULL` column with a default rewrote the whole table on older Postgres; on 11+ a constant default is metadata-only. Know your version.
- Backfill in batches with a bounded rate. A single `UPDATE` over ten million rows holds a long transaction, bloats WAL, and blocks vacuum.
- Every migration needs a rollback plan. A migration that drops a column has no rollback once the data is gone — that is what the contract phase is for, run a release later.
- Test migrations against production-scale data volumes, not a seed dataset. A migration that runs in 200 ms on a laptop can run for six hours on the real table.

## Scaling

In order — do not skip steps, because each later one adds permanent complexity:

1. **Index and query fixes.** Most "we need to shard" conversations end here.
2. **Connection pooling.** Postgres connections are processes; a service with 200 replicas each holding 20 connections will exhaust the server. PgBouncer or the managed equivalent. Note that transaction-mode pooling breaks session state, prepared statements, and advisory locks.
3. **Read replicas.** Cheap read scaling, at the price of replication lag. The trap is read-after-write: a user writes, is routed to a replica, and does not see their own change. Route reads that follow a write to the primary, or use a session token / LSN-based read.
4. **Vertical scaling.** Frequently the cheapest real answer. A larger instance is a config change; sharding is a project.
5. **Partitioning.** One logical table split by range or list within one database. Excellent for time-series with a retention policy — dropping a partition is instant where a `DELETE` is not. The partition key must appear in queries or you scan every partition.
6. **Sharding.** Multiple databases. Cross-shard queries and cross-shard transactions become application problems, and rebalancing is hard. Requires a shard key that distributes evenly and appears in nearly every query. Treat as a last resort with a specific, measured justification.

## N+1 and query patterns

- Loading a collection then querying per item. Batch it, join it, or use `WHERE id = ANY($1)`. ORMs make this easy to write accidentally — check the generated queries, not the code.
- Unbounded result sets. Every list endpoint needs a limit. Offset pagination degrades on deep pages (`OFFSET 100000` scans and discards); prefer keyset pagination on an indexed, stable sort.
- `SELECT *` across a service boundary couples the caller to the schema and drags large columns over the wire.
- Aggregations over full tables on the request path. Precompute, or use a materialized view with a stated refresh strategy.

## Output

```markdown
## Access patterns
| Pattern | Read/Write | Filter & sort | Volume | Latency need | Index serving it |

## Engine
<Choice, why, and what would change it.>

## Schema
<DDL or a clear table description. Types, constraints, keys, indexes.>

## Transactions
<What must commit atomically, isolation level assumed, and the concurrency
hazards handled: check-then-act, quota checks, lock ordering, idempotency.>

## Migration plan
<Steps, locking behaviour, backfill strategy, rollback, expected duration at
production volume.>

## Scaling path
<What the design does at 10x. Which step in the ladder is next, and the signal
that says it is time.>

## Retention
| Table | Grows with | Retention | Mechanism |

## BLOCKING / WARNING / CONSIDER

## Reversibility
| Decision | Cost to change later |
```

When invoked by `design-review`, return only the BLOCKING / WARNING / CONSIDER findings plus the access-patterns and reversibility tables; the coordinator assembles the document.

## Calibration

Severity levels are defined once in `design-review`. In this domain, reserve BLOCKING for: unindexed queries on tables that will grow, a check-then-act race on data that matters, a migration that locks a large table without a `lock_timeout`, money in a float, missing constraints on an invariant the business depends on, a unique constraint that omits `tenant_id` in a multi-tenant schema, and a schema that cannot answer a query the feature requires.

The highest-value finding in most database reviews is **an access pattern the schema cannot serve efficiently** — caught at design time it is a schema change on an empty table, and caught in production it is a migration with a maintenance window.

| Not a finding | A finding |
|---|---|
| "Consider adding indexes for performance." | "`GET /orders?status=open` filters `orders` by `status` with no index; at the stated 50M rows and 200 rps that is a sequential scan per request. Add `(tenant_id, status, created_at)`." |
| "The migration may lock the table." | "`ALTER TABLE orders ADD COLUMN ... NOT NULL DEFAULT now()` on Postgres 10 rewrites the 50M-row table under `ACCESS EXCLUSIVE`; on the stated instance that is roughly 40 minutes of writes blocked. Add nullable, backfill in 10k-row batches, then add the constraint `NOT VALID` and validate." |

Include the reversibility table. It tells the reader where to argue with you and where to just ship.
