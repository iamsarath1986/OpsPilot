# Milestone 12 — On-Premises Production and Azure Portability

**Goal:** OpsPilot has two deliberate deployment targets: a production-candidate on company infrastructure for real data, including self-hosted inference; and an Azure environment containing synthetic data only, proving cloud portability with Bicep, managed identity, managed services, monitoring, and deployment operations. Both are built from promoted artifacts and proven through failure drills, a restore, and a rollback.

**Why now:** everything either target demands is already true — the service is stateless, configuration is external, state is in backing services, images are hardened, health probes exist, inference saturation is measured, and telemetry is instrumented. Deploying is now a matter of provisioning and wiring, not changing the data-residency decision.

**Why Azure:** the job description names it. The Azure target is real, deployed, and operated, but it is not authorised for customer data under ADR-000. The concepts transfer to any major cloud, and you should be able to state the AWS or GCP equivalent of each service you use.

**Time:** 6–8 focused weeks. **Prerequisites:** Milestone 11 checklist green; ADR-012 records both targets and their data boundary; company infrastructure and GPU ownership are named. **Set an Azure budget alert before you provision anything.**

## Non-negotiable deployment boundary

| Target | Permitted data and inference path | Purpose |
|---|---|---|
| Company infrastructure | Approved real data; generation, embeddings, reranking, and judging all self-hosted inside the company boundary | Production-candidate and eventual internal pilot |
| Azure | Synthetic incidents, logs, runbooks, and evals only; use the same open-weight model/server on short-lived Azure compute if practical, otherwise a clearly labelled synthetic adapter | Demonstrate Azure architecture and operations without weakening ADR-000 |

No network route may allow the Azure environment to call the real-data inference or storage plane. A future written policy approval may change that boundary through a new ADR; this milestone does not assume it.

### Production egress is enforced by the network, not by CI

The `egress-policy` CI job from Milestones 6–11 proves that the *test* deployment cannot send customer content anywhere unapproved. It proves nothing about the real hosts. On the company target, enforcement must be a network control:

- **Outbound traffic is denied by default** on every host and container that handles real data — API, worker, inference, embedding, reranker, judge, and telemetry collector — using the platform's firewall or network-policy mechanism. Each permitted destination is listed explicitly, owned, and justified in ADR-012.
- **Model weights, tokenisers, and container images come from internal mirrors.** Inference servers commonly download weights or tokeniser files from a public model hub at startup, and some libraries send usage telemetry by default. Pre-stage weights in approved internal storage, pin them by revision and checksum, configure the serving stack for offline operation, and disable library telemetry. A server that can only start with internet access is not deployable on this target.
- **Verify it from the host, not from the diagram.** Run a probe from inside each real-data workload that attempts to reach a public address and the public model hub; both must fail. Repeat the probe after every infrastructure change, and alert on denied-egress events — an unexpected attempt is a finding even though it was blocked.

---

## 1. Topics to Learn

### Cloud fundamentals
| Topic | Why it matters |
|---|---|
| Regions, availability zones, latency | Where you deploy affects latency, cost, data residency, and service/GPU availability. |
| Shared responsibility model | Which failures are the provider's problem and which remain yours. Most people assume too much is handled. |
| IaaS / PaaS / serverless trade-offs | Control versus operational burden. You are choosing PaaS deliberately, and should be able to say why. |
| Consumption pricing and quotas | Cloud bills are a function of design decisions. Quotas are hard walls you must design around. |

### Azure specifics
| Service / concept | Why it is in this architecture |
|---|---|
| Subscriptions, resource groups, tags, RBAC | Lifecycle, cost attribution, and least-privilege boundaries |
| Microsoft Entra ID | Human identity — replaces the local accounts from Milestone 5 |
| Managed identities | Workload identity with **no stored secrets**. The single biggest security improvement available to you. |
| Azure Key Vault | The few non-identity secrets or certificates that remain in the synthetic Azure target, with rotation and access policy |
| Azure Container Registry | Stores the image built in Milestone 11; deployment references it by digest |
| Azure Container Apps | Managed containers with ingress, revisions, scale rules, and jobs — without operating Kubernetes |
| Azure Database for PostgreSQL Flexible Server | Managed Postgres with backups, PITR, HA options, and `pgvector` |
| Azure Blob Storage | Durable artifact storage with lifecycle rules; container disks are ephemeral |
| Azure Service Bus | The durable broker behind the Milestone 10 port |
| Azure Managed Redis | The shared cache and distributed limiter |
| Self-hosted open-weight inference | Company GPU hosts serve the real-data path; optional short-lived Azure GPU compute serves only the synthetic target. Model, quantisation, constrained decoding, and capacity are explicit deployment artifacts. |
| Azure Monitor, Log Analytics, Application Insights | The backend for your OpenTelemetry data; KQL for querying it |
| Bicep | Infrastructure as code: reviewable, repeatable, diffable |
| GitHub Actions OIDC federation | Deploy without storing cloud credentials in CI |
| Private endpoints, VNet integration, DNS | Keeping data services off the public internet |
| Autoscaling rules (KEDA-style) | Scale on HTTP concurrency and on queue depth — the number you measured in Milestone 10 |
| Budgets and cost alerts | The cloud lesson everyone learns once, expensively |
| Well-Architected Framework | A structured review across reliability, security, cost, operations, and performance |
| RTO and RPO | How much downtime and data loss the business can tolerate — the numbers that justify your HA and backup design |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Resource model and RBAC | **Implement** | Environments are separated, tagged, and least-privileged |
| Managed identity | **Productionize** | No connection string or key exists in app configuration |
| Bicep | **Implement** | You can destroy an environment and rebuild it from code |
| Container Apps | **Debug** | You can diagnose a failed revision from logs and probes |
| Managed PostgreSQL | **Productionize** | You have tuned connections, verified backups, and performed a restore |
| Autoscaling | **Design** | Your scale rules come from measurements, and you know what they will break |
| Monitoring | **Debug** | You can diagnose an incident using KQL and traces alone |
| Cost | **Design** | You can state cost per 1,000 diagnoses and the biggest lever |
| DR | **Productionize** | You have restored, failed over, and rolled back — and timed each |

## 3. Hands-On Learning Exercises

1. **Portal, then code.** Deploy one container through the portal. Then delete everything and reproduce it in Bicep. The contrast — clicking versus a reviewable file you can destroy and rebuild — is the entire argument for IaC, and you should feel it once.
2. **Delete the secret.** Connect to PostgreSQL with a connection string, then switch to managed identity and *remove the secret entirely*. Then revoke the role and watch the failure, so you understand what the identity was actually granting.
3. **Hit self-hosted saturation deliberately.** Drive the inference server beyond its measured concurrent-sequence or KV-cache limit. Observe queue growth and overload responses, confirm the Milestone 7 admission, fallback, and degradation logic holds, and record the effective tokens/s ceiling.
4. **Connection-limit collision.** Scale the API to ten replicas with a pool of ten each. Managed PostgreSQL has a connection cap by tier. Watch it break. This is the Milestone 4 exercise arriving as a production incident — fix it with pool sizing or a pooler.
5. **Scale test.** Load-test until autoscale triggers. Measure scale-out latency and cold-start effects. Then scale to zero and measure the first-request penalty; decide whether that is acceptable for your users.
6. **Point-in-time restore.** Restore the database to a timestamp fifteen minutes ago, into a new server. Verify the data. Time it. That is your RPO and RTO evidence, not an estimate.
7. **Kill a revision.** Deploy a version that fails its health probe. Confirm traffic never shifts to it. Then deploy one that passes probes but is functionally broken, and practise the rollback.
8. **Break identity.** Remove a role assignment and watch the failure mode. Is the error message diagnosable? Would your alerts have caught it?
9. **KQL practice.** Answer from Log Analytics alone: the slowest ten requests in the last hour, the error rate by endpoint, allocated inference use by tenant today, and every request in one trace.
10. **Two-target cost model.** From measured GPU time, tokens, request rates, storage, and operations effort, compute allocated monthly on-prem cost at 100, 1,000, and 10,000 diagnoses. Separately use the Azure pricing calculator for the synthetic cloud target. Identify the dominant line item in each rather than assuming it is inference.

## 4. Project Implementation

### Step 1 — Foundation

For the synthetic Azure target: resource groups per environment (`dev`, `demo`), consistent tagging for cost attribution, a budget with alerts **before** provisioning anything expensive, and Entra groups for human access. The company target uses the equivalent environment, ownership, and budget boundaries selected in ADR-012.

### Step 2 — Infrastructure as code

Bicep modules for the synthetic Azure target — network, data, messaging, cache, container environment, identity, monitoring — composed per environment with parameter files. Nothing configured by hand. The test is destructive and simple: delete the Azure dev resource group and rebuild it from code. Define the company target with its approved IaC/configuration mechanism too; document any platform capability that prevents parity rather than routing around it manually.

### Step 3 — Identity and secrets

| Concept | Engineering problem | Implementation |
|---|---|---|
| Workload identity | Stored credentials leak, expire, and get committed | Separate managed identities for the Azure API and worker; use the company platform's workload-identity equivalent on-prem, each least-privileged |
| Passwordless data access | Connection strings are long-lived secrets | Entra authentication to Azure data services; use the approved identity mechanism for company-hosted services |
| Remaining secrets | Certificates or legacy credentials may still exist | Key Vault in Azure and the approved internal secret store on-prem; never in environment variables in the repo |
| Human identity | Local password auth was a learning exercise | Entra ID OIDC for users; keep the local provider swappable for demos |
| CI identity | Long-lived deploy credentials are a standing risk | GitHub Actions OIDC federation — zero stored cloud secrets |

### Step 4 — Data services

For the synthetic Azure target: PostgreSQL Flexible Server with `pgvector` enabled, backup retention and PITR configured, an HA decision made explicitly against a stated RTO/RPO (and its cost), connection limits matched to replica count and pool size, and network restricted to private access. Blob Storage has lifecycle rules for synthetic artifacts; Service Bus has DLQ and alerting; Managed Redis is sized for the cache and limiter. Map each to the approved company-hosted service for the real-data target and record operational ownership.

### Step 5 — Compute and deployment

The Azure target uses Container Apps for the API and worker plus a Container Apps job for migrations. The company target deploys the same application and worker image digests to its approved container platform, with the self-hosted inference server deployed and versioned separately. The deployment sequence matters:

1. Build once in CI, record the digest, push to ACR, and mirror the same digest to the approved internal registry when direct pull is not allowed.
2. Run the migration job **first**, using backward-compatible (expand) migrations only.
3. Deploy a new revision with zero traffic; run smoke tests against it.
4. Shift traffic progressively — 10%, 50%, 100% — watching error rate and latency.
5. Roll back by shifting traffic to the previous revision, which is still there.
6. Run contract (contract-phase) migrations only after the old revision is retired.

### Step 6 — Monitoring

Point the Azure target's OpenTelemetry exporter at Application Insights and the company target at its approved internal backend, without prompt bodies. Rebuild the four Milestone 11 dashboards in both. Write runbooks in `runbooks/`: inference saturation/outage, database failover, DLQ growth, deployment failure, capacity/cost spike, and identity or secret-store failure.

### Step 7 — Scaling

HTTP-concurrency scaling for the API and queue-depth scaling for the worker, both parameterised from Milestone 10 and 11 measurements. Set minimum replicas deliberately — scale-to-zero saves money and costs cold starts where the platform supports it. Set maximum replicas from the *downstream* limits: database connections, inference concurrent-sequence capacity, KV cache, and sustainable tokens/s. Scaling stateless services beyond the fixed GPU capacity converts a queue into an outage.

**Re-measure on production hardware before trusting any limit.** The bulkhead and queue bounds in ADR-007 are tagged with the configuration they were measured on. If the production GPU, model revision, quantisation, serving-stack version, or context length differs from that tag in any way, re-run the Milestone 6 concurrency curve on the production inference host, re-derive the Milestone 7 limits, and update the tag before sending traffic. The startup check from Milestone 7 should refuse a mismatched configuration; do not work around it by editing the tag.

### Step 8 — Prove it operationally

- Failure drills: inference-server outage/saturation, database failover, Redis down, broker unavailable, identity failure, bad deployment.
- A point-in-time restore, verified and timed.
- A rollback, performed and timed.
- A load test against each target with a written comparison; only synthetic data enters Azure.
- A Well-Architected review across all five pillars, with accepted risks written down.
- Separate on-prem allocation and Azure synthetic-target cost models, with assumptions, measured usage, and the dominant driver identified.

## 5. Expected Project State

**Before:** a complete system running locally under Compose, with no proved deployment target.

**After:** a real-data production candidate on company infrastructure and a synthetic-only Azure environment, both reproducible from code, secretless where possible, scaled or admission-controlled from measured signals, monitored with runbooked alerts, and proven under deliberately induced failure with rehearsed restore and rollback.

**Architecturally:** portability is proved without pretending that cloud deployment overrides data residency. Call the company deployment a *production candidate* until it has real users and operational ownership; call Azure a *synthetic portability target*, never production.

## 6. Testing and Validation

- Post-deployment smoke tests against a zero-traffic revision before any traffic shifts.
- Infrastructure rebuild test: destroy and recreate dev from Bicep.
- Identity tests: managed identity works; removing a role assignment fails in a diagnosable way.
- Migration tests: forward against a copy of production-shaped data, with rollback rehearsed.
- Load test with autoscale enabled, measuring scale-out latency and the downstream limits reached.
- Failure drills with alerts firing and runbooks followed.
- Restore drill verifying data completeness and elapsed time.
- The full approved evaluation suite run inside the company boundary against the deployed candidate before release; only the synthetic subset runs in Azure.

## 7. Failure Scenarios to Reproduce Deliberately

1. Deployment fails health probes — confirm traffic never shifts.
2. Functionally broken revision passing probes — practise rollback under time pressure.
3. Database connection limit exhausted by scaled replicas.
4. Inference saturation under load — queue bounds, fallback, and degradation must hold.
5. Key Vault access denied after an identity change.
6. Private endpoint or DNS misconfiguration making the database unreachable from the app but reachable from your laptop.
7. Azure region capacity refusal for the optional synthetic inference host or a container revision.
8. Cost spike from a runaway retry loop — caught by the budget alert.
9. DLQ growth with no consumer — alert must fire.
10. Secret rotation while requests are in flight.
11. Scale-to-zero cold start on the first request after idle.
12. Migration applied to production before the code that requires it — the classic self-inflicted outage.

## 8. Skills Gained

- Azure resource model, identity, networking, managed data services, containers, and monitoring
- Infrastructure as code with reproducible environments, and OIDC-federated deployment with no stored secrets
- Managed identity and passwordless service authentication
- Autoscaling designed from measurements and constrained by downstream limits
- KQL-based diagnosis, alerting with runbooks, and rehearsed restore and rollback
- Two-target cost modelling and Well-Architected trade-off reasoning

**Job-description mapping:** completes *Azure or another major cloud platform* and *cloud-native solutions* with operational evidence rather than a deployment screenshot.

## 9. Milestone Completion Checklist

- [ ] I destroyed the dev environment and rebuilt it entirely from Bicep.
- [ ] No connection string or key exists in Azure application configuration; company workloads use their approved workload identity or secret-injection mechanism, with no secrets in the image or repository.
- [ ] CI deploys via OIDC with no stored cloud credentials.
- [ ] I saturated the self-hosted inference server deliberately, confirmed bounded-queue and fallback behaviour, and know the measured throughput ceiling.
- [ ] Capacity limits were re-measured on the production inference host, and their configuration tag matches what is actually running.
- [ ] Real-data workloads deny outbound traffic by default; model weights and tokenisers load from an internal mirror with no public-hub access; a probe from inside each workload confirms public destinations are unreachable.
- [ ] I hit the database connection limit under scale and fixed it.
- [ ] Scale rules come from measured numbers, and maximum replicas respect downstream limits.
- [ ] I performed a point-in-time restore and recorded how long it took.
- [ ] I deployed a broken revision and rolled back, and timed the recovery.
- [ ] Migrations run as a separate gated step and are backward compatible.
- [ ] I can diagnose a slow request on the company target using its internal telemetry, and on the Azure target using traces and KQL alone.
- [ ] Every alert has a runbook, and I have followed one during a drill.
- [ ] I can state allocated on-prem cost per 1,000 diagnoses, the Azure synthetic-target cost, and the dominant driver in each.
- [ ] I completed a Well-Architected review and wrote down the accepted risks.
- [ ] I can name the AWS or GCP equivalent of every Azure service I used.
- [ ] **ADR-012** records both deployment targets, their data boundary, hosting, identity, IaC, inference-capacity, and scaling decisions with alternatives.

---

## Final checkpoint — the whole roadmap

| Job requirement | Final evidence |
|---|---|
| Python | Typed FastAPI service and workers, async I/O, concurrency, profiling, layered tests |
| APIs and backend systems | Versioned REST contract, auth, RBAC, PostgreSQL, transactions, queues, cache, idempotency |
| AI applications | Provider abstraction, structured outputs, RAG with citations, guardrails, evaluation, bounded tools |
| Cloud-native solutions | Containers, IaC, CI/CD, autoscaling, observability, rollback, failure drills |
| Azure | Synthetic-only target using ACA, ACR, PostgreSQL Flexible Server, Blob, Service Bus, Managed Redis, Key Vault, Entra, and Monitor |
| Collaboration | ADRs, threat model, runbooks, evaluation reports, reviewable pipeline and infrastructure |

**You are done when you can answer all nine of these with evidence from the repository:** how it works, why it was designed this way, how it fails, how it is tested, how AI quality is measured, how it is deployed, how it scales, how it is monitored, and what you would improve next — with a measurement, not a trend, justifying the priority.

For what comes after — fine-tuning, a dedicated vector service, write-capable tools, multi-agent designs, Kubernetes, multi-region — see *Evidence-Driven Extensions* in `../production_ai_engineer_project_roadmap.md`. The rule there is the same one that governed this roadmap: baseline, hypothesis, smallest experiment, evaluation, decision, guarded rollout.
