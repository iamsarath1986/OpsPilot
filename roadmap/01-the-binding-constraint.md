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

Do not pick a model from a leaderboard. Use the Milestone 9 method in **two passes** so model selection does not depend on the post-demo request for closed tickets.

| Pass | Dataset | Decision |
|---|---|---|
| **Bootstrap, at the start of Milestone 6** | 12–20 synthetic cases covering the top ticket types — confirmed by ops where answered, otherwise the discovery hypotheses with ops reviewing the category list — including at least 4 unanswerable cases. Ops reviews the scenarios and reference facts; no customer data is required. | Choose a **provisional, reversible demo model**. These results compare candidates but make no claim about production quality. |
| **Confirmation, after the demo and completed in Milestone 9** | Approved sanitised closed tickets with known resolutions, expanded with synthetic boundary and adversarial cases into the full versioned evaluation set. | Confirm or replace the provisional model before any pilot or production claim. This is the release decision. |

For each pass:

1. Define the cases and quality bar before running candidates.
2. Shortlist 3–4 open-weight instruction-tuned models across two size tiers — something in the 7–14B range and something in the 24–32B range.
3. Run all of them against the same set, at temperature 0, with the same prompt and serving configuration. In the confirmation pass, also use the same constrained-decoding setup.
4. Record for each: correctness on the available slices, abstention behaviour, tokens/second under one request and under eight concurrent requests, and VRAM footprint at the quantisation you will deploy. Add groundedness once the runbook corpus exists.
5. Pick the smallest model that clears the stated bar. If none clears it, use the best bootstrap candidate only for the labelled demo and narrow the demo's claims.

The bootstrap set is checked into `evals/bootstrap/` with provenance `synthetic`, its limitations, and ops sign-off. It must never be silently promoted into evidence of live-ticket performance. The post-demo ticket request remains deliberately post-demo; it no longer blocks the demo that earns permission for that request.

Verify the current state of the open-weight field at the time you choose; it moves faster than any document. What is stable is the *method* above, and two rules of thumb: instruction-tuned beats base for this task, and a well-prompted smaller model with good retrieval usually beats a larger one with poor retrieval.

**Quantisation is a quality decision, not just a memory one.** Evaluate the quantised weights you will actually deploy, never the full-precision version, or your eval numbers describe a system you are not running.

**You also need an embedding model and probably a reranker**, both self-hosted, both chosen by the same method against retrieval metrics (recall@K, MRR) rather than by reputation. Embedding models are small and cheap to serve; this is not the expensive part.

## The runbook corpus is a programme dependency

Milestone 8 cannot manufacture its source material during an engineering sprint. Discovery has not yet established whether usable written runbooks exist — ops has not been asked directly. As a deliberately conservative planning assumption, plan as though they do not: begin a parallel authoring workstream during discovery, nominate an operations owner, reserve a recurring review slot, and track 20–30 runbooks through draft, technical review, approval, and versioned publication. Milestone 8 starts only after that corpus-readiness gate is green. See [Discovery](02-discovery-with-operations.md) and [Milestone 8](milestone-08-rag.md).

## Serving

| Option | Best for | Trade-off |
|---|---|---|
| **vLLM** | Production serving with real concurrency | Continuous batching and paged attention give far better throughput under load; GPU-oriented; more operational surface |
| **llama.cpp / Ollama** | Local development, your laptop, CPU fallback | Easiest to run, excellent grammar support, much lower throughput under concurrency |
| **TGI or similar** | Alternative production server | Comparable to vLLM; choose on ecosystem fit, not benchmarks |

A pragmatic split: develop *features* against the easy local option, and target the high-throughput server for anything the ops team will actually use. The provider port makes this a configuration change, which is the entire reason the port exists.

**What a laptop cannot give you is a capacity number.** Prompt construction, validation, retries, the state machine, and retrieval can all be built and tested against a local server or the fake adapter. But the concurrency knee, KV-cache pressure, tokens/s, and VRAM footprint are properties of one specific combination of GPU, model, quantisation, and serving stack. A limit measured on a laptop running llama.cpp says nothing about a GPU host running vLLM. Therefore:

- **GPU access is a gate, not an assumption.** Milestone 6's capacity exercises (the concurrency curve and bootstrap model selection) require access to target-class hardware — the GPU type and serving stack you expect to deploy on. Track this as an early workstream in the [README](README.md); ask for it during discovery, because hardware allocation has a longer lead time than code.
- **Every capacity limit is tagged with the configuration that produced it.** ADR-007 records the in-flight limit, queue depth, and queue-wait bound together with the GPU model, VRAM, model revision, quantisation, serving stack and version, and maximum context length.
- **Any change to that configuration invalidates the limits.** Re-run the Milestone 6 concurrency curve and re-derive the Milestone 7 bulkhead whenever any tagged component changes — including the move from a development GPU to production hardware in Milestone 12. A limit whose configuration no longer matches the deployment is treated as unknown, not as approximately right.

If target-class hardware is not yet available when Milestone 6 starts, build the milestone's software against the local server, mark the capacity exercises and the provisional model selection as **blocked**, and escalate the hardware request. Do not substitute laptop numbers.

**Hardware, in the range this project needs:** a quantised model in the 7–14B class fits comfortably on a single 24 GB GPU and will serve an internal support tool with modest concurrency. The 24–32B class needs more, or heavier quantisation, or both. CPU-only inference works for development and is too slow for interactive use. Before promising anything, ask what hardware actually exists and who controls it — see the open questions below.

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
5. **Does the constraint extend to the runbook corpus**, or only to customer log data? Runbooks are company IP but not customer data, and the distinction may matter.

## What would change this decision

Record these honestly, as with every ADR:

- Written legal sign-off that a specific hosted provider in a specific region with no-training, no-retention terms satisfies the policy. This would reopen option (2), not overturn the ADR.
- Self-hosted quality falling short of the Milestone 9 bar after a genuine selection pass across model sizes — with the eval numbers to prove it, not an impression.
- GPU capacity proving unobtainable, making the project undeployable in practice. Note this would be an *organisational* blocker, not a technical one, and should be escalated as such rather than quietly worked around.

---

**Next:** engineering starts with [Milestone 1](milestone-01-domain-and-parsing.md), which has no model in it. In parallel, start the runbook-authoring workstream in [Discovery](02-discovery-with-operations.md) now; its calendar lead time is longer than its code dependency suggests.
