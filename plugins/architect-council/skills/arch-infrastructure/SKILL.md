---
name: arch-infrastructure
description: Architect-level infrastructure design — compute and deployment topology, availability and failure domains, networking, scaling, IaC, CI/CD and release strategy, observability, disaster recovery, and cost. Use this when a feature adds services or deployment units, when the user mentions Kubernetes, containers, Terraform, cloud providers, load balancers, autoscaling, SLOs, uptime, deployment, or infrastructure cost. The design-review skill loads this when it selects the infrastructure perspective.
---

# Infrastructure architecture

Infrastructure decisions get made by default if nobody makes them deliberately — and the defaults are usually a platform chosen for a scale the system does not have, at a cost nobody sized. Two questions drive most of this: what availability does the business actually need, and what does the design cost to run.

## Start with the requirements, in numbers

Do not design before these are stated. If the user has not supplied them, ask once — and record what you assumed for the ones they cannot answer.

- **Availability target.** Not "high" — a number. 99.9% is 43 minutes of downtime per month and is achievable single-region with good practices. 99.99% is 4 minutes and needs multi-AZ with automated failover and no single points of failure. 99.999% needs multi-region active-active, and is very rarely what the business actually needs when priced.
- **RTO and RPO.** How long can it be down, and how much data can be lost. These two numbers determine the entire backup and DR design, and they are business decisions, not technical ones.
- **Traffic**: baseline, peak, ratio between them, and how fast peaks arrive. A 3x daily cycle is different from a 50x flash spike.
- **Latency target** at p99, and where the users are.
- **Data residency** constraints.
- **Budget**, or at least an order of magnitude.

An availability target stated without a cost is usually revised downward once the cost is shown. Show it.

## Compute topology

Match the platform to operational maturity, not to ambition. Kubernetes for three services and two engineers is a second full-time job.

| Platform | Fits | Real cost |
|---|---|---|
| Managed PaaS (App Runner, Cloud Run, Fly, Render) | Stateless HTTP services, small teams, early products | Less control; per-request pricing can exceed instances at steady high load |
| Serverless functions (Lambda, Cloud Functions) | Event-driven, spiky, glue | Cold starts; execution limits; local dev friction; costly at sustained high volume |
| Container orchestration (ECS, Nomad) | Several services, want control without Kubernetes | Less ecosystem than k8s |
| Kubernetes (EKS/GKE/AKS) | Many services, multiple teams, need the ecosystem | Substantial ongoing operational load — upgrades, networking, RBAC, cost control. Needs someone who owns it |
| VMs with autoscaling | Stateful workloads, specific hardware, licensed software | You own patching, images, config management |

Honest defaults: **a managed container platform for most new backend services**, Kubernetes when you have several teams and someone whose job includes the cluster, serverless for genuinely event-driven and spiky work.

## Availability and failure domains

For each component, name what happens when it fails and how the system behaves:

- **Single points of failure.** One database primary, one Redis instance, one NAT gateway, one region. Which of these is acceptable given the target?
- **AZ spread.** Multi-AZ is usually a small cost for a large availability gain. Multi-region is a large cost for a gain most systems do not need — and it forces a data consistency decision that is the hard part, not the infrastructure.
- **Blast radius.** Does one bad tenant, one bad query, or one hot partition take down everyone? Cell-based or shuffle-sharded architectures bound this; it is worth naming even if the answer is "not yet."
- **Graceful degradation.** When the cache, the search index, or a downstream dependency is gone, does the system serve reduced functionality or return 500? This is a design decision with product input, and it is frequently never made.
- **Timeouts, retries, and circuit breakers everywhere.** A retry without a budget and jitter amplifies an incident into an outage — the classic retry storm. Every outbound call needs a timeout shorter than the caller's.
- **Health checks that mean something.** A liveness check returning 200 unconditionally is decorative. A readiness check that hits the database turns a slow database into a full outage as every instance is pulled from the pool. Distinguish the two deliberately.

## Networking

- Private subnets for everything not deliberately public. Databases and caches never publicly reachable.
- One ingress path with TLS termination, WAF, and rate limiting.
- Egress: NAT gateways are a real cost line at volume and a single failure domain per AZ. VPC endpoints for cloud-provider services avoid both.
- Service-to-service: mutual TLS or a mesh if you have the scale to justify it. A mesh for five services is complexity without payoff.
- Security groups as the primary boundary, least privilege, and check that the default group is not permissive.

## Scaling

- **Autoscale on the metric that actually correlates with load.** CPU is a proxy that is often wrong for I/O-bound services — queue depth, request concurrency, or p99 latency are usually better.
- **Scale-up speed vs. traffic arrival speed.** If instances take 90 seconds to become ready and the spike arrives in 10, autoscaling does not save you. Either pre-warm, over-provision the baseline, or shed load.
- **Set a maximum.** Unbounded autoscaling converts a traffic spike or a retry storm into an unbounded bill, and often just moves the bottleneck to the database.
- **Scale the database too.** Autoscaling the application tier into a fixed-size database connection pool produces connection exhaustion instead of throughput.
- **Load shedding and backpressure.** Above capacity, rejecting some requests quickly is better than degrading all of them. This is a design decision that needs to exist before the incident.

## IaC and environments

- Everything in Terraform, Pulumi, CDK, or the cloud-native equivalent. No console changes — they are invisible, unreviewable, and lost on rebuild.
- Remote state with locking. Local state on a laptop is a single point of failure with no backup.
- Environment parity: staging should differ from production in scale, not in shape. Bugs that only appear in production usually live in the differences.
- Least-privilege IAM per service, derived from what it actually needs. A wildcard policy is a finding.
- Tag everything for cost attribution from day one; retrofitting tags across a live estate is tedious and never gets done.

## CI/CD and release

- Build once, promote the same artifact through environments. Rebuilding per environment means you did not test what you shipped.
- Automated tests, a security scan, and a dependency audit as gates.
- Deployment strategy sized to risk: rolling for most services, blue/green when you need instant rollback, canary when the change is risky and you have the metrics to evaluate it. Canary without automated analysis of the canary's metrics is just a slow rolling deploy.
- **Rollback must be tested, not assumed.** A rollback path that has never been exercised is a hypothesis. Note that a deploy including a breaking migration is not rollback-safe — which is the reason for expand-migrate-contract.
- Feature flags to decouple deploy from release, with a plan for removing them. Stale flags accumulate into untested code paths.

## Observability

Design this in, rather than adding it after the first incident:

- **Structured logs** with a correlation ID propagated across service boundaries. Unstructured logs at volume are unsearchable.
- **Metrics**: the four golden signals per service — latency (as a distribution, not a mean), traffic, errors, saturation. A mean latency hides the tail that users experience.
- **Traces** across service boundaries. The only practical way to find where latency goes in a multi-service request.
- **SLOs with error budgets**, derived from the availability target. Alert on symptoms users feel, not on causes — CPU at 80% is not necessarily a problem, and a page for it trains people to ignore pages.
- **Cost as a monitored metric** with an anomaly alert. Cost incidents are real incidents.

## Disaster recovery

- Automated backups, encrypted, with a retention policy that satisfies both compliance and RPO.
- **Restores tested on a schedule.** An untested backup is not a backup. This is the single most commonly skipped item in this document.
- Cross-region backup copies if a regional failure is in scope.
- A written runbook for the top failure modes. During an incident nobody derives the procedure.
- Point-in-time recovery where the RPO requires it.

## Cost

Produce an actual estimate, even a rough one. A design without a cost is not a complete design.

- Compute, storage, data transfer, managed service fees, observability ingest. **Data egress and log ingest are the two lines that surprise people** — both scale with traffic and neither appears in early estimates.
- Cost per unit of business value: per request, per tenant, per active user. This is the number that tells you whether the architecture is viable at 10x.
- Reserved capacity or savings plans for predictable baseline; on-demand or spot for burst. Spot for stateless workers is a large saving; spot for a database is not a plan.
- Note what the design costs at 10x traffic. Some architectures scale cost linearly and some superlinearly, and that difference matters more than the current bill.

## Output

```markdown
## Requirements
| Requirement | Value | Stated or assumed |
| Availability target | | |
| RTO / RPO | | |
| Traffic (baseline / peak) | | |
| p99 latency | | |
| Residency | | |
| Budget | | |

## Topology
<Components, deployment units, failure domains. A diagram description is fine.>

## Failure analysis
| Component | Failure mode | Impact | Detection | Mitigation | Degradation behaviour |

## Scaling
<Trigger metric, bounds, scale-up time vs. traffic arrival, what the next
bottleneck is after the app tier.>

## Deploy & rollback
<Strategy, gates, rollback procedure, migration safety.>

## Observability
<Logs, metrics, traces, SLOs, and the alerts that would have caught the top
three failure modes above.>

## DR
<Backup schedule, restore test cadence, runbooks, PITR.>

## Cost
| Line item | Monthly estimate | Scales with |
Cost at 10x traffic: <estimate>
Cost per <request / tenant / user>: <figure>

## BLOCKING / WARNING / CONSIDER
```

## Calibration

Reserve BLOCKING for: a single point of failure inconsistent with the stated availability target, no tested restore path, unbounded autoscaling or unbounded retries, secrets or databases publicly reachable, no rollback path for a breaking change, and no cost estimate on a design that will be expensive.

The two findings that recur most and are worth checking every time: **the availability target has never been stated as a number**, and **the restore has never been tested**. Everything else in this document is downstream of those.

Recommending less infrastructure is frequently the right call. A managed platform, one region, multi-AZ, and a tested backup covers a large majority of products, and saying so plainly is more useful than a Kubernetes design the team cannot operate.
