# Milestone 7 — Reliable AI Orchestration: Structured Output, Validation, Retries, Fallbacks

**Goal:** turn an unreliable model call into a dependable product capability. Structured output validated in four distinct layers, deadlines and bounded retries, a circuit breaker, per-tenant limits, a cache, and an explicit outcome state — `complete`, `degraded`, `abstained`, or `escalated` — that never lies to the caller.

**Why now:** Milestone 6 produced answers that are sometimes malformed, sometimes contradicted by your own parser, and sometimes absent because the self-hosted inference server was saturated. This is the milestone that turns a model demo into a reliable internal capability.

**Time:** 4–5 focused weeks. **Prerequisite:** Milestone 6 checklist green, with the bootstrap set recorded.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| Serving-layer constrained decoding (JSON schema or grammar) | Free text cannot drive a UI, a workflow, or a database column; the self-hosted sampler must enforce valid structure. |
| Why structured output is necessary but not sufficient | Schema-valid JSON can still be factually wrong, internally inconsistent, or cite evidence that does not exist. This distinction is the heart of the milestone. |
| Validation in two decoding configurations | **Constrained primary:** syntax and schema failures are unreachable invariants. **Unconstrained fallback/test:** syntax and schema remain defensive boundaries. Business and evidence failures remain reachable in both. |
| Pydantic validators for cross-field rules | "Confidence high with zero evidence" is schema-valid and semantically nonsense. |
| Vacuity under constrained decoding | A decoder can force the model to fill every field without making the content useful. Shape-valid empty, circular, or boilerplate answers must fail business validation or take the explicit unsupported path. |
| Prompt versioning and a prompt registry | A prompt edit is a production change. Untracked, it produces invisible regressions you cannot diagnose. |
| Overall deadlines versus per-attempt timeouts | Three retries at ten seconds each is a thirty-second request. The user's budget, not the attempt's, is what matters. |
| Retry, repair, and fallback classification | A transport blip may retry; queue timeout and OOM must not; context overflow requires rebuilding a smaller request rather than replaying the same one. |
| Exponential backoff with jitter | Synchronised retries from many clients turn a blip into an outage. Jitter is the fix, and it is one line. |
| Retry budgets and storms | Retries multiply load exactly when a dependency is weakest. |
| `Retry-After` | The inference or admission-control service is telling you when capacity may be available. Ignoring it deepens the queue. |
| Circuit breakers, and half-open recovery | Stop hammering a dead dependency; probe carefully before declaring it healthy. |
| Bulkheads and concurrency limits | One tenant's burst must not consume the entire GPU concurrency and KV-cache budget. |
| Token-bucket rate limiting per tenant | Fairness and capacity allocation, per customer rather than globally. |
| Idempotency for expensive calls | A retried request should not pay for a second inference. |
| Cache design and key construction | A cache key that omits tenant or prompt version serves the wrong answer to the wrong customer. This is a security bug, not a performance bug. |
| Fallback ladders and graceful degradation | The alternative to a good answer is an honest lesser answer, not a 500. |
| Abstention as a feature | "I don't have enough evidence" is often the most valuable output in an ops context. |
| SLIs, SLOs, and error budgets | Turns "it should be reliable" into a number you can design against. |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Structured output | **Productionize** | The constrained path makes syntax/schema failure unreachable, the unconstrained fallback still fails closed, and both support honest abstention |
| Validation layers | **Design** | You can state which failures are unreachable in each decoding configuration and which remain live |
| Retry policy | **Design** | You can state which errors retry, how many times, with what delays, inside what deadline — and why |
| Circuit breaker | **Implement** | You have tripped one and watched half-open recovery |
| Rate limiting | **Implement** | Per-tenant limits enforced, with correct `429` and `Retry-After` responses |
| Caching | **Debug** | You can explain every component of your cache key and what breaks if one is missing |
| Fallbacks | **Design** | Every path terminates in a defined outcome state, provably without loops |
| SLOs | **Understand** | You have written an SLI and target for the diagnosis path |

## 3. Hands-On Learning Exercises

1. **Run both decoding configurations.** With constrained decoding deliberately disabled on a fake or test-only endpoint, force prose, a missing field, an invalid enum, and confidence `1.7`; prove syntax/schema validation fails closed. Turn constraints on and prove those four fixtures cannot be generated under the schema. In both configurations, return valid JSON citing `ev_99`; prove the evidence layer still rejects it.
2. **Attack with vacuity.** Under constrained decoding, produce `unsupported: false` with empty causes, steps, and evidence, then produce syntactically filled but blank or boilerplate strings. Business validation must reject both. Next produce `unsupported: true` with empty arrays and a typed reason; it must be a valid document that terminates in abstention.
3. **Retry classification.** Take the self-hosted error list from Milestone 6 and encode the table in Step 3. Assert exact attempt counts: queue timeout and OOM get one attempt; context overflow gets one bounded context repair, not a replay; only classified transient failures retry.
4. **The jitter demonstration.** Simulate 50 clients retrying after a shared failure, with fixed delays and then with full jitter. Plot or tabulate request arrivals per second. The synchronised spike is the reason jitter exists — see it once and you will never omit it.
5. **Trip a breaker.** Fail a fake inference adapter 100% of the time. Watch the breaker open, requests fail fast, then a probe succeed and traffic resume. Measure how much latency and wasted inference capacity the open state saved.
6. **Deadline arithmetic.** Set a 10 s overall deadline, a 4 s per-attempt timeout, and 3 retries. Work out on paper what happens; then run it and check your prediction. Fix whichever was wrong.
7. **Fallback comparison.** Run the M6 bootstrap set through four strategies: primary only; primary then a smaller/faster local model; primary then deterministic-rules-only output; primary then abstain. Record quality, latency, inference capacity, and — for the model fallback — resident VRAM, cold-load time, and the effect on the primary model's KV-cache headroom. Choose from the table, not from intuition.
8. **Cache key attacks.** Deliberately omit, one at a time, `tenant_id`, prompt version, quantisation, and decoding/schema version. Prove a cross-tenant leak for the first and stale or incompatible hits for the others; then make every omission fail a test.
9. **Size admission control from evidence.** Use the M6 concurrency curve to choose the global in-flight limit at the knee before queue latency collapses. Add a bounded queue and maximum wait, exceed each deliberately, and confirm fast degradation rather than an ever-growing queue.
10. **Write an SLO.** "99% of diagnosis requests return a terminal state within 15 seconds." Measure your current number. Decide whether the gap is a bug or a target.

## 4. Project Implementation

### Step 1 — The structured contract

```python
class Diagnosis(BaseModel):
    severity: Severity | None
    summary: str
    likely_causes: list[Cause]
    recommended_steps: list[str]
    evidence_ids: list[str]
    confidence: Confidence | None
    needs_escalation: bool
    unsupported: bool
    unsupported_reason: UnsupportedReason | None
```

Pass the schema to the self-hosted serving layer for schema-guided decoding; use grammar-constrained decoding where that server lacks native schema support. Validate the result yourself regardless. Constrained decoding guarantees shape, not business correctness, usefulness, or evidential support.

The contract must make abstention representable. When `unsupported` is `true`, empty cause, step, and evidence arrays are valid; `severity` and `confidence` are null; `needs_escalation` is true; and a typed `unsupported_reason` is required (`insufficient_evidence`, `conflicting_evidence`, `out_of_scope`, or `safety_policy`). When `unsupported` is `false`, severity and confidence are required, the answer must contain non-vacuous supported content, and `unsupported_reason` is null. Enforce those cross-field rules in Pydantic and test the emitted JSON Schema against the actual serving stack.

### Step 2 — The validation pipeline

| Layer | Constrained primary path | Unconstrained fallback/test path | Implementation and testing |
|---|---|---|---|
| Syntax | **Unreachable invariant** if the server honoured the decoding contract | Live: prose, truncated JSON, markdown fences | Always parse. Constrained-path syntax failure is a serving/configuration fault and opens degradation; malformed fallback output gets one bounded repair, then abstains. |
| Schema | **Unreachable invariant** for fields, types, and enums expressible in the serving schema | Live: missing fields, wrong types, unknown enums | Always run Pydantic. Contract tests prove the server uses the exact schema version; one fallback fixture per violation. |
| Business | Live: shape-valid contradiction or vacuity | Live | Cross-field/domain rules reject high confidence without evidence, unsafe contradictions, and `unsupported: false` with empty, blank, or boilerplate content. They explicitly accept the typed unsupported document. |
| Evidence | Live: fabricated or unauthorised citations | Live | Check every ID against the exact authorised evidence set supplied; fabricated and cross-tenant fixtures fail closed. |

Anything failing its allowed repair becomes `abstained` or `degraded` with a typed reason. Never return an unvalidated answer, never silently drop the offending field, and never treat an unreachable syntax/schema failure on the constrained path as ordinary model variance.

### Step 3 — The resilience policy

One policy object wraps every inference call: overall deadline, per-attempt timeout, explicit classification, exponential backoff with full jitter, bounded `Retry-After` honouring, retry budget, and a circuit breaker per serving endpoint. Configuration lives in settings, not scattered literals.

| Failure | Action | Why |
|---|---|---|
| Connection reset before acceptance; transient healthy-server 5xx | Retry within the remaining deadline and retry budget | A later attempt may succeed without amplifying a capacity incident |
| Admission `429` with `Retry-After` | Retry only if the delay fits the remaining deadline and the bounded queue is below its limit; otherwise degrade | The server supplied a recovery signal, but waiting remains bounded |
| Queue-wait timeout | **Do not retry**; degrade or abstain | Re-enqueueing adds load to the already saturated resource |
| GPU OOM / KV-cache allocation failure | **Do not retry**; enter capacity degradation and alert | The identical request makes the same exhausted resource worse |
| Context-length exceeded | Rebuild once with a smaller deterministic context, under the original deadline; do not replay unchanged | This is request repair, not a transient retry |
| Authentication, licence, schema-contract mismatch, other permanent 4xx | **Do not retry**; fail closed and alert | Operator action is required |
| Constraint feature unavailable, with an explicitly enabled unconstrained fallback | Take the fallback once, validate all four layers, and mark `degraded: decoder_constraint_unavailable` | The caller and telemetry must expose the weaker path |
| Unconstrained syntax/schema failure | One bounded output repair, then abstain | Validation repair is tracked separately from infrastructure retries |

### Step 4 — Admission control

Per-tenant token-bucket limits protect fairness, but the global bulkhead and bounded queue protect the inference server. Set the in-flight limit from the knee of M6's measured concurrency curve, then define a maximum queue depth and queue-wait time; reject or degrade before latency enters the collapse region. Record the measurement and selected numbers in ADR-007, tagged with the exact configuration that produced them: GPU model and VRAM, model revision, quantisation, serving stack and version, and maximum context length. The controls are in-process for now; Milestone 10 makes them distributed without changing those limits — **for the same tagged configuration.**

A limit is only valid for the configuration it was measured on. If any tagged component changes — a new GPU, a different quantisation, a serving-stack upgrade, a longer context window, or the move to production hardware in Milestone 12 — re-run the Milestone 6 concurrency curve and re-derive the bulkhead and queue bounds before relying on them. Store the tag alongside the configured limits and fail startup when it does not match the running inference server's reported configuration, so a stale limit cannot be deployed silently.

### Step 5 — Caching

Cache-aside on the result, keyed by tenant, normalised input hash, prompt version, full model identity (name + revision), quantisation, decoding mode and configuration, output-schema version, generation settings, and — from Milestone 8 — corpus and retrieval configuration versions. Short TTL. Every component prevents a specific incorrect result; centralise key construction and make an omission fail a test.

### Step 6 — The outcome state machine

Every request terminates in exactly one state, returned explicitly to the caller:

- `complete` — validated answer from the primary path
- `degraded` — validated answer from a fallback, with a required typed reason and source
- `abstained` — insufficient evidence or repeated validation failure
- `escalated` — routed to a human, with a packet of what was known

Two fields describe every result, and they answer different questions — keep them separate:

- **`degradation_reason` — *why* the primary path did not produce the answer.** Present only on `degraded`. Initial values: `capacity_saturated`, `primary_unavailable`, `circuit_open`, `deadline_exhausted`, `decoder_constraint_unavailable`, and `primary_validation_failed` (the primary output failed business or evidence validation after its allowed repair, *and* a fallback then produced a validated answer). A value that names the fallback rather than the cause, such as "fallback model used", does not belong here: that is what `output_source` records.
- **`output_source` — *what* produced the answer.** Present on **every** result, so distributions sum to 100% and a shift between sources is measurable. Values: `primary_constrained`, `fallback_model`, `deterministic_rules`, `unconstrained_validated`, and `none`.

The permitted combinations are closed; enforce them in the domain model and test every row:

| Outcome | `output_source` | `degradation_reason` |
|---|---|---|
| `complete` | `primary_constrained` | must be null |
| `degraded` | `fallback_model` or `deterministic_rules` | any reason except `decoder_constraint_unavailable` |
| `degraded` | `unconstrained_validated` | `decoder_constraint_unavailable` only |
| `abstained` | `none` | must be null — the typed abstention reason lives in `unsupported_reason` or the abstention record |
| `escalated` | `none` | must be null |

A validation failure with no fallback left is therefore **abstained**, never degraded: "one bounded output repair, then abstain" in the Step 3 table produces `abstained` with `output_source: none`. Silent model or decoder switching is dishonest engineering: prove every combination in the table is reachable, every combination outside it is rejected, and the machine has no loops.

A fallback model is not a free rung. Record one explicit residency decision in ADR-007:

| Residency | Measure before choosing it | Consequence |
|---|---|---|
| Co-resident on the primary GPU | Fallback VRAM plus the new primary concurrency/KV-cache curve | Faster failover, but the reserved weights reduce the bulkhead that protects the primary |
| Loaded on demand on the primary GPU | Model unload/load time, memory fragmentation, and total fallback latency | Usually too slow for an in-request fallback; use only if the measured deadline still holds |
| Resident on separate approved hardware | End-to-end latency, capacity, utilisation, and operational cost | Preserves primary headroom but adds another service to deploy and operate |

If none fits the deadline and capacity budget, remove the fallback-model rung and use deterministic rules or abstention. Whenever residency changes, rerun the concurrency curve and update the Step 4 bulkhead rather than assuming the old limit survives.

### Step 7 — The prompt registry

Immutable, versioned prompt artifacts in `prompts/`, with a changelog. The version is stored with every diagnosis. Requirement to test: given a stored diagnosis row, you can reconstruct exactly which prompt, model, and settings produced it.

### Step 8 — The demo for operations

This is the demo the rest of the roadmap refers to. Schedule it for the final week of this milestone, once Steps 1–7 are working — not before. A Milestone 6 demo shows answers that are sometimes malformed; this one shows a system that validates, degrades honestly, and abstains when it should, and that is what earns trust.

- **Audience:** the operations manager and one or two first-level support engineers. 30 minutes.
- **Content:** only authored cases — preferably reproduced incidents, whose root cause is known. Show one `complete` diagnosis with cited evidence, one `degraded` result with its reason and source, one honest `abstained` case, and the rejection of a fabricated citation. Show the rule-baseline comparison from Milestone 6, including where rules still win.
- **Claims:** state explicitly that results come from cases we authored and reproduced, not from real tickets, and that runbooks marked `unreviewed` describe our procedure, not company policy. Do not demo a live model call on customer data.
- **The ask: review, not data.** Ask ops to check three or four of our runbooks ("is this how we handle it?"), to sanity-check a few `scenario` cases, to describe from memory a couple of recent cases that could become new scenarios, and to say which ticket categories matter most. Each costs minutes and involves no customer data.
- **Record:** the date, attendees, feedback, and which runbooks and scenarios were reviewed, in `02-discovery-with-operations.md`. Update the review status of each reviewed runbook.

Nothing later in the roadmap depends on this demo happening. If it cannot be scheduled, Milestones 8 and 9 proceed on unreviewed runbooks and authored cases, and the reports say so.

## 5. Expected Project State

**Before:** an experimental endpoint returning free text that might be wrong, malformed, or missing.

**After:** a validated, typed diagnosis with citations checked against real evidence; inference-server faults absorbed by deadlines, bounded retries, and a breaker; per-tenant limits and a cache controlling scarce capacity; and an explicit, honest outcome state on every response.

**Architecturally:** the model is now one step inside a controlled pipeline rather than the pipeline itself. This is the diagram from `production_ai_engineering_beyond_prompts.md` made real — and the point at which the AI feature stops being a demo.

## 6. Testing and Validation

- Two-configuration validation tests: syntax/schema are unreachable under constrained decoding, remain fail-closed boundaries when unconstrained, and business/evidence failures remain reachable in both.
- Contract tests proving `unsupported: true` with empty arrays is schema-valid and terminates in abstention; `unsupported: false` with vacuous content is rejected.
- Deterministic fault injection: assert exact attempt counts, backoff bounds, and honoured `Retry-After`.
- Breaker tests: open, fail-fast, half-open, close.
- Rate-limit tests: at the boundary, over it, burst, fairness across tenants, and correct `Retry-After` headers.
- Cache tests: hit, miss, expiry, and — most importantly — **no cross-tenant, cross-model-revision, cross-quantisation, cross-decoder, or cross-schema hits**.
- State-machine tests exhausting every transition and every degradation reason/source combination, proving termination.
- Re-run the Milestone 6 bootstrap set and record how many previously unusable outputs are now `complete`, `degraded`, or honestly `abstained`. That table is a portfolio artifact.

## 7. Failure Scenarios to Reproduce Deliberately

1. Test-only unconstrained path returns markdown-fenced JSON; validation fails closed. The constrained path cannot reproduce it unless its serving contract is broken.
2. Valid JSON citing `ev_99`, which does not exist. Confirm the evidence layer catches what schema validation cannot.
3. A citation referencing another tenant's document. This must fail closed.
4. Retry storm: remove jitter, fail or saturate the inference server, and watch synchronised spikes.
5. Deadline exceeded mid-retry — confirm the user gets a terminal state, not a hang.
6. Breaker stuck open after the inference server recovers (a half-open bug) — users see degradation long after the incident ended.
7. Fallback loop: primary fails to secondary, which fails back to primary. Prove your machine cannot do this.
8. Cache poisoning: stale answers served after prompt, quantisation, decoding configuration, or schema version changed.
9. One tenant consuming the entire GPU concurrency/KV-cache budget, starving everyone else — before and after the capacity-derived bulkhead and bounded queue.
10. Inference server returns a 200 with an empty body.
11. Queue-wait timeout and GPU OOM accidentally retried, worsening saturation; assert exactly one attempt for each.
12. Context-length failure replayed unchanged; replace it with one deterministic context repair.
13. Constrained decoding fills a supported-shaped document with empty or boilerplate content; business validation rejects the vacuity.
14. Unsupported output cannot satisfy the schema; fix the contract so typed abstention with empty arrays is valid.

## 8. Skills Gained

- Structured outputs with layered validation, including grounding checks that schemas cannot perform
- Retry, timeout, backoff, jitter, circuit-breaking, bulkhead, and rate-limiting policies — the standard distributed-systems toolkit, applied to AI
- Honest degradation and abstention as designed product behaviour
- Prompt and model versioning as release artifacts
- Cache design where a key omission is a security bug
- Writing an SLI and SLO for an AI path

**Job-description mapping:** this is the strongest single milestone for *real-world AI products*. The differentiator is a candidate who can describe what happens when the inference fleet saturates at 3 a.m. and can point at the code that handles it.

## 9. Milestone Completion Checklist

- [ ] I can state which validation failures are unreachable under constrained decoding, which remain reachable, and how the unconstrained fallback fails closed.
- [ ] `unsupported: true` with empty arrays is schema-valid and abstains; supported but vacuous output is rejected.
- [ ] Hallucinated evidence IDs are rejected, and I have a test with a fabricated citation.
- [ ] I can state my retry policy — which errors, how many attempts, what delays, inside what deadline — and justify each part.
- [ ] I demonstrated a retry storm without jitter and fixed it.
- [ ] I tripped a circuit breaker and observed half-open recovery.
- [ ] Every request ends in exactly one of four outcome states; every result has an `output_source`, every degraded result has a typed reason, and only the permitted state/source/reason combinations can be constructed.
- [ ] Queue timeout and GPU OOM never retry; context overflow takes one bounded repair rather than an unchanged replay.
- [ ] My global bulkhead and queue bounds come from M6's measured concurrency curve, are tagged with the configuration that produced them, and startup fails if that tag does not match the running inference server.
- [ ] My cache key includes tenant, prompt, model revision, quantisation, decoding/schema configuration, and corpus/retrieval versions — with omission tests.
- [ ] Per-tenant limits work and return correct `429` with `Retry-After`.
- [ ] Given a stored diagnosis, I can reconstruct the exact prompt, model revision, quantisation, decoding/schema configuration, and generation settings that produced it.
- [ ] I compared four fallback strategies on quality, latency, and inference capacity with real numbers.
- [ ] The fallback-model rung has an explicit residency choice, measured load latency and VRAM, and a re-measured primary bulkhead; if it does not fit, the rung is removed.
- [ ] I have written one SLI and SLO for the diagnosis path and measured current performance against it.
- [ ] I ran the demo for operations on authored cases, stated what the results can and cannot claim, and recorded which runbooks and scenarios ops reviewed.
- [ ] **ADR-007** records the reliability policy, the degradation ladder, and the capacity limits with their configuration tag.

---

**Next:** Milestone 8 gives the model something authoritative to reason from — your own runbooks, chunked, embedded, retrieved, ranked, and cited.
