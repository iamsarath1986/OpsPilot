# AI Engineer Roadmap - Discussion

*Compiled from our discussion about becoming an AI engineer, including
the roadmap and the follow-up on Evaluation & Guardrails.*

## 1. Initial Request

**User:** I want to become an AI engineer. Please check the image
attached. Based on this create a roadmap.

## 2. AI Engineer Roadmap

The core recommendation was to become an engineer who can build reliable
AI systems, rather than focusing mainly on prompting. The roadmap
prioritizes backend engineering, distributed systems, LLM engineering,
production reliability, cloud deployment, and advanced AI.

### Phase 1: Strong Backend Foundations (Months 1-2)

-   Advanced Python: asyncio, concurrency, multithreading vs
    multiprocessing, type hints, generators, context managers, and
    pytest.
-   Databases: PostgreSQL, indexes, query optimization, transactions,
    and data modeling.
-   APIs: FastAPI, REST APIs, authentication, and rate limiting.
-   Docker: Dockerfile, Docker Compose, and container fundamentals.
-   Project: Build a Personal Finance Backend API with expense/income
    tracking, categories, reports, JWT authentication, PostgreSQL, and
    Docker.

### Phase 2: System Design & Distributed Systems (Months 3-4)

-   Architecture: monolith vs microservices and event-driven
    architecture.
-   Tools: Redis, RabbitMQ, and Kafka basics.
-   Concepts: caching, scaling, load balancing, rate limiting, database
    sharding, and CAP theorem.
-   Project: Extend the application into a real-time expense analytics
    system using FastAPI, PostgreSQL, RabbitMQ, analytics workers, and
    Redis.

### Phase 3: LLM Engineering Fundamentals (Month 5)

-   LLM concepts: tokens, embeddings, context windows, temperature, and
    function/tool calling.
-   RAG: chunking, embeddings, vector search, and retrieval.
-   Vector storage: pgvector, Chroma, or Qdrant.
-   Project: Build an AI Knowledge Assistant that ingests PDFs, creates
    embeddings, retrieves relevant information, answers questions, and
    provides citations.

### Phase 4: Agent Engineering (Month 6)

-   Frameworks and tools: LangGraph, Pydantic AI, LiteLLM, and MCP.
-   Concepts: planning, tool calling, memory, workflows, and multi-agent
    systems.
-   Project: Build an AI assistant capable of using tools and business
    knowledge, with features such as recommendations, support
    assistance, and knowledge retrieval.

### Phase 5: Production AI Systems (Months 7-8)

-   Observability: OpenTelemetry, Grafana, and Prometheus.
-   AI observability: Langfuse and similar tracing/evaluation systems.
-   Evaluation: hallucination detection, benchmarking, regression
    testing, and AI quality measurement.
-   Project improvements: tracing, monitoring, cost tracking, latency
    tracking, evaluation pipelines, and reliability controls.

### Phase 6: Deployment & Cloud (Month 9)

-   Cloud fundamentals: compute, storage, managed databases, IAM, and
    container deployment.
-   Deployment: Nginx, CI/CD, and GitHub Actions.
-   Deploy the AI system with a production architecture involving an API
    backend, PostgreSQL/pgvector, Redis, LLM providers, and monitoring.

### Phase 7: Advanced AI Engineering (Months 10-11)

-   Fine-tuning, LoRA, model distillation, advanced agents, multi-agent
    systems, and long-term memory.
-   Read approximately one useful research paper per week and implement
    selected ideas rather than trying to consume everything.

### Phase 8: Flagship Project (Month 12)

-   Build a production-grade AI product that demonstrates backend
    engineering, RAG, agents, evaluation, guardrails, observability,
    deployment, and system design.
-   Possible directions: AI platform for hosting companies, AI personal
    finance assistant, or an AI SaaS product for small businesses.

### Recommended Learning Order

Backend Engineering -\> Distributed Systems -\> RAG -\> Agents -\>
Production AI -\> Cloud -\> Advanced AI

### Suggested Weekly Schedule

Weekdays: approximately 1 hour learning + 1 hour building.

Weekends: approximately 4-6 hours of focused project work.

The key principle is project-driven learning: learn a concept, implement
it, test it, deploy it, and document the engineering decisions.

## 3. Follow-up: Evaluation & Guardrails

**User:** I have seen Evaluation & Guardrails in some companies job
description.

### Why Evaluation & Guardrails Matter

Modern AI engineering is not only about calling an LLM API. Companies
need engineers who can determine whether an AI system is correct,
reliable, safe, measurable, and suitable for production. Evaluation and
guardrails are therefore important skills to add explicitly to the
roadmap.

### Evaluation (Evals)

Evals answer the question: **"How do we know our AI system is actually
working?"**

Unlike deterministic software, many LLM tasks have multiple acceptable
outputs, so AI systems need dedicated quality measurements and
regression tests.

-   Exact-match evaluation for deterministic answers.
-   Semantic similarity when different wording can express the same
    correct meaning.
-   LLM-as-a-judge evaluation, where another model scores an answer
    against defined criteria.
-   Human evaluation for nuanced quality, correctness, usefulness, and
    professionalism.
-   Production metrics such as accuracy, hallucination rate, retrieval
    quality, latency, reliability, and cost.

#### Example Evaluation Workflow

1.  Create a representative evaluation dataset of real questions and
    expected behavior.
2.  Run the AI system against the dataset whenever prompts, models,
    retrieval logic, or application code changes.
3.  Score the results using deterministic checks, semantic metrics,
    model-based judges, and/or human review.
4.  Compare the new version against the previous baseline.
5.  Block or investigate deployments when important quality metrics
    regress.

### Guardrails

Guardrails are validation and control layers placed around an AI system.
They can inspect user input, model output, retrieved context, tool
calls, structured data, and business rules before an action or response
is accepted.

-   Input guardrails: detect or reject disallowed, malicious,
    irrelevant, or prompt-injection-style input.
-   Output guardrails: validate model responses before returning them to
    the user.
-   Structured-output validation: enforce schemas and types, for example
    with Pydantic.
-   Grounding/hallucination controls: check whether claims are supported
    by retrieved or trusted information.
-   Privacy controls: prevent unintended exposure of sensitive or
    personally identifiable information.
-   Tool/action guardrails: validate parameters and permissions before
    an agent executes an external action.
-   Business-rule guardrails: enforce application-specific constraints
    that the model must not override.

### Fallback Mechanisms

A production AI system should define what happens when the primary path
fails. Depending on the use case, a fallback might retry with corrected
input, use another model, switch to retrieval/search, return a safe "I
don't know" response, or escalate to a human.

### AI Reliability Engineering - Add This as a Dedicated Roadmap Phase

-   Evals and evaluation datasets
-   Guardrails
-   Structured outputs and schema validation
-   Pydantic / Pydantic AI
-   DeepEval or equivalent evaluation tooling
-   Langfuse or equivalent tracing/observability
-   Confidence and grounding strategies
-   Fallback mechanisms
-   Human-in-the-loop workflows
-   Regression testing for prompts, RAG, and agents

### Recommended Reliability Project

Build a Customer Support AI Agent containing:

-   RAG
-   Guardrails
-   Evaluation suite
-   Tracing and monitoring
-   Fallback logic
-   Structured outputs
-   Human escalation
-   Quality, latency, and cost measurements

## 4. Updated Roadmap Summary

1.  Python + Backend Engineering
2.  Databases + APIs + Testing
3.  Docker + CI/CD
4.  System Design + Distributed Systems
5.  LLM Fundamentals
6.  RAG + Embeddings + Vector Search
7.  Agent Engineering + Tool Calling
8.  Evaluation + Guardrails + Structured Outputs
9.  Observability + Cost/Latency Monitoring + Fallbacks
10. Cloud + Kubernetes/Production Deployment
11. Advanced AI Engineering
12. Production-grade flagship AI project

**Key principle:** Do not treat Evaluation & Guardrails as optional
extras. Build them into your AI projects so your portfolio demonstrates
that you can create AI systems that are not only intelligent, but also
testable, reliable, observable, and production-ready.
