# GPU Sizing and Cost

*How much GPU OpsPilot needs in production, what else self-hosting requires, and how to replace these estimates with measurements for about the price of a lunch.*

**Status:** estimate, dated 2026-09-15. Every number below comes from rules of thumb and public pricing, not from measuring OpsPilot. The measured results table at the end replaces it; until that table is filled in, treat this document as a planning range, never as a purchase specification.

**Read with:** [ADR-000](01-the-binding-constraint.md) (why inference is self-hosted), [Milestone 11](milestone-11-observability-docker-cicd.md) (load testing), and [Milestone 12](milestone-12-azure-production.md) (the company deployment).

---

## Verdict

For an internal first-level support tool, OpsPilot needs **one GPU to run and two for redundancy**. Whether those are **24 GB or 48 GB cards** depends on which model size wins the Milestone 9 evaluation, not on ticket volume.

## 1. Size by concurrency, not by tickets per day

GPU demand depends on how many diagnoses run **at the same moment**. Little's Law (average requests in progress = arrival rate × time per request) turns staffing numbers into that figure:

| Assumption — replace with real numbers from ops | Value |
|---|---|
| First-level engineers on shift at peak | 50 |
| Diagnoses per engineer at peak | 1 every ~6 minutes |
| Arrival rate | ~8 per minute ≈ 0.14 per second |
| Time per diagnosis | ~10 seconds |
| **Average requests in progress** | 0.14 × 10 ≈ **1.4** |
| Burst allowance (×3) | **~4–5 at once** |

One modern GPU running vLLM handles 4–5 concurrent requests. Doubling the staff still means sizing one GPU, not a cluster.

**The two inputs to get from ops:** the number of first-level engineers at peak, and roughly how many technical tickets arrive in the busiest hour. Both are aggregate numbers and involve no customer data.

## 2. The cost of one diagnosis

| Part | Tokens |
|---|---|
| Instructions and output schema | ~500 |
| Incident and trimmed, redacted log excerpt | ~1,500–2,500 |
| Retrieved runbook sections (top 5) | ~2,000 |
| **Input total** | **~4–5k** |
| **Output — the JSON diagnosis** | **~300–500** |

**Output tokens dominate latency.** The model reads the whole input in about a second, then generates output one token at a time. Keeping the JSON compact is the cheapest latency improvement available — cheaper than any hardware upgrade.

## 3. Memory

GPU memory holds the model weights plus a KV cache for every request in progress.

| Model size | Weights at 4-bit | KV cache per request (~6k tokens)* | Total at 5 concurrent | Fits on |
|---|---|---|---|---|
| 7–8B | ~5 GB | ~0.75 GB | ~11 GB | 24 GB, easily |
| 14B | ~9 GB | ~1.1 GB | ~16 GB | 24 GB |
| 32B | ~19 GB | ~1.5 GB | ~29 GB | **48 GB** |

\* Varies with architecture (layers, KV heads, head dimension); these figures are typical of current open models with grouped-query attention.

Budget **3–4 GB more** for the self-hosted embedding model and reranker; both are small enough to share the GPU.

## 4. Speed

A useful upper bound: **single-request decode speed ≈ memory bandwidth ÷ weight size**, because generating each token reads every weight once.

| GPU | Memory bandwidth | 8B at 4-bit | 14B at 4-bit | 32B at 4-bit |
|---|---|---|---|---|
| NVIDIA L4 (24 GB) | ~300 GB/s | ~50 tok/s → ~9 s | ~30 tok/s → ~14 s | does not fit |
| NVIDIA L40S (48 GB) | ~860 GB/s | ~150 tok/s → ~3 s | ~90 tok/s → ~5 s | ~45 tok/s → ~10 s |

Times assume ~400 output tokens. These are single-request ceilings: batching lowers per-request speed but raises total throughput. Every combination that fits meets the Milestone 7 SLO of 99% of requests terminating within 15 seconds at the concurrency in section 1 — which is exactly the claim the load test must confirm.

## 5. Recommendation by stage

| Stage | GPUs | Reasoning |
|---|---|---|
| **Pilot** — one ticket category, a few engineers | **1 × 24 GB** (L4 class) | A 7–14B model plus embedding and reranker. If the GPU fails, the Milestone 7 fallback serves deterministic-rule output or abstains; acceptable for a pilot |
| **Production** — the whole first-level team | **2 × 24 GB** if the winning model is ≤14B; **2 × 48 GB** (L40S class) if 32B wins | The second GPU is for availability and maintenance, not capacity. Evaluation runs and the local judge use spare capacity overnight |
| **Only if it later becomes customer-facing** | 80 GB data-centre GPUs | Not needed for an internal support tool |

### Why two GPUs rather than one

Redundancy is the obvious reason. The less obvious one is **model upgrades**. A safe switch runs the old and new models side by side while traffic moves gradually and rollback stays instant, and that needs memory for both at once. On one GPU, changing models means downtime.

## 6. The GPU is not the whole platform

One or two GPUs are enough *compute*. Self-hosting in production also needs:

| Area | Requirement | Why |
|---|---|---|
| **Host server** | 8–16 CPU cores; system RAM at least twice the GPU memory (64–128 GB); 1 TB NVMe | Loading weights needs RAM. Several model versions, plus unquantised originals for re-quantising, need disk |
| **Power and cooling** | L4 draws ~72 W; L40S ~350 W. Both are passively cooled and depend on server airflow | Straightforward in a data centre, but it must be in the rack plan |
| **Driver stack** | NVIDIA driver, CUDA, and a GPU-enabled container runtime, pinned to versions the chosen vLLM release supports | Driver upgrades break inference servers more often than anything else. Treat them like database upgrades |
| **Model management** | Weights in an internal mirror, pinned by revision and checksum; licence approved; server configured for offline operation | Required by the Milestone 12 outbound-traffic rule |
| **Access control** | The inference server reachable only from the OpsPilot API, never by users directly | vLLM's built-in authentication is minimal; network isolation is the real control |
| **Monitoring** | GPU health via NVIDIA's DCGM exporter (utilisation, memory, temperature, hardware errors) plus vLLM's own metrics (queue length, KV-cache use) | Ordinary application monitoring cannot see GPU memory filling or a card failing |
| **Failure behaviour** | The Milestone 7 fallback ladder, plus a context-length cap on every request | One oversized log must never exhaust GPU memory for everyone |
| **Ownership** | A named team responsible for the GPU host: patching, hardware faults, on-call | The Milestone 12 prerequisite. Without an owner, the pilot quietly dies the first time the card fails |

## 7. When this estimate stops holding

Re-size — and re-run the measurement in section 8 — if any of these becomes true:

- **Much higher volume:** far more engineers, or customers using the tool directly.
- **Longer context:** whole log files instead of trimmed excerpts. KV cache per request grows linearly with context length.
- **Longer output:** prose explanations instead of compact JSON. Latency grows linearly with output length.
- **Judge runs during working hours:** evaluation competing with live traffic.
- **A larger model winning evaluation:** a 70B-class model needs 80 GB-class GPUs.

## 8. Replace the estimate: rent a GPU for an afternoon

The development machine's integrated GPU cannot predict server behaviour. ADR-000 permits synthetic data outside the company, so a few hours on a rented cloud L4 or L40S produces real numbers before anyone spends money on hardware.

### Rental prices, September 2026

| GPU | Budget marketplaces | Specialist GPU clouds | Large clouds (AWS, Azure, Google, CoreWeave) |
|---|---|---|---|
| **L4** (24 GB) | ~$0.40–0.60/hr | **median ~$0.89/hr** | ~$1–2/hr |
| **L40S** (48 GB) | ~$0.32–0.68/hr | **~$0.86–1.29/hr** (median ~$1.13) | ~$1.57–2.25/hr, and higher on some |

On-demand prices at the main specialist providers sit within about 20% of each other; regional availability usually matters more than the hourly rate. **Prices move — check current rates before renting.**

### The test session

| Activity | Time | L4 at median | L40S at median |
|---|---|---|---|
| Set up vLLM and download 2–3 candidate models | ~1 hr | ~$0.90 | ~$1.10 |
| Concurrency curve (Milestone 6 exercise 4) and Milestone 11 load test, per model | ~3–4 hrs | ~$3.50 | ~$4.50 |
| Mistakes and reruns | ~1–2 hrs | ~$1.80 | ~$2.30 |
| **Total** | **~5–7 hrs** | **~$6** | **~$8** |

Testing both GPUs on the same day costs **about $15**; at large-cloud rates, roughly double or triple that — still under $50.

For the later rent-versus-buy conversation: one GPU running continuously for a month costs about **$640 (L4)** or **$810 (L40S)** at median on-demand rates.

### Rules for the session

- **Synthetic data only.** Rented GPUs run on third-party hardware. Upload reproduced incidents, authored scenarios, and our runbooks — never a real log, ticket, or anything copied from production, even for a quick test. This is ADR-000, not a preference.
- **Delete the instance when finished; do not merely stop it.** Many providers keep billing attached storage, and some bill stopped machines.
- **Download results before deleting.** Concurrency curves, latency percentiles, and memory readings.
- **Spot or community instances are fine.** Cheaper and interruptible, which does not matter for a repeatable test.
- **Watch storage.** Several model versions can reach 50–100 GB.
- **Tag every result with its full configuration:** GPU, model revision, quantisation, serving stack and version, maximum context length. Untagged numbers are unusable (ADR-000).

### What the session must produce

For each GPU × model combination: maximum concurrency before latency collapses, tokens/s at concurrency 1 and at the knee, time to first token, p50/p95/p99 end-to-end latency, peak GPU memory, and whether the Milestone 7 SLO held at the section 1 concurrency.

## 9. Measured results

*Empty until the section 8 session runs. When filled in, these numbers supersede sections 1–5, and the recommendation in section 5 is revised from them.*

| Date | GPU | Model, revision | Quantisation | Serving stack, version | Max context | Knee (concurrent) | tok/s at 1 / at knee | TTFT p95 | E2E p95 | Peak memory | SLO held? |
|---|---|---|---|---|---|---|---|---|---|---|---|
| | | | | | | | | | | | |

---

## Sources for rental prices (retrieved 2026-09-15)

- [L4 Cloud Pricing: Compare 17+ Providers (2026) — getdeploying.com](https://getdeploying.com/gpus/nvidia-l4)
- [L40S Cloud Pricing: Compare 29+ Providers (2026) — getdeploying.com](https://getdeploying.com/gpus/nvidia-l40s)
- [NVIDIA L40 and L40S Pricing (September 2026) — Thunder Compute](https://www.thundercompute.com/blog/nvidia-l40-pricing)
- [NVIDIA L40S: Pricing, Specs, Best Uses — Fluence](https://fluence.ai/blog/nvidia-l40s/)
- [NVIDIA L40S GPU Rental — Spheron](https://www.spheron.network/gpu-rental/l40s/)
- [GPU Cloud Pricing Comparison 2026 — Spheron Blog](https://www.spheron.network/blog/gpu-cloud-pricing-comparison-2026/)
- [NVIDIA L40S GPU Price Comparison — gpus.io](https://gpus.io/en/gpus/l40s)
- [Rent L40S GPUs — Vast.ai](https://vast.ai/pricing/gpu/L40S)
- [Runpod GPU Pricing — ComputePrices.com](https://computeprices.com/providers/runpod)
