# Milestone 10 — Background Jobs, Queues, Caching, and Bounded Tools

**Goal:** move expensive work off the request path onto durable queues with idempotent workers, replace in-process limits with distributed ones, add a real cache, and give the AI a small set of authorised **read-only** tools — the safe version of agent engineering.

**Why now:** ingesting a runbook set, embedding 5,000 chunks, or running a full evaluation cannot happen inside an HTTP request. And Milestone 12 will run multiple replicas, at which point every in-process limiter and cache from Milestone 7 becomes wrong. Fixing that now, locally, is far cheaper than discovering it in the cloud.

**Time:** 4–5 focused weeks. **Prerequisite:** Milestone 9 checklist green.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| Synchronous versus asynchronous work | A 4-minute ingestion job cannot hold an HTTP connection. Users need a job ID and a status endpoint instead. |
| Job lifecycle and status endpoints | `queued → running → succeeded / failed`, with progress and errors visible to the caller. |
| Message brokers and delivery semantics | At-least-once is what you actually get. Exactly-once is mostly a marketing claim, and designing as if you have it causes duplicate charges. |
| Idempotent consumers | Since messages *will* be redelivered, handlers must be safe to run twice. Without this, one duplicate embeds 5,000 chunks a second time and doubles the bill. |
| The dual-write problem | Committing to the database and publishing to a broker are two operations that can half-fail. |
| The outbox pattern | Write the message to the same transaction as the data, publish from there. The standard fix, and a strong interview topic. |
| Dead-letter queues | Poison messages must leave the queue without blocking everything behind them — and someone must be told. |
| Visibility timeout and redelivery | A message being processed must be invisible, or two workers will do the same work. |
| Backpressure and queue depth | Queue depth is your earliest warning signal, and later your autoscaling input. |
| Worker concurrency and prefetch | Too high starves memory and the database pool; too low wastes capacity. |
| Eventual consistency in the UX | "Your analysis is being prepared" is a product decision forced by architecture. |
| Cache-aside, TTL, and invalidation | The Milestone 7 cache moves to shared Redis and must survive multiple replicas. |
| Cache stampede | When a hot key expires, 100 concurrent requests all miss and all call the model at once. |
| Distributed rate limiting | Five replicas each allowing 100 requests per minute is a 500-per-minute limit you did not intend. |
| Distributed locks and their pitfalls | Useful and easy to get subtly wrong; know the failure modes before relying on one. |
| Cache as a *disposable* dependency | If Redis dies, the system must slow down, not fall over. |
| Tool/function calling design | Give the model capabilities through a typed, authorised, budgeted registry — not raw access. |
| Tool authorization and parameter validation | The model proposes; your code decides. A tool call is an untrusted request that must pass the same authorization as an HTTP request. |
| Loop and cost budgets for tool use | Without a bound, a model can loop until your quota is exhausted. |
| State machine versus autonomous agent | A bounded workflow is testable, explainable, and safe. Full autonomy is neither — and you should be able to say why you chose the former. |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Delivery semantics | **Design** | You assume at-least-once and design handlers accordingly |
| Idempotency | **Productionize** | Every handler is provably safe to re-run, with a test that runs it twice |
| Outbox | **Implement** | You have reproduced the dual-write failure and fixed it |
| DLQ operations | **Productionize** | You can inspect, fix, and replay a dead-lettered message |
| Cache | **Debug** | You have caused and fixed a stampede |
| Distributed limits | **Implement** | Limits hold correctly across two replicas |
| Tool calling | **Design** | You can explain each tool's authorization, budget, and blast radius |

## 3. Hands-On Learning Exercises

1. **The dual-write failure.** Commit a row, then crash before publishing the message. The work is lost with no error anywhere. Implement the outbox and reproduce the crash — the message survives. This experiment teaches more about distributed systems than any article.
2. **Duplicate delivery.** Deliver the same embedding message twice. Measure the doubled cost. Add an idempotency key. Deliver it twice again and confirm one execution.
3. **Poison message.** Send one that always throws. Watch it redeliver forever and block the queue. Add retry limits and a DLQ, then replay it after a fix.
4. **Queue lag.** Publish 10,000 messages, run one worker, and graph depth over time. Add a second worker. Derive your throughput per worker — the number you will use for autoscaling in Milestone 12.
5. **Cache stampede.** Expire a hot key, fire 100 concurrent requests, and count model calls. Add single-flight or a lock. Count again. Then compute what the difference would cost per month at real traffic.
6. **Distributed limit.** Run two API replicas with your Milestone 7 in-process limiter. Observe double the intended rate. Move to a Redis token bucket and re-measure.
7. **Kill the cache.** Stop Redis with the system under load. It must degrade, not fail. If it fails, you have made the cache a hard dependency by accident — a very common production outage.
8. **Runaway tool loop.** Let the model call tools without a budget and watch it loop. Add a call limit and a cost ceiling.
9. **Hostile tool arguments.** Have the model request `get_incident` with another tenant's ID. Your authorization layer must reject it exactly as it would reject an HTTP request.

## 4. Project Implementation

### Step 1 — Jobs and the outbox

| Concept | Engineering problem | Implementation | Testing |
|---|---|---|---|
| Job records | Callers need visibility into long work | `jobs` table: type, status, progress, error, tenant, correlation ID; `GET /v1/jobs/{id}` | Status transitions; a failed job reports a useful reason |
| Outbox | Dual-write loses or duplicates work | `outbox` table written in the same transaction; a publisher process drains it | Crash between commit and publish; message still delivered |
| Broker adapter | The broker changes in Milestone 12 | A port with a local implementation now, Azure Service Bus later | Contract tests both implementations must pass |
| Idempotent handlers | Redelivery is guaranteed, not exceptional | A processed-message table or natural idempotency keys | Handler executed twice produces one effect |
| DLQ and replay | Poison messages must not block the queue | Retry limit, DLQ, and an operator replay command | Poison message lands in DLQ; replay succeeds after fix |

### Step 2 — Move work to workers

Ingestion, chunking, embedding, re-embedding after a corpus change, and scheduled evaluation runs all become background jobs. The API returns `202 Accepted` with a job ID. Long-running diagnosis stays synchronous for now — it is seconds, not minutes — but it now runs *behind* the shared limits below.

### Step 3 — Shared Redis

Replace the Milestone 7 in-process cache and limiter with Redis-backed implementations behind the same ports. Add single-flight protection on hot keys. Crucially: wrap every Redis call so that a failure degrades to "cache miss" or "limit not enforced, logged loudly" rather than an error. Decide deliberately which of those two the limiter should do when Redis is down — fail open or fail closed — and write down why.

### Step 4 — The tool registry

Three read-only tools, each with a typed schema, an authorization check, and an audit record:

| Tool | Purpose | Blast radius |
|---|---|---|
| `get_incident(id)` | Fetch incident metadata | Read, tenant-scoped |
| `search_runbooks(query, filters)` | Targeted retrieval beyond the initial context | Read, visibility-scoped |
| `check_service_status(service)` | Synthetic or internal status endpoint | Read-only, no customer data |

Rules, all enforced in code rather than in the prompt: every tool call is authorised against the *caller's* principal, never the model's request; arguments are validated like any untrusted input; the loop is bounded by call count, wall time, and token cost; every call is recorded with arguments, result size, latency, and outcome.

**Write no tools that change anything.** Document what would be required before a write tool could exist: an isolated environment, per-action authorization, dry-run with a diff, explicit human approval, idempotency, a compensating action, and adversarial evaluation. Being able to list those requirements is more impressive than having built a tool that runs `chmod`.

### Step 5 — Orchestrate as a state machine, not a free-running agent

Keep the deterministic pipeline: retrieve → optionally call a bounded tool → generate → validate → decide the outcome. Every transition is testable and traceable. Record in the ADR what evidence would justify a more autonomous design — and note that "it would be more impressive" is not evidence.

## 5. Expected Project State

**Before:** everything happened inside the request, with per-process limits and cache.

**After:** long work runs on durable queues with idempotent workers, DLQ handling, and job status endpoints; the outbox guarantees no lost work; limits and cache are shared and correct across replicas; and the AI can call three authorised read-only tools inside a bounded loop.

**Architecturally:** OpsPilot became a distributed system — with eventual consistency, at-least-once delivery, and shared state as real design constraints rather than vocabulary. It is now shaped correctly to scale horizontally in Milestone 12.

## 6. Testing and Validation

- Idempotency tests: every handler executed twice, asserting a single effect.
- Outbox test with a simulated crash between commit and publish.
- DLQ test: poison message lands there, and replay works after a fix.
- Broker contract tests passing against both the local and (later) cloud adapter.
- Cache tests: hit, miss, stampede protection, and correct degradation when Redis is down.
- Distributed limit test across two replicas.
- Tool tests: schema validation, cross-tenant rejection, budget exhaustion, and audit records.
- A load test on the job pipeline measuring throughput per worker and queue lag.

## 7. Failure Scenarios to Reproduce Deliberately

1. Dual-write loss: committed data, unpublished message, silent gap.
2. Duplicate delivery doubling embedding cost — measure the money.
3. Poison message blocking the queue.
4. DLQ filling with nobody watching. (Add the alert in Milestone 11 and note the dependency now.)
5. Worker OOM on an unusually large document; find the bound that prevents it.
6. Redis down: cache degrades correctly, limiter behaves as designed.
7. Cache stampede after a hot key expires.
8. Two replicas each enforcing a local limit, permitting double the intended rate.
9. Worker concurrency set too high, exhausting the database connection pool — the Milestone 4 failure, now arriving from a new direction.
10. Tool loop without a budget, burning quota.
11. Tool called with another tenant's incident ID.
12. A job that succeeded but whose status was never updated, leaving a caller polling forever.

## 8. Skills Gained

- Durable background processing: queues, workers, idempotency, outbox, DLQ, and replay
- Reasoning about at-least-once delivery and eventual consistency as design constraints
- Distributed caching and rate limiting that stay correct across replicas
- Treating the cache as disposable, and proving it
- Bounded, authorised, audited tool calling — agent engineering with a defensible safety posture
- Choosing a state machine over autonomy and defending the choice with reasons

**Job-description mapping:** *cloud-native solutions* and *backend systems*. Queues, workers, and idempotency are the vocabulary of distributed-systems interviews, and this is also where you can speak credibly about agents without hand-waving.

## 9. Milestone Completion Checklist

- [ ] I can explain at-least-once delivery and why exactly-once is not something I design around.
- [ ] Every handler is idempotent, with a test that runs it twice.
- [ ] I reproduced the dual-write failure and fixed it with an outbox.
- [ ] I have sent a message to the DLQ and replayed it after a fix.
- [ ] I know my throughput per worker and my queue-lag behaviour under burst.
- [ ] My cache is genuinely disposable — I killed Redis under load and the system degraded.
- [ ] I caused a cache stampede and fixed it, and can state the cost saved.
- [ ] Rate limits hold correctly across two replicas, and I can explain fail-open versus fail-closed for mine.
- [ ] Every tool call is authorised against the caller, argument-validated, budgeted, and audited.
- [ ] A tool call for another tenant's data is rejected — with a test.
- [ ] I can list what would be required before adding a *write* tool, and why I have not.
- [ ] **ADR-010** records the queue, cache, and bounded-tool decisions.

---

**Next:** Milestone 11 makes all of this visible — tracing, metrics, SLOs, load testing, hardened containers, and a CI/CD pipeline that builds one artifact and promotes it.
