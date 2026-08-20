# Milestone 12 — Azure: Infrastructure as Code, Deployment, Scaling, and Operations

**Goal:** OpsPilot runs on Azure, provisioned from code, deployed by a pipeline with no long-lived secrets, authenticated by managed identity, autoscaled from real signals, monitored with alerts and runbooks, and proven through failure drills, a restore, and a rollback.

**Why now:** everything the cloud demands is already true — the service is stateless, configuration is external, state is in backing services, images are hardened, health probes exist, and telemetry is instrumented. Deploying is now a matter of provisioning and wiring, not rearchitecting.

**Why Azure:** the job description names it. The concepts transfer to any major cloud, and you should be able to state the AWS or GCP equivalent of each service you use.

**Time:** 6–8 focused weeks. **Prerequisite:** Milestone 11 checklist green. **Set a budget alert before you provision anything.**

---

## 1. Topics to Learn

### Cloud fundamentals
| Topic | Why it matters |
|---|---|
| Regions, availability zones, latency | Where you deploy affects latency, cost, data residency, and which services and model deployments are even available. |
| Shared responsibility model | Which failures are the provider's problem and which remain yours. Most people assume too much is handled. |
| IaaS / PaaS / serverless trade-offs | Control versus operational burden. You are choosing PaaS deliberately, and should be able to say why. |
| Consumption pricing and quotas | Cloud bills are a function of design decisions. Quotas are hard walls you must design around. |

### Azure specifics
| Service / concept | Why it is in this architecture |
|---|---|
| Subscriptions, resource groups, tags, RBAC | Lifecycle, cost attribution, and least-privilege boundaries |
| Microsoft Entra ID | Human identity — replaces the local accounts from Milestone 5 |
| Managed identities | Workload identity with **no stored secrets**. The single biggest security improvement available to you. |
| Azure Key Vault | The few secrets that must exist (a third-party API key), with rotation and access policy |
| Azure Container Registry | Stores the image built in Milestone 11; deployment references it by digest |
| Azure Container Apps | Managed containers with ingress, revisions, scale rules, and jobs — without operating Kubernetes |
| Azure Database for PostgreSQL Flexible Server | Managed Postgres with backups, PITR, HA options, and `pgvector` |
| Azure Blob Storage | Durable artifact storage with lifecycle rules; container disks are ephemeral |
| Azure Service Bus | The durable broker behind the Milestone 10 port |
| Azure Managed Redis | The shared cache and distributed limiter |
| Azure OpenAI or an external provider | Region, quota (TPM/RPM), content filtering, and data-handling terms all differ — this is a decision, not a default |
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
3. **Hit a quota deliberately.** Drive your model deployment past its tokens-per-minute limit. Observe throttling, confirm your Milestone 7 retry and fallback logic handles it, and record the effective ceiling. Your capacity plan is now based on measurement.
4. **Connection-limit collision.** Scale the API to ten replicas with a pool of ten each. Managed PostgreSQL has a connection cap by tier. Watch it break. This is the Milestone 4 exercise arriving as a production incident — fix it with pool sizing or a pooler.
5. **Scale test.** Load-test until autoscale triggers. Measure scale-out latency and cold-start effects. Then scale to zero and measure the first-request penalty; decide whether that is acceptable for your users.
6. **Point-in-time restore.** Restore the database to a timestamp fifteen minutes ago, into a new server. Verify the data. Time it. That is your RPO and RTO evidence, not an estimate.
7. **Kill a revision.** Deploy a version that fails its health probe. Confirm traffic never shifts to it. Then deploy one that passes probes but is functionally broken, and practise the rollback.
8. **Break identity.** Remove a role assignment and watch the failure mode. Is the error message diagnosable? Would your alerts have caught it?
9. **KQL practice.** Answer from Log Analytics alone: the slowest ten requests in the last hour, the error rate by endpoint, token spend by tenant today, and every request in one trace.
10. **Cost model.** From measured tokens, request rates, and the Azure pricing calculator, compute monthly cost at 100, 1,000, and 10,000 diagnoses. Identify the dominant line item. It is usually the model, not the infrastructure — and that finding shapes where optimisation effort belongs.

## 4. Project Implementation

### Step 1 — Foundation

Resource groups per environment (`dev`, `prod`), consistent tagging for cost attribution, a budget with alerts **before** provisioning anything expensive, and Entra groups for human access.

### Step 2 — Infrastructure as code

Bicep modules — network, data, messaging, cache, container environment, identity, monitoring — composed per environment with parameter files. Nothing configured by hand. The test is destructive and simple: delete the dev resource group and rebuild it from code. Until you have done that, you have scripts, not infrastructure as code.

### Step 3 — Identity and secrets

| Concept | Engineering problem | Implementation |
|---|---|---|
| Workload identity | Stored credentials leak, expire, and get committed | Separate managed identities for API and worker, each with least-privilege role assignments |
| Passwordless data access | Connection strings are long-lived secrets | Entra authentication to PostgreSQL, Blob, Service Bus, and Redis |
| Remaining secrets | Third-party keys must live somewhere | Key Vault, referenced by the container app; never in environment variables in the repo |
| Human identity | Local password auth was a learning exercise | Entra ID OIDC for users; keep the local provider swappable for demos |
| CI identity | Long-lived deploy credentials are a standing risk | GitHub Actions OIDC federation — zero stored cloud secrets |

### Step 4 — Data services

PostgreSQL Flexible Server with `pgvector` enabled, backup retention and PITR configured, an HA decision made explicitly against a stated RTO/RPO (and its cost), connection limits matched to replica count and pool size, and network restricted to private access. Blob Storage with lifecycle rules for artifacts. Service Bus queues with DLQ and alerting. Managed Redis sized for the cache and limiter.

### Step 5 — Compute and deployment

Container Apps environment with an API app (HTTP ingress, TLS, custom domain) and a worker app (queue-based scaling), plus a Container Apps job for migrations. The deployment sequence matters, and this is where the Milestone 11 lesson becomes concrete:

1. Build once in CI, push to ACR, record the digest.
2. Run the migration job **first**, using backward-compatible (expand) migrations only.
3. Deploy a new revision with zero traffic; run smoke tests against it.
4. Shift traffic progressively — 10%, 50%, 100% — watching error rate and latency.
5. Roll back by shifting traffic to the previous revision, which is still there.
6. Run contract (contract-phase) migrations only after the old revision is retired.

### Step 6 — Monitoring

Point the OpenTelemetry exporter at Application Insights. Rebuild the four Milestone 11 dashboards in Azure Monitor. Configure the four alerts with action groups. Write runbooks in `runbooks/`: provider outage, database failover, DLQ growth, deployment failure, cost spike, identity or Key Vault failure.

### Step 7 — Scaling

HTTP-concurrency scaling for the API and queue-depth scaling for the worker, both parameterised from Milestone 10 and 11 measurements. Set minimum replicas deliberately — scale-to-zero saves money and costs you cold starts. Set maximum replicas from the *downstream* limits: database connections and model quota. Scaling beyond what your dependencies can absorb converts a slow system into a broken one.

### Step 8 — Prove it operationally

- Failure drills: provider outage, database failover, Redis down, broker unavailable, identity failure, bad deployment.
- A point-in-time restore, verified and timed.
- A rollback, performed and timed.
- A load test against staging with a written report.
- A Well-Architected review across all five pillars, with accepted risks written down.
- A cost model with assumptions, measured usage, and the dominant cost driver identified.

## 5. Expected Project State

**Before:** a complete system running locally under Compose.

**After:** a production-candidate deployment — reproducible from code, secretless where possible, autoscaled from measured signals, monitored with runbooked alerts, and proven under deliberately induced failure with a rehearsed restore and rollback.

**Architecturally:** the final architecture in `../production_ai_engineer_project_roadmap.md` is now real rather than drawn. Call it a *production candidate* until it has real users and operational ownership — that honesty is itself a signal of seniority.

## 6. Testing and Validation

- Post-deployment smoke tests against a zero-traffic revision before any traffic shifts.
- Infrastructure rebuild test: destroy and recreate dev from Bicep.
- Identity tests: managed identity works; removing a role assignment fails in a diagnosable way.
- Migration tests: forward against a copy of production-shaped data, with rollback rehearsed.
- Load test with autoscale enabled, measuring scale-out latency and the downstream limits reached.
- Failure drills with alerts firing and runbooks followed.
- Restore drill verifying data completeness and elapsed time.
- The full evaluation suite run against the deployed candidate before release.

## 7. Failure Scenarios to Reproduce Deliberately

1. Deployment fails health probes — confirm traffic never shifts.
2. Functionally broken revision passing probes — practise rollback under time pressure.
3. Database connection limit exhausted by scaled replicas.
4. Model quota (TPM) throttling under load — fallback and degradation must hold.
5. Key Vault access denied after an identity change.
6. Private endpoint or DNS misconfiguration making the database unreachable from the app but reachable from your laptop.
7. Region capacity refusal for a model deployment or a container revision.
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
- Cost modelling and Well-Architected trade-off reasoning

**Job-description mapping:** completes *Azure or another major cloud platform* and *cloud-native solutions* with operational evidence rather than a deployment screenshot.

## 9. Milestone Completion Checklist

- [ ] I destroyed the dev environment and rebuilt it entirely from Bicep.
- [ ] No connection string or key exists in application configuration; workloads use managed identity.
- [ ] CI deploys via OIDC with no stored cloud credentials.
- [ ] I hit the model quota deliberately and confirmed fallback behaviour; I know my throughput ceiling.
- [ ] I hit the database connection limit under scale and fixed it.
- [ ] Scale rules come from measured numbers, and maximum replicas respect downstream limits.
- [ ] I performed a point-in-time restore and recorded how long it took.
- [ ] I deployed a broken revision and rolled back, and timed the recovery.
- [ ] Migrations run as a separate gated step and are backward compatible.
- [ ] I can diagnose a slow request in production using traces and KQL alone.
- [ ] Every alert has a runbook, and I have followed one during a drill.
- [ ] I can state cost per 1,000 diagnoses and the dominant cost driver.
- [ ] I completed a Well-Architected review and wrote down the accepted risks.
- [ ] I can name the AWS or GCP equivalent of every Azure service I used.
- [ ] **ADR-012** records the hosting, identity, IaC, and scaling decisions with alternatives.

---

## Final checkpoint — the whole roadmap

| Job requirement | Final evidence |
|---|---|
| Python | Typed FastAPI service and workers, async I/O, concurrency, profiling, layered tests |
| APIs and backend systems | Versioned REST contract, auth, RBAC, PostgreSQL, transactions, queues, cache, idempotency |
| AI applications | Provider abstraction, structured outputs, RAG with citations, guardrails, evaluation, bounded tools |
| Cloud-native solutions | Containers, IaC, CI/CD, autoscaling, observability, rollback, failure drills |
| Azure | ACA, ACR, PostgreSQL Flexible Server, Blob, Service Bus, Managed Redis, Key Vault, Entra, Monitor |
| Collaboration | ADRs, threat model, runbooks, evaluation reports, reviewable pipeline and infrastructure |

**You are done when you can answer all nine of these with evidence from the repository:** how it works, why it was designed this way, how it fails, how it is tested, how AI quality is measured, how it is deployed, how it scales, how it is monitored, and what you would improve next — with a measurement, not a trend, justifying the priority.

For what comes after — fine-tuning, a dedicated vector service, write-capable tools, multi-agent designs, Kubernetes, multi-region — see *Evidence-Driven Extensions* in `../production_ai_engineer_project_roadmap.md`. The rule there is the same one that governed this roadmap: baseline, hypothesis, smallest experiment, evaluation, decision, guarded rollout.
