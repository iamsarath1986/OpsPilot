# Milestone 11 — Observability, Performance, Containers, and CI/CD

**Goal:** make the system explain itself. Distributed tracing across API, workers, retrieval, and the model; metrics and SLOs with meaningful alerts; a load test that finds the real bottleneck; hardened container images; and a pipeline that builds one artifact, gates it, and promotes it.

**Why now:** Milestone 12 deploys to a machine you cannot attach a debugger to. Everything you cannot see there, you must have instrumented here. Building observability after deployment means learning to debug production blind.

**Time:** 4–5 focused weeks. **Prerequisite:** Milestone 10 checklist green.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| Logs, metrics, traces — and why traces lead | "Which of my nine steps made this request slow?" is a question only a trace answers. |
| OpenTelemetry: spans, attributes, context propagation | A vendor-neutral standard, so your instrumentation survives a change of monitoring backend. |
| Context propagation across process boundaries | A trace that stops at the queue boundary hides exactly the part you need. |
| Sampling | 100% tracing at scale is unaffordable; head and tail sampling trade cost against the chance of capturing the rare bad request. |
| Structured logs with correlation IDs | The field you added in Milestone 2 finally pays off: log line → trace → request. |
| RED and USE metric methods | Rate, Errors, Duration for requests; Utilisation, Saturation, Errors for resources. Two checklists that cover most of what you need. |
| Metric cardinality | A label containing an incident ID creates millions of time series and a very large bill. This is the classic self-inflicted observability outage. |
| AI-specific telemetry | Prompt version, model revision/quantisation, decoding configuration, tokens in and out, queue time, tokens/s, allocated compute cost, retrieval hit rate, guardrail trips, abstention rate, outcome state, `degradation_reason`, and `output_source`. The last three are bounded enums, not free-text labels. |
| What *not* to record | Prompt and response bodies contain customer data. Sampling, redaction, and retention policy are mandatory, not optional. |
| SLIs, SLOs, error budgets | Turns reliability into a number that decides whether you ship a feature or fix stability. |
| Alerting on symptoms, not causes | Alert on "users are getting errors", not on "CPU is 80%". Cause-based alerts produce noise and get muted. |
| Load testing: open versus closed models | A closed-loop test with fixed users hides queueing behaviour that real traffic exposes. |
| Percentiles and tail latency | Averages hide the experience of your unluckiest users. p99 is a real customer. |
| Little's Law and saturation | Predicts queueing behaviour and tells you which resource to add. |
| Docker layers, multi-stage builds, `.dockerignore` | Build speed, image size, and not shipping your `.env` into a registry. |
| Non-root containers, minimal base images | Reduces blast radius and the CVE surface you must patch. |
| Health checks, `SIGTERM`, graceful shutdown | A container killed mid-request drops user work. Graceful shutdown is what makes rolling deploys invisible. |
| Image scanning and SBOM | Knowing what is inside what you deploy, and being told when it becomes vulnerable. |
| Pipeline stages and caching | Fast feedback, or developers route around the pipeline. |
| Build once, promote by digest | The artifact tested in staging must be *bit-identical* to the one in production. Rebuilding per environment defeats the entire point of testing. |
| Deployment strategies: rolling, blue-green, canary | Different trade-offs of risk, cost, and complexity. |
| Migration ordering and backward compatibility | The classic outage: new code deployed before its migration, or a migration that breaks the still-running old version. |
| Rollback | The most important deployment capability, and the least practised. |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Tracing | **Productionize** | One trace shows API, DB, retrieval, model, and worker spans, correlated |
| Metrics | **Design** | You chose your metrics and labels deliberately, with cardinality in mind |
| AI telemetry | **Productionize** | You can answer "what fraction degraded, why, and which fallback produced the answer?" as well as why quality, latency, or capacity changed |
| SLOs and alerts | **Design** | Every alert is symptom-based and has a runbook |
| Load testing | **Debug** | You found a real bottleneck and fixed or documented it |
| Docker | **Productionize** | Multi-stage, non-root, scanned, with health checks and graceful shutdown |
| CI/CD | **Design** | You can justify every gate and explain your promotion and rollback model |

## 3. Hands-On Learning Exercises

1. **One request, one trace.** Instrument, fire a diagnosis request, and view the waterfall. The first time you see retrieval taking 60% of a request you assumed was model-bound, the value of tracing stops being theoretical.
2. **Break propagation.** Publish a job without injecting trace context. Watch the worker's spans become orphans. Fix it. This is the most common tracing bug in queue-based systems.
3. **Cardinality bomb.** Add `incident_id` as a metric label. Watch the series count explode. Remove it, and write the rule you derived on the dashboard description.
4. **PII in a span.** Attach a raw prompt to a span attribute, then find customer data in your monitoring tool. Implement a redacting span processor and a sampling policy for prompt bodies.
5. **Find the bottleneck.** Load-test the diagnosis path with an open model at increasing arrival rates. Watch p50 hold steady while p99 explodes; find the saturating resource — usually the database pool, the model concurrency limit, or worker count.
6. **Kill it mid-request.** `SIGTERM` a container while it is serving. Does the request complete? Implement graceful shutdown with a drain period and try again.
7. **Image diet and hardening.** Build naive, then multi-stage with a slim base. Compare size and build time. Run as root, then as a non-root user. Scan both and compare CVE counts.
8. **Break the pipeline.** Push a change that fails types, then one that fails an eval gate, then one that produces a broken image. Confirm each is caught by the stage you expect.
9. **Practise rollback.** Deploy a deliberately broken version locally, then roll back. Time it. That number is your recovery time, and you should know it before you need it.

## 4. Project Implementation

### Step 1 — Instrumentation

| Concept | Engineering problem | Implementation | Testing |
|---|---|---|---|
| Tracing | Multi-step requests are opaque | OpenTelemetry SDK; auto-instrument FastAPI, SQLAlchemy, httpx, Redis; manual spans for retrieval, rerank, model call, validation, guardrails | Assert a full trace exists end to end for one request |
| Cross-process context | Traces die at the queue | Inject and extract trace context in message headers | Worker spans join the API's trace |
| Structured logs | Correlation between logs and traces | JSON logs carrying `trace_id`, `span_id`, `request_id`, tenant | Every request log line is correlatable |
| RED metrics | Basic service health | Request rate, error rate, duration histograms by route and status | Metrics appear with bounded label sets |
| AI metrics | Quality and capacity use are invisible otherwise | Tokens, queue time, tokens/s, allocated compute cost, latency by stage, retrieval hit rate, guardrail trips, abstention/escalation rates, and outcome counts broken down by bounded `degradation_reason` and `output_source`, sliced by prompt, model revision/quantisation, and decoding configuration | Dashboard answers "what fraction degraded, why, and what changed after the last release?" |
| Privacy controls | Telemetry becomes a data leak | Redacting span processor, no raw prompt/response body export, bounded redacted metadata, explicit retention | A test asserting no raw customer content reaches span attributes or exporter payloads |
| Egress-policy gate | Telemetry adds another outbound path after model, embedding, rerank, and judge calls | Extend the standing deny-by-default job with the configured telemetry exporter; allowlist the approved internal backend and assert raw marker content is redacted before export | CI fails on any unapproved destination or any unredacted marker in a telemetry export |

### Step 2 — Dashboards, SLOs, and alerts

Four dashboards: user experience (rate, errors, latency by endpoint); AI quality and capacity (outcome states, degraded fraction split by `degradation_reason` and `output_source`, abstention, tokens, GPU time and allocated cost per successful diagnosis); system health (pool usage, inference queue depth, worker throughput, cache hit rate); and dependencies (inference-server errors, breaker state, overload rejections).

Then four alerts, each with a runbook: elevated user-facing error rate; p95 latency beyond the SLO; queue lag or DLQ growth; and inference degradation beyond its reason-specific budget — including any `decoder_constraint_unavailable`, sustained `capacity_saturated`, primary failure, or circuit-breaker-open. Symptom-based only. An alert without a runbook is a 3 a.m. puzzle.

### Step 3 — Load testing

k6 or Locust scripts for the three main paths. Report format: environment, traffic model, arrival rates, p50/p95/p99, error rate, the saturating resource, and what you changed. Record the self-hosted inference ceiling: concurrent sequences, KV-cache pressure, tokens/s, queue growth, and the arrival rate at which latency collapses. Milestone 12 uses those measurements for admission control and maximum replica counts.

Once these scripts work locally, run them on a rented L4 and L40S for an afternoon (about $15, synthetic data only) as described in [GPU sizing and cost](03-gpu-sizing-and-cost.md#8-replace-the-estimate-rent-a-gpu-for-an-afternoon), and fill in its measured results table. The development machine's integrated GPU cannot tell you what a server GPU will do; this is the cheapest way to find out before anyone buys hardware.

### Step 4 — Containers

Multi-stage Dockerfiles for API and worker: build stage with toolchain, runtime stage slim and non-root, no secrets in layers, `.dockerignore` covering `.env`, `.git`, and tests. Health-check endpoints wired to the platform's probes. Graceful shutdown handling `SIGTERM`: stop accepting, drain in-flight work, close pools, exit. For workers, that means finishing or safely abandoning the current message so it is redelivered rather than lost.

A `docker-compose.yml` bringing up the full stack — API, worker, PostgreSQL with `pgvector`, Redis, broker, and an OTel collector with a local tracing backend — so a new contributor gets a complete system with one command.

### Step 5 — The pipeline

```
lint → types → unit → component → integration (real Postgres) → security scan
 → egress-policy → small eval suite → build image → scan image → push by digest → deploy (M12)
```

Plus nightly full evaluation and a scheduled dependency scan. Build once, tag by digest, promote the same digest. Secrets via OIDC federation rather than long-lived credentials — set up properly in Milestone 12. Branch protection requiring the pipeline to pass.

## 5. Expected Project State

**Before:** a capable system you could only understand by reading code.

**After:** every request traceable across processes, quality and inference-capacity use visible on dashboards, SLOs with symptom-based alerts and runbooks, a measured capacity limit and known bottleneck, hardened images with graceful shutdown, and a pipeline producing one gated, promotable artifact.

**Architecturally:** unchanged in shape, transformed in operability. This is the milestone that makes the difference between a system you built and a system you can run.

## 6. Testing and Validation

- A test asserting a complete trace for one diagnosis request, including worker spans.
- A test asserting no raw prompt or customer data appears in span attributes or logs.
- The standing `egress-policy` job now covers generation, embeddings, reranking, judging, and telemetry in one deny-by-default integration flow; raw marker content is expected only at allowlisted internal AI endpoints and must be absent from telemetry exports. In CI every allowlisted endpoint, including the telemetry backend, is a recording stub on the isolated network, so the job needs no GPU or live model.
- Metric cardinality check: assert label sets are bounded.
- Outcome telemetry test: every degraded result increments exactly one known reason/source combination; complete results do not acquire a degradation reason.
- Container tests: non-root user, health endpoints, and graceful shutdown under `SIGTERM` mid-request.
- Load-test report committed with the environment and conditions stated.
- Pipeline verification: deliberately fail each gate and confirm the expected stage catches it.
- An image scan with a documented policy for what severity blocks a release.

## 7. Failure Scenarios to Reproduce Deliberately

1. Trace context lost at the queue boundary, leaving orphaned worker spans.
2. Cardinality explosion from a high-uniqueness label.
3. Customer data in telemetry — find it in your own monitoring tool, then close it off.
4. Alert storm from a cause-based alert firing on every deploy.
5. p99 latency collapse under load while p50 looks perfectly healthy.
6. Container killed mid-request, dropping user work; then fixed with graceful drain.
7. Worker killed mid-message: confirm redelivery rather than loss, thanks to Milestone 10.
8. Image running as root; then non-root, and the permission problem that surfaces when you switch.
9. CI green but the image broken — because a stage tested source rather than the artifact.
10. Migration deployed after the code that requires it, breaking every request until it lands.
11. A rollback attempted for the first time under pressure, without practice.

## 8. Skills Gained

- OpenTelemetry instrumentation with cross-process context propagation
- Metric design with cardinality awareness, and AI-specific quality, capacity, and allocated-cost telemetry
- Privacy-aware observability — a genuinely differentiating skill in AI systems
- SLOs, symptom-based alerting, and runbooks
- Load testing to find real bottlenecks, and reasoning about tail latency
- Hardened container images, health probes, and graceful shutdown
- A CI/CD pipeline built on build-once-promote, with practised rollback

**Job-description mapping:** *cloud-native solutions* directly. Also the operational half of *real-world AI products* — most AI prototypes have no observability at all, so this is a strong differentiator to demonstrate.

## 9. Milestone Completion Checklist

- [ ] One trace shows API, database, retrieval, model, validation, and worker spans together.
- [ ] I broke trace propagation across the queue and fixed it.
- [ ] I caused a cardinality explosion and can state my labelling rule.
- [ ] No raw prompt, response, or customer data reaches logs or spans — with a test.
- [ ] The standing `egress-policy` CI job covers every outbound AI and telemetry surface, fails on unapproved destinations, and proves raw marker content is absent from telemetry exports.
- [ ] I can answer "what changed after the last prompt release?" from a dashboard.
- [ ] I can answer "what fraction of answers degraded, why, and from which output source?" and alert on reason-specific budgets.
- [ ] Every alert is symptom-based and has a runbook.
- [ ] I load-tested to saturation and can name the first bottleneck and the fix.
- [ ] I know the measured inference concurrency/KV-cache limit and tokens/s ceiling that cap my throughput.
- [ ] Images are multi-stage, non-root, scanned, and contain no secrets.
- [ ] A `SIGTERM` mid-request completes gracefully; a worker's message is redelivered, not lost.
- [ ] I deliberately failed each CI gate and confirmed which stage caught it.
- [ ] The same image digest is promoted, never rebuilt per environment.
- [ ] I have performed a rollback and timed it.
- [ ] **ADR-011** records the observability strategy, telemetry privacy policy, and release model.

---

**Next:** Milestone 12 proves the real-data on-premises target and the synthetic-only Azure portability target — infrastructure as code, workload identity, autoscaling/admission control, alerting, failure drills, restore, rollback, and a cost model.
