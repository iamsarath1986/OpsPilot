# Milestone 3 — HTTP, REST, and the FastAPI Contract

**Goal:** OpsPilot stops being a CLI and becomes a stateless HTTP service with a versioned, documented, tested REST contract — while its state is still in memory.

**Why now:** the API contract is the product's public surface. Everything after this (auth, persistence, AI) is added *behind* a contract that clients already depend on. Designing the contract before the database also stops the classic mistake of exposing your table structure as your API.

**Time:** 3–4 focused weeks. **Prerequisite:** Milestone 2 checklist green.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| HTTP methods and their semantics | `GET` must be safe, `PUT` idempotent, `POST` neither. Clients, proxies, and retry logic all depend on this being right. |
| Status codes | The difference between 400, 401, 403, 404, 409, 422, 429, 500, and 503 *is* your error contract. Returning 500 for bad input makes every client's retry logic wrong. |
| Headers, content negotiation, caching | `Content-Type`, `Accept`, `ETag`, `Retry-After`, `Idempotency-Key` — the parts of HTTP that most tutorials skip and production depends on. |
| Connection lifecycle, keep-alive, TLS | Latency and connection-pool behaviour later depend on understanding this. |
| REST resource modelling | Whether analysis is `POST /incidents/{id}/analyses` or `POST /analyze?id=` determines how the API evolves for the next year. |
| Idempotency | Networks retry. Without an idempotency key, one user tap creates three incidents. |
| Pagination and filtering | An unbounded list endpoint is a latency and memory incident waiting for your first busy tenant. |
| API versioning and compatibility | You will change this contract in Milestone 6. Knowing what breaks a client is what lets you change it safely. |
| Consistent error format (RFC 9457) | Ad-hoc error shapes force every client to write custom parsing per endpoint. |
| OpenAPI | A machine-readable contract enables generated clients, contract tests, and honest documentation. |
| FastAPI routing, dependencies, lifespan | Dependency injection is how auth, DB sessions, and settings arrive at a handler without globals. |
| Pydantic v2 at the boundary | Types are compile-time hints; Pydantic is the runtime gate for untrusted input. You need both, for different reasons. |
| DTO versus domain separation | If your API model *is* your domain model, every API change becomes a domain change. |
| Async endpoints and ASGI | An `async def` endpoint calling blocking code still blocks the loop — a bug you will only see under concurrency. |
| Health versus readiness | Platforms need different answers to "is it alive?" and "should it get traffic?" You'll rely on this in Milestone 12. |
| Request IDs and correlation | Support asks "what happened to my request at 14:20?" You need to be able to answer. |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| HTTP semantics | **Design** | You pick a method and status from the contract's meaning, not from habit |
| REST modelling | **Design** | You can justify a subresource over a query flag, and say how each evolves |
| FastAPI + Pydantic | **Debug** | You can explain exactly why a request produced 422 and reshape the model to change it |
| Error contract | **Productionize** | Every failure path returns a consistent body with a request ID and no stack trace |
| Async endpoints | **Debug** | You can demonstrate a blocked endpoint under load and fix it |
| API evolution | **Understand** | You can classify a proposed change as additive or breaking |

## 3. Hands-On Learning Exercises

1. **See the protocol.** `curl -v` your endpoints. Vary `Accept`, send the wrong `Content-Type`, omit the body, send malformed JSON. Read the raw request and response, including headers. Do this before writing a single route.
2. **Model the resource two ways.** Write down `POST /v1/incidents/{id}/analyses` versus `POST /v1/analyze` with an ID in the body. For each, answer: how do you fetch a past analysis, how do you paginate them, how do you add a second analysis type? One of them answers cleanly.
3. **Make validation fight you.** Have Pydantic reject an oversized title, an unknown severity enum, a missing field, an extra field (`extra="forbid"`), a wrong type, and a string where a nested object belongs. Read each 422 body carefully — that shape is what your clients will see.
4. **Idempotency.** Fire the same `POST /v1/incidents` twice. Two incidents appear. Add an `Idempotency-Key` header, store the key with the result, and return the original on replay. Then reason about what happens when two identical requests arrive *simultaneously*.
5. **Block the server.** Add an endpoint with `time.sleep(3)` inside `async def`. Fire 10 concurrent requests with `hey` or `ab`. Observe total time ≈ 30 s. Change it to `def` (FastAPI runs it in a threadpool) and measure again. Explain both results.
6. **Break compatibility on purpose.** Rename a response field. Diff the OpenAPI document before and after. Then do it additively instead. Write down the rule you derived.
7. **Error mapping.** Take each domain exception from Milestone 1 and decide its status code. `IncidentNotFound` → 404. `InvalidIncidentState` → 409. `UnsupportedLogFormat` → 422. Argue each one.

## 4. Project Implementation

### Step 1 — Application shape

An app factory (`create_app(settings)`) plus a `lifespan` handler for startup and shutdown. No module-level global app configured by import side effects — that pattern breaks tests and multi-worker startup.

### Step 2 — The contract

| Endpoint | Purpose | Notes |
|---|---|---|
| `POST /v1/incidents` | Create an incident | Accepts `Idempotency-Key` |
| `GET /v1/incidents` | List for the caller's tenant | Paginated, capped page size |
| `GET /v1/incidents/{id}` | Fetch one | 404 when absent |
| `PATCH /v1/incidents/{id}` | Update status/severity | 409 on illegal transition |
| `POST /v1/incidents/{id}/artifacts` | Attach a pasted log excerpt | Size-capped; file upload arrives in Milestone 5 |
| `POST /v1/incidents/{id}/analyses` | Run deterministic rule analysis | Returns findings + evidence |
| `GET /v1/incidents/{id}/analyses` | List past analyses | Ordered, paginated |
| `GET /health/live`, `GET /health/ready` | Lifecycle signals | Different semantics — see below |

Tenant comes from a header placeholder for now (`X-Tenant-Id`) with a loud comment that it is **not** authentication. Milestone 5 replaces it with a verified token claim. Write the ADR entry for that now so the shortcut can't be forgotten.

### Step 3 — Boundary models

| Concept | Engineering problem | Implementation | Testing |
|---|---|---|---|
| Request/response DTOs | Domain changes should not break clients, and vice versa | Separate Pydantic models in `api/schemas/`, mapped explicitly to domain objects | A domain field rename must not change the API response |
| Strict validation | Untrusted input must never reach domain logic | `extra="forbid"`, length caps, enum types, bounded integers | Invalid, extra, oversized, boundary, and wrong-type cases |
| Error contract | Inconsistent errors make clients fragile | One exception handler mapping domain errors to a problem-details body: `type`, `title`, `status`, `detail`, `request_id` | Assert every domain error maps correctly, and that no traceback leaks |
| Pagination | Unbounded lists degrade unpredictably | `limit`/`offset` with a documented maximum, plus a total count | Empty, first, last, past-the-end, and invalid parameters |
| Request ID middleware | Support and debugging need a correlation handle | Read `X-Request-ID` or generate one; put it in logs and every error body | Assert the header round-trips and appears in logs |
| Health endpoints | The platform needs two different answers | `live` = process is up. `ready` = dependencies usable | Readiness must fail when a fake dependency is marked down |

### Step 4 — Wire the use cases

Routes call *use cases* from `application/`, which call ports. A route body should be short enough to read at a glance: parse, delegate, map, respond. If business rules appear in a route, the layering has leaked — and it will leak into your tests next.

### Step 5 — API tests

Use `httpx.ASGITransport` so tests exercise the real app without a network. Cover, for every endpoint: success, invalid input, not found, conflict, and dependency failure. Snapshot the OpenAPI document and fail CI when it changes unexpectedly — that turns accidental breaking changes into a review conversation.

## 5. Expected Project State

**Before:** a local CLI package.

**After:** a stateless HTTP service with a documented, versioned contract; strict boundary validation; a uniform error format with request IDs; pagination; and health/readiness endpoints. The CLI still exists and now calls the API — proving the API is genuinely the product surface.

**Architecturally:** state is still in memory. Restart loses everything, and two replicas would disagree with each other. Name that limitation out loud; it is exactly what Milestone 4 fixes, and being able to articulate *why* stateless services need external state is a system-design interview answer.

## 6. Testing and Validation

- API tests for every endpoint across success, validation failure, not found, conflict, and dependency-down cases.
- OpenAPI snapshot test to catch unintended contract changes.
- Contract tests from Milestone 1 still pass unchanged — the domain didn't move.
- A concurrency test: fire 20 simultaneous requests at an analysis endpoint and assert none blocks another.
- A baseline latency measurement for a deterministic endpoint, recorded with the hardware and conditions. Not a benchmark, a baseline you can compare against after Milestone 6 adds an LLM.
- Confirm no error response contains a stack trace, a file path, or a library name.

## 7. Failure Scenarios to Reproduce Deliberately

1. `Content-Type: text/plain` on a JSON endpoint — what status, and is the message useful?
2. Malformed JSON body, then a 50 MB body. Does the server reject it before buffering it all?
3. An unknown enum value and an unexpected extra field.
4. An unhandled domain exception reaching the client as a 500 with a traceback — then fix it with the global handler and confirm the leak is closed.
5. Client disconnects mid-request. Does the server keep working on a response nobody will read?
6. Duplicate `POST /v1/incidents` creating duplicates, before and after the idempotency key.
7. A blocking call inside an `async def` endpoint under 20 concurrent requests.
8. `ready` returning 200 while a required dependency is down — then fix it, and note that this exact bug sends live traffic to a broken replica in Milestone 12.

## 8. Skills Gained

- Designing and versioning a REST contract, and reasoning about compatibility
- Runtime validation of untrusted input, separate from static typing
- A consistent, safe error contract with correlation IDs
- FastAPI dependency injection, lifespan management, and async endpoint behaviour under concurrency
- API test design against a real ASGI app

**Job-description mapping:** this is direct, demonstrable evidence for *APIs and backend development* — the second explicit requirement in the role.

## 9. Milestone Completion Checklist

- [ ] I can explain safe versus idempotent methods and choose status codes from semantics.
- [ ] Every endpoint appears in OpenAPI with its error responses documented.
- [ ] Invalid input cannot reach domain logic — and I have a test proving it for each endpoint.
- [ ] Every error response has the same shape and carries a request ID; none leaks internals.
- [ ] I demonstrated a blocked async endpoint under concurrency and fixed it.
- [ ] Duplicate creation is handled, and I can explain what my idempotency implementation does *not* cover.
- [ ] Readiness fails when a dependency is down; liveness does not.
- [ ] I can name one alternative to REST here (GraphQL, RPC, or async job-submission) and why I rejected it for this product.
- [ ] **ADR-003** records the REST and versioning decision, and the temporary `X-Tenant-Id` shortcut with its removal date.

---

**Next:** Milestone 4 gives the service a memory — PostgreSQL, transactions, migrations, and the query behaviour that decides whether your API stays fast.
