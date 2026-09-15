---
name: design-review
description: Run a multi-perspective architecture review of a feature or product before implementation, routing to the security, caching, database, infrastructure, and backend-service architect skills as relevant. Use this whenever the user is planning a feature, writing a design doc or RFC, asking "how should I build X", weighing an architectural trade-off, or starting work large enough that the design matters more than the code. Also use it when the user asks for a design review, architecture review, or wants the architect skills applied.
---

# Design review

Coordinate an architecture review across the specialist perspectives. The value is not five checklists run in sequence — it is finding the places where two perspectives disagree, because that is where the real design decision is.

## Step 0: look before you ask

You are in the repository. Most of what a design review needs is already on disk. Before asking the user anything, spend a few minutes establishing what exists:

- **Language and runtime**: `go.mod`, `package.json`, `pyproject.toml`, `Dockerfile`.
- **Services and deployment**: `docker-compose.yml`, `k8s/`, `helm/`, `terraform/`, `.github/workflows/`, `Procfile`, `fly.toml`.
- **Data**: `migrations/`, `schema.sql`, `prisma/schema.prisma`, `ent/`, `sqlc.yaml`. Read the tables that the feature will touch.
- **Caching and messaging**: grep for `redis`, `memcache`, `kafka`, `sqs`, `nats`, `rabbitmq`, `pubsub`.
- **Auth and tenancy**: grep for `tenant`, `org_id`, `middleware`, `jwt`, `session`.
- **Existing design docs**: `docs/`, `ADR`, `RFC`, `DESIGN.md`.

Record what you found in the Assumptions section as "observed" rather than "assumed". Then ask only for what the code cannot tell you.

## Step 1: establish what you are reviewing

Do not start the review until you can state these. Ask in one round, not an interrogation, and only for what Step 0 did not answer:

- **What changes for the user.** The capability, not the implementation.
- **Scale, with numbers.** Requests per second, data volume, growth rate, read/write ratio. Order of magnitude is enough; "unknown" is a finding.
- **Consistency requirement.** Where is stale data acceptable, and where is it not? This single answer drives most of the caching and database design.
- **Data sensitivity.** PII, payment data, health data, credentials, or none. Determines whether compliance is in scope.
- **What already exists.** Reviewing a change to a running system is a different job from a greenfield design, and the constraints of the existing system usually dominate.

## Step 2: select the perspectives that apply

Do not run all five by default. An unused perspective produces filler that dilutes the findings that matter.

| Skill | Bring it in when |
|---|---|
| `arch-security-compliance` | The feature touches user data, authn/authz, payments, third-party data sharing, file uploads, webhooks, or a regulated domain |
| `arch-database` | New tables or collections, schema changes, new query patterns, retention questions, or a storage engine choice |
| `arch-caching` | Read-heavy paths, expensive computations, external API calls, or latency targets under a few hundred ms |
| `arch-infrastructure` | New services or deployment units, scaling and availability targets, cost, networking, CI/CD, or observability changes |
| `arch-backend-services` | Service boundaries, API contracts, inter-service communication, long-running operations, or asynchronous workflows |

For a small, contained change, one perspective is the right answer. Say which you skipped and why — that is information, not an omission.

Load each selected skill with the Skill tool before writing its findings section. Do not write a perspective from memory: the specialist skill carries the checklist, the output table, and the calibration for that perspective, and a section written without it is a guess wearing the section heading.

**Output discipline under the coordinator.** Each specialist ends with a full output template designed for direct invocation. When you run a specialist from here, take from it only its BLOCKING / WARNING / CONSIDER findings and the one or two tables that carry an actual decision (the access-pattern table, the cache plan, the boundary table, the failure analysis). Do not paste five full templates into one document. A review that pads to look rigorous teaches the reader to skim the next one.

## Step 3: surface the conflicts

This is the step that makes the review worth running. Read the findings against each other and name the tensions explicitly. The recurring ones:

- **Caching vs. compliance.** A cache is a copy of data in a place your deletion pipeline probably does not reach. GDPR erasure and cache TTLs are the same conversation.
- **Caching vs. consistency.** Every cache is a bet on tolerable staleness. If nobody has stated the tolerance, the cache is unspecified rather than fast.
- **Denormalization vs. correctness.** Read performance bought with duplicated data spends it on update paths and invariant maintenance.
- **Service decomposition vs. transactions.** A boundary drawn between two things that must commit together turns a transaction into a saga, and a saga into an on-call rotation.
- **Availability vs. cost.** Multi-region active-active is a real answer to a real requirement and a large bill for an imaginary one.
- **Security vs. latency.** Per-request authorization checks against a remote policy service sit on the hot path.
- **Observability vs. cost.** Log and trace ingest scale with traffic, and a feature that emits a debug line per item at ten thousand items per second is a five-figure bill by Friday. Sampling and cardinality are design decisions.
- **Flexibility vs. operability.** A generic rules engine, a plugin system, a configurable pipeline — each is powerful, and each is the thing nobody can debug at three in the morning. The specific version is usually right until the third real variant arrives.

For each tension, state the options and what each one costs. Do not resolve it silently.

## Step 4: verification and rollout

Every review answers these two questions, regardless of which perspectives ran. Designs that skip them are the ones that survive review and fail in production.

**How will we know it works?**
- The test strategy at each level: what the unit tests prove, what the integration tests prove against real dependencies, and what only a load test or a canary can prove.
- The failure modes named elsewhere in this review, and which test exercises each. A failure mode with no test is an assumption.
- Contract tests for any API or event another team or service consumes.

**How does it reach production, and how does it come back?**
- Feature flag, canary, or dark launch. What percentage, what metric decides promotion, who watches it.
- Migration sequencing: expand-migrate-contract steps and which deploy each lands in.
- The rollback path, stated as steps, and whether it is still valid after the migration (a dropped column has no rollback).
- The metric or alert that would tell you it is going wrong before a user does.

## Step 5: write it up

Write the review to `docs/design/<feature-slug>.md` (create the directory if needed) as well as returning it in the conversation. A review that lives only in the chat is lost by the next session; a file in the repo is a decision record the next engineer can find.

```markdown
# Design review: <feature>

## What this does
<2-3 sentences.>

## Non-goals
<What this feature will not do, by design. Different from "deferred": these are
scope boundaries, not future work.>

## Assumptions
<Every number and constraint. Mark each "observed" (from the repo) or "assumed"
(invented). Flag which ones, if wrong, change the design rather than just the sizing.>

## Recommended approach
<The design, and the two or three decisions that define it.>

## Key decisions
### <Decision>
- Options considered:
- Chosen, and why:
- Reversibility: <cheap | expensive — what undoing it costs>
- What would change this: <the observation or number that should make you revisit>

## Tensions
<Each conflict between perspectives, the options, and the cost of each.>

## Risks
| Risk | Likelihood | Impact | Mitigation | Owner |

## Findings by perspective
### Security & compliance
### Database
### Caching
### Infrastructure
### Backend services
<Only the ones you ran. BLOCKING / WARNING / CONSIDER, plus at most two decision
tables per perspective.>

## Verification and rollout
<From Step 4. Test strategy per level, flag/canary plan, migration sequencing,
rollback path, the alert that catches it first.>

## Open questions
<Things the user must answer or decide. Be specific about who decides.>

## Deliberately deferred
<What you are choosing not to solve now, and the signal that means it is time to.>
```

## Severity, defined once

Every specialist uses these three levels. Their calibration sections say what qualifies in their domain; the definitions live here.

- **BLOCKING** — the design cannot ship as described. It will cause a breach, a data-loss or correctness bug at volume, a compliance violation, an outage inconsistent with the stated availability target, or a re-architecture within a year. The finding names the concrete failure, not a category.
- **WARNING** — the design will work but will cost more than it should: an expensive-to-reverse decision made without evidence, a missing answer to a question that will be asked in production, a boundary in a place that makes the next feature harder.
- **CONSIDER** — an alternative or an improvement the author should see and may reasonably decline. Never blocks, never nags.

A finding that cannot describe its failure concretely is a category, not a finding, and does not get a severity.

## How to be useful here

**Distinguish what you know from what you assumed.** A review built on invented traffic numbers is a plausible-sounding fiction. Put the assumptions in their own section where they can be corrected, and mark which came from the repo.

**Give the reversibility of each decision.** A schema choice on a table that will hold 500 million rows is expensive to undo; a cache TTL is a config change. Spend the review's attention proportionally, and say which decisions are cheap to get wrong.

**Recommend, do not enumerate.** Six options with no opinion moves the work back to the user. Pick one, say why, and say what would change your mind.

**Name what you are deliberately not building.** The most valuable line in most design reviews is the one that says a requirement does not justify its complexity yet, and states the threshold at which it will.

**Match the depth to the stakes.** A three-day feature does not need a fifteen-page review. If the design is straightforward, say so in a paragraph, fill in Verification and rollout, and stop.
