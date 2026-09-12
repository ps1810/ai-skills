---
name: design-review
description: Run a multi-perspective architecture review of a feature or product before implementation, routing to the security, caching, database, infrastructure, and backend-service architect skills as relevant. Use this whenever the user is planning a feature, writing a design doc or RFC, asking "how should I build X", weighing an architectural trade-off, or starting work large enough that the design matters more than the code. Also use it when the user asks for a design review, architecture review, or wants the architect skills applied.
---

# Design review

Coordinate an architecture review across the specialist perspectives. The value is not five checklists run in sequence — it is finding the places where two perspectives disagree, because that is where the real design decision is.

## Step 1: establish what you are reviewing

Do not start the review until you can state these. If the user has not supplied them, ask — one round of questions, not an interrogation:

- **What changes for the user.** The capability, not the implementation.
- **Scale, with numbers.** Requests per second, data volume, growth rate, read/write ratio. Order of magnitude is enough; "unknown" is a finding.
- **Consistency requirement.** Where is stale data acceptable, and where is it not? This single answer drives most of the caching and database design.
- **Data sensitivity.** PII, payment data, health data, credentials, or none. Determines whether compliance is in scope.
- **What already exists.** Reviewing a change to a running system is a different job from a greenfield design, and the constraints of the existing system usually dominate.

## Step 2: select the perspectives that apply

Do not run all five by default. An unused perspective produces filler that dilutes the findings that matter.

| Skill | Bring it in when |
|---|---|
| `arch-security-compliance` | The feature touches user data, authn/authz, payments, third-party data sharing, or a regulated domain |
| `arch-database` | New tables or collections, schema changes, new query patterns, or a storage engine choice |
| `arch-caching` | Read-heavy paths, expensive computations, external API calls, or latency targets under a few hundred ms |
| `arch-infrastructure` | New services or deployment units, scaling and availability targets, cost, networking, or CI/CD changes |
| `arch-backend-services` | Service boundaries, API contracts, inter-service communication, or asynchronous workflows |

For a small, contained change, one perspective is the right answer. Say which you skipped and why — that is information, not an omission.

Load each selected skill with the Skill tool before writing its findings section. Do not write a perspective from memory: the specialist skill carries the checklist, the output table, and the calibration for that perspective, and a section written without it is a guess wearing the section heading.

## Step 3: surface the conflicts

This is the step that makes the review worth running. Read the findings against each other and name the tensions explicitly. The recurring ones:

- **Caching vs. compliance.** A cache is a copy of data in a place your deletion pipeline probably does not reach. GDPR erasure and cache TTLs are the same conversation.
- **Caching vs. consistency.** Every cache is a bet on tolerable staleness. If nobody has stated the tolerance, the cache is unspecified rather than fast.
- **Denormalization vs. correctness.** Read performance bought with duplicated data spends it on update paths and invariant maintenance.
- **Service decomposition vs. transactions.** A boundary drawn between two things that must commit together turns a transaction into a saga, and a saga into an on-call rotation.
- **Availability vs. cost.** Multi-region active-active is a real answer to a real requirement and a large bill for an imaginary one.
- **Security vs. latency.** Per-request authorization checks against a remote policy service sit on the hot path.

For each tension, state the options and what each one costs. Do not resolve it silently.

## Step 4: write it up

```markdown
# Design review: <feature>

## What this does
<2-3 sentences.>

## Assumptions
<Every number and constraint you assumed rather than were told. Flag which ones,
if wrong, change the design rather than just the sizing.>

## Recommended approach
<The design, and the two or three decisions that define it.>

## Key decisions
### <Decision>
- Options considered:
- Chosen, and why:
- What would change this: <the observation or number that should make you revisit>

## Tensions
<Each conflict between perspectives, the options, and the cost of each.>

## Risks
| Risk | Likelihood | Impact | Mitigation |

## Findings by perspective
### Security & compliance
### Database
### Caching
### Infrastructure
### Backend services
<Only the ones you ran. BLOCKING / WARNING / CONSIDER.>

## Open questions
<Things the user must answer or decide. Be specific about who decides.>

## Deliberately deferred
<What you are choosing not to solve now, and the signal that means it is time to.>
```

## How to be useful here

**Distinguish what you know from what you assumed.** A review built on invented traffic numbers is a plausible-sounding fiction. Put the assumptions in their own section where they can be corrected.

**Give the reversibility of each decision.** A schema choice on a table that will hold 500 million rows is expensive to undo; a cache TTL is a config change. Spend the review's attention proportionally, and say which decisions are cheap to get wrong.

**Recommend, do not enumerate.** Six options with no opinion moves the work back to the user. Pick one, say why, and say what would change your mind.

**Name what you are deliberately not building.** The most valuable line in most design reviews is the one that says a requirement does not justify its complexity yet, and states the threshold at which it will.

**Match the depth to the stakes.** A three-day feature does not need a fifteen-page review. If the design is straightforward, say so in a paragraph and stop — a review that pads to look rigorous teaches the reader to skim the next one.
