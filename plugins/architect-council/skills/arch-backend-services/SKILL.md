---
name: arch-backend-services
description: Architect-level backend service design — service boundaries and decomposition, API contracts and versioning, synchronous vs event-driven communication, distributed transactions and sagas, idempotency, resilience patterns, and background job design. Use this when deciding where a feature lives, designing or changing an API, choosing between REST/gRPC/GraphQL, introducing a queue or event, splitting or merging services, or designing async workflows. The design-review skill loads this when it selects the service-architecture perspective.
---

# Backend service architecture

The decisions here are about boundaries: what lives where, how the pieces talk, and what happens when one of them is unavailable. A boundary drawn in the wrong place is the most expensive mistake in this document, because every subsequent feature pays for it.

## Where does this feature live?

Answer before designing anything. The bias should be strongly toward putting it in an existing service.

**Extend an existing service when** the feature shares data with it, changes together with it, and is owned by the same team. This is the default answer.

**Create a new service when** at least one of these is genuinely true:

- It has a genuinely different scaling profile (a CPU-bound transcoder next to an I/O-bound API).
- It has a different availability requirement (payments must stay up when reporting does not).
- A different team owns it end to end.
- It needs technology the existing service cannot host — the weakest of the four, because a sidecar, a library, or a subprocess usually covers it without a network boundary.

**Bad reasons that come up constantly:** the existing service is large (that argues for internal modularization first), microservices are modern, the team wants to try something, or the code feels unrelated (a package boundary expresses that at a fraction of the cost).

A new service costs: a deployment pipeline, monitoring, on-call surface, a network hop with its own failure mode, a data boundary that forbids joins and transactions, and a versioned contract you can no longer refactor atomically. **Name that cost in the review.** A modular monolith with clean internal boundaries is a better starting point than four services, and it is far easier to split later than to merge.

## Drawing boundaries

- **Boundaries follow data ownership.** Exactly one service writes a given piece of data. Two services writing the same table is not two services.
- **A boundary should not split a transaction.** If two operations must commit together, they belong together. Splitting them turns an ACID transaction into a saga with compensating actions, retry logic, and a partial-failure state that someone has to reconcile — a large, permanent cost.
- **Watch for chatty boundaries.** If serving one request requires five round trips across a boundary, the boundary is in the wrong place.
- **Shared databases across services** couple them at the schema level, which is tighter coupling than an API, while giving up the ability to refactor. Either they are one service or they need an API.
- **A service that is only a proxy to another service** should not exist.

## Synchronous vs. asynchronous

The default worth arguing for: **synchronous for reads and for anything the caller needs a result from; asynchronous for work that can complete later.**

Choose async when:
- The caller does not need the result to respond (sending email, generating a thumbnail, updating a search index).
- The work is slow or its duration is unpredictable.
- You need to absorb a spike (the queue is the buffer).
- Multiple consumers care about the same event.
- The downstream may be unavailable and the work must still happen.

Async is not free. It buys decoupling and pays in: eventual consistency the UI must handle, out-of-order and duplicate delivery, harder debugging, a queue to monitor, and a dead-letter queue that needs an owner and a process. **A dead-letter queue nobody reads is a data-loss mechanism with extra steps** — assign the owner in the design.

Every synchronous call to another service needs: a timeout shorter than the caller's own, a bounded retry with jitter for idempotent operations only, a defined behaviour when it fails (fail the request, or degrade), and a circuit breaker if the call is on a hot path.

## API contracts

**Protocol:**

| Protocol | Fits | Cost |
|---|---|---|
| REST/JSON | Public APIs, browser clients, broad compatibility | Verbose; no schema unless you add OpenAPI |
| gRPC | Internal service-to-service, streaming, polyglot with generated clients | Not browser-native without a proxy; binary payloads are harder to debug |
| GraphQL | Client-driven queries across many resources, mobile clients wanting fewer round trips | Query complexity and depth limits are mandatory or you have a DoS; caching is harder; N+1 needs dataloaders |
| Async events | Fan-out, decoupling, audit trails | Contract discipline matters more, not less |

**Design rules that reliably matter:**

- Version from the first release. `/v1` or a header. Retrofitting versioning onto an unversioned API in use is painful.
- Additive changes only within a version: new optional fields yes, removed or renamed or retyped fields no. Clients you do not control will break, and clients you do control will break at an inconvenient time.
- Every list endpoint is paginated, with a maximum page size enforced server-side. An unpaginated list is a future outage.
- Errors are structured and machine-readable: a stable code, a human message, and a field reference for validation failures. A 500 with an HTML body is not an API.
- Idempotency keys on every non-idempotent write a client can retry — which is all of them, because clients retry on timeout and a timeout does not tell them whether the write landed.
- The response shape is a contract, not a serialized database row. Returning the ORM entity directly leaks the schema and turns every column addition into an API change.

**For events**, the same discipline: a schema registry or at minimum a documented, versioned schema; an event ID for deduplication; a timestamp; enough payload that consumers do not need a synchronous callback to be useful. And decide deliberately between event-carried state (self-contained, larger, can be stale) and event-as-notification (small, forces a callback, always fresh) — the second reintroduces the coupling the event was meant to remove.

## Distributed transactions

If the design has an operation spanning two services or two datastores, address this explicitly. It is the most common source of correctness bugs in service architectures.

- **First: can you avoid it?** Moving the boundary so both writes are in one transaction is almost always better than any of the options below.
- **Saga with compensating actions.** Each step has an undo. Requires that undo is actually possible — you cannot un-send an email or un-charge a card without a refund that is itself visible to the user.
- **Outbox pattern.** Write the business change and the outgoing event to the same database in one transaction; a separate process publishes from the outbox. This is the standard answer to "the write succeeded but the event was lost," and it is worth recommending by default wherever a write must produce an event.
- **Idempotent consumers.** At-least-once delivery is what you get. Every consumer must tolerate duplicates — a processed-message table, or an operation that is naturally idempotent.
- **Two-phase commit**: effectively not an option across services in practice.

The design must state what a partial failure looks like and who reconciles it. "It should not happen" is not an answer; at volume it happens daily.

## Resilience

- **Timeouts on everything**, and the budget must decrease going down the call chain. If the client times out at 5s, a service 3 hops deep cannot have a 10s timeout — it will do work nobody is waiting for.
- **Retries only on idempotent operations**, with exponential backoff and jitter, and a bounded attempt count. Retries without jitter synchronize into a thundering herd; retries without a budget amplify a partial failure into an outage.
- **Circuit breakers** on calls that can saturate. Failing fast preserves the caller's capacity.
- **Bulkheads.** Separate connection pools or concurrency limits per dependency, so one slow downstream cannot consume all of your workers. This is what turns "the recommendation service is slow" into a degraded feature instead of a full outage.
- **Graceful degradation.** For each dependency, state what the system does without it. Most designs never decide this, and the decision then gets made at 3am by whoever is on call.
- **Graceful shutdown.** Stop accepting new work, drain in-flight requests, then exit. Without it, every deploy drops requests.

## Background jobs and scheduling

- Jobs are idempotent and safely retryable. They will be retried.
- Every job has a timeout and a maximum attempt count, then goes to a dead-letter destination with an owner.
- Scheduled jobs need a distributed lock or leader election, or every replica runs them simultaneously.
- Long-running jobs checkpoint their progress; a job that must restart from zero after a deploy will never finish if it takes longer than your deploy interval.
- Separate the queues by priority and by latency requirement. A bulk backfill sharing a queue with password-reset emails means password resets wait behind the backfill.
- Instrument queue depth, oldest-message age, and processing rate. **Oldest-message age is the metric that catches a stuck consumer**; depth alone does not.

## Output

```markdown
## Where this lives
<Existing service or new. If new, the specific reason and the accepted cost.>

## Boundaries
| Service | Owns (data) | Exposes | Depends on |

## Communication
| Interaction | Sync/Async | Protocol | Timeout | Retry | On failure |

## API contract
<Endpoints or events. Request/response shapes, versioning, pagination, errors,
idempotency.>

## Consistency
<What is transactional, what is eventual, and where the user observes the
difference. For every cross-boundary operation: the mechanism (saga, outbox),
what partial failure looks like, and who reconciles it.>

## Resilience
| Dependency | Timeout | Retry | Breaker | Behaviour when unavailable |

## Background work
| Job | Trigger | Idempotent? | Retry/DLQ | Owner |

## BLOCKING / WARNING / CONSIDER

## What we are not building yet
<And the signal that says it is time.>
```

## Calibration

Reserve BLOCKING for: a boundary that splits an operation that must be atomic, a cross-service write with no defined partial-failure behaviour, missing idempotency on a retryable write, an unpaginated list endpoint, no timeout on an outbound call, and a dead-letter queue with no owner.

Two findings worth checking on every review because they are almost always absent: **what the system does when each dependency is unavailable**, and **who reconciles a partial failure across a boundary**.

The most valuable recommendation you can make here is often **"do not split this yet."** A modular monolith with enforced internal boundaries preserves the option to split later at a fraction of the cost, and the boundary you would draw today — before the feature has been built and the access patterns are known — is probably the wrong one.
