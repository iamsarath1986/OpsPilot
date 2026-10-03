# OpsPilot Roadmap

One product, built in twelve milestones. Written to be *done*, not just read.

For the complete development sequence, work packages, acceptance gates, parallel workstreams, and progress tracking, use the [OpsPilot Development Plan](development-plan.md).

## Start here

| File | What it covers |
|---|---|
| [00-what-you-are-building.md](00-what-you-are-building.md) | The project in plain language: the problem, an example request and response, who uses it, how it grows |
| [01-the-binding-constraint.md](01-the-binding-constraint.md) | ADR-000: customer data stays on company infrastructure; inference, embeddings, reranking, and judging are self-hosted |
| [02-discovery-with-operations.md](02-discovery-with-operations.md) | Questions for the operations team, shared-hosting pain points, and what ops has told us so far |
| [03-gpu-sizing-and-cost.md](03-gpu-sizing-and-cost.md) | How much GPU production needs (estimate: 1 to run, 2 for redundancy), what else self-hosting requires, and a ~$15 rented-GPU test that replaces the estimate with measurements |

## The milestones

| # | Milestone | You build | Weeks |
|---|---|---|---:|
| 1 | [Domain and parsing](milestone-01-domain-and-parsing.md) | Typed domain model, streaming log parsers, deterministic rules, CLI | 3–4 |
| 2 | [Production Python](milestone-02-production-python.md) | Async boundaries, test architecture, typed config, profiling, first CI | 3–4 |
| 3 | [HTTP and the API](milestone-03-http-and-api.md) | Versioned FastAPI REST contract, Pydantic validation, error contract, health | 3–4 |
| 4 | [PostgreSQL and persistence](milestone-04-postgresql-and-persistence.md) | Schema, transactions, migrations, indexes, integration tests | 4–5 |
| 5 | [Identity, tenancy, privacy](milestone-05-identity-tenancy-and-privacy.md) | Auth, RBAC, tenant isolation, secure upload, redaction, threat model | 4–5 |
| 6 | [First LLM slice](milestone-06-first-llm-slice.md) | Provider port, prompt builder, first diagnosis, usage ledger | 3–4 |
| 7 | [Reliable AI orchestration](milestone-07-reliable-ai-orchestration.md) | Structured output, four validation layers, retries, breaker, fallbacks | 4–5 |
| 8 | [RAG](milestone-08-rag.md) | Ingestion, chunking, embeddings, pgvector, hybrid retrieval, citations | 5–6 |
| 9 | [Evaluation and guardrails](milestone-09-evaluation-and-guardrails.md) | Eval datasets, judges, guardrails, release gates, escalation | 5–6 |
| 10 | [Async work and tools](milestone-10-async-work-and-tools.md) | Queues, workers, outbox, DLQ, shared cache, bounded read-only tools | 4–5 |
| 11 | [Observability, Docker, CI/CD](milestone-11-observability-docker-cicd.md) | Tracing, metrics, SLOs, load testing, hardened images, pipeline | 4–5 |
| 12 | [On-prem production and Azure portability](milestone-12-azure-production.md) | Real-data on-prem target, synthetic-only Azure target, IaC, identity, scaling, drills | 6–8 |

Roughly 9–12 months at 10–15 focused hours per week. The weeks are planning ranges, not permission to advance.

## Cross-cutting work that starts early

The numbered milestones describe the engineering sequence; they are not the schedule for work that depends on other people.

| Workstream | Start | Gate it must clear |
|---|---|---|
| Runbook corpus (we write it) | Now, in parallel with Milestones 1–7, into `corpus/runbooks/` | 20–30 runbooks written to the template, versioned, review status recorded, before Milestone 8 implementation begins. Ops review is optional |
| Evaluation cases (we build them) | Now: fault scripts in `evals/faults/` and reproduced incidents alongside Milestones 1–5 | 12–20 cases by Milestone 6; 60–100 with a frozen held-out split by Milestone 9 |
| Provisional model selection | Start of Milestone 6, on the development machine | 12–20 authored cases are enough to choose a reversible demo model |
| Demo for operations | Final week of Milestone 7 (Step 8) | Ops has seen validated, degraded, and abstained results, and has reviewed a sample of runbooks and scenarios |
| Model confirmation | Milestone 9, Step 6; repeated on server hardware in Milestone 12 | The full authored set, including the held-out split, clears the bars recorded before the run |
| GPU sizing measurement | After Milestone 11's load test exists; before any hardware request | A few hours on a rented L4/L40S with synthetic data only fills the measured results table in [GPU sizing and cost](03-gpu-sizing-and-cost.md) |
| Server GPU access | Only when the company deployment becomes a real plan | Needed for Milestone 12's company target, not for development. Request hardware from measured numbers, not from the estimate |

**Two directories are called "runbooks" and they must never mix.** `corpus/runbooks/` is the *retrieval corpus* — hosting procedures the model retrieves and cites, ingested in Milestone 8. `runbooks/` is *operational* — how to restore the database, what to do when an alert fires — created in Milestone 4 and extended in Milestones 11 and 12. Only `corpus/` is ever ingested.

**No step depends on real customer tickets.** Both the runbooks and the evaluation cases are authored by us — see [Data we create ourselves](01-the-binding-constraint.md#data-we-create-ourselves), including the rules that stop the evaluation from grading its own homework.

**Development runs on a local machine with an integrated GPU**, using llama.cpp. Capacity limits are measured, not carried: whenever the GPU, model revision, quantisation, serving stack, or context length changes, the Milestone 6 concurrency curve is re-run and the Milestone 7 limits re-derived.

The 5–6 weeks shown for Milestone 8 are engineering time for ingestion and retrieval. They do **not** include writing the runbooks.

## How every milestone is structured

1. **Topics to learn** — and the engineering problem each one solves
2. **Learning depth** — Understand → Implement → Debug → Design → Productionize
3. **Hands-on exercises** — small experiments, run before the project code
4. **Project implementation** — ordered steps, each mapping concept → problem → implementation → testing
5. **Expected project state** — what changed architecturally, not just functionally
6. **Testing and validation**
7. **Failure scenarios** — reproduce deliberately, diagnose, fix
8. **Skills gained** — mapped to the job description
9. **Completion checklist** — measurable, and gated on explanation rather than on the code running

## The rules

- **A milestone is finished when the checklist is honestly green** — not when the code runs.
- **Write the ADR.** Context, options, decision, consequences, and what evidence would change your mind. Interviews are won on those five lines.
- **Measure before you claim.** Every performance, quality, and cost statement in your README should have a number you produced behind it.
- **Break it on purpose.** The failure scenarios are not optional; they are where most of the learning is.

## Related

`../production_ai_engineer_project_roadmap.md` — a dense end-to-end reference for the same product: final architecture, technology stack, testing strategy, AI evaluation strategy, Azure design, repository structure, learning resources, portfolio strategy, interview topics, and the skills checklist. Use it to look ahead; use `roadmap/` to do the work.
