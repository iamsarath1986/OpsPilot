# ADR-000 — Data Residency and Self-Hosted Inference

**Status:** Accepted. Set by the CEO, not by engineering.
**Date:** 2026-08-21
**Supersedes:** nothing. **Constrains:** every ADR from 001 to 012.

## The decision

Customer data — log content above all — must not leave company infrastructure. Inference runs on models we host ourselves.

This is a business decision, and it is not ours to relitigate. What follows is the engineering consequence.

## Why this is good news

It removes the single largest open question at the top of the project, and it removes it in the direction that produces the more interesting system.

- **It kills a whole class of rework.** The alternative — building against a hosted API and discovering the constraint in month seven — would have invalidated the provider adapter, the cost model, the capacity plan, and the entire Milestone 12 deployment. Learning this now costs nothing.
- **It is the harder engineering, which is the point.** Anyone can call a hosted API. Serving your own model, getting reliable structured output out of it without a vendor's JSON mode, sizing GPU concurrency, and proving quality on your own eval set is a materially rarer skill set. It reads better on a CV and interviews better, because there is more to say.
- **It fits the employer exactly.** A hosting company owns racks and runs its own data centre. Self-hosting is not an exotic burden here — it is the house competency. The project stops being "an AI thing Sarath is trying" and starts being something the company could actually operate.
- **The unit economics invert in your favour.** Per-token billing punishes exactly the workload this is: long log excerpts, many tickets, retries, evaluation runs. Self-hosting removes the marginal provider invoice, although GPU time, energy, contention, and operations are still real costs that the capacity ledger must expose.

## What actually changes — and what does not

Most of the roadmap is untouched. The constraint bites in four places.

| Milestone | Impact | What changes |
|---|---|---|
| 1–5 | **None** | Domain, parsing, async, API, PostgreSQL, auth and tenancy are all local by construction. Build exactly as written. |
| 6 | **Moderate** | The `DiagnosisProvider` port was designed for this. The adapter targets a local OpenAI-compatible server instead of a vendor. The token ledger stays — it now measures capacity, not billing. |
| 7 | **Significant** | Structured output can no longer lean on a vendor's guaranteed JSON mode. See below — this is the main technical delta. |
| 8 | **Moderate** | `pgvector` was always local. But the **embedding model must be self-hosted too** — see the trap below. |
| 9 | **Moderate** | LLM-as-judge now means a *local* judge. Weaker judges shift weight onto deterministic graders and human review. Honest, and arguably better discipline. |
| 10–11 | **None** | Queues, caching, observability, CI/CD are unaffected. |
| 12 | **Significant** | The deployment target is in question. See "The Azure tension". |

## The trap: it is not just the LLM

The most common way this constraint gets violated by accident is that someone self-hosts the generation model and then quietly calls a hosted API for everything else. Every one of these sends customer log text or runbook content off-premises:

- **Embeddings** (Milestone 8). Embedding a log excerpt to retrieve against transmits the log excerpt. This is the one people miss.
- **Reranking** (Milestone 8). A cross-encoder rerank sends the query *and* the candidate chunks.
- **The judge model** (Milestone 9). Judging sends the input, the output, and often the reference answer.
- **Tokenisers and telemetry SDKs** that phone home, and any observability vendor receiving prompt or completion bodies (Milestone 11).

Write this list into the ADR as a checklist and audit against it before each phase checkpoint. "No outbound calls carrying customer data" should be a CI test that inspects egress in the integration suite, not a promise.

## The main technical delta: structured output without a vendor JSON mode

Milestone 7's four validation layers were designed assuming a model that mostly emits well-formed JSON. Open-weight models are less reliable at this, especially smaller ones under long prompts. The fix is better than the thing it replaces: **constrained decoding**, where the sampler is only permitted to emit tokens that keep the output valid against a grammar or JSON schema.

Approaches, roughly in order of how much you should prefer them:

1. **Schema-guided decoding at the serving layer** — the server is given your JSON schema and mechanically cannot produce output that violates it. This makes the syntax and schema validation layers structurally impossible to fail, rather than merely unlikely.
2. **Grammar-constrained decoding** (GBNF-style) — same idea expressed as a grammar. Useful where the serving stack supports grammars but not schemas directly.
3. **Prompt-and-validate-and-retry** — what you would do against a hosted API. Keep it as the fallback path so the adapter behaves identically if you ever switch.

**This does not make Milestone 7 redundant — it moves the work.** Constrained decoding guarantees *shape*, never *truth*. Layers 3 and 4 (business rules and evidence integrity) still catch a syntactically perfect diagnosis citing `ev_9` when only `ev_1`–`ev_4` exist. If anything the constraint sharpens the lesson: valid JSON was never the same thing as a correct answer, and now you can prove it, because you will have a model that reliably produces the former and unreliably produces the latter.

## Choosing the model

Do not pick a model from a leaderboard. Use the Milestone 9 method in **two passes**. Both use evaluation data you create yourselves — see [Data we create ourselves](#data-we-create-ourselves) below — because real customer tickets will not be released.

| Pass | Dataset | Decision |
|---|---|---|
| **Bootstrap, at the start of Milestone 6** | 12–20 authored cases covering the top ticket types — confirmed by ops where answered, otherwise the discovery hypotheses — including at least 4 unanswerable cases. | Choose a **provisional, reversible demo model**. The set is too small to support any quality claim beyond "suitable for a demo". |
| **Confirmation, completed in Milestone 9** | The full versioned evaluation set of 60–100 authored cases, anchored by reproduced incidents, with a held-out split the prompt and retrieval were never tuned against. | Confirm or replace the provisional model. This is the release decision. |

For each pass:

1. Define the cases and quality bar before running candidates.
2. Shortlist 3–4 open-weight instruction-tuned models across two size tiers that fit the hardware the pass runs on. On the development machine that means roughly 3–4B and 7–8B at 4-bit; on a server GPU in Milestone 12, something in the 7–14B range and something in the 24–32B range.
3. Run all of them against the same set, at temperature 0, with the same prompt and serving configuration. In the confirmation pass, also use the same constrained-decoding setup.
4. Record for each: correctness on the available slices, abstention behaviour, tokens/second under one request and under eight concurrent requests, and VRAM footprint at the quantisation you will deploy. Add groundedness once the runbook corpus exists.
5. Pick the smallest model that clears the stated bar. If none clears it, use the best bootstrap candidate only for the labelled demo and narrow the demo's claims.

The bootstrap set is checked into `evals/bootstrap/` with each case's provenance and its limitations recorded. Its cases may later join the Milestone 9 dev split, but never the held-out split: they were visible while the prompt was being built.

Verify the current state of the open-weight field at the time you choose; it moves faster than any document. What is stable is the *method* above, and two rules of thumb: instruction-tuned beats base for this task, and a well-prompted smaller model with good retrieval usually beats a larger one with poor retrieval.

**Quantisation is a quality decision, not just a memory one.** Evaluate the quantised weights you will actually deploy, never the full-precision version, or your eval numbers describe a system you are not running.

**You also need an embedding model and probably a reranker**, both self-hosted, both chosen by the same method against retrieval metrics (recall@K, MRR) rather than by reputation. Embedding models are small and cheap to serve; this is not the expensive part.

## Data we create ourselves

**Decision:** real customer tickets will not be released for this project, and we will not ask for them. Both the runbook corpus (Milestone 8) and the evaluation dataset (Milestones 6 and 9) are authored by us. This removes the two external data dependencies from the schedule. It also creates one serious risk that must be designed against from the first case.

### The runbook corpus

A runbook is a written procedure for diagnosing one known problem: symptoms, evidence to collect, diagnosis branches, safe next steps, and when to escalate. The corpus is the versioned collection of them. Milestone 8 retrieves from it, and Milestone 7's evidence check verifies that citations point into it.

- **Written by us** from our own hosting experience, public technical knowledge, and — where it exists — the company's own public help content.
- **Every runbook carries a review status:** `unreviewed` or `ops-reviewed`, with reviewer and date. Engineering proceeds on unreviewed runbooks. Ops review is an upgrade, not a prerequisite: asking "is this how we do it?" takes an engineer minutes, whereas asking them to author takes hours.
- **The claim follows the status.** An answer citing an unreviewed runbook is grounded in *our* procedure, not in company policy, and the demo and reports must say so.
- **Never model-generated.** An LLM may help with formatting, but diagnosis branches and safe next steps are written and checked by a person. A model-written runbook retrieved by a model is the model citing itself.

### The evaluation dataset

"Closed tickets with known resolutions" is replaced by **authored incident cases**, each tagged with one provenance:

| Provenance | How it is made | What makes it trustworthy |
|---|---|---|
| `reproduced` — **the anchor** | Break a real stack on purpose — local PHP/WordPress, web server, database, mail — using a scripted fault: exhausted PHP memory, corrupt `.htaccess`, wrong permissions or ownership, full disk or inode quota, wrong database credentials, expired certificate, redirect loop. Capture the logs the fault actually produced. | **Ground truth is known by construction:** the root cause is the fault you injected, not someone's opinion. The logs are real output, in real formats, with real noise. Keep the fault script, so the case can be regenerated. |
| `scenario` | A written case modelled on a real incident someone remembers — the symptom, the misleading clues, the eventual cause — with no customer details. | Captures the messy patterns nobody would think to reproduce. Weaker ground truth, so each one needs a second person to agree the reference answer. |
| `adversarial` | Injection in the ticket or log, requests for another tenant's data, destructive-command bait, out-of-scope requests. | Tests safety behaviour rather than diagnosis. |
| `unanswerable` | A plausible incident no runbook covers, or whose logs do not contain the cause. | The correct answer is abstention; tests the most dangerous failure an ops assistant has. |

### The risk: grading your own homework

When the same people write the runbooks, the test cases, and the reference answers, an evaluation can quietly measure *how well the system repeats our own wording back to us*. Retrieval scores look excellent because the question uses the runbook's vocabulary; answers look correct because the reference answer was copied from the runbook. None of it predicts behaviour on a real customer's ticket. These rules are mandatory, not advice:

1. **Ground truth never comes from the runbook text.** For `reproduced` cases it is the injected fault. For `scenario` cases it is written before looking at the matching runbook section.
2. **Tickets are written in the customer's voice, not the runbook's.** "My shop is broken since this morning, nothing loads" — not "HTTP 500 due to PHP memory limit exhaustion". A case whose ticket text shares distinctive phrases with its target runbook is rewritten. Check this mechanically with a vocabulary-overlap score per case, and report retrieval results for high- and low-overlap cases separately.
3. **A held-out split exists from day one and is never used for tuning.** At least 30% of cases, frozen before prompt and chunking experiments begin, and run only at confirmation. Tuning against it once turns it into a dev set; if that happens, write new held-out cases.
4. **Separate authorship where possible.** If a second person is available — a colleague, a support engineer contributing scenarios — their cases go into the held-out split. Record the author of every case and every runbook, and report results by author.
5. **Some answers must be missing from the corpus.** At least 15 unanswerable cases in the Milestone 9 set. If everything the tests ask is in the runbooks, abstention is never tested.
6. **Label every result with its provenance.** Reports state counts per provenance and per author, and never describe authored-case accuracy as real-ticket accuracy.

### What we can and cannot claim

| We can say | We cannot say |
|---|---|
| "Correct root cause on N of M reproduced incidents across these categories" | "X% accurate on customer tickets" |
| "Retrieval recall@5 of Y on low-overlap, customer-voice queries" | "Works on our support queue" |
| "Abstained correctly on K of 15 unanswerable cases" | "Reduces escalations" — without an in-company pilot |

The only route to the right-hand column is measurement inside the company boundary: support engineers using the tool on live tickets and rating answers, with only the aggregate ratings leaving their workflow. That is a pilot decision for later, and it is a different — and much easier — approval than releasing tickets.

**Why this is not a weaker position.** "I built the evaluation set by reproducing real hosting failures, so every case has a known root cause, and I kept a held-out split written in the customer's voice" is a stronger interview answer than "my employer gave me twenty tickets". Real tickets frequently have unclear or wrong resolutions; reproduced incidents never do.

## Serving

| Option | Best for | Trade-off |
|---|---|---|
| **vLLM** | Production serving with real concurrency | Continuous batching and paged attention give far better throughput under load; GPU-oriented; more operational surface |
| **llama.cpp / Ollama** | Local development, your laptop, CPU fallback | Easiest to run, excellent grammar support, much lower throughput under concurrency |
| **TGI or similar** | Alternative production server | Comparable to vLLM; choose on ecosystem fit, not benchmarks |

A pragmatic split: develop *features* against the easy local option, and target the high-throughput server for anything the ops team will actually use. The provider port makes this a configuration change, which is the entire reason the port exists.

**Development hardware.** OpsPilot is developed on a local machine with an integrated AMD Radeon 680M GPU sharing 30 GB of system RAM. That means:

- **Serving stack for development: llama.cpp with its Vulkan backend.** vLLM targets discrete GPUs and does not support this chip. llama.cpp also supports JSON-schema and grammar-constrained decoding, so Milestone 7 works unchanged.
- **Models for development:** 3–4B for fast iteration; 7–8B at 4-bit quantisation for quality checks. Larger models are impractical here.
- **Concurrency is effectively one request at a time**, and long log excerpts are slow to process. Measure it; do not assume.
- Everything in Milestones 6–11 can be built and learned on this machine.

**Capacity numbers are only valid for the configuration that produced them.** The concurrency knee, KV-cache pressure, tokens/s, and memory footprint are properties of one specific combination of GPU, model, quantisation, and serving stack. Measuring them on the development machine is worthwhile — the shape of the curve and the way latency collapses past the knee teach the concept on any hardware — but the numbers never transfer to other hardware. Therefore:

- **Every capacity limit is tagged with the configuration that produced it.** ADR-007 records the in-flight limit, queue depth, and queue-wait bound together with the GPU model, memory, model revision, quantisation, serving stack and version, and maximum context length. Development numbers are tagged as such.
- **Any change to that configuration invalidates the limits.** Re-run the Milestone 6 concurrency curve and re-derive the Milestone 7 bulkhead whenever any tagged component changes — including the move to production hardware in Milestone 12. A limit whose configuration no longer matches the deployment is treated as unknown, not as approximately right.
- **Model selection on the development machine is also provisional in size.** A model that fits a 680M may not be the one worth deploying on a server GPU. Milestone 12 re-runs the confirmation pass on the production hardware and may choose a larger model.
- **Server GPUs are needed only for the company deployment in Milestone 12.** That is a prerequisite for that milestone, not for development.

**Production hardware, in the range this project needs:** a quantised model in the 7–14B class fits comfortably on a single 24 GB GPU and will serve an internal support tool with modest concurrency. The 24–32B class needs more, or heavier quantisation, or both. CPU-only inference works for development and is too slow for interactive use. Before promising anything, ask what hardware actually exists and who controls it — see the open questions below. The full sizing estimate, the rest of what self-hosting requires, and how to measure it cheaply are in [GPU sizing and cost](03-gpu-sizing-and-cost.md).

## Capacity: the ceiling moves, it does not disappear

Milestone 6 measures the self-hosted throughput ceiling directly rather than treating provider RPM/TPM as the constraint. The ceiling is arguably more instructive because you own both sides of it:

- Concurrency is bounded by **GPU memory** — model weights plus KV cache per active request. Long log excerpts inflate KV cache fast.
- Throughput is bounded by **batching efficiency**. This is why continuous batching matters.
- There is no burst capacity to buy. When the GPU is saturated, requests queue. **Milestone 10's queue and backpressure work stops being an academic exercise and becomes the actual admission-control mechanism.**
- Latency under concurrency degrades differently from a hosted API: gentler at first, then sharply past saturation. Measure your own curve in Milestone 11's load test; do not assume.

The token ledger from Milestone 6 survives intact. It stops answering "what is the invoice?" and starts answering "how close are we to saturation, and which tenant is causing it?" — which is the more useful question anyway.

## The Azure tension — resolve it deliberately

Your target job description asks for Azure or another major cloud. Your employer wants inference on-premises. These are in tension and it is worth being explicit rather than letting it resolve by accident.

**They are less opposed than they look.** "Data must not leave company infrastructure" is a statement about *customer data*, not about every component. There are three defensible resolutions:

1. **On-premises deployment, cloud-portable design.** Build with containers, IaC, OIDC-federated CI, managed identity patterns and OpenTelemetry throughout — all of which apply identically on-premises — and deploy to the company's own infrastructure. Then stand up the Azure path separately with **synthetic data only**, as a documented, working second deployment target. You demonstrate both, and you can explain the trade-off in an interview, which is worth more than either alone.
2. **Azure with the model still private** — Azure OpenAI in a chosen region with no-training terms, or your own model on Azure GPU compute inside a VNet with private endpoints. This may or may not satisfy the CEO. It is a *policy* question about what "leave our infrastructure" means, and the answer must come from them in writing.
3. **Hybrid** — control plane and non-sensitive services in Azure, inference and log data on-premises. More architecture to explain, and honestly the most realistic shape for a company in this position.

**Recommendation: build for (1), and ask the question that decides whether (2) is available.** Option 1 is the only one you can start today without permission, it satisfies your employer completely, and it costs you nothing on the job-readiness side as long as you genuinely build the Azure target for the synthetic-data version. Milestone 12 stays in the roadmap; it acquires a second deployment target rather than losing its first.

The interview answer this produces is strong: *"Data residency was a hard constraint, so inference is self-hosted on our own hardware with schema-constrained decoding. I built the deployment as IaC with a cloud target as well, so the same artifact deploys to Azure Container Apps for environments where the constraint doesn't apply. Here's the cost and latency comparison between the two."* That is a better answer than "I used Azure OpenAI."

## Redaction is still required

A tempting and wrong conclusion is that self-hosting makes Milestone 5's redaction work unnecessary. It does not, and the reasons are worth writing down:

- Log excerpts still land in your **database**, your **eval datasets**, your **telemetry**, and your **cached responses** — all of which have wider access than the log file did.
- Eval datasets are the most-copied artifact in any AI project. They end up on laptops.
- Prompt and completion bodies in traces (Milestone 11) are a genuine exposure even when the model is local.
- Least-privilege handling of customer data is a GDPR posture question, not a "did it cross the network boundary" question.

Redaction moves from *gating whether you may call the model at all* to *limiting blast radius internally*. Keep the measured miss rate. Keep the false-positive rate too.

## Open questions this creates

Route these to the ops/technology channel, and route the hardware one through your manager:

1. **What does "leave our infrastructure" mean precisely** — our data centre, our legal entity, our country, or the EU? The answer decides whether resolution (2) above exists at all. Get it in writing.
2. **What GPU hardware exists today, who owns it, and what would it take to get access** to a modest allocation for an internal tool?
3. **Is there an existing internal model deployment** anyone is already running? If so, the fastest correct move is to consume it rather than stand up a second one.
4. **Who signs off on the model choice** — is there an approved-software process, and does an open-weight model with a permissive licence pass it? Check the licence terms for commercial use before you get attached to a model.
5. **Does the constraint extend to the runbook corpus**, or only to customer log data? Our runbooks are self-authored, but once ops reviews them they describe company procedure, and the distinction may matter.

## What would change this decision

Record these honestly, as with every ADR:

- Written legal sign-off that a specific hosted provider in a specific region with no-training, no-retention terms satisfies the policy. This would reopen option (2), not overturn the ADR.
- Self-hosted quality falling short of the Milestone 9 bar after a genuine selection pass across model sizes — with the eval numbers to prove it, not an impression.
- GPU capacity proving unobtainable, making the project undeployable in practice. Note this would be an *organisational* blocker, not a technical one, and should be escalated as such rather than quietly worked around.

---

**Next:** engineering starts with [Milestone 1](milestone-01-domain-and-parsing.md), which has no model in it. In parallel, start the runbook-authoring workstream in [Discovery](02-discovery-with-operations.md) now; its calendar lead time is longer than its code dependency suggests.
