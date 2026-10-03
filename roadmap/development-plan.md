# OpsPilot Development Plan

Created: 2026-10-03. Status: planned; implementation has not started.

This document turns the twelve milestone roadmaps into an ordered development and learning plan. The goal is an API-first, read-only incident diagnosis service for hosting support engineers, with evidence-backed results, honest fallback and abstention, and a deployment that respects the company data boundary.

Use this plan to choose the next task and track evidence. Use each linked milestone for its full lessons, exercises, failure scenarios, and completion checklist. A working feature does not by itself complete a milestone.

## Contents

- [Product and constraints](#product-and-constraints)
- [How we will work together](#how-we-will-work-together)
- [Schedule and dependencies](#schedule-and-dependencies)
- [Architecture and repository structure](#architecture-and-repository-structure)
- [Parallel workstreams](#parallel-workstreams)
- [Milestone implementation plans](#milestone-implementation-plans)
- [First four weeks](#first-four-weeks)
- [Verification and release gates](#verification-and-release-gates)
- [Decisions and external prerequisites](#decisions-and-external-prerequisites)
- [Progress tracking](#progress-tracking)
- [Project completion](#project-completion)

## Product and constraints

### Product behavior

A first-level support engineer submits an incident and log excerpt. OpsPilot identifies deterministic signals, retrieves relevant hosting procedures when RAG is available, and returns a structured diagnosis with references to the exact log evidence and authorized runbook passages. A senior engineer receives a useful escalation packet when the system cannot answer safely.

The final response exposes severity, summary, likely causes, recommended steps, evidence, citations, escalation and unsupported flags, execution outcome, output source, and sufficient prompt/model/configuration provenance to reconstruct the run. Token use, queue time, latency, and generation throughput support capacity analysis.

Sources: [Product definition](00-what-you-are-building.md), [discovery findings](02-discovery-with-operations.md), and [system design](images/opspilot-system-design.svg).

### Binding constraints

1. Customer data stays on approved company infrastructure. Generation, embeddings, reranking, and judging are self-hosted. Telemetry must not export raw prompts, completions, or customer content.
2. Development uses synthetic examples and incidents reproduced in disposable local environments. Real customer tickets and log samples are not requested or committed.
3. Azure and rented GPU tests contain synthetic data only. Azure has no route into the real-data storage or inference plane.
4. The product diagnoses and recommends. It does not execute shell commands, change customer servers, browse the internet, or automatically apply fixes.
5. Runbook diagnosis branches and safe steps are written and checked by a person. Codex may assist with structure and formatting, without generating the operational knowledge used as authority.
6. Capacity and quality claims come from measurements of the exact deployed configuration. Estimates in the GPU document are not purchase specifications.

Source: [ADR-000](01-the-binding-constraint.md). The accepted residency decision takes precedence over implementation convenience.

### Scope of this build

Build the backend, CLI, deterministic baseline, self-hosted AI pipeline, RAG, evaluation system, bounded read-only tools, background processing, observability, delivery pipeline, and both deployment targets.

A frontend, ticketing-system integration, fine-tuning, automatic remediation, autonomous agents, Kubernetes, and multi-region deployment are outside the twelve-milestone implementation. Consider extensions only after evidence identifies a need.

## How we will work together

The agreed approach is learning-first guided implementation. You own the product decisions, write the learning-critical core code, author runbooks, perform exercises, and explain the results. Codex helps break down work, explain concepts, scaffold when requested, review your implementation, debug failures, and design meaningful verification.

For each work package:

1. Read the relevant roadmap section and state the behavior to implement.
2. Run the relevant small learning exercise before changing the project code.
3. Define the inputs, outputs, invariants, and expected failures.
4. Implement one reviewable change and its meaningful tests.
5. Review the code and reproduce the failure cases in an isolated environment.
6. Record measurements, decisions, and remaining gaps.
7. Update the task status and discuss the next package.

Keep commits small and focused when you choose to commit. Do not batch later milestones into the current implementation. Codex should only create application code for the task you explicitly ask it to implement; presenting a plan or explaining a concept does not authorize a bulk implementation.

### Definition of milestone completion

Every milestone needs four kinds of evidence:

| Area | Evidence required |
|---|---|
| Implementation | The milestone behavior works through its intended interface and respects the architecture. |
| Verification | Required automated checks, experiments, and failure drills pass or have an explicitly documented disposition. |
| Learning | You can explain the concepts, alternatives, and failure behavior without relying on a pasted answer. |
| Documentation | ADR, measurements, examples, limitations, and the source milestone checklist are up to date. |

Automation cannot mark authorship or understanding as complete. In particular, the M1 requirement to write the core yourself remains a learning gate even if an example implementation exists.

## Schedule and dependencies

The individual roadmap ranges add up to **48–61 focused weeks**, roughly 11–14 months before interruptions, at 10–15 hours per week. This corrects the earlier conversational total of 47–61 weeks. The roadmap index's 9–12 month estimate is optimistic relative to its own milestone ranges.

These are planning ranges, not deadlines or permission to skip gates. Runbook authoring is additional work that must fit into the weekly budget; the M8 estimate covers retrieval engineering, not writing the source material.

| Phase | Milestone | Weeks | Deliverable |
|---|---|---:|---|
| Backend foundation | M1 Domain and parsing | 3–4 | Typed package, deterministic analysis, session CLI |
| Backend foundation | M2 Production Python | 3–4 | Async boundaries, configuration, tests, initial CI |
| Backend foundation | M3 HTTP and API | 3–4 | Versioned API and client CLI |
| Backend foundation | M4 Persistence | 4–5 | Durable PostgreSQL state and migrations |
| Backend foundation | M5 Identity and privacy | 4–5 | Secure tenant-scoped backend |
| Controlled AI | M6 First LLM slice | 3–4 | Experimental diagnosis with measured inference |
| Controlled AI | M7 Reliable orchestration | 4–5 | Validated results, fallback, abstention, demo |
| Controlled AI | M8 RAG | 5–6 | Authorized retrieval and verified citations |
| Controlled AI | M9 Evaluation and guardrails | 5–6 | Quality gates and model confirmation |
| Production operations | M10 Jobs and tools | 4–5 | Durable workers, shared limits, bounded tools |
| Production operations | M11 Observability and delivery | 4–5 | Measured capacity, hardened images, pipeline |
| Production operations | M12 Deployment targets | 6–8 | Company production candidate and Azure portability |

The backend phase is 17–22 weeks, controlled AI is 17–21, and production operations is 14–18. The operations demo falls at the end of M7, approximately 24–31 engineering weeks from the start if earlier gates clear.

### Dependency gates

- M1 through M12 proceed sequentially after the prior completion review.
- M6 additionally requires 12–20 bootstrap incident cases, including at least four unanswerable cases, and a measured redaction pipeline.
- M8 additionally requires 20–30 versioned runbooks with recorded authorship/review status and enough coverage for at least 40 labelled retrieval questions.
- M9 starts with at least 40 authored incident cases and a frozen held-out split; it expands the set to 60–100, including at least 15 unanswerable cases.
- M12 additionally requires measured server GPU results, a company deployment decision, named infrastructure owners, and a written target/data-boundary ADR.
- Ops review and the M7 demo are valuable but do not block M8 or M9 if ops is unavailable. Record the missing review and qualify claims accordingly.

## Architecture and repository structure

### Architecture progression

Start with a modular monolith. Keep pure domain behavior separate from use cases and external adapters. Add distribution only when the roadmap introduces durable workers in M10.

| Milestones | Architectural change |
|---|---|
| M1 | Pure domain, synchronous use cases, injected ports, in-memory adapters |
| M2 | Async I/O boundaries; domain and deterministic rules remain synchronous |
| M3 | HTTP and client CLI adapters around the same use cases |
| M4–M5 | External durable state, transactions, identity, mandatory tenant scope |
| M6–M7 | Self-hosted inference behind a controlled, validated pipeline |
| M8–M9 | Retrieval and evaluation as separately measurable subsystems |
| M10 | Durable messaging, workers, shared cache and admission controls |
| M11–M12 | Instrumentation, immutable release artifacts, reproducible deployment |

Domain imports must not depend on application, adapters, parsing implementations, the ORM, or the web framework. Application code depends on domain types and ports. The composition root wires adapters and parsing implementations into those ports.

### Shared result model

Use one domain claim/result shape for deterministic and model output:

- `Incident`: identity, tenant, title, description, status, severity, creation time.
- `LogArtifact`: identity, incident, origin, byte/line/encoding metadata, content reference.
- `Evidence`: stable identity, artifact, original line number, exact source text.
- `Finding`: category, severity, confidence, statement, evidence references, producer identity.
- `Analysis`: findings, evidence, recommended steps, derived severity, provenance, creation time, nullable summary initially.

Map API `likely_causes` entries onto `Finding`; map `cause` onto `statement`. Add diagnosis outcomes, citations, unsupported reasons, and escalation fields as later milestones arrive. Keep Pydantic boundary DTOs separate from domain entities rather than creating a competing AI-only domain model.

### Repository conventions

Create directories when their milestone needs them. This is the intended structure, not a claim that the files already exist:

```text
pyproject.toml
src/opspilot/
  domain/                 entities, enums, errors, invariants
  application/            use cases and ports
  parsing/                streaming formats and deterministic rules
  adapters/               CLI, files, memory, HTTP, AI, storage, messaging
  infrastructure/         database mappings, migrations integration, composition
tests/
  unit/
  component/
  contract/
  integration/
experiments/              isolated learning and measurement scripts
samples/                  deliberately synthetic fixtures
corpus/runbooks/          human-authored hosting guidance for retrieval
evals/
  faults/                 isolated fault reproduction scripts
  bootstrap/
  datasets/
  retrieval/
  graders/
  runners/
  reports/
prompts/                  immutable versioned prompt artifacts
architecture/
  decisions/
  notes/
runbooks/                 procedures for operating OpsPilot itself
infra/                    target-specific infrastructure as code
roadmap/                  learning roadmaps and this plan
```

`corpus/runbooks/` and `runbooks/` serve different audiences. Only corpus sources may enter ingestion.

### Implementation defaults

These are proposed starting choices to record and defend in the relevant ADR:

| Concern | Starting choice | Revisit condition |
|---|---|---|
| Python and packaging | Python 3.12, `src/` layout, `pyproject.toml`, editable installation | Deployment compatibility or measured benefit from a runtime change |
| Developer checks | pytest, Ruff, mypy; dependency locking in M2 | A concrete missing capability |
| M1 CLI state | An interactive or scripted single-process session | M3 replaces it with an API client |
| Dependencies | Minimal runtime initially; add FastAPI in M3 and SQLAlchemy/Alembic in M4 | A milestone demonstrates the need |
| Incident identities | UUIDs; inject clock and identifier generation | Storage or performance evidence |
| Log input | Bounded UTF-8 streaming; reject compression and unsupported encoding initially | A demonstrated support need with safe decoding limits |
| Database values | Checked status/severity values with explicit migrations | Measured benefit from a different representation |
| Artifact content | File reference through a storage port; later approved object storage | Target requirements; do not place large logs in database text columns |
| Identity | Local Argon2 accounts first; swappable validation and later OIDC | Company identity and deployment requirements |
| Tenant isolation | Mandatory repository tenant scope; assess RLS as defense in depth | Compliance, isolation requirements, or a demonstrated bypass |
| AI fallback | Validated deterministic rules, then abstention | A measured secondary-model strategy fits latency and capacity budgets |
| Local serving | llama.cpp with Vulkan on the documented development machine | Different measured hardware/serving requirements |
| Retrieval | PostgreSQL full-text plus pgvector and rank fusion | Measured scale or relevance limitations |
| Local durable broker | RabbitMQ behind a broker port | Company-approved platform requirements; Azure uses Service Bus |
| Shared cache | Redis; cache failures become misses | Measured operational requirements |
| Shared admission failure | Fail closed for new inference when distributed coordination is unavailable; preserve safe rule fallback | A reviewed policy that still protects the global capacity ceiling |
| Load testing | Locust with an open arrival model | A required capability the harness cannot provide |

Do not preselect model names, quantization, embedding models, index tuning, timeout values, or replica counts from estimates. Their selection procedure is part of the implementation, and the measurement-backed settings must be committed before a gate clears. Validate current dependency versions and platform capabilities when their milestone starts.

## Parallel workstreams

### Hosting runbook corpus

Start during M1 and continue through M7. Begin with the four deterministic-rule categories, then expand using discovery hypotheses and available aggregate ops feedback: quotas, database limits, SSL, DNS, mail delivery, redirects, PHP compatibility, slow sites, compromised accounts, and outbound spam. High-risk categories may document escalation rather than a fix.

Each runbook needs scope, symptoms, evidence to collect, diagnosis branches, safe next steps, escalation conditions, author, review status, version, and source references. Record reviewer/date if reviewed; publish a new version when content changes. A versioned release can remain `unreviewed`.

Suggested checkpoints within the overall time budget: four foundational drafts by M2, 10–15 by M5, and 20–30 before M8. These intermediate counts are planning targets; the M8 readiness gate is required.

Ops-reviewed guidance can describe company procedure. Unreviewed guidance describes the author's procedure and must be labelled accordingly in demos and reports. Reassess whether reviewed company procedure may leave the company boundary before using it in cloud measurements.

### Incident cases and fault library

Start with small reproduced faults during M1–M5. A script runs only against disposable local infrastructure, captures the logs actually produced, and records the injected fault as ground truth. Never create faults on production servers.

Each case records stable ID, provenance, author, customer-voice ticket, log artifact reference, injected fault or independent reference facts, expected findings, forbidden claims, answerability, slice/risk tags, and applicable corpus version. Add expected retrieval sources when the corpus is available.

| Checkpoint | Required or planned data |
|---|---|
| Before M6 | Required: 12–20 bootstrap cases with at least four unanswerable |
| Before M6 prompt experiments | Required: held-out membership reserved and frozen for available non-bootstrap cases |
| Before M8 experiments | Required: at least 40 labelled retrieval questions; held-out cases remain excluded from tuning |
| Before M9 | Required: at least 40 authored incident cases with a frozen held-out split |
| During M9 | Required: 60–100 cases, at least 15 unanswerable, held-out share at least 30% |

Retrieval questions and incident cases are different datasets. Meeting the 40-question retrieval gate does not automatically meet the 40-incident gate.

Follow the six anti-contamination rules in [ADR-000](01-the-binding-constraint.md#the-risk-grading-your-own-homework): independent ground truth, customer language, early held-out split, separate authorship where possible, missing-answer cases, and provenance-labelled reports. Bootstrap cases can join development data, never held-out data. Scenario references need another person's agreement; unreviewed scenarios may remain drafts without blocking progress on reproduced cases.

Reserve held-out membership as independent cases are created from the start, before any prompt or chunking tuning. Assign new independent cases to a split before running them; keep at least 30% of the eventual full incident dataset held out. Freezing starts early even though the full dataset does not exist yet.

### Operations discovery and demo

Use the existing findings rather than reopening the confirmed residency decision. Seek aggregate ticket mix, first-level resolution baseline, escalation triggers, risk tolerance, and the precise infrastructure/runbook boundary. Record confirmed answers separately from hypotheses. You own outreach; this plan does not send messages.

At the end of M7, prepare a 30-minute demo for the operations manager and one or two first-level engineers. Show complete, degraded, and abstained results, fabricated-evidence rejection, and cases where rules still win. Request reviews of a few runbooks and remembered scenarios, not customer tickets. Record attendees, feedback, and review changes, or record that the demo could not be scheduled.

### Model and GPU measurements

M6 selects a provisional local demo model. M9 confirms the model using deployable quantized weights, constrained decoding, and the full authored dataset. M11 measures server-class GPU performance using synthetic data and fills the [GPU sizing results table](03-gpu-sizing-and-cost.md#9-measured-results). M12 repeats model confirmation and capacity sizing on the actual company host.

Tag every run with GPU/memory, model revision, quantization, serving stack/version, and maximum context. A change invalidates the old capacity limits. Record token throughput, queue time, TTFT, end-to-end percentiles, memory, and the concurrency knee. Check rental rates when booking; the roadmap prices are dated estimates.

## Milestone implementation plans

Work packages below are ordered within each milestone. Every package is initially not started. The final gate also includes the full source roadmap checklist.

### Milestone 1 Domain and deterministic analysis

Source: [M1 roadmap](milestone-01-domain-and-parsing.md). Entry: none. Duration: 3–4 weeks.

**Learn:** generators, context managers, dataclasses/enums, composition, protocols, exceptions, static typing, pytest, packaging, and regex limitations.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M1.1 | Package skeleton, minimal dependencies, pytest/Ruff/mypy configuration; reconcile requirements and document the obsolete scratch layout | Editable package imports outside the project directory; no web/DB/AI runtime dependency required |
| M1.2 | Frozen entities, enums, explicit incident transitions, injected clock/IDs, shared finding/analysis shape | Invalid and boundary values fail with specific errors; illegal transitions cannot occur |
| M1.3 | Error hierarchy and artifact-reader/parser ports | Unsupported input, empty content, byte limits, state conflicts, and missing incidents are distinguishable |
| M1.4 | Bounded reader, nginx and PHP-FPM parsers, format detection, explicit generic mode, parse/skipped counts | Streaming retains original line numbers; bad lines are counted; unreadable formats fail loudly |
| M1.5 | Four rules: permission denied, disk full, upstream timeout, PHP fatal; stable evidence; conservative recommendations | Matching/nonmatching/near-miss tests pass; claims cite exact lines without inventing root causes |
| M1.6 | Create/attach/analyze/show use cases, repository protocols, in-memory implementations and contracts | Business logic does not depend on CLI or concrete repository; errors leave coherent state |
| M1.7 | Session CLI, synthetic fixtures, usage documentation, rules baseline and ADR-001 | Complete workflow runs in one process; restart loses state explicitly; checks pass |

Bound both reading and retained analysis output. A generator is insufficient if every parsed line or match is accumulated afterward. Preserve bounded evidence samples and report omitted matches rather than silently dropping them. Verify referenced content has not changed between attachment and analysis, or copy it through a controlled artifact port.

**Exercises:** 500 MB eager-versus-streaming memory comparison; dictionary-versus-dataclass invalid states; injected-clock tests; parser abuse; transition graph; nginx versus unsupported Apache/JSON inputs. The generic fallback must be selected explicitly so auto-detection never disguises an unknown format as success.

**Tests and drills:** legal/illegal transitions; zero-byte and oversized files; UTF-16/latin-1 rejection; malformed timestamp mid-file; truncated final line; oversized single line; renamed gzip; duplicate evidence IDs; fabricated evidence references; every rule's near misses. Repeat analysis 100 times with fixed clock and IDs to verify byte-identical domain results. Remove a validation branch in an isolated mutation experiment and demonstrate a test failure, then restore it.

**Exit evidence:** clean lint/types/tests, repository contracts, measured memory report, at least five documented source failure scenarios, reviewed CLI example, frozen `rules@v1` baseline, ADR-001, and your explanation/authorship review. Do not claim human learning gates from generated code.

### Milestone 2 Production Python

Source: [M2 roadmap](milestone-02-production-python.md). Entry: M1 complete. Duration: 3–4 weeks.

**Learn:** event loop, tasks, cancellation, bounded concurrency, threads/processes, fakes, typed settings, logging, profiling, locking dependencies, and CI.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M2.1 | Async repository/artifact I/O ports and fakes; synchronous domain/rules | Same external behavior and repository semantics; async contracts pass |
| M2.2 | Parsing offload, bounded fan-out, per-port deadlines and translated errors | Concurrent work remains responsive; configured bounds hold |
| M2.3 | Resource cleanup and cooperative cancellation for offloaded work | Cancellation closes resources and releases capacity; timed-out work does not keep consuming unbounded resources |
| M2.4 | Injected pydantic-settings configuration and structured logging | Invalid settings fail at startup; secrets never appear in repr/logs; correlation field exists |
| M2.5 | Unit/component/contract markers, fixture isolation, dependency lock | Clean environments resolve the same dependencies; unit suite stays under one second on the recorded machine |
| M2.6 | GitHub Actions lint/types/tests/coverage; profile one optimization; ADR-002 | A deliberate bad branch fails the expected gate; before/after numbers recorded |

Offload streaming work as a bounded operation or bounded producer/consumer flow; do not materialize all lines to cross the async boundary. Cancelling an awaiting coroutine does not automatically terminate its worker thread. Use an explicit stop/cleanup strategy and test resource release, not only that the caller stopped waiting.

**Exercises:** sequential/async/thread timing on I/O and CPU tasks; blocked loop; cancellation debris; semaphore sweep; fake-versus-mock defect; randomized ordering; parser profiling.

**Tests and drills:** unawaited coroutine, blocked event loop, unbounded fan-out, timeout with continuing work, leaked handles/semaphores, shared mutable fixtures, missing setting, secret in logs. Repeat unit tests in randomized order and verify isolated state.

**Exit evidence:** async boundaries and deadlines, cleanup mutation test, configuration privacy checks, timing/profile report, reproducible install, effective CI/branch protection where available, ADR-002, and concurrency explanation review.

### Milestone 3 HTTP and API contract

Source: [M3 roadmap](milestone-03-http-and-api.md). Entry: M2 complete. Duration: 3–4 weeks.

**Learn:** HTTP semantics/statuses/headers, REST resources, idempotency, pagination, compatibility, problem details, ASGI, validation, and dependency lifecycle.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M3.1 | `create_app(settings)` composition and lifespan-managed dependencies | Independent tests/apps do not share state; startup/shutdown clean up |
| M3.2 | Incident CRUD, pasted artifact attachment, analysis creation/listing | Routes delegate to use cases and appear in versioned OpenAPI |
| M3.3 | Separate request/response DTOs with strict validation and input caps | Invalid/extra/oversized inputs cannot enter domain logic |
| M3.4 | Problem-details error mapping, request IDs, bounded pagination | Every error shape is consistent, correlated, and free of internal details |
| M3.5 | Tenant placeholder, create idempotency, liveness/readiness | Placeholder is documented as unauthenticated; concurrent replay semantics tested; readiness reflects required dependencies |
| M3.6 | ASGI tests, OpenAPI snapshot, concurrency baseline, CLI API client, ADR-003 | Twenty concurrent analysis requests remain responsive; API is the product surface |

Public routes:

| Route | Purpose |
|---|---|
| `POST /v1/incidents` | Create; accept `Idempotency-Key` |
| `GET /v1/incidents` | Tenant-scoped listing with capped `limit`/`offset` and total |
| `GET /v1/incidents/{id}` | Fetch incident |
| `PATCH /v1/incidents/{id}` | Change status/severity; illegal transition returns 409 |
| `POST /v1/incidents/{id}/artifacts` | Attach bounded pasted log text |
| `POST /v1/incidents/{id}/analyses` | Create deterministic analysis |
| `GET /v1/incidents/{id}/analyses` | List previous analyses |
| `GET /health/live`, `GET /health/ready` | Process life and dependency readiness |

Use `X-Tenant-Id` only as a temporary local-development shortcut; ADR-003 sets removal at M5. Scope idempotency to tenant and request identity; same key with incompatible input must not return an unrelated result. In-memory state and deduplication are process-local until M4.

**Exercises:** inspect raw HTTP with curl; compare resource shapes; force Pydantic failures; race duplicate creates; deliberately block async endpoints; demonstrate additive/breaking OpenAPI changes; justify error mappings.

**Tests and drills:** every endpoint's success/invalid/missing/conflict/dependency-down paths; malformed JSON, wrong content type, oversized body rejected before full buffering; disconnect handling; request correlation; idempotency races; readiness failure; accidental contract change.

**Exit evidence:** documented OpenAPI/error contract, thin routes, API-driven CLI, concurrency/latency baseline, temporary tenancy recorded, ADR-003, and HTTP/compatibility explanation review.

### Milestone 4 PostgreSQL and persistence

Source: [M4 roadmap](milestone-04-postgresql-and-persistence.md). Entry: M3 complete. Duration: 4–5 weeks.

**Learn:** relational modeling, SQL, constraints, indexes, query plans, transactions/MVCC, optimistic locking, deadlocks, pools, ORM sessions, migrations, and restore.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M4.1 | Local PostgreSQL Compose service with pinned compatible version, health check, volume | Reproducible database startup |
| M4.2 | Tenants, incidents, artifacts, analyses, findings, evidence, idempotency schema | FKs/checks prevent invalid relationships; evidence retains source identity |
| M4.3 | ORM/domain mapping, repository adapters, storage references, use-case unit of work | Round trips and existing contracts pass; failure rolls back the entire use case |
| M4.4 | Session lifecycle, optimistic version checks, query-matched indexes | Pool returns connections after errors; one conflicting update returns 409 |
| M4.5 | Reviewed Alembic migrations and real-PostgreSQL integration tests | Empty DB reaches head; newest migration can be downgraded in test; existing data survives forward migration |
| M4.6 | Seed/query benchmarks, timed backup/restore, ADR-004 | Recorded plans justify indexes; restored data is complete |

Map findings and evidence without changing the M1 shared claim shape. Evidence belongs to the analysis/artifact and findings reference it, allowing multiple findings to cite the same line without duplicating ambiguous IDs. Preserve confidence, statement, producer, and recommended steps in persistence. Commit transactions at use-case boundaries, not inside repository methods. Compensate or clean up artifact writes that cannot share the database transaction.

**Exercises:** SQL before ORM; 200,000-incident list/index experiment; index write cost; N+1 query count; lost update; deadlock; pool exhaustion; restore; migration backfill/rollback.

**Tests and drills:** duplicate-create race, rollback after partial writes, FK/check violations, cascade policy, concurrent updates, leaked session, DB down/readiness, lock-heavy migration on disposable data, unindexed query. Use PostgreSQL rather than SQLite for integration tests.

**Exit evidence:** durable state across restart, repository contracts, transaction/concurrency tests, migration CI, measured query improvements, `runbooks/restore.md`, ADR-004, and SQL/session/isolation explanation review.

### Milestone 5 Identity tenancy and privacy

Source: [M5 roadmap](milestone-05-identity-tenancy-and-privacy.md). Entry: M4 complete. Duration: 4–5 weeks.

**Learn:** authentication versus authorization, password hashing, JWT validation, identity delegation, RBAC, tenant isolation, object authorization, upload abuse, redaction, audit, and threat modeling.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M5.1 | Users/roles/memberships, Argon2 login, `POST /v1/auth/token`, replaceable identity validator | Passwords/hashes never returned or logged; credentials fail safely |
| M5.2 | Verified principal with user/tenant/roles; signature, issuer, audience, expiry checks | Missing/forged/expired/wrong-audience tokens rejected |
| M5.3 | Repository tenant chokepoint and role policy; remove header trust | All cross-tenant routes fail; full role-by-endpoint matrix passes |
| M5.4 | Bounded streamed uploads, generated storage names, type/encoding checks, cleanup | Traversal, disguised files, compression bombs, oversized bytes/lines rejected safely |
| M5.5 | Redaction before durable storage, artifact reports, audit trail, login limits | Raw originals never persisted; seeded-secret recall/false positives measured; sensitive reads/writes audited |
| M5.6 | Isolation alternatives/RLS assessment, threat model, key-rotation policy, ADR-005 | Accepted risks and boundaries are explicit; auth survives tested rotation behavior |

Use redaction on both pasted text and uploads. Temporary raw data must remain bounded and controlled, with no raw copy left in durable quarantine, caches, logs, or audit records. `viewer`, `engineer`, and `admin` are starting roles; define permissions explicitly in the matrix before implementation. Do not trust a client-supplied tenant header after this milestone.

**Exercises:** forge tokens, IDOR hunt, compare enforcement locations, upload attack corpus, 200-line planted-secret corpus, login timing, STRIDE review.

**Tests and drills:** every route/resource across tenants and roles; token algorithms/signatures/claims; new-route scoping omission; disk pressure; secret missed by redaction; brute-force lockout/recovery; key rotation with in-flight tokens. Repeat restore verification with new tables.

**Exit evidence:** auth and tenant tests, upload suite, recorded redaction rates/residual risks, immutable audit behavior, threat model, ADR-005, and security design explanation review. This is the backend phase checkpoint; customer-data use still requires an approved company environment.

### Milestone 6 First self hosted LLM slice

Source: [M6 roadmap](milestone-06-first-llm-slice.md). Entry: M5 complete and bootstrap/redaction gates clear. Duration: 3–4 weeks.

**Learn:** tokens/context, sampling variance, GPU/KV-cache capacity, prompt roles, injection, serving errors, licenses, and replaceable inference boundaries.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M6.1 | Local llama.cpp serving and 3–4 candidate comparison across fitting size tiers | Smallest provisional model clears a predeclared demo bar; actual quantized weights evaluated |
| M6.2 | Async `DiagnosisProvider`, typed redacted request/response, real and fake adapters | Tests need no live model/GPU; serving metadata and failure categories retained |
| M6.3 | Prompt builder with policy/task/data separation, numbered evidence, token budget | Prompt snapshots and oversize tests preserve signal and record dropped context |
| M6.4 | `POST /v1/incidents/{id}/diagnoses`, per-tenant feature flag, experimental result | Authorized end-to-end call uses stored redacted artifacts and rules |
| M6.5 | `llm_calls` ledger for every attempt/outcome and configuration identity | Success, timeout, overload, refusal, and error recorded with correlation |
| M6.6 | Isolated egress-policy job, local capacity measurements, baseline report, ADR-006 | Marker data reaches only approved recording stubs; local limits are measured and tagged |

The provider boundary must retain model name/revision, quantization, serving version, input/output tokens, queue/generation/end-to-end timing where available, finish reason, and server request ID. Missing server measurements remain explicitly unavailable; do not manufacture GPU-seconds or throughput figures.

Keep this slice experimental and free-form. Retries, structured validation, and RAG arrive in subsequent milestones so their effects can be measured independently.

**Exercises:** log tokenization, variance at temperatures 0/1, context ablation, 50-call concurrency sweeps at 1/2/4/8 or safe stopping point, error stubs, injection attempt, rules-versus-model comparison on the bootstrap set.

**Tests and drills:** hanging server, overload/auth failure, context overflow, empty/refused output, contradiction with deterministic evidence, large-input capacity spike. Use deny-by-default recording stubs for CI egress; CI proves the test network policy, not production enforcement.

**Exit evidence:** frozen bootstrap dev cases with provenance, per-case rule/model wins and losses, provisional-model limits, token/context and tagged capacity reports, ledger completeness, egress gate, ADR-006, and inference concepts explanation review.

### Milestone 7 Reliable AI orchestration

Source: [M7 roadmap](milestone-07-reliable-ai-orchestration.md). Entry: M6 complete with recorded capacity curve. Duration: 4–5 weeks.

**Learn:** constrained decoding, cross-field/evidence validation, vacuity, deadlines, error classification, retry budgets, jitter, circuit breaking, bulkheads, caching, and terminal states.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M7.1 | Extend shared Analysis; Pydantic DTO and serving schema/grammar; typed unsupported result | Actual serving schema supports abstention; supported-but-empty content rejected |
| M7.2 | Always validate syntax, schema, business rules, and authorized evidence | Fabricated/cross-tenant IDs fail closed; broken constrained contract is a serving fault |
| M7.3 | One resilience policy with overall/attempt deadlines, classified retry/repair, jitter, breaker | Exact attempt counts and recovery transitions pass deterministic tests |
| M7.4 | Measured in-flight bound, queue depth/wait, per-tenant token bucket | Burst overload ends promptly; startup rejects stale capacity configuration |
| M7.5 | Cache port/in-process implementation, full configuration key, expensive-call idempotency | Tenant/input/model/prompt/decoder/schema separation and expiry tests pass |
| M7.6 | Outcome state machine, fallback comparison, immutable prompt registry | All results terminate; stored provenance reconstructs exact run artifacts |
| M7.7 | Bootstrap rerun, SLO measurement, ops demo, ADR-007 | Gains/degradation are reported honestly; reviewed content and feedback recorded |

Resilience policy: retry only classified pre-acceptance connection failures and healthy-server transient failures within remaining deadline/budget. Honor admission `429` only when its bounded delay fits. Queue timeout and GPU OOM do not retry. Context overflow permits one smaller deterministic rebuild. Permanent auth/license/schema-contract failures fail closed and alert. Explicit unconstrained mode gets all validation layers and at most one bounded output repair. No fallback may start work beyond the original deadline.

Default fallback is the already available validated rules result, then abstention. Compare a secondary local model experimentally, but enable it only if residency, loading latency, memory, and primary KV-cache effects fit the measured budget.

| Outcome | Output source | Degradation reason |
|---|---|---|
| `complete` | `primary_constrained` | Null |
| `degraded` | `fallback_model` or `deterministic_rules` | Required; excludes decoder-constraint-unavailable |
| `degraded` | `unconstrained_validated` | `decoder_constraint_unavailable` only |
| `abstained` | `none` | Null; typed abstention reason recorded separately |
| `escalated` | `none` | Null; escalation reason recorded separately |

Allowed degradation reasons initially are `capacity_saturated`, `primary_unavailable`, `circuit_open`, `deadline_exhausted`, `decoder_constraint_unavailable`, and `primary_validation_failed`. Unsupported results require a reason such as insufficient/conflicting evidence, out of scope, or safety policy; severity/confidence are null, cause/step/evidence arrays may be empty, and escalation is required. Validation failure with no valid fallback becomes abstention.

Cache keys include tenant, normalized input hash, prompt version, full model identity, quantization, decoding/configuration/schema version, generation settings, and later corpus/retrieval/embedding versions. Derived limits carry the full measurement tag and are invalidated by any configuration change.

**Exercises:** constrained/unconstrained comparison, vacuous output, retry classification/jitter, breaker recovery, deadline arithmetic, four fallback strategies, cache omissions, measured admission saturation, SLO definition.

**Tests and drills:** every permitted and forbidden outcome/source/reason combination; unknown evidence; retry storm; deadline mid-retry; stuck breaker; fallback loop; cross-version cache; unfair tenant burst; OOM/queue retries; context replay; empty 200 response; schema incapable of representing abstention.

**Exit evidence:** validated terminal behavior, measured limits and configuration check, reason/source observability, reproducible prompts, bootstrap improvement report, fallback-residency decision, ADR-007, and reliability explanation review. Target 99% of requests reaching a terminal state within 15 seconds at stated load; record gaps without claiming that fast fallback equals successful AI diagnosis.

### Milestone 8 Retrieval augmented generation

Source: [M8 roadmap](milestone-08-rag.md). Entry: M7 complete plus corpus readiness. Duration: 5–6 engineering weeks.

**Learn:** corpus hygiene, chunking, embeddings, vector metrics/indexes, lexical search, hybrid ranking, authorization, reranking, grounding, and retrieval evaluation.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M8.1 | Freeze versioned corpus, validate metadata/provenance, path-scoped loaders | Only approved corpus sources ingested; review status preserved |
| M8.2 | Normalization and structural chunking; idempotent/resumable ingestion | Procedures remain intact; rerun skips unchanged content; partial failure is recoverable |
| M8.3 | Local embedding port/adapter, bounded batches, identity/version metadata | Model/revision/dimensions/configuration mismatch fails closed |
| M8.4 | Documents/chunks, pgvector and lexical columns, measured HNSW setup | Correct metadata/indexes; query performance and recall documented |
| M8.5 | Authorized vector/lexical queries, reciprocal rank fusion, optional reranker | Unauthorized/deprecated chunks are excluded in SQL before retrieval |
| M8.6 | Token-budget context builder, deduplication, stable citations, low-relevance abstention | Citations resolve to exactly supplied authorized chunks; irrelevant retrieval cannot trigger generic answers |
| M8.7 | Retrieval harness, ablations, egress extension, ADR-008 | RAG improvement measured by slice; cache invalidation covers corpus/embedding/retrieval changes |

Support Markdown first, then the roadmap's HTML/PDF loader fixtures. Keep multi-step procedures intact; an over-budget procedure is handled explicitly rather than silently split into unsafe fragments. New embedding identity creates a separate index namespace and requires re-embedding before activation.

**Exercises:** 200/500/1000-token chunks with 0/10/25% overlap; two local embedding models; exact-token lexical/vector comparison; K=1/3/5/10/20; outdated-runbook conflict; deliberate procedure split; 50,000-chunk index comparison; RAG versus no RAG. Report high- and low-vocabulary-overlap queries separately.

**Tests and drills:** unauthorized retrieval under all filter combinations, fake citation, irrelevant retrieval, stale/deprecated source, mixed vector spaces, context overflow, duplicates, half-ingested document, exact error-code retrieval, operating runbook entering corpus. Extend egress tests to embeddings and reranking.

**Exit evidence:** at least 40 labelled queries, measured recall@K/precision@K/MRR/nDCG and latency, selected chunk/embedding/index settings, authorized citations, measured reranker decision, RAG ablation, ADR-008, and pipeline/failure-attribution explanation review.

### Milestone 9 Evaluation guardrails and escalation

Source: [M9 roadmap](milestone-09-evaluation-and-guardrails.md). Entry: M8 complete, at least 40 incident cases and frozen split. Duration: 5–6 weeks.

**Learn:** dataset/slice design, deterministic grading, judge bias/calibration, noise floor, guardrails, abstention trade-offs, human review, and operational-quality regressions.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M9.1 | Expand to 60–100 provenance-labelled cases, ≥15 unanswerable, ≥30% held out | Bootstrap excluded from held out; independent facts/customer language retained |
| M9.2 | Capacity-capped cached runner and deterministic/retrieval graders | Graders tested against deliberately right/wrong results; reports reconstruct configuration |
| M9.3 | Versioned local judge, blinded comparisons, human calibration | Agreement on 30 human-labelled answers and position bias documented |
| M9.4 | Input/output guardrails, destructive-action detection, confidence/grounding thresholds | Safety violations cannot reach callers; legitimate-input false positives measured |
| M9.5 | Escalation packet and authorized review endpoint | Human has incident/findings/passages/partial result/reason/correlation; resolutions remain dataset candidates |
| M9.6 | Quality release gates, three-run noise measurement, valid-but-degraded regression drill | Answer and operational-quality regressions fail expected gates |
| M9.7 | Full model confirmation, judge-path egress tests, ADR-009 | Smallest deployable candidate clears predeclared bars; claims limited to authored/reproduced data |

Run small development/regression suites on relevant PRs and the full non-held-out suite nightly/pre-release. Reserve held-out runs for controlled M9/M12 confirmation. This resolves the tension between routine full-suite CI and ADR-000's rule that held-out data is never used for tuning. If a held-out result informs tuning, replace that split before claiming another independent confirmation.

Ground truth is not copied from runbooks. Reports include counts by provenance/author/slice, safety findings, recall/grounding, noise, abstention, outcome mix, degradation reason/source, latency, and measured capacity. A company engineer's live resolution is not automatically exported into a repository dataset.

**Exercises:** three identical eval runs, 30-answer judge calibration, swapped-order pairwise comparison, intentionally degraded prompt, 25 adversarial inputs including retrieved-document injection, abstention threshold sweep, self-consistency capacity comparison, decoder-fallback regression, customer-versus-runbook wording experiment.

**Tests and drills:** wrong graders, fluent unsupported claims, noisy gate, ambiguous rubric, held-out contamination, guardrail false positives, over-abstention, incomplete escalation, leaked report secret, unsafe step, valid-but-weaker decoder behavior. No CI model requirement: deterministic tests use fakes; actual model evaluations are controlled GPU runs with versioned reports consumed by release gates.

**Exit evidence:** tested graders, calibrated judge, red-team suite, measured threshold/noise policy, escalation review, working hard blocks, model confirmation, ADR-009, and evaluation/statistical-limits explanation review. This is the controlled-AI phase checkpoint.

### Milestone 10 Async work shared controls and tools

Source: [M10 roadmap](milestone-10-async-work-and-tools.md). Entry: M9 complete. Duration: 4–5 weeks.

**Learn:** at-least-once delivery, outbox, idempotent consumers, visibility/redelivery, DLQ, backpressure, shared caching/limits, and authorized bounded tool calls.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M10.1 | Jobs and outbox tables; job state/progress/error API | Tenant-scoped `GET /v1/jobs/{id}`; coherent terminal transitions |
| M10.2 | Broker port, local durable adapter, publisher, worker acknowledgement/retry/DLQ | Commit/publish crash loses no work; poison messages leave active queue |
| M10.3 | Idempotent ingestion/embedding/re-embedding/eval handlers and replay command | Redelivery produces one committed effect; replay is explicit and observable |
| M10.4 | Long-work submission returns 202/job ID; worker bounds and correlation | Requests do not hold connections for minutes; queue/DB/inference bounds measured |
| M10.5 | Redis cache/limits, global concurrency leases, stampede protection | Two replicas share one capacity ceiling; cache loss degrades safely |
| M10.6 | Typed tool registry and deterministic orchestration state machine | Caller principal authorizes every tool; loop/time/token/capacity budgets terminate |
| M10.7 | Worker throughput/lag report, dependency-failure policies, ADR-010 | Explicit cache-versus-admission failure behavior, DLQ handling, and tool limits |

Diagnosis stays synchronous and deadline-bounded. Ingestion, embeddings, re-embedding, and scheduled evals become durable jobs. The transactional outbox guarantees recoverable publication; workers assume duplicates. Crash recovery may repeat an external inference call if its completion could not be durably recorded, so report repeated compute honestly rather than claiming exactly-once inference.

Tools are `get_incident(id)`, `search_runbooks(query, filters)`, and `check_service_status(service)`. Status lookup uses configured service identities/destinations rather than arbitrary model-supplied URLs. Audit sanitized arguments and bounded results. No write-capable tool is introduced.

A cache outage becomes a cache miss. A distributed limiter/concurrency-coordination outage must not permit all replicas to independently use the global GPU budget: default to no new inference admission with rules/abstention available. Expiring leases and worker cleanup prevent permanent capacity loss after a crashed replica; test ownership/release behavior.

**Exercises:** dual-write crash, duplicate delivery and compute waste, poison/DLQ/replay, 10,000-message queue lag, hot-key stampede, two-replica limiter, Redis outage, runaway tool loop, hostile tool arguments.

**Tests and drills:** crash before/after publish, worker death before ack, stale job status, duplicate effects, stuck/expired lease, document OOM, pool exhaustion, broker unavailable, Redis failure, unauthorized tool ID, invalid parameters, budget exhaustion. Ensure cache degradation does not become an inference stampede.

**Exit evidence:** durable job flow and contracts, single committed effects, DLQ replay, measured throughput, safe shared capacity/admission, three authorized audited tools, ADR-010, and delivery/failure-policy explanation review.

### Milestone 11 Observability containers and delivery

Source: [M11 roadmap](milestone-11-observability-docker-cicd.md). Entry: M10 complete. Duration: 4–5 weeks.

**Learn:** traces/logs/metrics, propagation, sampling/privacy, cardinality, RED/USE, SLOs/error budgets, open-model load tests, containers, shutdown, scanning, digest promotion, migration ordering, and rollback.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M11.1 | OpenTelemetry automatic/manual spans and cross-process propagation | Diagnosis trace covers relevant stages; job trace preserves its originating context |
| M11.2 | Correlated logs, bounded RED/AI/resource metrics, privacy/retention controls | No raw bodies/marker content in telemetry; labels remain bounded |
| M11.3 | Four dashboards and four symptom alerts with operating runbooks | User experience, AI quality/capacity, system, dependencies visible; alerts exercised |
| M11.4 | Open-arrival load harness, local bottleneck analysis, server GPU measurements | Report includes load, environment, percentiles, queue/KV-cache behavior, first saturation point |
| M11.5 | Hardened API/worker Dockerfiles and full Compose stack | Non-root, minimal runtime, no image secrets, health checks, graceful drain/redelivery |
| M11.6 | Pipeline gates, image scan/SBOM, immutable digests, rollback drill, ADR-011 | Tested image is the promoted artifact; deliberately broken stages fail |

Pipeline sequence: lint → types → unit → component → PostgreSQL integration → security scan → egress policy → small regression eval evidence → image build → image tests/scan → push by digest. Deployment is added in M12. Nightly non-held-out evaluation and dependency scanning complement fast PR checks.

Use a shared trace across API/worker only when a job originates from that request; do not require an unrelated scheduled worker to appear inside a synchronous diagnosis trace. Keep request/incident IDs in logs/traces rather than high-cardinality metrics. Version labels must be restricted to active configured versions, not arbitrary input.

Extend the isolated egress job through inference, embeddings, reranker, judge, and telemetry. AI stubs may receive authorized marker payloads; telemetry stubs must not receive raw marker content. Production egress enforcement remains an M12 host/network responsibility.

**Exercises:** trace waterfall, broken queue propagation, isolated cardinality bomb, sensitive span leak, saturation curve, SIGTERM mid-request/job, image-size/root comparison, failed pipeline gates, timed rollback.

**Tests and drills:** orphan spans, telemetry leaks, unbounded labels, alert noise, p99 collapse, killed worker/API, root-to-nonroot permissions, green-source/broken-image mismatch, bad migration ordering, rollback. Run rented GPU tests with synthetic data only after checking current rates; download results and remove resources/storage that would continue billing.

**Exit evidence:** dashboards/alerts/runbooks, privacy/egress tests, measured local and server GPU limits, updated sizing table, scanned images, shutdown tests, digest-based pipeline, timed rollback, ADR-011, and operability explanation review.

### Milestone 12 Company deployment and Azure portability

Source: [M12 roadmap](milestone-12-azure-production.md). Entry: M11 complete, measurements and ownership ready. Duration: 6–8 weeks.

**Learn:** target ownership, IaC, cloud identity/networking, managed services, quotas/budgets, revisions, downstream-limited scaling, KQL, RTO/RPO, disaster recovery, and cost allocation.

| Task | Implementation and deliverable | Acceptance |
|---|---|---|
| M12.1 | ADR-012 target map, approved company services/owners, Azure environment budget | Data boundary and responsibilities written; Azure budget alert exists before provisioning |
| M12.2 | Bicep modules/parameters and approved company IaC/configuration | Disposable dev target rebuilt from code; parity gaps documented |
| M12.3 | API/worker workload identities, OIDC CI, OIDC human auth, target secret stores | Least privilege verified by removing role assignments; no image/repository secrets |
| M12.4 | Private PostgreSQL/storage/broker/cache, retention/backup/HA/pool decisions | Services accessible only from intended workloads; RTO/RPO and ownership explicit |
| M12.5 | Internal model/image mirrors, offline runtime, deny-by-default egress | Public/model-hub probes fail from each real-data workload; approved internal calls succeed |
| M12.6 | Separate migrations, zero-traffic revision smoke tests, progressive deployment | Expand migrations precede code; same digest promoted; rollback shifts to retained revision |
| M12.7 | Target dashboards, measured scaling, model/capacity reconfirmation | Limits match actual host configuration; replicas respect DB and GPU ceilings |
| M12.8 | Restore/rollback/failure drills, target comparisons, cost models, final review | Timed recovery with verified data; synthetic Azure tests; documented accepted risks |

Azure service map: Container Apps API/worker and migration job; ACR; PostgreSQL Flexible Server with pgvector; Blob Storage; Service Bus; Managed Redis; Key Vault; Entra identity; Monitor/Log Analytics/Application Insights. Use optional short-lived self-hosted GPU serving for synthetic workloads if practical; otherwise label a synthetic adapter explicitly. It does not prove live-model GPU performance.

The company target uses approved internal equivalents, an internal telemetry backend, and internally mirrored model/tokenizer/image artifacts pinned by revision/checksum. Target vLLM for production serving on a compatible discrete GPU, subject to measured fit and the approved platform; keep llama.cpp as the local development adapter. Disable library telemetry and public startup downloads. No real-data workload uses Azure storage, inference, or telemetry.

Deployment order: build once → mirror identical digest as needed → backward-compatible expand migration → new revision with zero traffic → smoke tests → 10%/50%/100% traffic with monitoring → retire old revision → contract-phase migrations. Where the company platform lacks weighted traffic, document and validate its approved equivalent rather than claiming parity.

Re-run both the M6 capacity curve and M9 model confirmation on company hardware before use. Consider larger fitting tiers only when they improve quality beyond noise at acceptable latency/throughput. Derive API/worker replica maxima from database pool/connection caps, GPU concurrent sequences, KV cache, tokens/s, and measured queue behavior. Use separate allocated on-prem and synthetic Azure cost models.

**Exercises and drills:** rebuild disposable Azure dev; identity removal; GPU saturation; scaled DB pool exhaustion; autoscale/cold start; PITR into a new database; bad revision/rollback; private DNS failure; broker/cache/identity outage; secret rotation; quota refusal; cost spike. All induced destructive faults use disposable or explicitly approved drill environments, never an unapproved live customer system.

**Exit evidence:** reproducible targets, isolated data planes, workload/OIDC identity, enforced egress, measured capacity/configuration match, model reconfirmation, verified restore and timed rollback, fired alerts/runbooks, load/cost reports, Well-Architected review, ADR-012, and deployment/scale/recovery explanation review. Call the company environment a production candidate until it has real users and operational ownership.

## First four weeks

This is the initial execution schedule. If a gate needs more time, keep the task open and shift later work. Parallel writing stays within the weekly time budget.

| Week | Project work | Exercises and evidence | Parallel work |
|---|---|---|---|
| 1 | M1.1–M1.2: skeleton/tooling, entities/invariants, clocks/IDs | Dataclass versus dict, transition graph, typed tests, ADR-001 draft | Define runbook/case templates; list initial fault categories; define held-out policy |
| 2 | M1.3–M1.4: errors, bounded reader, supported parsers | UTF-8/encoding/compression/line-cap cases, malformed timestamp counts, memory comparison | Draft foundational hosting guidance yourself; capture first reproduced case |
| 3 | M1.5–M1.6: rules, evidence, use cases, in-memory contracts | Rule near misses, evidence integrity, repeatability with fixed dependencies | Add independently worded cases; record provenance; reserve held-out membership |
| 4 | M1.7: session CLI, synthetic examples, docs/baseline | End-to-end session, validation mutation, failure notes, complete M1 review | Review draft corpus/case inventory and schedule writing through M7 |

A suggested ten-hour week allocates three hours to concepts/exercises, four to implementation, two to tests/debugging/review, and one to corpus/case documentation. At fifteen hours, add implementation and parallel-data time. Treat this as a starting allocation and adjust using actual effort logs.

The first task is **M1.1**, discussed and implemented in a small step. No application implementation is created by saving this document.

## Verification and release gates

### Test progression

| Layer | Starts | What it protects |
|---|---|---|
| Unit | M1 | Domain invariants, parsers, rules; later validators, graders, policies |
| Component | M1/M2 | Use cases with injected fakes and deterministic dependencies |
| Adapter contract | M1 | Shared semantics across memory/Postgres, storage, provider, broker implementations |
| API contract | M3 | Real ASGI behavior, validation, error mapping, compatibility |
| Database integration | M4 | PostgreSQL transactions, constraints, locking, migrations, query behavior |
| Security/privacy | M5 | Tenant/role matrix, token/upload abuse, redaction/audit |
| Egress integration | M6 onward | Destination policy with recording stubs; extended per new outbound surface |
| Model/retrieval evaluation | M6 onward | Measured quality/capacity; independent datasets and versioned reports |
| Distributed faults | M10 | Outbox/redelivery, shared limits, lease recovery, DLQ/cache/tool behavior |
| Artifact/deployment | M11/M12 | Actual images, graceful shutdown, target identity/networking, restore/rollback |

Routine CI must not depend on a GPU or live model. Separate opt-in smoke/model evaluation jobs from deterministic tests. Real quality gates consume controlled, reproducible model-run results; canned responses cannot establish model quality. Bind evaluation reports to the exact candidate prompt, model, corpus, and configuration artifacts, and reject stale or unrelated evidence rather than letting an old passing report approve a new candidate.

### Hard blocks introduced in M9

- Any cross-tenant leak, secret leak, destructive recommendation, or fabricated citation.
- Unexpected `decoder_constraint_unavailable`, an unknown state/source/reason combination, or a degraded result without a reason.
- A critical-slice quality regression beyond the measured noise floor.
- Outcome or reason/source rates exceeding their versioned operational-quality budgets.

Latency and allocated-cost changes are reported with an explicit release decision, as the M9 roadmap specifies. Safety blockers have no override. The later deployment still has to satisfy its measured operating/capacity envelope.

### Measurement rules

Record the exact input/dataset/corpus split, machine, model/revision/quantization, serving/decoding/generation settings, context, concurrency/arrival model, timestamps, outcomes, and limitations. Compare rules, model-only, and RAG behavior on the same eligible cases. Do not turn authored-case accuracy into a customer-ticket claim.

Track both latency and useful completion. The 15-second terminal-state target can be met by abstention or fallback; it is not a quality score. Pair it with complete/degraded/abstained/escalated distributions and per-slice wrong-answer/abstention measures.

Retrieval recall thresholds, quality floors, redaction miss tolerance, retry budgets, queue bounds, SLO load envelope, RTO/RPO, retention, and image-scan severity policy are written before the applicable experiment/release gate. Measure to select them; do not invent unsupported numbers in this plan or tune a bar after seeing held-out results.

## Decisions and external prerequisites

### ADR sequence

| ADR | Decision area |
|---|---|
| ADR-000 | Accepted residency/self-hosting constraint; keep the roadmap source canonical |
| ADR-001 | Modular monolith, dependency direction, shared claim model, M1 session behavior |
| ADR-002 | Async boundaries, concurrency/offload and cancellation strategy |
| ADR-003 | REST/versioning/error contract, temporary tenant header, idempotency limits |
| ADR-004 | PostgreSQL, identities, constraints, artifact storage, transactions/deletion behavior |
| ADR-005 | Identity/token policy, roles, tenant isolation, redaction and residual risks |
| ADR-006 | Provider boundary, provisional model/quantization/serving, data path |
| ADR-007 | Validation/retry/deadline policy, outcomes/fallback residency, tagged capacity/cache |
| ADR-008 | Corpus/chunking, embeddings, hybrid retrieval, indexes/reranking, authorization |
| ADR-009 | Dataset splits, graders/judge, noise, safety/quality budgets, abstention/escalation |
| ADR-010 | Broker/outbox/workers, shared cache/admission failure policy, bounded tools |
| ADR-011 | Telemetry privacy, metrics/SLOs, images/scanning, promotion/rollback |
| ADR-012 | Company/Azure boundaries, services/ownership, identity/IaC, scaling/recovery/cost |

Every ADR states context, realistic alternatives, decision, consequences, and the evidence that would justify revisiting it. Link ADRs to actual measurements rather than copying the proposed defaults as if already validated.

### External questions and gates

| Question or risk | When to resolve | Behavior while unresolved |
|---|---|---|
| Exact company data boundary and runbook-IP policy | Discovery; mandatory before external use of reviewed material or company deployment | Use synthetic/self-authored public-safe material and the strict residency interpretation |
| Ticket mix, first-level baseline, wrong-answer cost | Early discovery and M7 feedback | Keep category priorities provisional; report per-slice results without business-effect claims |
| Ops availability to review/demo | Prepare during M6; request demo for final M7 week | Continue on labelled unreviewed corpus; record unavailable reviews |
| Scenario reference second reviewer | Before accepting a scenario into evaluated data | Keep it as a draft or use reproduced ground truth |
| Model license/approval and offline artifacts | Before using each candidate; formal company approval before M12 | Do not deploy unapproved weights or silently substitute hosted APIs |
| Server hardware, services, access and owners | After M11 measurement, as company deployment becomes concrete | Continue local/synthetic work; company M12 track remains pending |
| Azure budget, subscription/identity, region/service compatibility | Before M12 provisioning | Prepare IaC and tests without incurring resources |
| Latency/quality bar cannot be met | M6/M7 experiments, M9/M12 confirmation | Narrow scope, improve evidence/retrieval, or abstain; do not weaken residency/safety to pass |
| Corpus/cases lag behind prerequisites | Review every milestone | Allocate authoring time before dependent implementation |

The older root reference documents mentioned in some roadmaps are currently absent. This plan is grounded in the available `roadmap/` sources and does not restore or depend on those deleted files.

## Progress tracking

Initial state: planning document saved; application implementation is not started. The earlier trial implementation was rolled back, so it contributes no completed milestone or validated baseline.

| Milestone | Implementation | Verification | Learning review | ADR and evidence | Overall |
|---|---|---|---|---|---|
| M1 | Not started | Not started | Pending | Pending | Not started |
| M2 | Not started | Not started | Pending | Pending | Not started |
| M3 | Not started | Not started | Pending | Pending | Not started |
| M4 | Not started | Not started | Pending | Pending | Not started |
| M5 | Not started | Not started | Pending | Pending | Not started |
| M6 | Not started | Not started | Pending | Pending | Not started |
| M7 | Not started | Not started | Pending | Pending | Not started |
| M8 | Not started | Not started | Pending | Pending | Not started |
| M9 | Not started | Not started | Pending | Pending | Not started |
| M10 | Not started | Not started | Pending | Pending | Not started |
| M11 | Not started | Not started | Pending | Pending | Not started |
| M12 | Not started | Not started | Pending | Pending | Not started |

Update the current task and progress table after a reviewed work package. Use `Not started`, `In progress`, `Awaiting review`, `Blocked`, or `Complete`, with the blocking dependency or evidence link recorded. A milestone is complete only after its source checklist and all four evidence areas clear.

Current task: **M1.1 Package skeleton and development tooling**. Next checkpoint: editable package, minimal dependency setup, clean lint/type/test configuration, and your explanation of dependency direction.

For each milestone, create a working note in `architecture/notes/` when implementation begins:

```text
Milestone and task:
Status and date:
Behavior being implemented:
Source roadmap sections:
Implementation or commit links:
Checks run and results:
Exercises and measured values:
Failure reproduced, diagnosis, and resolution:
ADR and alternatives:
Learning explanation reviewed:
Open gaps or external dependencies:
Next concrete task:
```

Keep summaries factual. Do not check a box because a script exists; record that it ran and what it demonstrated. Do not label a core authored by Codex as code you wrote yourself.

## Project completion

The project is complete when all twelve milestone gates have evidence and you can demonstrate and explain:

1. The incident-to-evidence-to-diagnosis path and authorization boundaries.
2. Why the layers, storage, inference, retrieval, queue, and deployment choices were made.
3. How the system responds to malformed input, overload, outages, unsafe output, and uncertainty.
4. Which tests protect contracts, privacy, tenant isolation, transactions, and distributed behavior.
5. How authored-case quality, retrieval, judge agreement, abstention, and degradation are measured.
6. How one artifact is deployed, promoted, rolled back, and restored.
7. How downstream database/GPU limits constrain scaling and admission.
8. How an operator diagnoses failures using telemetry and runbooks without exposing raw content.
9. Which improvement comes next, supported by a measurement.

Final evidence includes a reproducible package/stack, API contract, ADR-001–012, threat model, human-authored corpus, provenance-labelled evaluation data and reports, capacity/load measurements, dashboards/alerts, operational runbooks, hardened image digests, target IaC, identity/egress tests, and timed restore/rollback drills.

Use the term **company production candidate** until real users and operational ownership exist. Use **synthetic Azure portability target** for the cloud environment. Measuring a live improvement in first-level resolution, handle time, or escalation quality requires a separately approved in-company pilot; it is not implied by authored-case evaluation results.
