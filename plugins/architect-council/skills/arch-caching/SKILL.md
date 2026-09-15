---
name: arch-caching
description: Architect-level caching design — whether to cache at all, cache topology and layers, invalidation strategy, key design, stampede and hot-key protection, and choosing between Redis, Valkey, Memcached, CDN, and in-process caches. Use this when designing read-heavy or latency-sensitive features, when the user mentions caching, Redis, Memcached, CDN, TTL, cache invalidation, stampedes, or slow reads. The design-review skill loads this when it selects the caching perspective.
---

# Caching architecture

A cache is a second copy of your data with its own consistency model, its own failure modes, and its own compliance surface. Design it deliberately. Most cache-related outages are not cache misses — they are invalidation bugs, stampedes, and hot keys.

## First: should you cache at all?

Ask these before designing a cache, because the answer is often no:

- **Is the read actually slow, measured?** Caching to fix a query that needs an index adds a consistency problem on top of the original problem, and hides it until the cache is cold.
- **What is the read/write ratio?** Below roughly 10:1 the invalidation cost starts to dominate the benefit.
- **How stale can this be, in seconds, per field?** If nobody can answer, the cache is unspecified rather than fast. This single number determines the entire design.
- **What breaks if it serves stale data?** A stale follower count is cosmetic. A stale permission check is a security incident. A stale account balance is a support ticket and possibly a regulatory one.
- **Is the working set small enough to matter?** A cache with a 5% hit rate is pure added latency and complexity.

The alternatives that are often better: add the missing index, fix the N+1, denormalize a read model, precompute on write, or put a CDN in front of it. Say so when one of those is the real answer.

## Choose the layer

Each layer has a different invalidation story. Getting the layer wrong is more expensive than getting the TTL wrong.

| Layer | Latency | Invalidation | Use for |
|---|---|---|---|
| Client / browser | 0 | Effectively impossible before expiry | Immutable, content-hashed assets |
| CDN / edge | 10-50 ms | Purge API, seconds to minutes to propagate | Static assets, public API responses, images |
| Request-scoped (per-request map, dataloader) | ns | None needed; dies with the request | The same entity loaded N times while serving one request; N+1 across resolvers or nested handlers |
| In-process (bounded LRU) | ns | Per-instance, no coordination | Config, feature flags, hot reference data, tiny working sets |
| Distributed (Redis/Valkey/Memcached) | 0.5-2 ms | Coordinated, immediate | Session data, computed results, shared hot data |
| Materialized read model | Query cost | Rebuilt on write or on schedule | Complex aggregations, denormalized reads |
| Database buffer pool | Already there | Automatic | Free; tune it before adding a cache layer |

**Request-scoped memoization is the cache with no invalidation problem**, because nothing outlives the request. It is the correct first answer to N+1 in GraphQL resolvers, nested service calls, and any handler that loads the same user or tenant row from three places. A dataloader that batches and dedupes within one request often removes the need for the distributed cache the design was about to add. Check for it before anything below.

**In-process caches are per-instance.** With twenty replicas you have twenty independently stale copies and no way to invalidate them synchronously. That is fine for feature flags with a 30-second TTL and wrong for anything a user expects to see change immediately after their own write.

**Never cache authorization decisions in-process without a short, hard TTL.** Revoking access that stays live for an hour on some instances is a security finding.

## Pick an invalidation strategy

The genuinely hard part. Order of preference, and the reasoning is that each step down adds coordination:

**1. Immutable keys.** Put the version or content hash in the key: `user:123:v7`, `asset:sha256-abc`. Nothing to invalidate — writes create a new key and old keys expire on their own. This is the best answer whenever you can construct it.

**2. TTL only.** Simple, self-healing, bounded staleness. Add jitter to the TTL, or a thundering herd of simultaneous expiries will hit the database in lockstep. Right whenever the staleness window is acceptable.

**3. Write-through / write-around.** Update or delete the cache entry in the same operation as the write. Two systems, no transaction between them, so decide what happens when the second step fails: prefer deleting the cache entry over writing it, because a delete that fails leaves a stale entry that TTL will fix, while a write that partially succeeds can leave a wrong entry that persists.

**4. Event-driven invalidation.** Publish change events; consumers invalidate. Handles multi-service and multi-layer cases. Now you own event ordering and delivery guarantees — an out-of-order invalidation can resurrect stale data.

**5. Tag or dependency-based.** Cache entries carry tags; invalidate by tag. Powerful for fan-out invalidation, and the bookkeeping is where the bugs live.

Review question: **for every cached entity, name what invalidates it and what happens when that mechanism fails.** If the answer for any entry is "nothing, it just has a TTL," confirm the TTL matches the stated staleness tolerance. If the answer is "we delete it on write," confirm every write path does — including migrations, admin tools, background jobs, and the other service that also writes this table.

## Key design

- **Namespace and version every key**: `svc:entity:v2:id`. The version lets you change the serialization format without a flush and without deserializing garbage into a struct.
- **Include every input that affects the value.** A key missing the locale, tenant, or permission scope serves one user's data to another. This is a real and recurring cross-tenant leak.
- **Tenant-scope keys in multi-tenant systems.** `user:123` across tenants is a leak with a plausible-looking key.
- **Bound key length and cardinality.** Keys built from unbounded user input become a memory leak.
- Document the key schema somewhere that is not only the code that writes it, so the code that invalidates it can be checked against it.

## Failure modes to design against

**Stampede / dogpile.** A popular key expires and a thousand concurrent requests all miss and all hit the database. Fixes: request coalescing (one in-flight fill per key per instance — `singleflight` in Go, a per-key promise map in Node, `cachetools` locks in Python), a short lock on the fill, or serve-stale-while-revalidate. For a genuinely hot key, coalescing per instance still lets N instances through, so combine with jittered TTLs.

**Hot key.** One key gets a disproportionate share of traffic and saturates a single shard. Fixes: a small in-process tier in front of the distributed cache, or key splitting (`counter:x:{0..9}` summed on read).

**Cache penetration.** Repeated requests for keys that do not exist bypass the cache every time. Cache the negative result with a short TTL, or use a bloom filter for high-volume cases.

**Cold start after deploy or failover.** An empty cache means full database load at the moment you are least able to absorb it. Design for it: can the database survive a cold cache at peak? If not, that is a BLOCKING finding — you have a cache the system cannot run without, which is a dependency, not an optimization. Options: warm on startup, roll deploys gradually, or keep the cache external to the deploy unit.

**Cache as system of record by accident.** Data that only exists in the cache, or a system that cannot serve at all without it. Decide explicitly whether the cache is an optimization or a dependency, and if it is a dependency, give it the availability design of one.

**Stale-forever entries.** A cache entry with no TTL and a broken invalidation path. Always set a TTL, even with active invalidation, as the backstop.

## Tool selection

**Redis / Valkey** — data structures beyond key-value (sorted sets for leaderboards and rate limiting, streams, hashes), Lua scripting for atomic multi-step operations, pub/sub, optional persistence. The default for a distributed cache. Valkey is the BSD-licensed fork after Redis's licence change; API-compatible, and the relevant question for a new system is which one your managed provider supports.

**Memcached** — simpler, multithreaded, very good at pure key-value at high throughput with lower memory overhead per entry. No data structures, no persistence. Choose it when you genuinely only need `get`/`set` at scale.

**CDN** (CloudFront, Fastly, Cloudflare) — anything cacheable by URL. Also handles TLS termination and DDoS absorption. Getting `Cache-Control`, `Vary`, and `Surrogate-Key` right matters more than the vendor choice. A `Vary: Cookie` on an otherwise cacheable response drops the hit rate to near zero.

**Eviction policy** is a design decision the defaults hide. Redis defaults to `noeviction` (writes fail at `maxmemory`), managed offerings often default to `volatile-lru` (only keys with a TTL are evicted, so keys without one accumulate until the instance is full), and `allkeys-lru` is what most people assume they have. State which one, and what happens at the memory limit: failed writes, evicted sessions, or an out-of-memory restart are three very different incidents. Set `maxmemory` explicitly and alert on headroom.

## HTTP caching semantics

For any public or client-facing API, the cheapest cache is the one the browser and the CDN already have. Most designs ignore it.

- **`Cache-Control`** on every response, deliberately: `no-store` for anything personal, `private, max-age=N` for per-user data the browser may keep, `public, max-age=N, s-maxage=M` for shared responses the CDN may keep, `immutable` for content-hashed assets.
- **`ETag` and `If-None-Match`** turn a repeat request into a 304 with no body. Cheap to generate from a version column or a content hash, and it makes "is it still current?" a free question.
- **`stale-while-revalidate` and `stale-if-error`** let the CDN serve a slightly old response while it refreshes, or while the origin is down — which is serve-stale for free.
- **`Vary`** lists the request headers that change the response. Missing `Vary: Accept-Encoding` or `Vary: Authorization` serves one client's response to another; a needless `Vary: Cookie` or `Vary: User-Agent` makes the cache useless.
- **Cache keys include the query string** by default at most CDNs; a tracking parameter with a unique value per visit defeats the cache. Normalise or strip.

**In-process** (an LRU library for your language — `hashicorp/golang-lru` in Go, `lru-cache` in Node, `cachetools` in Python — or a plain map with a lock) — nanosecond reads, no network. Bound the size, or it is a memory leak. A concurrent map type such as Go's `sync.Map` is not an LRU and has no eviction.

Managed vs. self-hosted: managed unless you have a specific reason and someone who wants to own failover. Cluster mode adds cross-slot operation constraints — multi-key operations must hash to the same slot, which changes key design, so decide this before writing the keys.

## Compliance interaction

Raise this explicitly, because it is the thing most caching designs miss: **a cache is a copy of data in a store your deletion pipeline probably does not reach.**

- Regulated data in a cache is in scope for erasure requests. Either the deletion path invalidates cache entries, or TTLs are short enough to satisfy the obligation, or do not cache it.
- Cached data may sit in a different region than the primary store. Check residency requirements.
- Redis persistence (RDB/AOF) means cached PII lands on disk and in snapshots. If persistence is on and the data is regulated, that disk is in scope for encryption and retention.
- Cache keys built from email addresses or names put PII in key names, which appear in monitoring, `SLOWLOG`, and `KEYS` output. Hash the identifier.

## Output

```markdown
## Should we cache?
<The measured problem, the read/write ratio, the stated staleness tolerance,
and whether an index or read model is the better answer.>

## Cache plan
| Data | Layer | Key schema | TTL | Invalidated by | Staleness tolerance | On failure |

## Failure-mode coverage
| Mode | Applies | Mitigation |
| Stampede | | |
| Hot key | | |
| Penetration | | |
| Cold start | | |

## Is the cache an optimization or a dependency?
<And if a dependency, what availability design that implies.>

## Compliance
<Regulated data in cache, deletion path, residency, persistence.>

## BLOCKING / WARNING / CONSIDER

## Metrics to instrument
<Hit rate per key class, p99 latency with and without cache, eviction rate,
memory headroom, database load at cold start.>
```

When invoked by `design-review`, return only the BLOCKING / WARNING / CONSIDER findings plus the cache-plan table; the coordinator assembles the document.

## Calibration

Severity levels are defined once in `design-review`. The most valuable finding in this domain is often **"do not cache this yet — add the index and measure again."** A cache added to a system that has not measured its slow path adds a consistency problem and hides the original one.

Second most valuable: **naming the staleness tolerance the design never stated.** Almost every caching bug traces back to nobody having written down how stale is acceptable, per field.

Reserve BLOCKING for: no invalidation path on data that must be fresh, a cache the database cannot survive losing, missing scope in a key (cross-tenant or cross-user leak), regulated data cached with no deletion path, and an eviction policy that fails writes at the memory limit on a path that cannot tolerate it.

| Not a finding | A finding |
|---|---|
| "Consider caching the permissions lookup." | "Permissions are read on every request (stated 2k rps) with a 40 ms query; a per-request memo removes the 3× repeat within one request, and a 30 s in-process TTL bounds revocation lag to 30 s, which the security review accepted. No distributed cache needed." |
| "Cache invalidation should be handled carefully." | "`user:{id}` is written by the profile service on update but also by the admin bulk-import job, which does not touch the cache; imported changes stay invisible for the full 24 h TTL. Either the import job deletes the keys or the TTL drops to the stated 5-minute tolerance." |
