# OpsPilot Roadmap

One product, built in twelve milestones. Written to be *done*, not just read.

## Start here

| File | What it covers |
|---|---|
| [00-what-you-are-building.md](00-what-you-are-building.md) | The project in plain language: the problem, an example request and response, who uses it, how it grows |
| [01-the-binding-constraint.md](01-the-binding-constraint.md) | ADR-000: customer data stays on company infrastructure; inference, embeddings, reranking, and judging are self-hosted |
| [02-discovery-with-operations.md](02-discovery-with-operations.md) | Questions for the operations team, shared-hosting pain points, and what ops has told us so far |

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
| Runbook corpus | Discovery, in parallel with Milestones 1–7 | 20–30 reviewed, versioned runbooks before Milestone 8 implementation begins |
| GPU access | Discovery; request it well before Milestone 6 | Access to target-class hardware (the GPU type and serving stack you expect to deploy on) before Milestone 6's concurrency curve and model selection. Without it those exercises are blocked — laptop numbers are not a substitute |
| Provisional model selection | Start of Milestone 6, on target-class hardware | 12–20 synthetic, ops-reviewed cases are enough to choose a reversible demo model |
| Synthetic-data demo | Final week of Milestone 7 (Step 8) | Ops has seen validated, degraded, and abstained results on synthetic data, with no real-ticket quality claimed |
| Closed-ticket request | Same week as the demo, through your manager | Approved sanitised closed tickets arrive in time for labelling at the start of Milestone 9 |
| Production model confirmation | Milestone 9, Step 6 | Approved tickets plus synthetic/adversarial cases support the full evaluation pass |

Capacity limits are measured, not carried: whenever the GPU, model revision, quantisation, serving stack, or context length changes, the Milestone 6 concurrency curve is re-run and the Milestone 7 limits re-derived.

The 5–6 weeks shown for Milestone 8 are engineering time for ingestion and retrieval. They do **not** include waiting for operations staff to author and approve the source material.

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
