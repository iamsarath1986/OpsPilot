# Milestone 1 — The Domain Model and Deterministic Log Analysis

**Goal:** OpsPilot exists as a typed, tested Python package with a CLI that can create an incident, attach a log file, and produce **rule-based** findings — with no AI, no database, no web framework.

**Why start here and not with FastAPI or an LLM:** every later layer (API schemas, DB tables, prompts, evaluation cases) is a *projection of the domain model*. If "what is an Incident, what is a Finding, what is Evidence" is unclear now, you will re-litigate it at every milestone. Starting with the LLM instead produces a demo you cannot test, cannot persist, and cannot explain.

**Time:** 3–4 focused weeks. **Prerequisite:** none.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| Python data model: iterators, generators, comprehensions | A 500 MB log file must not be loaded into memory to be parsed. Generators give you streaming for free. |
| Context managers (`with`, `contextlib`) | Files and, later, DB sessions and HTTP clients must close even when an exception is raised mid-parse. |
| `dataclasses`, `enum`, frozen types | Dictionaries permit invalid states (`{"serverity": "hihg"}` is a valid dict). Typed entities make illegal states unrepresentable. |
| Composition over inheritance, dependency injection | You will swap the in-memory store for PostgreSQL in M4 and a rule engine for an LLM in M6. Anything hard-wired now becomes a rewrite later. |
| `typing` + a static checker (mypy or pyright) | Types are contracts between modules. They catch integration mistakes without running the program — but they do **not** validate untrusted input (that's Pydantic's job in M3). |
| Custom exception hierarchies | `except Exception` destroys information. Callers need to distinguish "unsupported log format" from "incident already closed". |
| `pytest`: assertions, fixtures, `parametrize` | Deterministic logic is cheap to test. Build the habit before AI makes correctness fuzzy. |
| Packaging: `src/` layout, `pyproject.toml`, editable install | "Works on my machine because of `sys.path`" is not shippable. |
| Git: small commits, branches, meaningful messages | Your commit history is portfolio evidence that the architecture *evolved* rather than appeared. |
| Regex — and its limits | Log parsing is the classic case where regex is right; also where it silently fails on a format variant. |

**Not in this milestone, on purpose:** FastAPI, SQL, Docker, any LLM. Adding them now would hide the domain behind a framework.

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Python core | **Implement / Debug** | You write a streaming parser without copying from a tutorial, and can read a traceback to the real cause |
| Generators vs lists | **Design** | You can state *when* eager evaluation is correct and when it exhausts memory |
| OOP / dependency injection | **Design** | You can explain why the CLI passes a repository *in* rather than importing it |
| Type hints | **Implement** | All public functions typed, checker clean, and you can say where types stop protecting you |
| Exceptions | **Design** | Your error taxonomy maps cleanly onto HTTP status codes later — without you having thought about HTTP |
| pytest | **Implement** | You test invalid and boundary input, not just the happy path |
| Git | **Implement** | You can rebase/resolve a conflict and explain a commit before pushing |

## 3. Hands-On Learning Exercises

Do these in `experiments/` — small, throwaway, *not* part of the package. Keep the notes; they are interview material.

1. **Memory: list vs generator.** Generate a 500 MB synthetic log. Parse it once returning `list[str]`, once with `yield`. Measure peak RSS with `tracemalloc` or `/usr/bin/time -v`. Write down the two numbers. This is the single clearest demonstration of why generators exist.
2. **Dict vs dataclass.** Model an incident both ways. Then list every invalid state each permits (typo'd key, missing field, severity `"urgent"` when your enum only has 4 values). Count them.
3. **Injected clock.** Write a service that stamps `created_at`. First call `datetime.now()` directly, then inject a clock. Try to write a deterministic test for both. Only one is possible.
4. **Break the parser on purpose.** Feed it: a UTF-16 file, a line with no timestamp, a truncated final line, a 2 MB single line, and a file that is actually gzip. Record which exception each produces and whether the message tells you what happened.
5. **State machine.** Write down every incident status transition. Then try to move `closed → analyzing` and confirm your code refuses it.
6. **Regex limits.** Parse nginx error logs with regex. Then feed it an Apache line and a JSON-formatted log. Note exactly how it fails — silently returning nothing is worse than raising.

## 4. Project Implementation

Build in this order. Each step is a commit.

### Step 1 — Repository skeleton

```
opspilot/
├── pyproject.toml
├── src/opspilot/
│   ├── __init__.py
│   ├── domain/          # entities, enums, errors — zero I/O, zero imports from other layers
│   ├── application/     # use cases + ports (protocols)
│   ├── adapters/        # in-memory repo, file reader, CLI
│   └── parsing/         # log format parsers + rules
├── tests/unit/
├── experiments/
├── samples/             # synthetic log excerpts; never a real customer log
├── corpus/runbooks/     # the retrieval corpus (Milestone 8) — written in parallel, not by this milestone
├── evals/faults/        # fault scripts that generate reproduced incidents and the samples above
└── architecture/decisions/
```

The dependency rule: `domain` imports nothing from the others. `application` imports `domain`. `adapters` imports both. Never the reverse. Enforce it by trying to write a domain test that needs no filesystem — if you can't, the layering leaked.

**Two directories named "runbooks" would be a bug waiting to happen**, so they are separated from the start: `corpus/runbooks/` holds the *retrieval corpus* that Milestone 8 ingests and the model cites, while a top-level `runbooks/` — created in Milestone 4 — holds *operational* procedures for the people running OpsPilot (restore drills, alert responses). Ingestion must never pick up the latter.

**Reconciling the repository as it stands.** The project currently carries a root `requirements.txt` and a `note.txt` describing an `app/main.py` FastAPI layout. Resolve both now rather than living with two conventions:

- **Dependencies move into `pyproject.toml`** with optional groups — a minimal runtime group, and a `dev` group. This milestone needs roughly `pytest`, `ruff`, and `mypy` (or `pyright`) and nothing else; FastAPI, SQLAlchemy, and Alembic arrive with Milestones 3 and 4 and are pinned then. Install with `pip install -e ".[dev]"`. A lockfile follows in Milestone 2.
- **Install the static checker before writing the domain model**, not at the end. The checklist below requires it clean, and retrofitting types to code written without it is the slow path.
- **`note.txt` is personal scratch, not project guidance.** Its `app/main.py` layout is superseded by the tree above; keep the file if the command snippets are useful to you, but the roadmap is the authority on structure.

### Step 2 — Domain model

| Concept | Engineering problem | Implementation | Testing |
|---|---|---|---|
| `Severity`, `IncidentStatus` enums | Free-text status values drift (`"open"`, `"Open"`, `"OPEN"`) | Enums with explicit allowed transitions | Assert an illegal transition raises |
| `Incident` entity | Ad-hoc dicts permit invalid states | Frozen dataclass: `id`, `tenant_id`, `title`, `description`, `status`, `severity`, `created_at` | Invariants: title non-empty, severity in enum |
| `LogArtifact` | A log is not a string — it has origin, size, encoding, line count | Dataclass with metadata + a reference to content | Reject zero-byte and oversized artifacts |
| `Evidence` | A finding with no source line is unverifiable | `id`, `artifact_id`, `line_number`, `text` — stable IDs | IDs unique within an analysis |
| `Finding` | Rules and the model must produce *the same* claim type, or they cannot be compared | One claim about the incident: `id`, `category`, `severity`, `confidence`, `statement`, `evidence_ids`, `produced_by` | One rule → deterministic finding; a finding with no evidence IDs is rejected |
| `Analysis` | Results need a container with provenance | `incident_id`, `findings`, `recommended_steps`, `produced_by` (`"rules@v1"`), `created_at` | Same input → identical analysis |

**Design this now with M6 and M7 in mind — this is the decision that is expensive to get wrong.** The model's structured diagnosis in Milestone 7 is not a parallel type; it is *this* `Analysis`, with fields added. The mapping is fixed here and must hold:

| Milestone 7 diagnosis field | Milestone 1 domain | Note |
|---|---|---|
| one entry in `likely_causes` | one `Finding` | `cause` → `statement`; `confidence` and `evidence_ids` are already on `Finding` |
| `recommended_steps` | `Analysis.recommended_steps` | Rules may populate it too; a rule-based fix is still a recommended step |
| `severity` | `Analysis.severity`, derived from its findings | One rule severity per finding, one rolled-up severity per analysis |
| `summary` | added in Milestone 7 | Rules do not write prose; the field is nullable from the start |
| `citations`, `needs_escalation`, `unsupported`, `unsupported_reason` | added in Milestones 7–8 | Additive; they do not reshape `Finding` |

So a rule finding and a model cause are the *same* type distinguished by `produced_by` (`rules@v1` versus a model identity). That is what makes the Milestone 6 baseline comparison a field-by-field diff rather than a translation exercise, and what lets Milestone 7's validation layers check both producers with one set of rules. Record it in ADR-001.

### Step 3 — Error taxonomy

```python
class OpsPilotError(Exception): ...
class InvalidIncidentState(OpsPilotError): ...
class UnsupportedLogFormat(OpsPilotError): ...
class ArtifactTooLarge(OpsPilotError): ...
class IncidentNotFound(OpsPilotError): ...
```

Each maps to an HTTP status in M3 and to a user-facing message. Never `raise Exception("bad")`.

### Step 4 — Parsers

Support **two** formats only (nginx error log, PHP-FPM), plus a generic fallback. Two is enough to force a real abstraction; five is busywork.

- A `LogParser` protocol: `def parse(self, lines: Iterable[str]) -> Iterator[ParsedLine]`
- Streaming — accept an iterable, yield results. Never `readlines()`.
- Format detection that *fails loudly* (`UnsupportedLogFormat`) rather than returning an empty result.

### Step 5 — Deterministic rules

Start with four rules drawn from real hosting incidents:

| Rule | Trigger | Category |
|---|---|---|
| `permission-denied` | `(13: Permission denied)` / `open() ... failed` | permissions |
| `disk-full` | `No space left on device` | capacity |
| `upstream-timeout` | `upstream timed out` / `504` | performance |
| `php-fatal` | `PHP Fatal error` | application |

Each rule emits a `Finding` with the exact evidence lines. **These rules are your permanent baseline** — in M6 you will ask "does the LLM beat this?", and in M9 that comparison becomes an evaluation metric. Without a baseline, "the AI works well" is unfalsifiable.

### Step 6 — Repository port + in-memory adapter

```python
class IncidentRepository(Protocol):
    def add(self, incident: Incident) -> None: ...
    def get(self, incident_id: str) -> Incident: ...
    def list_for_tenant(self, tenant_id: str) -> list[Incident]: ...
```

Implement `InMemoryIncidentRepository`. Write **contract tests** against the protocol — in M4 the exact same tests will run against PostgreSQL. That's the payoff for the abstraction.

### Step 7 — CLI

```
opspilot incident create --title "500 after deploy" --tenant acme
opspilot incident attach <id> --file samples/nginx-error.log
opspilot incident analyze <id>
opspilot incident show <id>
```

The CLI is an *adapter*. It parses arguments and prints; it contains no business logic. Prove it: your use-case tests should never touch the CLI.

Use synthetic sample logs in `samples/`. Never commit a real customer log.

## 5. Expected Project State

**Before:** nothing but Markdown files.

**After:** an installable package (`pip install -e .`) with a working CLI, a clean three-layer architecture, a deterministic analysis pipeline, and a test suite that runs in under a second.

**Architecturally:** you have a domain core with I/O pushed to the edges. There is no network, no persistence (restart = data gone), no auth, no AI. Every one of those is a deliberate gap that a later milestone fills — and you can now name what each one will cost you.

## 6. Testing and Validation

- **Unit tests** for every rule, parametrized across matching, non-matching, and near-miss lines.
- **Parser tests** across both formats plus malformed input, wrong encoding, empty file, and a truncated last line.
- **State-transition tests**: every legal transition passes, every illegal one raises the specific error.
- **Contract tests** for `IncidentRepository`, written against the protocol so they are reusable in M4.
- **Determinism check**: run the same analysis 100 times, assert byte-identical output. (Note this — you lose it in M6, and understanding *why* is the point.)
- Run `mypy`/`pyright` and `ruff` locally. Treat a type error as a failing test.
- Use coverage as a **diagnostic**, not a target. 100% coverage of code with no assertions proves nothing.

**Manual verification of the test suite itself:** delete one validation branch on purpose. If no test fails, your tests are decorative. Restore it and add the missing test.

## 7. Failure Scenarios to Reproduce Deliberately

Reproduce each, diagnose it from the traceback alone, then fix or handle it. Write one paragraph per failure in `architecture/notes/`.

1. Parse a 500 MB log with an eager parser → watch memory climb; fix with a generator.
2. Feed a UTF-16 or latin-1 file to a UTF-8 reader → `UnicodeDecodeError`. Decide: reject, or decode with `errors="replace"`? Justify it.
3. Malformed timestamp on line 4,001 of 10,000 → does the whole parse die, or does it skip and record? (Production answer: skip, count, and surface the count.)
4. Two findings generated with the same evidence ID → downstream citation becomes ambiguous. Add the uniqueness invariant.
5. `closed → analyzing` without an explicit reopen → must raise `InvalidIncidentState`.
6. A log line 2 MB long (no newlines) → your "read line by line" assumption breaks. Add a line-length cap.
7. A gzip file renamed to `.log` → your detector must fail loudly, not return zero findings.

The lesson to internalize: **a system that silently returns nothing is worse than one that fails.** Zero findings and "I could not read this file" look identical to the caller unless you make them different.

## 8. Skills Gained

- Modelling a real operational domain in typed Python, with I/O pushed to the boundaries
- Streaming data processing and the memory reasoning behind it
- Designing an error taxonomy that survives contact with an HTTP API
- Dependency inversion via protocols — the mechanism that will let you swap in PostgreSQL and an LLM without touching business logic
- Writing tests that catch real defects, including on malformed input
- Git discipline and the ADR habit

**Job-description mapping:** this is the foundation of the *Python* requirement, and — less obviously — of the *collaboration* requirement: layered code, typed contracts, and ADRs are what make your work reviewable by other engineers.

## 9. Milestone Completion Checklist

Do not advance because the CLI runs. Advance when you can tick every box honestly.

- [ ] I can explain generators, context managers, composition over inheritance, and dependency inversion **without looking anything up**.
- [ ] I wrote every line of the domain model, parsers, and rules myself.
- [ ] I can state the memory difference between my eager and streaming parsers **with the numbers I measured**.
- [ ] Every public function is typed and the static checker passes clean.
- [ ] Tests cover valid, invalid, boundary, and illegal-state-transition cases.
- [ ] I reproduced at least five of the seven failure scenarios and diagnosed each from the traceback.
- [ ] I deleted a validation branch and a test failed.
- [ ] I can swap `InMemoryIncidentRepository` for a different implementation without editing any domain code — and my contract tests prove it.
- [ ] I can explain one alternative to my layering (e.g. a flat script, or an active-record ORM model as the domain) and why I rejected it.
- [ ] A rule finding and a model-produced cause are the **same type**, distinguished only by `produced_by` — and I can point at the Milestone 7 field mapping that stays true.
- [ ] **ADR-001** records the modular-monolith + layered-domain decision, the shared claim shape, the alternatives, and what evidence would make me change them.
- [ ] My rule-based analysis is frozen as the **baseline** the LLM must later beat, and I know how I'd measure that.

---

**Next:** Milestone 2 turns this into production-grade Python — async I/O boundaries, real test architecture, typed configuration, profiling, and a first CI pipeline. Ask for it when the checklist above is genuinely green.
