# Milestone 7 — Reliable AI Orchestration: Structured Output, Validation, Retries, Fallbacks

**Goal:** turn an unreliable model call into a dependable product capability. Structured output validated in four distinct layers, deadlines and bounded retries, a circuit breaker, per-tenant limits, a cache, and an explicit outcome state — `complete`, `degraded`, `abstained`, or `escalated` — that never lies to the caller.

**Why now:** Milestone 6 produced answers that are sometimes malformed, sometimes contradicted by your own parser, and sometimes absent because the provider was throttling. This is the milestone that most separates AI engineers from people who call an API — and it is the reliability layer the job description's "real-world AI products" phrase is actually pointing at.

**Time:** 4–5 focused weeks. **Prerequisite:** Milestone 6 checklist green, with the baseline set recorded.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| Structured output modes (JSON schema, tool/function calling, `response_format`) | Free text cannot drive a UI, a workflow, or a database column. |
| Why structured output is necessary but not sufficient | Schema-valid JSON can still be factually wrong, internally inconsistent, or cite evidence that does not exist. This distinction is the heart of the milestone. |
| The four validation layers | **Syntax** (is it JSON), **schema** (right fields and types), **business** (values make sense together), **evidence** (claims trace to real, authorised source lines). |
| Pydantic validators for cross-field rules | "Confidence high with zero evidence" is schema-valid and semantically nonsense. |
| Prompt versioning and a prompt registry | A prompt edit is a production change. Untracked, it produces invisible regressions you cannot diagnose. |
| Overall deadlines versus per-attempt timeouts | Three retries at ten seconds each is a thirty-second request. The user's budget, not the attempt's, is what matters. |
| Transient versus permanent errors | Retrying a 400 wastes money and time; not retrying a 429 wastes availability. |
| Exponential backoff with jitter | Synchronised retries from many clients turn a blip into an outage. Jitter is the fix, and it is one line. |
| Retry budgets and storms | Retries multiply load exactly when a dependency is weakest. |
| `Retry-After` | The provider is telling you when to come back. Ignoring it is how you get rate-limited harder. |
| Circuit breakers, and half-open recovery | Stop hammering a dead dependency; probe carefully before declaring it healthy. |
| Bulkheads and concurrency limits | One tenant's burst must not consume the entire provider quota. |
| Token-bucket rate limiting per tenant | Fairness and cost control, per customer rather than globally. |
| Idempotency for expensive calls | A retried request should not pay for a second inference. |
| Cache design and key construction | A cache key that omits tenant or prompt version serves the wrong answer to the wrong customer. This is a security bug, not a performance bug. |
| Fallback ladders and graceful degradation | The alternative to a good answer is an honest lesser answer, not a 500. |
| Abstention as a feature | "I don't have enough evidence" is often the most valuable output in an ops context. |
| SLIs, SLOs, and error budgets | Turns "it should be reliable" into a number you can design against. |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Structured output | **Productionize** | Provider-side schema enforcement *and* your own validation, because you do not trust either alone |
| Validation layers | **Design** | You can name which layer catches which real failure, with an example of each |
| Retry policy | **Design** | You can state which errors retry, how many times, with what delays, inside what deadline — and why |
| Circuit breaker | **Implement** | You have tripped one and watched half-open recovery |
| Rate limiting | **Implement** | Per-tenant limits enforced, with correct `429` and `Retry-After` responses |
| Caching | **Debug** | You can explain every component of your cache key and what breaks if one is missing |
| Fallbacks | **Design** | Every path terminates in a defined outcome state, provably without loops |
| SLOs | **Understand** | You have written an SLI and target for the diagnosis path |

## 3. Hands-On Learning Exercises

1. **Break the output five ways.** Force the model to return: prose instead of JSON; JSON missing a required field; an invalid enum; confidence `1.7`; and — the important one — perfectly valid JSON citing `ev_99` when only `ev_1`–`ev_8` exist. Note that only the last requires *your* evidence layer. Schema validation is blind to it.
2. **Retry classification.** Take the error list from Milestone 6 and sort each into retry or do-not-retry. Now implement it and assert attempt counts per error type.
3. **The jitter demonstration.** Simulate 50 clients retrying after a shared failure, with fixed delays and then with full jitter. Plot or tabulate request arrivals per second. The synchronised spike is the reason jitter exists — see it once and you will never omit it.
4. **Trip a breaker.** Fail a fake provider 100% of the time. Watch the breaker open, requests fail fast, then a probe succeed and traffic resume. Measure how much latency and cost the open state saved you.
5. **Deadline arithmetic.** Set a 10 s overall deadline, a 4 s per-attempt timeout, and 3 retries. Work out on paper what happens; then run it and check your prediction. Fix whichever was wrong.
6. **Fallback comparison.** Run the baseline 30 cases through four strategies: primary only; primary then a smaller/cheaper model; primary then deterministic-rules-only output; primary then abstain. Record quality, latency, and cost for each. Choose from the table, not from intuition.
7. **Cache key attack.** Deliberately build a cache key without `tenant_id`. Fetch tenant A's answer as tenant B. Fix it, then add the test that would have caught it. Do the same for prompt version.
8. **Write an SLO.** "99% of diagnosis requests return a terminal state within 15 seconds." Measure your current number. Decide whether the gap is a bug or a target.

## 4. Project Implementation

### Step 1 — The structured contract

```python
class Diagnosis(BaseModel):
    severity: Severity
    summary: str
    likely_causes: list[Cause]        # each with confidence + evidence_ids
    recommended_steps: list[str]
    evidence_ids: list[str]
    confidence: Confidence            # low | medium | high
    needs_escalation: bool
    unsupported: bool = False
```

Request it with provider-side schema enforcement where available — and validate it yourself regardless. Provider enforcement can change, degrade, or be unavailable on a fallback model.

### Step 2 — The validation pipeline

| Layer | Catches | Implementation | Testing |
|---|---|---|---|
| Syntax | Prose, truncated JSON, markdown fences | Parse; on failure, one bounded re-ask with the parse error appended | Malformed-output fixtures |
| Schema | Missing fields, wrong types, unknown enums | Pydantic model validation | One fixture per violation |
| Business | Contradictions: high confidence with no evidence; `needs_escalation` false on a critical severity; steps that contradict findings | Pydantic model validators plus domain rules | One test per rule |
| Evidence | **Hallucinated citations** — IDs not in the prompt, or belonging to another tenant | Check every ID against the evidence set actually supplied | Fabricated and cross-tenant ID fixtures |

Anything failing all repair attempts becomes `abstained` with a reason. Never return an unvalidated answer, and never silently drop the offending field.

### Step 3 — The resilience policy

One policy object wrapping every provider call: overall deadline, per-attempt timeout, retry classification, exponential backoff with full jitter, `Retry-After` honouring, retry budget, and a circuit breaker per provider. Configuration lives in settings, not scattered literals.

### Step 4 — Admission control

Per-tenant token-bucket limits on requests and on tokens, plus a global concurrency bulkhead. In-process for now; Milestone 10 replaces it with a distributed limiter once there are multiple replicas — and knowing *why* that swap is necessary is the lesson.

### Step 5 — Caching

Cache-aside on the result, keyed by tenant, normalised input hash, prompt version, model, and (from Milestone 8) corpus version. Short TTL. Every one of those key components prevents a specific incorrect result; write a comment saying which.

### Step 6 — The outcome state machine

Every request terminates in exactly one state, returned explicitly to the caller:

- `complete` — validated answer from the primary path
- `degraded` — answer from a fallback, with the reason stated
- `abstained` — insufficient evidence or repeated validation failure
- `escalated` — routed to a human, with a packet of what was known

Silent model switching is dishonest engineering: the caller must be able to tell a first-class answer from a fallback. Prove the machine has no loops with a test that exhausts every transition.

### Step 7 — The prompt registry

Immutable, versioned prompt artifacts in `prompts/`, with a changelog. The version is stored with every diagnosis. Requirement to test: given a stored diagnosis row, you can reconstruct exactly which prompt, model, and settings produced it.

## 5. Expected Project State

**Before:** an experimental endpoint returning free text that might be wrong, malformed, or missing.

**After:** a validated, typed diagnosis with citations checked against real evidence; provider faults absorbed by deadlines, bounded retries, and a breaker; per-tenant limits and a cache controlling cost; and an explicit, honest outcome state on every response.

**Architecturally:** the model is now one step inside a controlled pipeline rather than the pipeline itself. This is the diagram from `production_ai_engineering_beyond_prompts.md` made real — and the point at which the AI feature stops being a demo.

## 6. Testing and Validation

- One focused test per validation layer, plus combined cases where several fail at once.
- Deterministic fault injection: assert exact attempt counts, backoff bounds, and honoured `Retry-After`.
- Breaker tests: open, fail-fast, half-open, close.
- Rate-limit tests: at the boundary, over it, burst, fairness across tenants, and correct `Retry-After` headers.
- Cache tests: hit, miss, expiry, and — most importantly — **no cross-tenant and no cross-version hits**.
- State-machine tests exhausting every transition and proving termination.
- Re-run the Milestone 6 baseline set and record how many previously unusable outputs are now `complete`, `degraded`, or honestly `abstained`. That table is a portfolio artifact.

## 7. Failure Scenarios to Reproduce Deliberately

1. Model returns markdown-fenced JSON — a classic that breaks naive parsers.
2. Valid JSON citing `ev_99`, which does not exist. Confirm the evidence layer catches what schema validation cannot.
3. A citation referencing another tenant's document. This must fail closed.
4. Retry storm: remove jitter, fail the provider, watch synchronised spikes.
5. Deadline exceeded mid-retry — confirm the user gets a terminal state, not a hang.
6. Breaker stuck open after the provider recovers (a half-open bug) — users see degradation long after the incident ended.
7. Fallback loop: primary fails to secondary, which fails back to primary. Prove your machine cannot do this.
8. Cache poisoning: a stale answer served after the prompt version changed.
9. One tenant consuming the entire provider quota, starving everyone else — before and after bulkheads.
10. Provider returns a 200 with an empty body.

## 8. Skills Gained

- Structured outputs with layered validation, including grounding checks that schemas cannot perform
- Retry, timeout, backoff, jitter, circuit-breaking, bulkhead, and rate-limiting policies — the standard distributed-systems toolkit, applied to AI
- Honest degradation and abstention as designed product behaviour
- Prompt and model versioning as release artifacts
- Cache design where a key omission is a security bug
- Writing an SLI and SLO for an AI path

**Job-description mapping:** this is the strongest single milestone for *real-world AI products*. Anyone can call an LLM; the differentiator is a candidate who can describe what happens when the provider throttles at 3 a.m. and can point at the code that handles it.

## 9. Milestone Completion Checklist

- [ ] I can name the four validation layers and give a real failure each one uniquely catches.
- [ ] Hallucinated evidence IDs are rejected, and I have a test with a fabricated citation.
- [ ] I can state my retry policy — which errors, how many attempts, what delays, inside what deadline — and justify each part.
- [ ] I demonstrated a retry storm without jitter and fixed it.
- [ ] I tripped a circuit breaker and observed half-open recovery.
- [ ] Every request ends in exactly one of four outcome states, and the caller can tell them apart.
- [ ] My cache key includes tenant, prompt version, and model — and I have a test for each omission.
- [ ] Per-tenant limits work and return correct `429` with `Retry-After`.
- [ ] Given a stored diagnosis, I can reconstruct the exact prompt and model that produced it.
- [ ] I compared four fallback strategies on quality, latency, and cost with real numbers.
- [ ] I have written one SLI and SLO for the diagnosis path and measured current performance against it.
- [ ] **ADR-007** records the reliability policy and the degradation ladder.

---

**Next:** Milestone 8 gives the model something authoritative to reason from — your own runbooks, chunked, embedded, retrieved, ranked, and cited.
