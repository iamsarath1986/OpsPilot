# OpsAssistant — What You Are Building

*Plain-language project definition. Read this once. Everything in `roadmap/` builds this one product.*

## The problem (a real one, from your own business)

A hosting customer opens a ticket: *"My site shows a 500 error."*

Today, a support engineer:

1. SSHes in or opens WebFTP and hunts for the right log file.
2. Reads a few hundred lines looking for the actual error.
3. Remembers — or fails to remember — that this pattern usually means wrong file permissions, a full disk, or a PHP-FPM upstream timeout.
4. Searches the internal wiki for the fix, if it was ever written down.
5. Escalates to a senior engineer when unsure.

Steps 2–4 are slow, inconsistent between engineers, and the knowledge lives in people's heads.

## What OpsAssistant does

**OpsAssistant is a read-only incident diagnosis service for hosting operations.** It takes an incident plus a log excerpt, and returns a structured, evidence-cited diagnosis that a support engineer can act on — or an honest "I don't have enough evidence, escalate this."

It is an API-first backend product, not a chatbot.

### One request, end to end

A client posts an incident and a log excerpt:

```http
POST /v1/incidents/8f2a.../diagnoses
Authorization: Bearer <token>
```

The service responds with **structured data**, not prose:

```json
{
  "status": "complete",
  "severity": "high",
  "summary": "PHP-FPM cannot read the document root because the directory mode is 0555 for the web user.",
  "likely_causes": [
    {
      "cause": "Incorrect directory permissions after deployment",
      "confidence": "high",
      "evidence_ids": ["ev_3", "ev_7"]
    }
  ],
  "recommended_steps": [
    "Verify ownership of /var/www/arenacore/public with ls -la",
    "Restore directory mode to 0755 for the site owner",
    "Reload php-fpm and re-test the failing URL"
  ],
  "citations": [
    {"runbook": "runbook-permissions.md", "section": "500 after deploy", "chunk_id": "rb_12"}
  ],
  "evidence": [
    {"id": "ev_3", "line": 142, "text": "2026-08-19 14:22:01 [error] 2231#0: *5 open() \"/var/www/arenacore/public/index.php\" failed (13: Permission denied)"}
  ],
  "escalate": false,
  "prompt_version": "diagnose@v4",
  "model": "…",
  "latency_ms": 2140,
  "tokens": {"input": 1832, "output": 214}
}
```

Every field in that response exists for a reason you will be able to defend:

| Field | Why it exists |
|---|---|
| `status` | The caller must know whether this is a full answer, a degraded one, or an abstention |
| `evidence_ids` / `evidence` | A claim with no supporting log line is a hallucination; this makes grounding checkable |
| `citations` | The answer must be traceable to *your* runbooks, not the model's memory |
| `escalate` | The system's most valuable output is sometimes "a human should do this" |
| `prompt_version`, `model` | When quality changes next week, you must know what produced this answer |
| `tokens`, `latency_ms` | AI features have a per-request cost and a latency budget; invisible cost is a production incident waiting to happen |

### Who uses it

- **Tier-1 support engineers** — get a consistent first diagnosis instead of guessing.
- **Senior engineers** — receive a clean escalation packet instead of "it's broken".
- **You, the operator** — can measure answer quality, abstention rate, latency, and cost per diagnosis.

### What it deliberately does NOT do

- It does not execute shell commands or change any customer server.
- It does not browse the internet.
- It does not "fix" anything — it diagnoses, cites, and recommends.

This restriction is intentional and defensible: an LLM that can run commands on production hosting infrastructure is an unacceptable risk with today's reliability. Late in the roadmap it gains a few **read-only** tools (fetch incident, check a synthetic service status, search a runbook) so you can demonstrate agent/tool-calling engineering safely.

## Why this project is the right one for the job description

| Job requirement | What forces you to learn it here |
|---|---|
| Python | Log parsing, domain modelling, async I/O, workers, tests |
| APIs and backend systems | The whole product is a versioned REST API with auth, validation, and a database |
| AI applications | LLM diagnosis, structured output, RAG over runbooks, guardrails, evaluation |
| Cloud-native solutions | Containers, CI/CD, queues, cache, scaling, monitoring |
| Azure | Deployed on Azure Container Apps with managed Postgres, Blob, Key Vault, Monitor |
| Working with data scientists / product | Evaluation datasets, quality metrics, measurable user journeys |

A todo app teaches none of the AI parts. A "chat with your PDF" demo teaches none of the backend parts. This one requires both.

## How the product grows

| Stage | What exists at the end of it |
|---|---|
| M1 | A typed Python package + CLI: create an incident, attach a log, get **rule-based** findings. No AI, no DB, no API. |
| M2 | Same behaviour, now with async boundaries, real tests, config, and basic CI |
| M3 | It's an HTTP service (FastAPI) with a versioned contract — but state is still in memory |
| M4 | PostgreSQL persistence, transactions, migrations — restarts stop losing data |
| M5 | Multi-tenant auth: users, tenants, roles, secure log upload |
| M6 | **First LLM diagnosis** end-to-end, marked experimental, with token/latency logging |
| M7 | Structured output + schema validation + retries + timeouts + fallback/abstain |
| M8 | RAG: your runbooks ingested, chunked, embedded, retrieved, cited |
| M9 | Evaluation datasets, guardrails, regression gates, human escalation |
| M10 | Background workers, queue, cache, bounded read-only tools |
| M11 | Observability (traces/metrics/logs), Docker, full CI/CD |
| M12 | Deployed on Azure with IaC, scaling, alerting, rollback, failure drills |

**The rule:** each milestone changes the *architecture*, not just adds a feature. You should be able to draw the difference between M(n-1) and M(n) on a whiteboard.

## The one habit that makes this work

For every non-trivial decision, write a short ADR in `architecture/decisions/`:

```
Context   — what problem forced a decision
Options   — the 2–3 realistic alternatives
Decision  — what you chose
Consequences — what you gave up
Revisit when — the measurement that would change your mind
```

Interviews are won on these five lines, not on the code.
