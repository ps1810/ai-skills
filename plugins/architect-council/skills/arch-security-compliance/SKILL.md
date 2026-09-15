---
name: arch-security-compliance
description: Architect-level security and compliance review of a feature design — threat modelling, authentication and authorization, data classification, secrets, tenancy isolation, audit logging, and GDPR/SOC 2/PCI/HIPAA obligations. Use this when designing or reviewing any feature that touches user data, login, permissions, payments, third-party data sharing, file uploads, or a regulated domain, and whenever the user asks about security, privacy, compliance, or threat modelling during planning. The design-review skill loads this when it selects the security perspective.
---

# Security and compliance architecture

Review the design, not the code. Code-level review catches injection and missing bounds checks. Design-level review catches the authorization model that cannot express the permission the product needs, and the data flow that puts regulated data somewhere the deletion pipeline does not reach. Those are the ones that cost a re-architecture.

## Start with data classification

Everything downstream depends on this, so do it first and write it down:

| Class | Examples | Consequence |
|---|---|---|
| Public | Marketing copy, docs | No controls needed |
| Internal | Aggregate metrics, logs | Access control, no encryption mandate |
| Confidential | Business data, non-public customer content | Encrypt at rest and in transit, audit access |
| Regulated | PII, payment data, health data, biometrics | Above, plus retention limits, erasure, residency, breach notification |
| Secret | Credentials, keys, tokens | Never in logs, source, or a database column |

Then trace each class through the design: where it enters, every store it lands in, every service that reads it, every third party it reaches, every log and metric it might appear in, and how it gets deleted. **The stores people forget are caches, message queues, search indexes, analytics warehouses, backups, and log aggregation.** Regulated data in any of those is in scope for retention and erasure, and a design that only deletes from the primary database has a compliance gap regardless of how correct the primary deletion is.

## Threat model

Use STRIDE as a prompt list, not a ceremony. For each trust boundary in the design — client to service, service to service, service to third party — ask:

- **Spoofing** — how is the caller's identity established? What happens if the token is replayed, or is valid but stale?
- **Tampering** — what can the client change that the server trusts? Prices, quantities, IDs, role claims, redirect URLs, webhook payloads.
- **Repudiation** — is there an audit trail that survives the actor deleting their own data?
- **Information disclosure** — error messages, timing differences, enumerable IDs, over-broad API responses, unfiltered logs.
- **Denial of service** — unbounded queries, unbounded uploads, unbounded fan-out, expensive operations reachable unauthenticated.
- **Elevation of privilege** — can a tenant reach another tenant's data? Can a user act as an admin by changing a parameter?

The highest-value question in most reviews: **what is the worst thing a legitimately authenticated user can do?** Most real breaches are authorization failures, not authentication failures.

## Authorization

This is where design review earns its keep, because the model is expensive to change later.

- **Where is the decision made?** Every read and write path, or scattered through handlers? Scattered means one path will be missed.
- **Is authorization enforced at the data layer or the handler layer?** Handler-level checks are one forgotten endpoint away from a leak. A query that cannot be constructed without a tenant scope is structurally safer than one where the caller must remember the `WHERE tenant_id = ?`.
- **Can the model express what the product needs in twelve months?** Role-based access breaks the first time a customer asks for "this user, but only for these three projects." Retrofitting relationship-based access onto an RBAC schema is a migration, not a patch.
- **Object-level checks on every object, not just the collection.** `GET /orders` filtered by user but `GET /orders/{id}` unfiltered is the most common API vulnerability there is.
- **Deny by default.** New endpoints and new fields should be unreachable until explicitly permitted.

## Multi-tenancy isolation

If the system is multi-tenant, state the isolation model explicitly: shared tables with a tenant column, schema per tenant, or database per tenant. Then check:

- Is the tenant identity derived from the authenticated session, never from a request parameter or header the client controls?
- Can a query be written without the tenant predicate? If yes, one will be.
- Are cache keys tenant-scoped? A cache key of `user:123` across tenants is a cross-tenant leak with a plausible-looking key.
- Do background jobs, exports, and admin tooling carry tenant scope? These are the usual gap.

## Authentication and sessions

Delegate to a proven implementation rather than designing one. Then check the parts that are yours: token lifetime and refresh, revocation (can you actually invalidate a session, or does it live until expiry?), MFA coverage for privileged actions, and account recovery — which is frequently the weakest authentication path in a system that is otherwise well built.

For service-to-service: mutual TLS or short-lived workload identity, not a long-lived shared secret in an environment variable.

## Secrets and keys

- Managed secret store with rotation, not environment variables baked into images, not config files in the repo.
- Rotation must be possible without downtime. If rotating the database password requires a coordinated restart, it will not happen.
- Nothing secret in logs, error messages, traces, or URLs. Query strings end up in access logs and browser history.
- Scan for committed secrets in CI, and assume anything ever committed is compromised.

"Encrypt at rest" is not a design until the keys have a home:

- **Where do the keys live?** A KMS (cloud provider's, or Vault), not a column, not a config file. The application should hold a reference to a key, never the key material.
- **Envelope encryption** for anything encrypted in the application: a per-record or per-tenant data key, itself encrypted by the KMS master key. Rotating the master key then re-wraps data keys instead of re-encrypting every row.
- **Who can use each key?** IAM on the KMS is the real access control for encrypted data. A service with `kms:Decrypt` on every key has access to everything, regardless of database permissions.
- **Per-tenant keys** turn tenant erasure into a key deletion, which is the only erasure that also covers backups. Worth naming as an option when regulated data and multi-tenancy meet.

## File uploads

An upload endpoint is an attacker writing bytes onto your infrastructure. If the feature has one, check every line:

- **Validate the content, not the extension or the client-supplied `Content-Type`.** Sniff magic bytes; for images, re-encode through a library so a polyglot file does not survive.
- **Bound the size at the edge** (load balancer or proxy), before the application reads it into memory. Stream to storage; never buffer an unbounded body.
- **Store outside the web root, under a name you generated**, not the client's filename. Serve through a signed URL or a proxy that sets `Content-Disposition: attachment` and a safe `Content-Type`; never execute or render user content from the same origin as the application.
- **Scan** for malware where the file is shared with other users or opened by staff. A scanner with no quarantine path is decorative.
- **Direct-to-storage uploads** (presigned S3/GCS URLs) keep the bytes off your servers entirely and are the default for anything over a few megabytes. Constrain the presigned policy: size, content type, key prefix, expiry.
- **Uploaded PII is regulated data**: it needs the same classification, retention, and deletion path as a database column, and object storage is one of the stores the erasure pipeline usually forgets.

## Inbound webhooks

A webhook endpoint is an unauthenticated POST unless you make it otherwise:

- **Verify the signature** with the provider's secret over the raw body, using a constant-time compare, before parsing anything.
- **Reject replays**: check the timestamp header against a window, and dedupe on the provider's event ID.
- **Do the work asynchronously.** Acknowledge fast, enqueue, process. A slow handler causes the provider to retry, which causes duplicates, which is why the dedupe exists.
- **Never trust the payload's claims about identity or amounts.** Fetch the canonical object from the provider's API by ID if the decision matters (payment succeeded, subscription changed).

## Supply chain

- **Lockfiles committed and honoured** (`go.sum`, `package-lock.json`/`pnpm-lock.yaml`), with `--frozen-lockfile` or the equivalent in CI. A build that resolves fresh is a build you have not tested.
- **Dependency audit in CI** (`govulncheck`, `npm audit`/`pnpm audit`, Dependabot or Renovate) with a policy for what blocks: known-exploited critical vulnerabilities block; the rest have an owner and a deadline.
- **Pin third-party CI actions and images by digest**, not by mutable tag. A compromised `v1` tag on a popular action has exfiltrated secrets from thousands of repos.
- **CI secrets are the crown jewels.** Deploy credentials in a workflow that also runs on pull requests from forks is a leak. Separate the build job from the deploy job and scope secrets to the latter.
- **SBOM** where a customer or regulator will ask; generating one is cheap once the lockfile discipline exists.

## Audit logging

For regulated data and privileged actions, log who did what to which object when, from where — to an append-only destination the actor cannot modify. Separate audit logs from application logs; they have different retention requirements and different readers.

Do not log the sensitive values themselves. Log that a field was read, not what it contained.

## Compliance obligations

Only bring in the regimes that apply. Naming all four when one is relevant is noise.

**GDPR / similar privacy law** — lawful basis for each purpose; data minimisation (is every field collected actually used?); erasure that reaches every store including caches, backups, and analytics; export in a portable format; residency constraints; data processing agreements with sub-processors; retention limits with actual enforcement rather than an intention.

**SOC 2** — access reviews, change management with approvals, encryption at rest and in transit, monitoring and alerting, incident response, vendor management. Mostly a question of whether the design produces the evidence an auditor will ask for.

**PCI DSS** — the decisive design question is whether card data touches your systems at all. A hosted payment page or tokenizing provider keeps you out of most of the scope. If card data enters your network, scope expands dramatically — treat that as a BLOCKING finding unless it is a deliberate, resourced decision.

**HIPAA** — BAAs with every vendor touching PHI, encryption, access controls with audit trails, minimum necessary access. PHI in a logging or analytics vendor without a BAA is a violation, and logging pipelines are where this usually happens by accident.

## Output

```markdown
## Data classification
| Data | Class | Stores it lands in | Deletion path |

## Threat model
| Boundary | Threat | Likelihood | Impact | Mitigation | Status |

## Authorization model
<Where decisions are made, what the model can express, where it will break.>

## BLOCKING
<Design-level problems that must be resolved before build. Each with the concrete
attack or violation, not a category name.>

## WARNING
## CONSIDER

## Compliance
| Regime | Applies? | Obligations triggered | Gap |

## Assumptions
<What you assumed about data sensitivity, tenancy, and existing controls.>
```

When invoked by `design-review`, return only the BLOCKING / WARNING / CONSIDER findings plus the data-classification table; the coordinator assembles the document.

## Calibration

Severity levels are defined once in `design-review`. In this domain, reserve BLOCKING for things that will cause a breach, a violation, or an expensive re-architecture: a missing authorization boundary, regulated data with no deletion path, card data entering scope by accident, cross-tenant reachability, an upload endpoint that serves user content from the application origin. Everything else is WARNING or CONSIDER.

Name the concrete attack. If you cannot describe the attack, you have found a category, not a finding.

| Not a finding | A finding |
|---|---|
| "Insufficient input validation on the signup endpoint." | "The `role` field in the signup payload is written straight to the user record, so any user can POST `role: admin` and get it." |
| "Multi-tenancy risks should be considered." | "`GET /reports/{id}` loads by primary key with no tenant predicate; a user in tenant A who guesses an ID reads tenant B's report." |

Say when a control is disproportionate. An internal tool for eight employees does not need the threat model of a payment system, and recommending one anyway trains the reader to discount the next review.
