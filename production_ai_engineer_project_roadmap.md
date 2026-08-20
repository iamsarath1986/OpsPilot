# Production AI Engineer Roadmap: Build and Operate OpsPilot

## Purpose

This roadmap synthesizes the three supplied documents and the target job description into one project-driven path. Their shared message is consistent: strong Python and backend engineering come first; an LLM call is only one unreliable dependency inside a larger system; and a credible AI engineer must be able to test, deploy, observe, secure, and explain the whole product.

The roadmap is ordered by dependency, not fashion. Complete milestones by demonstrating the exit criteria, not by waiting for a date or merely making a feature work. At roughly 10–15 focused hours per week, the path will commonly take 9–12 months, but evidence of mastery is the real schedule.

## The One Project: OpsPilot

**OpsPilot is an evidence-grounded incident and support copilot for hosting operations teams.** A support engineer creates an incident or uploads sanitized server logs. OpsPilot extracts signals, retrieves relevant runbooks and support policies, proposes a structured diagnosis with citations, recommends safe next steps, and escalates when evidence is insufficient.

The initial product is deliberately read-only. It does not execute shell commands or modify customer infrastructure. Near the end, it gains a small set of authorized read-only tools such as fetching an incident, checking a synthetic service status, or searching a runbook. This demonstrates agent engineering without turning an immature model into an unsafe administrator.

### Real-world users and value

- Tier-1 support engineers get a consistent first diagnosis and cited runbook steps.
- Senior engineers receive better incident summaries and structured escalation packets.
- Support managers can measure answer quality, abstention, latency, cost, and common incident types.
- The organization retains deterministic business controls even when the model is unavailable or wrong.

### Core user journeys

1. A user authenticates and creates an incident.
2. The user uploads a bounded, validated log file or pastes an error excerpt.
3. A background worker redacts likely secrets, parses deterministic signals, and stores an analysis job.
4. The user requests a diagnosis.
5. OpsPilot validates the request, retrieves tenant-authorized runbook passages, constructs a versioned prompt, invokes an LLM, validates structured output, applies policy checks, and returns an evidence-linked response or abstains.
6. The user rates the response or escalates it; feedback becomes curated evaluation data only after review.

### Non-goals for the portfolio release

- No autonomous production changes, unrestricted web browsing, or arbitrary code execution.
- No microservice split merely to claim microservices experience.
- No fine-tuning until prompt/RAG/evaluation evidence shows a specific gap that fine-tuning could solve.
- No dedicated vector database until PostgreSQL/`pgvector` is measured to be insufficient.
- No multi-agent system unless a single bounded workflow is demonstrably unable to meet a measured requirement.

## How to Use the Roadmap

For every topic, use this depth ladder:

| Depth | Evidence of mastery |
|---|---|
| **Understand** | Explain the mechanism, vocabulary, and problem it solves without a tutorial. |
| **Implement** | Build a small version yourself and explain each important line or configuration. |
| **Debug** | Reproduce at least one realistic failure, locate it with evidence, and fix or mitigate it. |
| **Design** | Compare alternatives and justify a choice from requirements and constraints. |
| **Productionize** | Add security, tests, observability, operational procedures, and a rollback/fallback path. |

Use a repeating weekly loop: learn one concept, run a small experiment, implement it in OpsPilot, test the happy and unhappy paths, document the decision, and demo it. Maintain an `architecture/decisions/` directory of short Architecture Decision Records (ADRs): context, options, decision, consequences, and evidence that would cause reconsideration.

## Architecture Evolution at a Glance

| Milestone | Architectural state |
|---|---|
| 1 | In-memory Python domain model and CLI |
| 2 | Typed package with tested sync/async boundaries |
| 3 | Stateless FastAPI service with a stable HTTP contract |
| 4 | PostgreSQL persistence, transactions, migrations |
| 5 | Multi-tenant identity, authorization, secure uploads |
| 6 | First provider-neutral LLM diagnosis vertical slice |
| 7 | Reliable AI orchestration with schemas, retries, fallbacks, and prompt versions |
| 8 | Full RAG ingestion and cited retrieval using `pgvector` |
| 9 | Evaluation, guardrails, regression gates, and human escalation |
| 10 | Durable background work, cache, bounded tools, and distributed-system controls |
| 11 | Observable, containerized system with CI/CD-quality build artifacts |
| 12 | Azure-hosted, infrastructure-as-code, scalable and operable production candidate |

### Suggested effort by gate

| Milestone | Typical focused weeks | Do not leave until |
|---|---:|---|
| 1–2 | 6–8 total | You can build, type, test, profile, and debug Python without tutorial dependence |
| 3 | 3–4 | The API contract and unhappy paths are explainable and tested |
| 4–5 | 8–10 total | Persistence, transactions, identity, tenant isolation, and secure input survive failure tests |
| 6–7 | 7–8 total | The AI baseline has explicit contracts, telemetry, validation, resilience, and degraded modes |
| 8–9 | 9–11 total | RAG beats a baseline on defined metrics and regression/safety gates work |
| 10–11 | 9–10 total | Async work is idempotent and the whole system is observable, containerized, and release-gated |
| 12 | 6–8 | Azure deploy/scale/failure/restore/rollback evidence is documented |

These are planning ranges, not permission to advance. If a checklist takes longer, stay with it; if prior experience lets you prove it faster, proceed.

---

# Phase I — Software and Backend Engineering Foundations

## Milestone 1 — Python Foundations and the Incident Domain

### 1. Topics to Learn

- Python values, control flow, functions, collections, comprehensions, iterators, generators, exceptions, file/context managers, modules, and packaging. These are the raw materials for expressing the product cleanly.
- Object-oriented design: encapsulation, composition over inheritance, dataclasses, protocols, and separation of domain logic from I/O. The problem solved is change isolation, not “using classes.”
- Type hints and a static checker. Types make contracts visible and catch integration mistakes before runtime; they do not replace tests or validation of external data.
- Git fundamentals: small commits, feature branches, pull-request-style self-review, `.gitignore`, and meaningful history. The repository is part of the portfolio evidence.
- Domain modeling: incident, log artifact, analysis, recommendation, evidence, tenant, and user. Good models make later API and database design easier.

### 2. Learning Depth

- **Python core — Implement/Debug:** solve small transformations without copying, read tracebacks, and debug mutable-state and boundary errors.
- **OOP and modular design — Design:** explain why composition and dependency injection make testing easier than deep inheritance.
- **Types — Implement:** type all public functions and make the checker pass; explain where runtime validation is still required.
- **Git — Implement:** recover from an ordinary merge conflict and explain a commit before publishing it.

### 3. Hands-On Learning Exercises

1. Parse three sample log formats with pure functions; compare a list-returning parser with a generator on a large generated file.
2. Model an `Incident` first as a dictionary and then as a dataclass. Write down which invalid states each permits.
3. Inject a clock and ID generator into a service instead of calling globals; test deterministic timestamps and IDs.
4. Trigger and diagnose `KeyError`, invalid encoding, malformed timestamps, and a partial file read.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Domain model | Ad hoc dictionaries drift and permit invalid states | Define typed incident, artifact, finding, evidence, and analysis entities | Unit tests for invariants and transitions |
| Layering | File/CLI concerns contaminate business rules | Create `domain`, `application`, and `adapters` packages | Test domain without filesystem or console |
| Error taxonomy | Callers cannot react to generic failures | Define domain errors such as `InvalidIncidentState` and `UnsupportedLogFormat` | Assert error type and message, not blanket `except` |
| Repository interface | Storage will change later | Define an in-memory incident repository behind a protocol | Contract tests that can later run against PostgreSQL |

Implement a CLI that creates an incident, attaches a local sample log, runs deterministic parsing rules, and displays a summary. Keep sample logs synthetic and free of real credentials.

### 5. Expected Project State

OpsPilot is an installable, typed Python package with a CLI and in-memory state. Compared with no application, it now has stable domain language and boundaries, but no network, persistence, authentication, or AI.

### 6. Testing and Validation

- Use `pytest` for domain and parser unit tests, parametrized across log formats and malformed inputs.
- Add coverage reporting as a diagnostic, not a target that substitutes for meaningful assertions.
- Run a type checker and formatter/linter locally.
- Verify deterministic results on fixed inputs and property-like invariants such as “severity must be an allowed value.”

### 7. Failure Scenarios

- A log uses an unknown encoding or contains a malformed line.
- A 500 MB input would exhaust memory if fully loaded.
- Two findings share an identifier.
- An incident transitions from closed back to analyzing without an explicit reopen operation.

### 8. Skills Gained

You can write maintainable Python, model a real operational domain, use Git, and test deterministic logic. This directly supports the job’s Python requirement and begins the collaborative engineering practices implied by working with experienced engineers and product teams.

### 9. Milestone Completion Checklist

- [ ] I can explain iterators, generators, context managers, composition, and dependency inversion.
- [ ] I implemented the domain model and CLI myself with public functions fully typed.
- [ ] I can replace the in-memory repository without changing domain logic.
- [ ] Tests cover valid, invalid, boundary, and state-transition cases.
- [ ] I intentionally broke parsing and diagnosed the traceback.
- [ ] ADR-001 explains the modular-monolith choice and when it should be revisited.

## Milestone 2 — Production Python, Testing, Async, and Concurrency

### 1. Topics to Learn

- `asyncio`, coroutines, tasks, cancellation, timeouts, and structured concurrency. Async helps while waiting on network I/O; it does not make CPU-heavy parsing faster.
- Threads versus processes versus async: the GIL, I/O-bound versus CPU-bound work, shared-state hazards, and backpressure.
- Testing layers, fixtures, fakes, mocks at system boundaries, contract tests, and test determinism.
- Configuration, dependency injection, resource lifecycle, logging basics, and packaging/locking dependencies.
- Profiling CPU and memory before optimizing. Performance work should start from measurement.

### 2. Learning Depth

- **Async/concurrency — Implement/Debug:** run concurrent I/O, impose deadlines, propagate cancellation, and avoid blocking the event loop.
- **Testing — Design:** choose unit versus integration tests based on the risk and boundary being checked.
- **Resource lifecycle — Implement:** reliably open/close files and clients even on exceptions.
- **Profiling — Understand/Implement:** use measurements to identify a real bottleneck.

### 3. Hands-On Learning Exercises

1. Simulate ten 300 ms network calls synchronously and with bounded async concurrency; measure wall time and explain the result.
2. Put a blocking `time.sleep` inside async code, observe stalled tasks, then move blocking work appropriately.
3. Cancel a task halfway through and verify cleanup executes.
4. Compare thread and process pools on a synthetic I/O task and a CPU-heavy parsing task.
5. Write one brittle over-mocked test, replace it with a fake adapter, and explain why the second checks behavior more safely.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Async ports | Future database and AI calls wait on I/O | Make application repository/provider interfaces async while keeping pure domain logic sync | Async unit tests with deterministic fakes |
| Bounded concurrency | Unlimited tasks exhaust sockets/memory | Add a semaphore-controlled artifact parser experiment | Stress test bounds and cancellation |
| Configuration | Hard-coded settings differ by environment | Add typed settings loaded from environment with safe defaults | Tests for missing/invalid values; secrets never logged |
| Test architecture | Slow/flaky suites stop providing confidence | Separate unit and future integration markers | Run unit suite repeatedly with identical result |
| Basic CI | Local checks are easy to forget and differ by developer | Add a small GitHub Actions workflow for lint, types, and unit tests | Push a deliberately failing change on a branch and confirm it is blocked |

### 5. Expected Project State

The same CLI behavior now sits behind explicit async I/O contracts, dependable configuration, and basic automated checks. The architecture has not become distributed; it has become ready to integrate network resources without spreading concurrency concerns through the domain. Milestone 11 will expand CI into full integration, security, evaluation, image, and deployment gates.

### 6. Testing and Validation

- Unit-test cancellation, timeout translation, and cleanup.
- Run a simple memory profile against a large generated log and retain the before/after evidence.
- Add mutation-style checks manually: remove an important validation branch and confirm a test fails.
- Ensure no test depends on real time, network, or a developer machine’s environment.

### 7. Failure Scenarios

- A coroutine is created but never awaited.
- Cancellation leaves a partially written artifact.
- An unbounded `gather` spikes memory.
- A blocking parser freezes the event loop.
- Tests pass alone but fail in a different order due to shared state.

### 8. Skills Gained

You can reason about production Python behavior, concurrency, resource management, and test design—skills needed for API backends that coordinate databases and AI providers.

### 9. Milestone Completion Checklist

- [ ] I can choose among async, threads, and processes for a stated workload and defend the choice.
- [ ] I can demonstrate event-loop blocking and fix it.
- [ ] Timeout and cancellation paths have tests.
- [ ] Configuration is typed and environment-specific; no secret is committed.
- [ ] A profile, not intuition, supports at least one optimization.
- [ ] I understand why asynchronous code is not automatically faster.

## Milestone 3 — HTTP, REST, FastAPI, and API Contracts

### 1. Topics to Learn

- HTTP methods, status codes, headers, content types, caching semantics, connection lifecycle, TLS, and request/response anatomy.
- REST resource modeling, idempotency, pagination, filtering, versioning, consistent errors, and OpenAPI.
- FastAPI routing and dependency injection; Pydantic request/response schemas and boundary validation.
- Sync versus async endpoints and server concurrency. An async endpoint that calls blocking code still blocks.
- API testing, client timeouts, CORS, request IDs, and health/readiness concepts.

### 2. Learning Depth

- **HTTP/REST — Design:** explain semantics and select methods/statuses from the contract rather than habit.
- **FastAPI/Pydantic — Implement/Debug:** build validation, dependencies, errors, and async endpoints; debug a `422`, `500`, and timeout.
- **API evolution — Understand:** distinguish additive changes from breaking changes and document compatibility.

### 3. Hands-On Learning Exercises

1. Inspect requests with `curl -v`; vary `Accept`, `Content-Type`, authorization, and timeouts.
2. Design `POST /incidents`, `GET /incidents/{id}`, and `POST /incidents/{id}/analyses`; explain why analysis is a subresource rather than a query flag.
3. Make Pydantic reject oversized strings, unknown enums, missing fields, and extra fields.
4. Send duplicate POST requests and compare non-idempotent behavior with an idempotency-key experiment.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Resource API | Clients need a stable product contract | Expose incident create/list/get/update and deterministic analysis endpoints under `/v1` | API tests assert status, body, headers, and schema |
| Boundary models | Untrusted inputs can corrupt domain state | Separate Pydantic API DTOs from domain entities | Invalid, extra, oversized, and boundary-value cases |
| Error contract | Random error shapes make clients fragile | Use a problem-details-style response with code, message, request ID, and safe details | Assert domain-to-HTTP mapping; no stack traces leaked |
| Pagination | Unbounded lists become slow and expensive | Add cursor or simple limit/offset initially, with a documented limit | Empty, first, middle, last, and invalid pages |
| Health | Platforms need lifecycle signals | Add liveness and readiness endpoints with different semantics | Readiness fails when a required fake dependency is unavailable |

### 5. Expected Project State

OpsPilot changes from a local CLI to a stateless network service. State is still in memory, so restarts lose data and multiple replicas would disagree. The API contract and application boundaries are now testable independently.

### 6. Testing and Validation

- Use FastAPI’s test client or an async HTTP client for endpoint tests.
- Validate the generated OpenAPI document and keep representative request/response examples.
- Add contract tests for repository adapters and API error mapping.
- Measure baseline latency for deterministic endpoints and record hardware/context rather than presenting it as a universal benchmark.

### 7. Failure Scenarios

- Wrong content type, malformed JSON, an oversized payload, or unknown enum.
- Client disconnects or times out mid-request.
- An unhandled domain exception becomes an unsafe `500` response.
- Duplicate creation produces duplicate incidents.
- The service is alive but its required storage dependency is not ready.

### 8. Skills Gained

You can design and test a Python REST API, validate untrusted data, and reason about HTTP and async behavior. This is direct evidence for the job’s APIs and backend-development requirement.

### 9. Milestone Completion Checklist

- [ ] I can explain safe/idempotent HTTP methods and common status-code choices.
- [ ] OpenAPI describes every public endpoint and error response.
- [ ] Invalid input cannot reach domain logic.
- [ ] Endpoint tests include success, authorization placeholder, invalid input, not found, conflict, and dependency failure.
- [ ] I diagnosed a blocking async endpoint under concurrent requests.
- [ ] ADR-002 justifies REST and the initial versioning strategy.

## Milestone 4 — SQL, PostgreSQL, Transactions, and Data Access

### 1. Topics to Learn

- Relational modeling, keys, constraints, normalization/denormalization, joins, SQL query plans, indexes, and transactions.
- Isolation levels, locking, deadlocks, optimistic concurrency, connection pooling, and the N+1 problem.
- PostgreSQL operation basics: roles, backups, restore, extensions, and `EXPLAIN ANALYZE`.
- SQLAlchemy mapping/session boundaries and Alembic forward/backward migrations.
- Repository and unit-of-work patterns: useful when they preserve business boundaries, harmful when they merely hide SQL.
- Docker and Docker Compose fundamentals for a reproducible local PostgreSQL instance: image versus container, port, volume, network, and health check. Application-image hardening comes in Milestone 11.

### 2. Learning Depth

- **SQL/schema design — Design/Debug:** write joins and constraints, read query plans, and identify an inappropriate index.
- **Transactions — Implement/Debug:** make multi-write operations atomic and reproduce a concurrency conflict.
- **SQLAlchemy/Alembic — Implement:** map entities without leaking sessions into route code and migrate an existing database safely.
- **Database operations — Understand:** perform and verify backup/restore in development.

### 3. Hands-On Learning Exercises

1. Draw the schema and cardinalities before using an ORM; implement several queries in raw SQL first.
2. Populate 100,000 synthetic incidents, compare an unindexed and indexed query plan, and explain write/storage trade-offs.
3. Reproduce a lost update with two sessions, then add version-based optimistic concurrency.
4. Create a migration that fails halfway in a disposable database and practice recovery.
5. Backup, delete a disposable database, restore it, and verify row counts/checksums.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Relational schema | In-memory data disappears and cannot enforce relationships | Add users/tenants placeholders, incidents, artifacts, findings, analyses, evidence, and feedback tables | Constraint and repository integration tests |
| Transaction boundary | Incident and artifact metadata must commit together | Use one unit of work per application command | Force second write failure and assert rollback |
| Migrations | Teams need repeatable schema evolution | Establish Alembic baseline and upgrade/downgrade policy | Apply migrations from empty and previous schema in CI |
| Indexes | List/search queries degrade with volume | Index measured filter/sort paths such as tenant/status/created time | Compare query plans and latency with fixture volume |
| Concurrency control | Two operators can overwrite incident state | Add a version column or conditional update | Concurrent update test yields a conflict, not lost data |
| Local data environment | Team machines need the same database behavior | Add Docker Compose for PostgreSQL with health check and disposable test configuration | Start from a clean machine, preserve intended dev data, recreate test data |

### 5. Expected Project State

OpsPilot is now a durable API backed by PostgreSQL. Compared with in-memory storage, multiple API instances could share consistent state, though identity, secure tenant isolation, AI, and production operations are not complete.

### 6. Testing and Validation

- Run integration tests against real PostgreSQL, not SQLite, because SQL and transaction behavior differ.
- Roll back each test or use isolated schemas/databases while still testing real commits where needed.
- Verify foreign keys, uniqueness, check constraints, transaction rollback, pagination, and concurrency.
- Test every migration on a snapshot of the previous schema and rehearse restoration.

### 7. Failure Scenarios

- Database unavailable at startup or during a request.
- Pool exhaustion under concurrency.
- Deadlock or serialization conflict.
- A migration holds a table lock too long.
- N+1 queries make list endpoints slow.
- Backup exists but restore procedure is broken.

### 8. Skills Gained

You can design and operate the persistence layer of a real backend, use an ORM without losing SQL understanding, and reason about consistency and performance. This strengthens the backend requirement and prepares for a managed Azure database.

### 9. Milestone Completion Checklist

- [ ] I can draw the schema, explain every constraint, and write the important queries in SQL.
- [ ] Repository contract tests pass for both in-memory and PostgreSQL adapters.
- [ ] Transaction rollback and a concurrent update conflict are tested.
- [ ] I can read the important query plans and justify each nontrivial index.
- [ ] Migrations work from empty and previous schema; restore has been rehearsed.
- [ ] ADR-003 records PostgreSQL over NoSQL and its reconsideration conditions.

## Milestone 5 — Authentication, Authorization, Multi-Tenancy, and Secure Inputs

### 1. Topics to Learn

- Authentication versus authorization; sessions, OAuth 2.0 concepts, OpenID Connect, JWT validation, key rotation, issuer/audience/expiry, and why decoding is not verification.
- Role-based access control (RBAC), object-level authorization, least privilege, and multi-tenant isolation.
- Password hashing only for a local learning identity provider; production identity should use a maintained provider such as Microsoft Entra ID rather than custom auth.
- Secure uploads: byte and file-count limits, MIME/content mismatch, archive bombs, malicious filenames, path traversal, malware scanning concepts, redaction, and retention.
- Threat modeling, OWASP API risks, secret handling, audit logs, privacy, and data minimization. Hosting logs commonly contain tokens, IPs, usernames, or customer content.

### 2. Learning Depth

- **Identity protocols — Understand/Debug:** trace a token’s creation and validation and diagnose expired, wrong-audience, wrong-issuer, and rotated-key cases.
- **Authorization/tenant isolation — Design/Productionize:** enforce access in application/repository boundaries and test cross-tenant denial.
- **File security — Implement/Debug:** stream bounded uploads, sanitize names, hash content, redact secrets, and reject unsafe inputs.
- **Threat modeling — Design:** identify assets, trust boundaries, attackers, abuse cases, and mitigations.

### 3. Hands-On Learning Exercises

1. Validate locally issued learning JWTs, then tamper with payload/signature and change issuer, audience, clock, and key.
2. Create two tenants and attempt every endpoint with the other tenant’s object ID.
3. Test `../../secret`, double extensions, fake MIME types, huge streams, compressed bombs in a safe synthetic fixture, and duplicate uploads.
4. Run a secret-redaction corpus containing API-key-like values and measure false positives/negatives.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Identity dependency | Anonymous callers must not access customer incidents | Add token validation dependency and a development identity adapter | Missing, invalid, expired, wrong issuer/audience/key cases |
| Object authorization | Valid users may still access the wrong tenant | Pass actor/tenant context into every use case and tenant-scope every query | Cross-tenant matrix for read/write/upload/analyze |
| RBAC | Support agents and admins have different powers | Define small explicit roles and permission checks | Deny-by-default tests for every protected operation |
| Safe artifact ingestion | Logs are untrusted and sensitive | Stream to controlled storage, enforce limits, generate server-side names, hash, redact, and retain metadata | Traversal, size, type, duplication, redaction, cleanup |
| Audit trail | Security events need accountability | Record actor, action, object, decision, timestamp, and request ID without sensitive payloads | Assert allow/deny events and tamper-resistant append behavior conceptually |

### 5. Expected Project State

The durable API is now multi-user and tenant-aware. Security is an architectural boundary rather than route decoration. It can safely accept bounded synthetic artifacts, but its final production identity integration and cloud blob storage arrive later.

### 6. Testing and Validation

- Build an authorization decision table and automated matrix; “authenticated” must never imply “authorized.”
- Use dependency integration tests for token validation and repository tests for tenant predicates.
- Run static dependency/secret scanning and verify logs never include tokens or raw sensitive artifacts.
- Complete a lightweight STRIDE-style threat model and keep mitigations linked to tests or backlog items.

### 7. Failure Scenarios

- Token expiry/clock skew, signature-key rotation, or identity provider outage.
- Insecure direct object reference exposes another tenant’s incident.
- Filename traversal or MIME spoofing.
- Logs contain credentials that would be sent to an LLM.
- An admin permission is accidentally granted to a support-agent role.

### 8. Skills Gained

You can secure a multi-user backend, reason about cloud identity, protect sensitive inputs, and collaborate using a threat model. These are critical real-world skills not stated explicitly in the short job description but expected of an engineer building cloud AI applications.

### 9. Milestone Completion Checklist

- [ ] I can explain OAuth/OIDC/JWT roles without claiming JWT itself is authentication.
- [ ] Every protected object access is tenant-scoped and has a cross-tenant denial test.
- [ ] Invalid tokens fail closed; logs contain no token or sensitive payload.
- [ ] Uploads are streamed, bounded, server-named, hashed, and redacted before AI use.
- [ ] I intentionally reproduced an object-level authorization bug and demonstrated the fix.
- [ ] The threat model identifies assets, boundaries, abuse cases, and residual risks.

### Phase I Job-Readiness Evidence

| Job requirement | Evidence after Milestone 5 |
|---|---|
| Python | Typed modular package, async boundaries, profiling, robust error handling, comprehensive tests |
| APIs and backend development | Versioned FastAPI REST API, validation, auth/RBAC, PostgreSQL, migrations, secure uploads |
| Real-world application development | Multi-tenant incident workflow shaped by operational users and failure cases |
| Collaboration | Git history, ADRs, API contract, threat model, and reviewable tests |
| Cloud preparation | Stateless service, externalized configuration, readiness checks, identity concepts |

---

# Phase II — AI Application and Reliability Engineering

## Milestone 6 — LLM Fundamentals and the First AI Vertical Slice

### 1. Topics to Learn

- Tokens, context windows, sampling/temperature, model/provider differences, latency, usage-based cost, rate limits, and non-determinism.
- Prompt roles, instructions, few-shot examples, delimiters, context management, and prompt injection as an untrusted-data problem.
- LLM API lifecycle: credentials, client reuse, timeouts, response metadata, token usage, provider errors, and data-handling policy.
- Structured output concepts and function/tool calling at a conceptual level; reliability layers arrive in the next milestone.
- Provider abstraction: isolate product semantics from a specific SDK without reducing every provider to a lowest-common-denominator interface.

### 2. Learning Depth

- **LLM behavior — Understand/Implement:** predict how context, sampling, and instructions affect results, while recognizing non-determinism.
- **Provider API — Implement/Debug:** make a server-side call, classify errors, record usage/latency, and avoid exposing credentials.
- **Prompting — Implement:** produce an initial prompt from explicit product requirements, not vibes.
- **Provider boundary — Design:** explain direct SDK use versus an adapter and what capabilities the adapter may leak.

### 3. Hands-On Learning Exercises

1. Run the same 20 incident prompts repeatedly at several temperatures; measure output stability rather than judging one example.
2. Vary irrelevant context, conflicting log text, and context length; observe instruction-following and cost/latency.
3. Calculate expected per-request and monthly cost from measured input/output tokens and traffic assumptions.
4. Stub 429, timeout, authentication, malformed response, and content-refusal cases.
5. Put an instruction such as “ignore prior rules” inside a synthetic log and document why log content is data, not authority.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Provider port | SDK details spread through business logic | Define `DiagnosisProvider` request/result types and one provider adapter | Contract tests with recorded synthetic responses/fakes |
| Prompt builder | String concatenation becomes unreviewable | Build prompts from system policy, task, bounded sanitized signals, and version metadata | Snapshot/semantic assertions on required sections and limits |
| First diagnosis | Deterministic rules cannot explain complex patterns | Add `POST /incidents/{id}/diagnoses` using sanitized log excerpt only | End-to-end test with fake provider; manual exploratory set with real provider |
| Usage ledger | Cost and latency are invisible | Store model/deployment, prompt version, tokens, latency, outcome, and correlation ID | Assert every call records success or classified failure |

Keep the response intentionally simple but label it **experimental**. Do not add RAG yet; first establish an understandable baseline against which retrieval can later be measured.

### 5. Expected Project State

OpsPilot can produce its first AI-assisted diagnosis end to end. The LLM is behind an interface and observable at a basic level, but outputs are not yet safe enough to be treated as dependable product data.

### 6. Testing and Validation

- Unit-test prompt construction and provider error mapping with fakes.
- Run an initial frozen exploratory dataset of at least 30 representative, boundary, and adversarial incidents.
- Record output, latency, tokens, and human notes; this becomes the baseline, not a claim of production accuracy.
- Never make CI depend on a live paid provider for normal deterministic tests; keep opt-in smoke tests separate.

### 7. Failure Scenarios

- LLM timeout, 429 rate limit, provider authentication failure, or refusal.
- Prompt exceeds the context budget.
- Model follows an instruction embedded in a log.
- Response contradicts deterministic parsed signals.
- Token use or latency is unexpectedly high.

### 8. Skills Gained

You can integrate an LLM into a backend responsibly, measure its basic behavior, and keep provider concerns isolated. This is the first direct evidence for “AI applications,” while preserving the backend practices the role also expects.

### 9. Milestone Completion Checklist

- [ ] I can explain tokens, context, sampling, rate limits, latency, and cost trade-offs.
- [ ] The provider credential exists only server-side and no sensitive log reaches the provider unredacted.
- [ ] Provider errors become explicit application outcomes, not generic `500`s.
- [ ] Each call records prompt/model version, tokens, latency, and outcome.
- [ ] I ran the same frozen dataset multiple times and can describe non-determinism observed.
- [ ] ADR-004 justifies the provider boundary and initial provider choice.

## Milestone 7 — Reliable AI Orchestration and Structured Outputs

### 1. Topics to Learn

- JSON Schema and Pydantic structured output: syntax validity, schema validity, semantic/business validity, and evidence validity are separate layers.
- Prompt templates as versioned product artifacts; model/deployment versions and configuration form part of the release.
- Deadlines, transient versus permanent errors, bounded retry with exponential backoff and jitter, `Retry-After`, circuit breakers, and retry budgets.
- Fallbacks: same request retry, smaller/alternate model, deterministic partial result, cached result with freshness label, abstention, and human escalation.
- Rate limiting, quotas, concurrency limits, idempotency, and safe output handling.
- Reliability targets: availability, latency, cost, schema-pass rate, and successful/abstained outcomes.

### 2. Learning Depth

- **Structured output — Productionize:** enforce provider-side schema where available and still validate application/business/evidence rules.
- **Retries/timeouts/fallbacks — Design/Debug:** retry only safe transient failures, prevent retry storms, and expose degraded modes honestly.
- **Prompt/version control — Implement:** reproduce which prompt, model, knowledge, and configuration produced an answer.
- **Reliability — Design:** define a service-level indicator (SLI), target, and error budget for important paths.

### 3. Hands-On Learning Exercises

1. Force invalid JSON, missing fields, invalid enums, impossible confidence values, and a syntactically valid but unsupported claim.
2. Simulate `429`, `500`, slow responses, connection reset, and permanent `400`; inspect which cases retry.
3. Add random jitter to many simulated clients and compare retry synchronization with fixed delays.
4. Trip a circuit breaker and demonstrate half-open recovery.
5. Compare primary, alternate-model, deterministic-only, and abstention fallbacks on the baseline dataset; measure quality/cost/latency.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Structured diagnosis | Free text cannot safely drive UI/workflows | Define severity, summary, hypotheses, evidence IDs, next steps, confidence band, and escalation fields | Invalid syntax/schema/semantics/evidence-reference cases |
| Validation pipeline | Valid JSON can still be wrong | Validate schema, business rules, authorized evidence IDs, forbidden secrets, and uncertainty policy | One focused test per layer plus combined integration tests |
| Resilience policy | Provider faults propagate to users | Centralize overall deadline, per-attempt timeout, bounded retry+jitter, rate-limit handling, and circuit breaker | Deterministic fault injection and attempt-count assertions |
| Admission control | One tenant or traffic burst can exhaust shared provider capacity | Apply per-actor/tenant request and token budgets plus a global concurrency bulkhead; begin in-process, then use a distributed limiter when replicas are added | Boundary, burst, fairness, `Retry-After`, limiter-outage, and multi-replica tests |
| Fallback state machine | Silent model switching hides quality changes | Return explicit `complete`, `degraded`, `abstained`, or `escalated` outcome with reason | Exhaust every transition and prevent loops |
| Prompt registry | Untracked prompt edits create invisible regressions | Store immutable prompt IDs/versions and changelog; persist version with diagnosis | Reproduce output configuration from stored metadata |

Important decision: a provider’s constrained structured output solves format adherence, not factual correctness. Pydantic solves parsing and some business constraints, not grounding. Keep those failure classes separate in code and metrics.

An alternate provider is not a free resilience switch: its schema capabilities, safety behavior, data region/retention, authorization, cost, and evaluation baseline can differ. Send data to a second provider only when policy permits it and that exact fallback is evaluated; otherwise prefer a deterministic degraded result or human escalation.

### 5. Expected Project State

The experimental LLM call becomes an AI orchestration component with explicit contracts and degraded behavior. Compared with Milestone 6, model output is no longer returned directly to the user; it must traverse validation and policy layers.

### 6. Testing and Validation

- Unit-test policies with a scripted fake provider that returns a sequence of faults/results.
- Integration-test provider serialization separately from product behavior.
- Add schema contract and prompt-version tests.
- Define initial targets from measured baseline, for example p95 latency and schema-validity target, and label them provisional. Do not choose impressive numbers without user needs and evidence.

### 7. Failure Scenarios

- Invalid JSON or schema-valid nonsense.
- Evidence IDs that were never supplied.
- Rate limiting across many concurrent users.
- Retry storm worsens an outage.
- Primary and fallback both fail.
- Circuit remains open after provider recovery.
- A stale cached answer is presented as fresh.

### 8. Skills Gained

You can build the reliability layers around an LLM: structured contracts, multi-layer validation, prompt versioning, deadlines, retries, circuit breaking, fallback, cost/latency recording, and honest degradation. These are high-value production AI skills beyond the terse job description.

### 9. Milestone Completion Checklist

- [ ] I can explain four validation layers and what each cannot prove.
- [ ] Retry policy distinguishes transient, throttling, permanent, and validation failures.
- [ ] Overall deadlines bound all attempts; tests prove no infinite retry/fallback loop.
- [ ] Every diagnosis is reproducible by prompt/model/configuration version.
- [ ] Degraded, abstained, and escalated outcomes are visible to clients and metrics.
- [ ] I intentionally failed primary and fallback paths and can narrate the state transition.

## Milestone 8 — RAG: Ingestion, Retrieval, Ranking, Grounding, and Citations

### 1. Topics to Learn

- Complete pipeline: documents → extraction → normalization → chunking → embeddings → vector storage → retrieval → metadata filtering → optional reranking → context construction → LLM → cited answer.
- Embeddings, vector distance/similarity, approximate versus exact nearest-neighbor search, lexical search, hybrid retrieval, top-K, and metadata filters.
- Chunk boundaries, size/overlap, tables/code, document versions, deduplication, access control, deletion, and re-indexing.
- PostgreSQL `pgvector` first; compare with Qdrant, Pinecone, and Azure AI Search. A dedicated service can add scale/hybrid/ranking features but also cost, network hops, and operational complexity.
- Retrieval evaluation: recall@K, precision@K, mean reciprocal rank (MRR), normalized discounted cumulative gain (nDCG), and failure categorization.

### 2. Learning Depth

- **Embeddings/vector search — Implement/Debug:** build an index and explain similarity, normalization, and why nearest does not mean relevant.
- **Chunking/retrieval — Design:** choose parameters through experiments on the project corpus.
- **RAG authorization — Productionize:** enforce tenant/document permissions before context enters the prompt.
- **Citations/grounding — Implement:** bind claims to immutable document/version/chunk evidence.

### 3. Hands-On Learning Exercises

1. Create 40–60 synthetic runbook questions with known relevant passages before tuning retrieval.
2. Compare several chunk sizes and overlaps; record index size, recall@K, noise, latency, and answer effects.
3. Compare lexical, vector, and hybrid retrieval on exact error codes, paraphrases, and ambiguous terms.
4. Sweep top-K and metadata filters; show that more context can reduce answer quality.
5. Remove the relevant document, insert a contradictory old version, and test expected abstention/version preference.
6. Compare embedding models only after controlling corpus, chunking, and evaluation queries.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Ingestion pipeline | Raw runbooks are not searchable or reproducible | Store document/version/checksum; extract, normalize, chunk, embed, and track job state | Golden extraction fixtures, idempotent re-ingestion, delete/reindex tests |
| `pgvector` index | Semantic queries need vector similarity | Add tenant-scoped chunks and embeddings to PostgreSQL | Known-neighbor and access-control integration tests |
| Hybrid retrieval | Error codes favor lexical; paraphrases favor vectors | Combine lexical and vector candidates, then fuse/rerank with a simple documented method | Offline retrieval benchmark by query category |
| Context builder | Too much or unauthorized context harms quality/security | Deduplicate, filter versions/tenant, enforce token budget, and label untrusted text | Token-limit, ordering, injection, and cross-tenant tests |
| Citation binding | Unsupported prose looks authoritative | Require cited chunk IDs and render document title/version/section links | Citation existence, entailment sample, and stale-version tests |

Create a retrieval trace containing query transformation, filters, candidate IDs/scores, selected context, and index version. Do not log sensitive raw content by default.

### 5. Expected Project State

OpsPilot now answers from tenant-authorized runbooks and cites exact source versions. Compared with the no-RAG baseline, it has an inspectable knowledge path and can abstain when retrieval does not support an answer. RAG has not “solved hallucination”; it has created evidence that can be evaluated.

### 6. Testing and Validation

- Separate retrieval evaluation from answer evaluation so a bad answer can be attributed correctly.
- Report recall@K/MRR overall and by query category; review misses manually.
- Compare no-RAG and RAG on the same frozen answer dataset for correctness, groundedness, citation precision/recall, latency, and cost.
- Add data lifecycle tests: update, deletion, duplicate, failed embedding, and unauthorized document.

### 7. Failure Scenarios

- Relevant chunk not retrieved; irrelevant chunk ranks first.
- Exact error code is lost by semantic-only retrieval.
- Old policy version conflicts with current policy.
- Cross-tenant chunk leaks into context.
- Embedding provider is unavailable during ingestion.
- Oversized context increases cost/latency or distracts the model.
- Retrieved text contains prompt injection.

### 8. Skills Gained

You can design, implement, debug, and evaluate a complete RAG system rather than calling a vector database blindly. This supplies strong AI-application evidence and demonstrates database, security, and systems thinking.

### 9. Milestone Completion Checklist

- [ ] I can draw and explain every stage of the ingestion and query pipelines.
- [ ] Chunking, embedding, top-K, filters, and retrieval type were selected from recorded experiments.
- [ ] Retrieval metrics are reported separately from generation metrics.
- [ ] Citations identify immutable document versions/chunks and cannot cross tenants.
- [ ] RAG beats the no-RAG baseline on defined quality measures without an unacceptable cost/latency regression.
- [ ] ADR-005 justifies `pgvector` and defines thresholds/features that would motivate a dedicated search service.

## Milestone 9 — AI Evaluation, Guardrails, Safety, and Human Escalation

### 1. Topics to Learn

- Evaluation design: task definition, representative datasets, train/dev/test separation concepts, slices, baselines, regression, confidence intervals, and human review.
- Deterministic graders, schema/business checks, retrieval metrics, semantic similarity limitations, rubric-based model judges, pairwise comparison, and judge bias/calibration.
- Answer metrics: task correctness, groundedness/faithfulness, citation precision/recall, completeness, actionability, abstention quality, and safety.
- Guardrails at input, retrieval, output, and tool/action boundaries; content safety, prompt injection, PII/secrets, policy enforcement, and business rules.
- Human-in-the-loop: escalation thresholds, evidence packet, reviewer action, auditability, feedback curation, and incident response.
- Red teaming and abuse-case testing. A guardrail reduces risk; it does not prove safety.

### 2. Learning Depth

- **Evaluation datasets/graders — Design/Productionize:** build representative frozen suites, document rubrics, calibrate automated judges with humans, and gate regressions.
- **Guardrails — Implement/Debug:** test false positives and false negatives, fail safely, and keep deterministic policy outside prompts.
- **Human escalation — Design:** define when, why, and with what evidence a case leaves automation.
- **Quality decisions — Productionize:** compare candidate to baseline by slices, not one aggregate score.

### 3. Hands-On Learning Exercises

1. Build at least 100 reviewed cases across common incidents, rare cases, no-answer cases, conflicting evidence, injection, sensitive data, and policy violations.
2. Have two humans independently score a subset with a written rubric; resolve disagreement and revise ambiguous criteria.
3. Compare exact checks, retrieval metrics, a rubric judge, and human ratings; measure judge-human agreement and inspect disagreements.
4. Blind model/prompt labels in pairwise judging and swap response order to detect position bias.
5. Generate adversarial inputs: hidden instructions in documents, citation laundering, plausible unsupported claims, obfuscated secrets, and requests for dangerous commands.
6. Tune an abstention threshold on a development set, then evaluate once on a held-out test set.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Versioned eval dataset | Anecdotes cannot show improvement | Store case ID, input, corpus version, expected evidence/behavior, tags, and rubric | Dataset schema, duplicates, leakage, and review-status checks |
| Evaluation runner | Prompt/RAG changes can silently regress | Run candidates against frozen configurations and produce per-slice comparison reports | Golden grader tests and reproducible seeded settings where supported |
| Layered guardrails | Model instructions alone cannot enforce policy | Add deterministic limits/redaction/auth; injection/context checks; output evidence/policy checks | Adversarial corpus with false-positive/negative reporting |
| Escalation workflow | Low-evidence cases need accountable handling | Create review queue with reason, sanitized input, retrieved evidence, model metadata, and reviewer decision | Low-confidence, conflicting, safety, outage, and user-requested escalation |
| Release gate | Teams need a ship/no-ship rule | Fail CI/nightly promotion on critical safety failures or statistically/practically meaningful slice regression | Deliberately introduce a bad prompt and confirm the gate blocks it |

Suggested metric set: retrieval recall@K and MRR; groundedness and citation accuracy; task correctness and actionability rubric; correct abstention on unanswerable cases; unsafe-output and secret-leak rate; p50/p95 latency; tokens/cost per successful diagnosis; provider error/fallback rate. Report sample size and uncertainty.

### 5. Expected Project State

OpsPilot is now measurable and governed. Compared with Milestone 8, changes to prompts, models, chunking, retrieval, or policies have a repeatable quality gate, and low-confidence or unsafe cases have an explicit human path.

### 6. Testing and Validation

- Run a fast deterministic/small eval subset on pull requests and the full suite on schedule or before release.
- Keep the held-out set restricted; avoid repeatedly tuning against it.
- Calibrate model judges against human labels and retain judge prompt/model version.
- Review regressions by slice and case, not only average score.
- Treat user thumbs-up/down as noisy product signals, not ground truth; curate before inclusion.

### 7. Failure Scenarios

- Hallucinated answer with real-looking but irrelevant citations.
- Guardrail blocks legitimate support content or misses obfuscated sensitive data.
- LLM judge prefers verbosity, response position, or its own model style.
- Evaluation dataset contains duplicates or does not represent production traffic.
- Prompt improves average correctness but destroys correct abstention for unanswerable cases.
- Human review backlog grows without an operational owner.

### 8. Skills Gained

You can answer “How do we know the AI works?” with a dataset, metrics, calibrated review, regressions, and operational decisions. Evaluation, guardrails, safety, and human escalation are among the most important production AI skills absent from many short job descriptions.

### 9. Milestone Completion Checklist

- [ ] The dataset has reviewed cases, explicit slices, expected evidence/behavior, and a held-out component.
- [ ] I can explain limitations and calibration of every automated grader.
- [ ] Retrieval, answer, safety, operational, and cost metrics are separated.
- [ ] A deliberately regressive prompt/model/retrieval change is blocked by the release gate.
- [ ] Guardrail false positives and false negatives are measured, not hidden.
- [ ] Every abstention/escalation includes a machine-readable reason and useful evidence packet.

### Phase II Job-Readiness Evidence

| Job requirement | Evidence after Milestone 9 |
|---|---|
| AI applications | End-to-end structured diagnosis and cited RAG assistant |
| Python/backend | Provider adapters, orchestration state machine, persisted AI metadata, tested APIs |
| Real-world product quality | Baseline and RAG comparison, versioned prompts/corpus, human escalation |
| Reliability and safety (important beyond JD) | Timeouts, retries, circuit breaker, fallback, layered validation, guardrails |
| Evaluation (important beyond JD) | Reviewed dataset, retrieval/answer/safety metrics, calibrated graders, regression gate |

---

# Phase III — Distributed, Observable, and Cloud-Native Engineering

## Milestone 10 — Background Jobs, Messaging, Caching, and Bounded Agent Tools

### 1. Topics to Learn

- Background jobs and message queues: producer/consumer, at-least-once delivery, acknowledgment, visibility/lock timeout, retry, poison messages, dead-letter queues (DLQs), ordering, and backpressure.
- Idempotency and deduplication. In distributed systems, “exactly once” is usually an end-to-end effect created with durable state and idempotent operations, not a magical queue setting.
- Transactional outbox and eventual consistency. A database commit and message publish cannot be assumed atomic.
- Redis cache-aside, TTL, invalidation, eviction, stampede protection, and why a cache is not the source of truth.
- Tool/function calling, schema validation, authorization, least privilege, bounded workflows, stop limits, and action audit. An “agent” is useful only where runtime tool choice solves a user problem.
- Monolith versus microservices and sync versus async workflows. Distribution adds network failures, deployment units, and consistency problems.
- Distributed-system foundations: network partitions, partial failure, time/ordering, consistency versus availability (including what the CAP theorem does and does not say), replication, leader/follower concepts, load balancing, horizontal scaling, and database partitioning/sharding. Understand and design around these; do not implement sharding before measurements require it.

### 2. Learning Depth

- **Queues/workers — Implement/Debug:** process jobs, renew locks if necessary, retry with limits, inspect a DLQ, and handle duplicates.
- **Idempotency/outbox — Design/Productionize:** make observable effects correct under redelivery and producer failure.
- **Caching — Design/Debug:** choose safe keys/TTLs, prevent cross-tenant leakage/stampede, and tolerate cache outage.
- **Agent tools — Implement/Productionize:** validate and authorize every call, bound iterations/time/cost, and return a safe failure.
- **Architecture — Design:** justify remaining a modular monolith with separately scaled API and worker processes.

### 3. Hands-On Learning Exercises

1. Deliver the same job twice and prove only one analysis effect is committed.
2. Crash a worker after database commit but before acknowledgment; observe redelivery and verify idempotency.
3. Publish a message outside a database transaction, reproduce a missing/orphan event, then implement an outbox relay.
4. Send a poison message until it reaches the DLQ; build a safe inspect/replay procedure.
5. Measure a cache miss, hit, expiry, stale entry, cache outage, and 100 concurrent misses for one key.
6. Give the model one allowed read tool and one invented/forbidden tool; verify only the authorized schema can execute and enforce a maximum-call limit.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Durable ingestion jobs | Extraction/embedding can exceed HTTP deadlines | API creates a job and outbox event; worker processes artifact/chunks and updates status | Commit/publish crash, duplicate delivery, timeout, poison, DLQ |
| Idempotent worker | At-least-once delivery repeats work | Use operation/idempotency keys, unique constraints, and atomic state transitions | Concurrent duplicate messages create one logical result |
| Cache-aside | Repeated safe reads/retrieval plans add latency/load | Cache selected tenant-safe metadata or deterministic retrieval results with versioned keys | Hit/miss/TTL/invalidation/outage/stampede tests |
| Read-only tool registry | Some questions need live incident/status data | Add typed `get_incident`, `get_synthetic_service_status`, and `search_runbooks` tools | Authorization, argument validation, timeout, max calls, fabricated tool |
| Bounded workflow | Open-ended agents loop and spend unpredictably | Implement a small state machine: interpret → optional tool → validate → answer/escalate | Step/time/token budgets and repeated-call detection |

Use a broker abstraction with a simple local broker for development and Azure Service Bus in the cloud. Do not use Redis as both disposable cache and critical durable queue without explicitly accepting its delivery/durability trade-offs.

### 5. Expected Project State

Long-running ingestion and evaluation no longer occupy request workers. The API remains stateless and the worker can scale independently, but both use the same codebase and domain. Compared with Milestone 9, OpsPilot now has deliberate eventual consistency, job status, idempotent effects, and a small auditable tool workflow—not a collection of microservices or uncontrolled agents.

### 6. Testing and Validation

- Use integration tests with the real local broker/cache equivalents and real PostgreSQL.
- Test state-machine transitions and outbox publication deterministically.
- Run chaos-style process termination at each job checkpoint.
- Load-test queue lag, job throughput, duplicate rates, and database connections while scaling worker concurrency.
- Extend AI eval cases to include tool selection, correct arguments, permission denial, tool-error recovery, call count, and final-answer grounding.

### 7. Failure Scenarios

- Duplicate, delayed, out-of-order, expired, or poison message.
- Worker dies mid-job or repeatedly loses its message lock.
- Queue is available but database is down, or the reverse.
- Cache is stale, evicted, unavailable, or leaks a result across tenants due to a bad key.
- Model fabricates a tool, calls repeatedly, requests another tenant’s incident, or passes dangerous parameters.
- Queue backlog grows faster than workers can drain it.

### 8. Skills Gained

You can design asynchronous backend work, handle eventual consistency and redelivery, use caching safely, and build bounded agentic tool use with authorization. This demonstrates system-design and cloud-native skills that distinguish production AI engineering from prompt demos.

### 9. Milestone Completion Checklist

- [ ] I can explain at-least-once delivery, idempotency, outbox, DLQ, backpressure, and eventual consistency.
- [ ] A crash at every important worker checkpoint does not corrupt or duplicate logical results.
- [ ] The DLQ has an inspect, fix, replay, and audit procedure.
- [ ] Cache failure reduces performance but does not break correctness; tenant/version are part of cache keys.
- [ ] Tools are read-only, schema-validated, authorized at execution time, and bounded by budgets.
- [ ] ADR-006 explains the queue and modular-monolith choices, plus triggers for service extraction.

## Milestone 11 — Observability, Performance, Docker, and CI/CD

### 1. Topics to Learn

- Observability: structured logs, metrics, traces, correlation/trace IDs, OpenTelemetry, dashboards, alerts, and the difference between telemetry and useful diagnosis.
- AI observability: prompt/model/corpus versions, provider attempts, retrieval trace, token/cost, validation/guardrail outcomes, tool calls, and user feedback—with privacy-aware sampling/redaction.
- SLIs/SLOs, error budgets, alert quality, runbooks, and incident response.
- Performance/load testing: latency distributions, throughput, saturation, queue lag, database pools, cold starts, profiling, and Little’s Law intuition.
- Docker images/layers, multi-stage builds, non-root users, immutable artifacts, Docker Compose, networking, volumes, health checks, and graceful shutdown.
- CI/CD: lint/type/test/security/migration/eval gates; build once, promote the same artifact; provenance; deployment strategies and rollback.

### 2. Learning Depth

- **Logs/metrics/traces — Productionize:** trace one request through API, retrieval, provider, database, and worker while controlling sensitive data.
- **SLO/alerting — Design:** make user-centered indicators and alerts with actionable runbooks.
- **Performance — Debug:** find the saturated resource from evidence and test an improvement.
- **Docker/Compose — Implement/Debug:** build a minimal secure image and run the entire local stack reproducibly.
- **CI/CD — Design/Productionize:** create gated artifacts, fail safely, and rehearse rollback.

### 3. Hands-On Learning Exercises

1. Trace one diagnosis end to end and find its logs, spans, database calls, retrieval candidates, LLM attempts, tokens, and outcome from one ID.
2. Break trace propagation across the worker and repair it.
3. Define SLOs from user journeys; create one good symptom-based alert and one noisy cause-based alert, then compare.
4. Load-test at increasing concurrency until latency bends; identify whether API CPU, DB pool, provider quota, or queue lag saturates first.
5. Inspect image layers/size and run as non-root with a read-only filesystem where practical.
6. Deliberately fail lint, type, unit, integration, migration, security, and AI regression stages and confirm promotion stops.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Structured telemetry | Failures cannot be diagnosed across dependencies | Instrument API/worker/provider/retrieval/database using OpenTelemetry conventions and structured logs | Trace propagation and sensitive-field redaction tests |
| Product/AI metrics | Uptime alone hides bad or expensive AI | Measure request/job outcomes, latency, errors, queue lag, retrieval quality samples, guardrail/fallback rates, tokens/cost | Metric presence/labels/cardinality review; synthetic failures |
| SLOs/runbooks | Teams need operational decisions, not dashboards only | Define availability/latency/job-freshness/quality/cost indicators and response playbooks | Alert fires on controlled symptom and links to usable runbook |
| Containers | “Works on my machine” blocks deployment | Add separate API/worker targets and Compose for PostgreSQL, cache, broker, and telemetry stack | Clean-machine build/start, health, migration, graceful stop |
| CI pipeline | Human memory is an unreliable release process | Run static checks, unit/integration tests, migration rehearsal, image/security scans, and small eval suite; publish immutable image on success | Negative tests for every gate and artifact digest verification |
| CD design | Bad releases need containment | Define dev/staging/prod environments, approvals, backward-compatible migrations, smoke tests, canary/blue-green plan, and rollback | Rehearse rollback with an intentionally broken candidate |

Avoid logging complete prompts and documents by default. Prefer IDs, hashes, versions, sizes, timing, token counts, outcome codes, and carefully sampled/redacted content in a restricted store when debugging requires it. Avoid high-cardinality metric labels such as incident or user IDs.

### 5. Expected Project State

OpsPilot is locally reproducible as containers, emits connected telemetry, has provisional SLOs/runbooks, and produces an immutable, tested image through CI. Compared with Milestone 10, it is diagnosable and releaseable, but the final cloud resources and production identity/network controls are not yet provisioned.

### 6. Testing and Validation

- Preserve a test pyramid: many unit tests, focused adapter integration tests, fewer end-to-end tests, and a separate eval pyramid.
- Run load tests in a representative non-production environment; never infer production capacity from laptop results.
- Verify graceful shutdown stops accepting work, finishes or safely abandons in-flight operations, and releases message/database locks.
- Scan dependencies, source secrets, container vulnerabilities, and infrastructure configuration; triage findings rather than claiming zero risk.
- Rehearse a rollback and a backward-compatible expand/migrate/contract database change.

### 7. Failure Scenarios

- Trace context disappears at queue boundary.
- Logs accidentally contain customer text or access tokens.
- Metrics create a cardinality/cost explosion.
- Alert fires constantly or misses a user-visible outage.
- Container runs as root, ignores termination, or reports ready before dependencies/config are usable.
- Migration passes from empty but fails on existing data.
- CI succeeds while AI quality regresses because eval gate was skipped.

### 8. Skills Gained

You can observe and performance-test an AI backend, package it securely, automate quality gates, and reason about deployment/rollback. These are direct cloud-native product-development capabilities in the target role.

### 9. Milestone Completion Checklist

- [ ] One trace follows an API request through retrieval/provider and an async job through the queue.
- [ ] Logs/metrics are useful, privacy-aware, and have reviewed cardinality/retention.
- [ ] SLOs identify users, indicators, targets, and measurement windows; alerts have owners/runbooks.
- [ ] A clean environment can start the full stack from documented commands.
- [ ] Images run non-root, are immutable/versioned, and shut down gracefully.
- [ ] CI blocks code, migration, security, and AI-quality regressions; rollback has been rehearsed.

### Phase III Job-Readiness Evidence

| Job requirement | Evidence after Milestone 11 |
|---|---|
| APIs/backend systems | Stateless API, workers, durable jobs, outbox, cache, PostgreSQL, health and graceful shutdown |
| AI applications | Bounded tools, evaluation-aware orchestration, AI-specific telemetry |
| Cloud-native solutions | Secure containers, external configuration, independent scaling boundaries, immutable artifacts |
| Collaboration | Automated review gates, ADRs, runbooks, dashboards, incident and rollback rehearsals |
| Production skills beyond JD | Distributed tracing, SLOs, load/failure testing, idempotency, cost and privacy controls |

---

# Phase IV — Azure Deployment, Scaling, and Production Operations

## Milestone 12 — Azure Foundations to a Production Candidate

### 1. Topics to Learn

Learn cloud concepts before clicking through deployment screens:

- **Cloud fundamentals:** regions, availability zones, shared responsibility, IaaS/PaaS/serverless containers, elasticity, consumption pricing, quotas, resource lifecycle, and recovery objectives.
- **Azure organization/governance:** subscriptions, resource groups, naming, tags, budgets, policies, environments, and least-privilege RBAC.
- **Networking:** DNS, TLS, ingress/egress, virtual networks/subnets, network security groups, private endpoints, service discovery, and why “inside Azure” is not automatically private.
- **Identity:** Microsoft Entra ID for people, managed identities for workloads, workload identity federation for CI, and Key Vault only for secrets that cannot be replaced by identity.
- **Storage/data:** Azure Blob Storage for artifacts, Azure Database for PostgreSQL Flexible Server for system-of-record and `pgvector`, Azure Managed Redis for cache, and backup/restore/retention.
- **Containers/messaging:** Azure Container Registry (ACR), Azure Container Apps (ACA) for API/worker/jobs, and Azure Service Bus for reliable queues/DLQ.
- **AI/operations:** an Azure-hosted LLM deployment or provider endpoint behind the existing adapter; Azure Monitor and Application Insights via OpenTelemetry.
- **Infrastructure and delivery:** Bicep as the selected Azure-native IaC, with Terraform as a valid multi-cloud alternative; revision traffic, smoke tests, rollback, autoscaling, quotas, and cost governance.

Why these services: managed PaaS reduces patching and operational burden while preserving the architecture learned locally. ACA fits HTTP and event-driven containers without requiring Kubernetes administration; PostgreSQL keeps relational and vector data together at this scale; Blob Storage is better than container disks for durable artifacts; Service Bus provides durable delivery/DLQ; Managed Redis accelerates safe reads but is not authoritative; managed identity reduces long-lived credentials.

ACA ingress supplies TLS termination, routing, and load balancing across replicas; its HTTP and event rules provide horizontal scaling. Learn those mechanisms rather than adding Nginx or a separate load balancer by habit. Learn Kubernetes concepts only well enough to compare ACA with AKS; operate AKS when requirements demand cluster-level control, not as a portfolio checkbox.

### 2. Learning Depth

- **Cloud/resource model/networking — Understand/Design:** draw traffic, identity, data, and trust boundaries and explain shared responsibility.
- **Azure services — Implement/Debug:** provision, connect, observe, scale, back up, and diagnose access/network/quota failures.
- **Identity/secrets — Productionize:** use Entra/workload identities and RBAC; rotate the remaining secrets.
- **IaC/CI/CD — Productionize:** recreate environments, detect drift conceptually, use federated CI identity, deploy immutable revisions, and rollback.
- **Reliability/scalability/cost — Design:** set realistic SLO, RTO/RPO, capacity, scaling, and budget assumptions; load/failure test them.

### 3. Hands-On Learning Exercises

1. Draw the Azure control plane versus data plane and identify who can deploy, read secrets, read blobs, query the DB, and call the model.
2. Deny a managed identity a required role, diagnose the `403` from logs/traces, grant least privilege, and verify recovery.
3. Compare public endpoint plus firewall with private endpoint/VNet integration for database, storage, and cache; document cost and complexity.
4. Deploy a revision with zero traffic, smoke-test it, shift a small percentage, then roll back.
5. Drive HTTP and queue load separately; observe API replica count, worker scale, database connections, provider quotas, queue lag, latency, and cost.
6. Simulate Service Bus unavailability, database failover/restart in a safe environment, model throttling, and a broken deployment.
7. Restore PostgreSQL and Blob data into a non-production environment and measure recovery time and recovered point.

### 4. Project Implementation

| Concept | Engineering problem | OpsPilot implementation | Testing |
|---|---|---|---|
| Resource organization/IaC | Manual resources drift and cannot be reviewed | Bicep modules for resource group-scoped dev/staging/prod resources, tags, diagnostics, roles, and budgets | Validate/preview changes, redeploy idempotently, peer-review diffs |
| Registry/runtime | Containers need a managed execution platform | Push immutable API/worker images to ACR; deploy separate ACA components with probes and revision settings | Image digest, startup/readiness, graceful shutdown, revision smoke test |
| Managed identity/Key Vault | Embedded credentials create rotation/leak risk | Give each workload a narrow identity; use Entra auth where supported and Key Vault references for residual secrets | Deny/allow matrix, rotation test, no static Azure credential in repo/CI |
| Managed data | Container disks and self-hosted DB/cache are fragile | Use Blob, PostgreSQL Flexible Server + `pgvector`, Managed Redis, and backup/retention policies | Connectivity, tenant isolation, restore, cache outage, storage lifecycle |
| Durable messaging | Local broker is not a cloud operations plan | Implement Azure Service Bus adapter, DLQ monitoring, and queue-based worker scaling | Duplicate/lock expiry/DLQ/outage/backlog tests |
| Monitoring | Local telemetry must support on-call diagnosis | Export OpenTelemetry to Application Insights/Azure Monitor; dashboards and alerts for SLOs, quota, DLQ, queue lag, cost | Synthetic transaction, alert drill, trace-to-runbook workflow |
| Federated CD | Static deployment secrets are dangerous | GitHub Actions uses OIDC/workload federation, environments, approvals, IaC preview, deploy, smoke/canary, promote/rollback | Permission denial, failed smoke test, rollback, audit trail |
| Autoscaling/capacity | Demand varies and dependencies have limits | Scale API on HTTP concurrency and worker on queue depth within min/max and DB/provider quotas | Stepped load, saturation, cold start, scale-down and cost observation |

### 5. Expected Project State

OpsPilot is an Azure-hosted production candidate. It accepts authenticated requests, stores artifacts and relational/vector data in managed services, processes durable jobs, performs evaluated AI diagnosis with grounded citations, degrades safely, emits end-to-end telemetry, deploys via federated CI/CD, and scales API and worker independently. The final architecture remains understandable because it evolved from the same modular application rather than being rebuilt as disconnected services.

### 6. Testing and Validation

- Run unit/integration/eval gates before build; deploy the same image through staging to production.
- Run post-deploy smoke tests and synthetic transactions without using real customer secrets.
- Load-test against explicit user/traffic assumptions and dependency quotas; capture the first bottleneck and mitigation.
- Conduct failure drills for provider throttling/outage, DB/cache/broker failure, bad revision, identity denial, and queue backlog.
- Run DAST/API authorization checks in staging plus dependency/image/IaC/secret scans.
- Verify backup restore, RTO/RPO, alert delivery, rollback, data deletion, and cost budget alerts.
- Complete an Azure Well-Architected review across reliability, security, cost optimization, operational excellence, and performance efficiency.

### 7. Failure Scenarios

- ACA revision never becomes ready or has a broken migration.
- Managed identity/RBAC or network rule blocks a dependency.
- Model deployment is throttled, unavailable, or exceeds budget.
- High traffic exhausts database connections before autoscaling helps.
- Worker scale creates a provider-rate-limit storm.
- Service Bus lock expires, message reaches DLQ, or backlog violates freshness SLO.
- Cache outage increases database load; database failover exposes bad retry behavior.
- Region/zone issue, DNS/TLS failure, or accidental public exposure.
- Telemetry pipeline fails during the incident or logs sensitive content.

### 8. Skills Gained

You can design, provision, deploy, secure, evaluate, scale, observe, and operate a real AI backend on Azure. This directly completes the job requirements for Python, APIs/backend, AI applications, Azure/cloud-native solutions, and collaboration. It also demonstrates reliability, security, distributed systems, evaluation, cost governance, and incident response that the job description leaves implicit.

### 9. Milestone Completion Checklist

- [ ] I can draw the cloud architecture, trust boundaries, identity flows, data flows, and failure domains from memory.
- [ ] Infrastructure is reproducible from reviewed IaC with no long-lived Azure credentials in CI.
- [ ] API and worker deploy as immutable revisions and can roll back without data corruption.
- [ ] A real restore drill meets documented provisional RTO/RPO or produces an improvement plan.
- [ ] Load tests identify capacity and first bottleneck; scaling respects DB/provider quotas and cost limits.
- [ ] Failure drills prove safe degradation for provider, DB, cache, broker, identity, and deployment faults.
- [ ] Dashboards/alerts answer user impact, quality, latency, errors, queue health, tokens/cost, and saturation.
- [ ] The held-out AI regression suite and security gates pass for the release candidate.
- [ ] I completed a Well-Architected review and documented accepted risks and next investments.

### Phase IV Job-Readiness Evidence

| Job requirement | Final evidence |
|---|---|
| Python | Production-style typed FastAPI/worker code, async I/O, concurrency, profiling, packaging, tests |
| APIs and backend systems | REST contracts, auth/RBAC, PostgreSQL, transactions, queues, cache, idempotency, rate controls |
| AI applications | Provider abstraction, structured outputs, RAG, bounded tools, guardrails, evals, fallback/escalation |
| Cloud-native solutions | Stateless containers, managed data/messaging, IaC, autoscaling, health, observability, rollback |
| Azure | Resource groups, ACR, ACA, PostgreSQL, Blob, Service Bus, Managed Redis, Key Vault, Entra identity, Monitor/Application Insights |
| Collaboration | ADRs, threat model, docs, CI checks, reviewable IaC, runbooks, incident/evaluation reports |
| Product teamwork | Measurable user journey, feedback workflow, quality/cost/latency trade-offs, documented non-goals |

### Skills the Job Description Leaves Implicit

Job descriptions list the hiring filter, not the daily work. These skills are rarely written down, are expected in real AI teams, and are already produced by this roadmap—so name them explicitly in your CV, README, and interviews.

| Implicit skill | Why teams need it | Where the roadmap proves it |
|---|---|---|
| AI evaluation and regression gating | Without measurement, "the answer looks fine" is the only quality signal, and every prompt or model change is an unmeasured risk | Milestones 6, 8, 9; AI Evaluation Strategy |
| Guardrails, safety, and prompt-injection defence | Retrieved documents and user logs are untrusted input that can redirect the model | Milestones 5, 8, 9, 10 |
| Cost and token governance | Usage-based pricing turns an inefficient prompt or retry storm into a budget incident | Milestones 6, 7, 10, 12 |
| Data privacy and redaction | Customer logs contain secrets and PII that must never reach a provider, a log line, or a trace | Milestones 5, 6, 11 |
| Non-determinism literacy | Debugging, testing, and support all change when the same input can produce different output | Milestones 2, 6, 7, 9 |
| Failure-mode design and degraded modes | Users need an honest degraded answer, not a 500, when a provider throttles or fails | Milestones 7, 10, 12 |
| Idempotency and delivery semantics | At-least-once messaging duplicates expensive LLM work unless the system is designed for it | Milestones 3, 10 |
| Observability and incident response | You are on the hook for diagnosing a latency or quality regression in production, not just shipping the feature | Milestones 11, 12 |
| Capacity, quota, and scaling judgement | Model deployments, database connections, and container replicas all have hard limits that shape architecture | Milestones 10, 12 |
| Infrastructure as code and release discipline | Reproducible environments, promotion by digest, and rollback are the difference between a demo and a service | Milestones 11, 12 |
| Trade-off communication and written decisions | Data scientists, product managers, and engineers must be able to review why the system is shaped this way | ADR practice throughout |
| Requirements and scope negotiation | The most valuable AI engineering decision is often what not to build autonomously | Non-goals, bounded read-only tools |
| Vendor and framework evaluation | Provider APIs and AI frameworks change quickly; the ability to compare and swap them is the durable skill | Provider adapter, optional-framework policy |
| Prompt and corpus versioning as release artifacts | A prompt edit is a production change and needs the same traceability as code | Milestones 7, 8, 9 |
| Human-in-the-loop workflow design | Escalation quality determines whether an AI product is trusted or bypassed | Milestones 9, 12 |

---

# Final Architecture

```text
API consumer / support UI
          |
      HTTPS + JWT
          v
Azure Container Apps ingress
          |
          v
FastAPI API (stateless, independently scalable)
  |  request validation / authn / tenant authz / rate & size limits
  |  business use cases / idempotency / audit / job creation
  |
  +----> Azure Database for PostgreSQL
  |        incidents, users/tenants, jobs, prompt/model metadata,
  |        documents/chunks + pgvector, eval cases/results, outbox
  |
  +----> Azure Blob Storage
  |        encrypted, lifecycle-managed sanitized artifacts
  |
  +----> Azure Managed Redis
  |        disposable, tenant/version-scoped cache and coordination
  |
  +----> Azure Service Bus ----> Worker Container App / Job
  |        durable jobs/DLQ       parse, redact, chunk, embed, evaluate
  |
  v
AI orchestration layer
  prompt + model + corpus versions
  context budget / provider adapter / deadline / retry / circuit breaker
          |
          +----> RAG: authorized hybrid retrieval -> rerank -> context
          |
          +----> bounded read-only tool registry
          |
          +----> primary LLM deployment -> alternate/degraded path
          |
          v
structured output -> Pydantic schema -> business/evidence validation
-> safety/secret/policy checks -> complete | degraded | abstain | escalate
          |
          v
versioned diagnosis + citations + feedback

Cross-cutting: managed identities + Key Vault, OpenTelemetry,
Application Insights/Azure Monitor, SLOs/alerts/runbooks, audit trail,
GitHub Actions OIDC, Bicep, ACR, immutable ACA revisions and rollback.
```

## Important Architecture Decisions and Alternatives

| Problem | Possible approaches | Selected approach and trade-off |
|---|---|---|
| Application boundaries | Single script, modular monolith, microservices | **Modular monolith with API/worker processes.** Preserves clear modules and independent compute scaling without distributed deployment overhead. Extract a service only for independently owned/scaled/released boundaries proven by data. |
| Operational database | PostgreSQL, document NoSQL, mixed stores | **PostgreSQL.** Transactions, constraints, joins, mature operations, and `pgvector`; accept schema migration work and scale-up/replica planning. |
| Vector retrieval | `pgvector`, dedicated vector DB, Azure AI Search | **`pgvector` first.** One authorization/backup/transaction boundary. Move when corpus/query scale, hybrid ranking, operational ownership, or measured latency/recall requires it. |
| Long work | Hold HTTP request, in-process background task, durable queue | **Durable Service Bus jobs.** Survives restarts and scales workers; introduces eventual consistency, idempotency, DLQ, and more operations. |
| Cache | No cache, Redis, semantic answer cache | **Cache-aside in Managed Redis only for measured safe paths.** Exact/versioned keys first; semantic answer caching stays optional because similarity can return incorrect or unauthorized answers. |
| AI SDK | Provider calls everywhere, generic framework, narrow port/adapters | **Narrow product-owned adapter.** Keeps tests and fallback controllable while allowing provider-specific capability via explicit extensions. |
| AI workflow | Free-text prompt, structured pipeline, autonomous/multi-agent | **Deterministic state machine with structured model steps and bounded read-only tools.** Easier to test and operate; less flexible than open-ended autonomy by design. |
| Hosting | VM/App Service/ACA/AKS | **Azure Container Apps.** Managed ingress/revisions/autoscale and lower Kubernetes burden; less cluster-level control than AKS. Use AKS only if platform requirements justify operating it. |
| Identity | Custom accounts, static keys, Entra + managed identity | **Entra for people and managed identities for workloads.** Less secret handling; adds identity/RBAC setup and external dependency. |
| Infrastructure | Portal scripts, Bicep, Terraform | **Bicep.** Deep Azure integration and smaller initial learning surface. Terraform is appropriate if multi-cloud/provider ecosystem is a real requirement. |

# Recommended Technology Stack

| Layer | Selection | Why it earns its place |
|---|---|---|
| Language/tooling | Python, type checker, `pytest`, formatter/linter, dependency lock | Target-role alignment, async ecosystem, visible contracts, reproducible quality |
| API/contracts | FastAPI, Pydantic, OpenAPI | Typed boundary validation and documented async REST APIs |
| Data | PostgreSQL, SQLAlchemy, Alembic, `pgvector` | Transactional system of record, migrations, vector retrieval without premature extra service |
| Artifacts | Local object-store adapter, then Azure Blob Storage | Durable files should not live on ephemeral container disks |
| Cache | Redis locally, Azure Managed Redis in cloud | Low-latency disposable cache after measurement |
| Messaging | Local broker/emulator adapter, Azure Service Bus in cloud | Durable asynchronous processing, redelivery, DLQ, scalable workers |
| AI | Provider-owned SDK behind adapter; Azure-hosted LLM/embedding deployments where appropriate | Current features without coupling product logic; supports controlled fallback and testing |
| Observability | OpenTelemetry, structured logs, local collector/dashboard stack, Application Insights/Azure Monitor | Vendor-neutral instrumentation with Azure operations backend |
| Runtime | Docker/Compose locally; ACR + Azure Container Apps | Reproducible artifacts, managed revisions and HTTP/event scaling |
| Security/identity | Entra ID, managed identities, Azure RBAC, Key Vault, scanning tools | Least privilege and reduced static-secret exposure |
| CI/CD/IaC | GitHub Actions with OIDC, Bicep | Automated gates, federated deployment identity, reviewable/reproducible cloud |

Frameworks such as LangChain/LangGraph, PydanticAI, LiteLLM, DeepEval, Ragas, Langfuse, or Promptfoo are **optional comparison implementations**, not mandatory architecture. First implement the core orchestration/evaluation boundary yourself. Adopt a framework only if it removes measured maintenance cost without hiding authorization, retries, traces, versions, or eval semantics.

# End-to-End Testing Strategy

## Test portfolio

| Layer | What it proves | Typical frequency |
|---|---|---|
| Static checks | Types, formatting/lint, dependency and source-secret issues | Every change |
| Domain unit tests | Deterministic rules/invariants/state transitions | Every change |
| Component tests with fakes | Orchestration, retries, fallback, guardrails, prompt construction | Every change |
| Adapter contract tests | All repository/provider/broker/cache implementations honor the same product contract | Every change or adapter update |
| Integration tests | Real PostgreSQL, broker/cache, migrations, auth dependency, serialization | Every change in CI |
| API tests | HTTP contract, validation, authz, pagination, idempotency, errors | Every change |
| AI small eval | Critical deterministic, retrieval, safety, and representative cases | Pull request |
| Full offline eval | Quality by slice, candidate versus baseline, cost/latency | Scheduled and pre-release |
| End-to-end/smoke | Deployed user journeys and managed-service wiring | Each deployment |
| Load/soak | Capacity, saturation, leaks, queue lag, dependency quotas | Before major release/change |
| Failure/chaos drills | Degraded behavior, recovery, alerts, runbooks | Scheduled |
| Security tests | Authz matrix, input abuse, prompt injection, secret/PII leakage, DAST/scans | PR subset and pre-release full |
| Restore/rollback | Data and release recovery actually work | Scheduled and after material change |

## Key testing rules

1. Test the same database engine and important broker semantics used in production.
2. Mock at expensive/external boundaries, not inside the logic under test.
3. Every production incident or escaped AI failure should create a regression case when safe.
4. Do not make ordinary CI flaky or costly by calling a live model; use fakes/recorded synthetic fixtures plus controlled provider smoke/eval runs.
5. Test unhappy paths intentionally: timeout, cancellation, malformed data, partial commit, duplicate event, denied permission, outage, and rollback.
6. Track flaky tests as defects; a test no one trusts is not a gate.

# AI Evaluation Strategy

## Dataset design

Each case should include: stable ID, sanitized user/log input, tenant/document fixture, corpus version, expected relevant chunk IDs, expected behavior or reference facts, forbidden claims/actions, answerability, tags/slices, risk level, human-review state, and provenance. Important slices include common web-server errors, database issues, permissions, unknown/insufficient evidence, conflicting/outdated docs, exact error codes, paraphrases, multilingual/noisy input if relevant, prompt injection, secret/PII, and cross-tenant attempts.

Keep three functional sets:

- **Development:** used frequently while designing prompts/retrieval.
- **Regression:** reviewed known failures and representative cases, run continuously.
- **Held-out:** used sparingly for release decisions so repeated tuning does not overfit it.

Production feedback may propose cases, but a human must sanitize, label, deduplicate, and approve it. Never automatically train or gate on raw thumbs-up/down.

## Evaluation layers

1. **Pipeline validity:** completion/refusal/error state, schema pass, business-rule pass, valid evidence IDs, tool-call validity, and no secret leakage.
2. **Retrieval:** recall@K, precision@K, MRR/nDCG, authorized-context rate, stale-version rate, latency, and query-category slices.
3. **Answer:** factual/task correctness, groundedness, citation precision/recall, completeness, actionability, severity correctness, and consistency with deterministic signals.
4. **Behavior:** correct abstention, correct escalation, safe handling of unsupported/dangerous requests, injection resistance, and tool authorization.
5. **Operations:** p50/p95 latency, time to first/useful response if streaming, availability, error/fallback rate, tokens, cost per request and per successful diagnosis, queue freshness.
6. **Human/product:** reviewer agreement, escalation usefulness, support resolution contribution, user feedback, and recurring failure themes.

## Grading hierarchy

- Prefer deterministic checks when the requirement is deterministic.
- Use retrieval metrics when ground-truth passages exist.
- Use rubric-based model graders for nuanced attributes, pinning and versioning judge configuration.
- Blind candidate labels and use pairwise comparison where it reduces inconsistent absolute scoring.
- Calibrate judges on a human-scored sample, inspect disagreements, and report agreement/uncertainty.
- Reserve human review for high-risk, ambiguous, sampled production, and judge-calibration cases.

## Release decision

Compare a candidate with the currently released baseline on the same data and corpus. Block any critical authorization, secret-leak, dangerous-action, or evidence-integrity failure. For quality metrics, require no practically meaningful regression in critical slices and an explicit trade-off decision for latency/cost changes. Publish the dataset size, slice counts, confidence/variance where possible, and known limitations; never advertise a single context-free “accuracy” number.

# Azure Deployment Architecture and Progression

## Learning progression

1. **Fundamentals:** region/zone, resource group, PaaS/shared responsibility, cost model, quotas.
2. **Resources/governance:** dev resource group, tags, budgets, IaC and least-privilege roles.
3. **Networking:** DNS/TLS/ingress, outbound dependencies, VNet/private endpoint decision.
4. **Identity:** developer login, Entra token validation, managed workload identities, CI federation.
5. **Storage/data:** Blob lifecycle, PostgreSQL backup/HA/connection pool, Redis cache semantics.
6. **Containers/messaging:** ACR, ACA environment/API/worker/jobs, Service Bus queues/DLQ.
7. **Deployment:** revision probes, zero-traffic smoke, progressive traffic, migration, rollback.
8. **Monitoring/scaling:** Application Insights/Monitor, SLO alerts, queue/HTTP scaling, quota and cost tests.

## Environment shape

- Separate dev, staging, and production configuration and ideally separate subscriptions or at minimum resource groups and identities according to budget/organization maturity.
- Build one image and promote by digest; do not rebuild different code per environment.
- Public HTTPS reaches only API ingress. Data services use network restrictions/private connectivity when the risk/cost analysis supports it.
- API and worker receive different managed identities and least-privilege roles.
- Database remains authoritative; Redis is disposable; Blob and database have retention/restore procedures; queue DLQ is monitored.
- Production model deployment, quotas, region availability, content policies, and data handling must be verified for the actual Azure account before finalizing the design.

# Suggested GitHub Repository Structure

```text
opspilot/
├── README.md
├── pyproject.toml
├── lockfile
├── .env.example                 # names and safe examples, never secrets
├── Makefile                     # or task runner with documented commands
├── src/opspilot/
│   ├── domain/                  # entities, value objects, domain errors/rules
│   ├── application/             # use cases, ports, orchestration state machines
│   ├── api/                     # FastAPI routes, dependencies, DTOs, error mapping
│   ├── infrastructure/
│   │   ├── db/                  # SQLAlchemy mappings/repositories/unit of work
│   │   ├── ai/                  # provider adapters, prompts, output schemas
│   │   ├── retrieval/           # chunking, embeddings, search/rerank/context
│   │   ├── messaging/           # outbox, broker adapters, workers
│   │   ├── cache/
│   │   ├── storage/
│   │   └── telemetry/
│   ├── security/                # authz policy, redaction, guardrail code
│   └── settings.py
├── prompts/                     # immutable/versioned prompt artifacts + changelog
├── evals/
│   ├── datasets/                # sanitized versioned cases and corpus fixtures
│   ├── graders/
│   ├── runners/
│   └── reports/                 # selected reproducible reports, not secrets
├── tests/
│   ├── unit/
│   ├── component/
│   ├── integration/
│   ├── contract/
│   ├── api/
│   ├── e2e/
│   ├── security/
│   └── load/
├── migrations/
├── infra/bicep/
│   ├── modules/
│   └── environments/
├── deploy/                      # Dockerfiles, Compose, collector config
├── architecture/
│   ├── diagrams/
│   ├── decisions/
│   ├── threat-model.md
│   └── capacity-cost-model.md
├── docs/
│   ├── api-and-local-development.md
│   ├── evaluation.md
│   ├── security-privacy.md
│   └── demos/
├── runbooks/                    # outage, rollback, restore, DLQ, key/identity issues
└── .github/workflows/           # CI, scheduled eval, deploy, rollback helpers
```

Do not create every directory on day one. Grow the repository at the milestone that creates the concept; the history should show the architecture evolving.

# Learning Resources by Major Topic

Prefer official documentation for behavior that changes and use books for durable mental models. Read selectively, then apply each section through the milestone experiment.

| Topic | Primary resources | Use them for |
|---|---|---|
| Python | [Python tutorial](https://docs.python.org/3/tutorial/), [asyncio](https://docs.python.org/3/library/asyncio.html), *Effective Python* by Brett Slatkin, *Fluent Python* by Luciano Ramalho | Language depth, async mechanics, idioms, data model |
| Testing/types | [`pytest` docs](https://docs.pytest.org/), [Python typing docs](https://docs.python.org/3/library/typing.html) | Fixtures/parametrization, type contracts and limitations |
| HTTP/API/FastAPI | [MDN HTTP overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview), [FastAPI docs](https://fastapi.tiangolo.com/), [Pydantic docs](https://docs.pydantic.dev/), [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457) | Protocol semantics, APIs, validation, consistent errors |
| SQL/PostgreSQL | [PostgreSQL tutorial](https://www.postgresql.org/docs/current/tutorial.html), [indexes](https://www.postgresql.org/docs/current/indexes.html), [concurrency control](https://www.postgresql.org/docs/current/mvcc.html), [SQLAlchemy tutorial](https://docs.sqlalchemy.org/en/20/tutorial/), [Alembic tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html) | SQL first, query plans, transactions, ORM/migrations |
| Security | [OWASP API Security Top 10](https://owasp.org/API-Security/), [OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700), [Microsoft identity platform docs](https://learn.microsoft.com/en-us/entra/identity-platform/) | Threat model, API abuse, current identity protocol guidance |
| System design | *Designing Data-Intensive Applications* by Martin Kleppmann, [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/), [Cloud Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/) | Consistency, queues, cache, failure, scale, trade-offs |
| LLM APIs/reliability | [Official OpenAI API docs](https://developers.openai.com/api/docs/), [structured outputs](https://developers.openai.com/api/docs/guides/structured-outputs), [evals](https://developers.openai.com/api/docs/guides/evals), [safety best practices](https://developers.openai.com/api/docs/guides/safety-best-practices) | Current API mechanics, schemas, evaluation, safety; verify chosen Azure/provider feature parity |
| RAG/vector search | [`pgvector`](https://github.com/pgvector/pgvector), [PostgreSQL full-text search](https://www.postgresql.org/docs/current/textsearch.html), [Azure retrieval/RAG guidance](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview) | Vector + lexical/hybrid retrieval, indexes, RAG design alternatives |
| Messaging/cache | [Azure Service Bus Python client](https://learn.microsoft.com/en-us/python/api/overview/azure/servicebus-readme), [Service Bus queues/topics](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-queues-topics-subscriptions), [Redis docs](https://redis.io/docs/latest/), [Azure Managed Redis overview](https://learn.microsoft.com/en-us/azure/redis/overview) | Delivery semantics, DLQ, cache patterns and managed service |
| Observability | [OpenTelemetry docs](https://opentelemetry.io/docs/), [Azure Monitor OpenTelemetry](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable), [Google SRE book](https://sre.google/sre-book/table-of-contents/) | Instrumentation, SLIs/SLOs, alerts, incident operations |
| Docker/CI | [Docker guides](https://docs.docker.com/guides/), [Docker build best practices](https://docs.docker.com/build/building/best-practices/), [GitHub Actions docs](https://docs.github.com/en/actions) | Images, Compose, secure builds, automated pipelines |
| Azure deployment | [Azure fundamentals learning path](https://learn.microsoft.com/en-us/training/paths/describe-cloud-service-types/), [Container Apps overview](https://learn.microsoft.com/en-us/azure/container-apps/overview), [Container Apps revisions](https://learn.microsoft.com/en-us/azure/container-apps/revisions), [PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/service-overview), [managed identities](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity), [Key Vault references](https://learn.microsoft.com/en-us/azure/container-apps/manage-secrets), [Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview) | Resource model, hosting, data, passwordless identity, IaC, revisions/rollback |
| Architecture quality | [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/what-is-well-architected-framework) | Reliability, security, cost, operations, performance review |

# Portfolio and Demonstration Strategy

## README structure

Lead with the problem and evidence, not a list of technologies:

1. One-paragraph user problem and product demo GIF/video.
2. Architecture diagram and a request trace through reliability layers.
3. “Why this design” summary linking to 5–8 strong ADRs.
4. Local setup with one reproducible path and synthetic demo data.
5. API examples and the complete/abstain/escalate response contract.
6. Evaluation methodology and a versioned report with honest limitations.
7. Reliability/security section: failure matrix, threat model, data handling, and non-goals.
8. Cloud/deployment section: IaC, CI/CD, monitoring, rollback, restore, and cost assumptions.
9. Roadmap/backlog based on measured gaps rather than trendy features.

## Demonstrations to record

- **Three-minute product demo:** create incident, upload synthetic log, receive cited diagnosis, inspect source, then escalate.
- **Reliability demo:** force malformed output, provider throttling/outage, and show validated retry/degraded/abstention behavior plus trace.
- **RAG/eval demo:** change chunking or prompt, run candidate versus baseline, and explain why a regression blocks release.
- **Cloud operations demo:** deploy a revision, smoke/canary it, inspect telemetry, create a controlled failure, use a runbook, and roll back.

## Evidence artifacts

- Architecture and deployment diagrams.
- A small set of high-quality ADRs rather than dozens of trivial ones.
- Sanitized eval dataset schema and one reproducible report.
- Load-test report stating environment, traffic model, bottleneck, and change made.
- Threat model and authorization matrix.
- Incident postmortem from a simulated failure.
- SLO dashboard screenshot and alert/runbook link.
- Cost model with assumptions and measured tokens/requests, never undisclosed provider/customer data.

## CV bullet templates

Replace brackets only with measurements you actually reproduced:

- “Built and deployed a multi-tenant AI incident copilot in Python/FastAPI on Azure Container Apps, integrating PostgreSQL/`pgvector`, Blob Storage, Service Bus, and managed identity through reproducible Bicep infrastructure.”
- “Designed a versioned RAG and evaluation pipeline across [N] reviewed cases, measuring retrieval recall@K, groundedness, citation accuracy, abstention, latency, and cost; blocked releases on critical safety or evidence regressions.”
- “Improved [p95 latency/job throughput/cost] from [measured baseline] to [measured result] by [specific change], while maintaining [quality/SLO constraint].”
- “Engineered failure controls for LLM throttling/timeouts and duplicate background delivery using bounded retries, circuit breaking, idempotency/outbox, DLQ operations, and human escalation.”
- “Implemented privacy-aware OpenTelemetry tracing and Azure monitoring across API, RAG, LLM, database, and workers; rehearsed [rollback/restore/outage] against documented SLO/RTO/RPO targets.”

Never claim “production,” “99.9%,” “X% accuracy,” or scale numbers merely because the architecture could support them. Say “production-style” or “production candidate” until it has real users, operational ownership, and measured operation.

# Final Skills Checklist

## Python and software engineering

- [ ] I write typed, modular Python and can explain composition, dependency inversion, packaging, exceptions, and resource lifecycle.
- [ ] I choose sync, async, threads, or processes from workload characteristics and can debug event-loop blocking/cancellation.
- [ ] I design unit, component, integration, contract, end-to-end, load, and failure tests deliberately.
- [ ] I profile before optimizing and communicate changes through Git, ADRs, reviews, and documentation.

## Backend and data

- [ ] I design HTTP/REST contracts, validation, consistent errors, pagination, idempotency, health, and API evolution.
- [ ] I design PostgreSQL schemas, constraints, transactions, indexes, migrations, pools, concurrency controls, backup/restore.
- [ ] I implement authentication, RBAC/object authorization, multi-tenancy, secure uploads, audit, and privacy controls.
- [ ] I can use queues, workers, outbox, DLQ, caching, backpressure, and eventual consistency safely.

## AI engineering

- [ ] I explain model/context/sampling/cost/latency limitations and isolate provider SDKs behind testable boundaries.
- [ ] I version prompts/models/configuration and validate syntax, schema, business rules, evidence, and safety separately.
- [ ] I implement deadlines, bounded retry/jitter, rate controls, circuit breaker, fallback, abstention, and escalation.
- [ ] I build and tune the entire RAG pipeline using retrieval and answer evidence, with access-controlled citations.
- [ ] I build representative eval datasets, calibrated graders, per-slice regression gates, and human review.
- [ ] I implement bounded, authorized, observable tool calling and can justify why autonomy is limited.

## Cloud-native and operations

- [ ] I build secure reproducible containers, CI/CD gates, immutable artifacts, progressive deployment, and rollback.
- [ ] I instrument logs/metrics/traces and define SLIs/SLOs, dashboards, alerts, and actionable runbooks.
- [ ] I use Azure resource organization, networking, Entra/RBAC, managed identities, Key Vault, managed data, ACA/ACR, Service Bus, and Monitor.
- [ ] I provision with IaC, use federated CI identity, test scale/quotas/cost, and perform restore/failure drills.
- [ ] I can review reliability, security, cost, operational excellence, and performance trade-offs honestly.

# Interview Topics to Be Ready to Explain

1. Walk through one diagnosis from HTTP request to cited response, including every trust and validation boundary.
2. Why a modular monolith was chosen, and concrete evidence that would justify a microservice split.
3. Why PostgreSQL/`pgvector` was selected over NoSQL or a dedicated vector database; how the migration path would work.
4. How async Python works, what blocks an event loop, and where threads/processes/workers fit.
5. Transaction boundaries, isolation, optimistic concurrency, connection-pool exhaustion, and migration safety.
6. Authentication versus authorization; JWT validation; cross-tenant enforcement; managed identity versus secrets.
7. HTTP idempotency versus job idempotency; at-least-once delivery, outbox, DLQ, and duplicate handling.
8. Cache-aside, invalidation, stampede, stale data, semantic-cache risk, and cache-outage behavior.
9. Why structured output does not guarantee a factually correct output; the validation layers around the model.
10. Which provider errors are retried and why; how deadlines, jitter, circuit breaking, quotas, and fallback interact.
11. The complete RAG pipeline; chunk/top-K/hybrid/reranking experiments; retrieval versus generation failure attribution.
12. How groundedness, citation accuracy, abstention, and hallucination risk are measured without relying on “looks good.”
13. Dataset representativeness, judge bias/calibration, leakage/overfitting, slices, uncertainty, and release gates.
14. Prompt injection and untrusted retrieved content; deterministic guardrails and tool authorization.
15. Why the tool workflow is bounded/read-only and what controls would be required before write actions.
16. Trace propagation, safe AI observability, metric cardinality, SLO design, and diagnosing a latency spike.
17. Container image security, probes, graceful shutdown, build-once/promote, migrations, canary/blue-green, rollback.
18. Azure architecture, identity/network flow, scaling signals, dependency quotas, RTO/RPO, cost/reliability trade-offs.
19. A failure you intentionally introduced, the evidence used to find it, the fix, regression test, and operational lesson.
20. What you would improve next and which measurements—not trends—justify the priority.

# Evidence-Driven Extensions After the Core Roadmap

These topics appear in broader AI-engineering roadmaps, but they should follow—not displace—the production fundamentals above:

- **Fine-tuning/SFT, LoRA, DPO, or distillation:** first identify a stable behavior gap that retrieval, prompt, tools, or validation cannot solve; build training/evaluation splits; compare the tuned model with the existing baseline on quality, safety, latency, and cost. Learn the concepts even if the project evidence says not to deploy them.
- **Dedicated search/vector service:** migrate a shadow index and compare recall, ranking features, filtering, ingestion throughput, tail latency, cost, operational burden, and cutover/rollback against `pgvector`.
- **Write-capable remediation tools:** require isolated test environments, per-action authorization, parameter allowlists, dry-run/diff, explicit human approval, idempotency, compensating action, audit, and adversarial evaluation before any production action.
- **More complex agent graphs or multi-agent workflows:** add only when task decomposition or ownership boundaries create a measured benefit over the bounded state machine; include loop/cost budgets, shared-state consistency, traceability, and end-to-end evals.
- **AKS/Kubernetes or service extraction:** adopt when ACA/platform limits, organizational ownership, isolation, deployment cadence, or scaling evidence justify the operational cost.
- **Multi-region/high availability:** start from business RTO/RPO and provider/data constraints; design replication, traffic routing, failover, consistency, drills, and the additional cost instead of drawing a multi-region diagram without operating it.

For each extension, repeat the same rule: baseline → hypothesis → smallest experiment → evaluation → architecture decision → guarded rollout → operational proof.

# Definition of Done for the Entire Roadmap

The project is complete only when you can give a coherent, evidence-backed answer to all of these:

- **How it works:** trace the domain, API, data, queue, retrieval, model, validation, and response paths.
- **Why it is designed this way:** compare realistic alternatives through requirements, trade-offs, and ADRs.
- **How it fails:** demonstrate provider, retrieval, database, cache, broker, identity, deployment, and load failures.
- **How it is tested:** show deterministic tests, integration/API/security/load/failure tests, restore and rollback drills.
- **How AI quality is measured:** reproduce dataset, retrieval/answer/safety/operations metrics, calibration, and release decision.
- **How it is deployed:** rebuild Azure environments from IaC and promote an immutable image through gated CI/CD.
- **How it scales:** show stateless API and worker scaling, capacity constraints, backpressure, pools/quotas, and cost effects.
- **How it is monitored:** use one trace, dashboards, SLO alerts, and runbooks to diagnose a controlled incident.
- **How it would improve:** prioritize the next change from user feedback, eval slices, incidents, performance, security, or cost evidence.

If you can do those things without hiding behind a framework or tutorial—and can show the repository evidence—you have built not just an AI demo, but the practical foundation for the AI Engineer role described.
