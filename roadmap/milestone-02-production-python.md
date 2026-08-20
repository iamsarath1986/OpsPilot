# Milestone 2 — Production Python: Async, Test Architecture, Configuration, and CI

**Goal:** the same CLI behaviour as Milestone 1, now sitting behind explicit async I/O boundaries, a real test architecture, typed configuration, measured performance, and an automated pipeline that runs on every push.

**Why now:** Milestone 3 puts this code behind HTTP, where every request waits on a database and, later, on an LLM that takes two seconds to answer. If you learn concurrency *after* adding a web framework, you will never be able to tell whether a bug is yours or FastAPI's. Learn it while the system is still small enough to hold in your head.

**Time:** 3–4 focused weeks. **Prerequisite:** Milestone 1 checklist green.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| The event loop, coroutines, tasks | One process must handle many requests that are all *waiting*. Async gives you concurrency without a thread per request. |
| `await`, `asyncio.gather`, `TaskGroup` | Ten sequential 200 ms calls take two seconds. Concurrent ones take 200 ms. This is the entire reason async exists in your project. |
| Cancellation and `asyncio.timeout` | A client disconnects or an LLM hangs. Work must stop and clean up rather than run on invisibly, consuming quota. |
| Semaphores / bounded concurrency | Unbounded `gather` over 10,000 log files opens 10,000 file handles and dies. Bounds are not optional in production. |
| GIL, threads, processes, `asyncio.to_thread` | Async does **not** speed up CPU-bound parsing. Knowing which of the three tools fits which workload is a standard interview question and a real design decision here. |
| Blocking the event loop | One `time.sleep` or one big regex in a coroutine stalls *every* concurrent request. You must be able to recognise and prove this. |
| Test layering: unit, component, contract | A test suite you don't trust is worse than none. Layering tells you what each test is actually protecting. |
| Fakes versus mocks | Over-mocked tests pass while production burns. A fake adapter tests behaviour; a mock often just asserts you called your own code. |
| `pytest` fixtures, `parametrize`, markers | Fast unit runs on every save; slower integration runs in CI. Markers make that split possible. |
| Typed configuration (`pydantic-settings`) | Settings differ per environment, and a missing variable should fail loudly at startup, not at 2 a.m. inside a request. |
| Structured logging | `print` is invisible in a container. Structured logs are what you will later correlate with traces in Milestone 11. |
| Profiling: `cProfile`, `tracemalloc`, `perf_counter` | Optimising without measuring is guessing. Every performance claim you make later must have a number behind it. |
| Dependency locking | "Works on my machine" is a reproducibility failure. CI and your laptop must resolve the same versions. |
| GitHub Actions basics | Local checks get skipped under deadline pressure. A pipeline makes quality non-optional. |

**Not yet:** web frameworks, databases, Docker. Async here is exercised through file I/O and simulated network calls.

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Event loop and coroutines | **Debug** | You can look at a stalled program and say *why* it stalled |
| Concurrency choice | **Design** | Given a workload, you pick async / threads / processes and defend it in one sentence |
| Cancellation and timeouts | **Implement** | Your cleanup runs on cancellation, and you have a test proving it |
| Test architecture | **Design** | You can justify why a given test is a unit test and not an integration test |
| Configuration | **Productionize** | Invalid config fails at startup with a clear message, and no secret appears in a log |
| Profiling | **Implement** | You have before/after numbers for one real optimisation |
| CI | **Implement** | You can explain every step in your workflow file |

## 3. Hands-On Learning Exercises

1. **The headline experiment.** Simulate ten 300 ms network calls three ways: sequentially, with `asyncio.gather`, and with a thread pool. Record wall time for each. Then repeat with a CPU-heavy parse instead of a sleep and record it again. Those six numbers explain async better than any article — keep them.
2. **Block the loop on purpose.** Put `time.sleep(2)` inside a coroutine that runs alongside nine others. Watch all ten stall. Replace it with `await asyncio.sleep(2)` and watch them overlap. Now do it with a real CPU-bound regex and fix it with `asyncio.to_thread`.
3. **Cancellation cleanup.** Start a task that writes a file, cancel it mid-write, and confirm the partial file is removed. Then remove the `finally` block and watch the partial file survive. That orphan file is a production bug in miniature.
4. **Bounded concurrency.** `gather` 5,000 file reads unbounded — observe file-handle or memory pressure. Add a `Semaphore(20)`. Measure throughput at bounds of 1, 5, 20, 100. Note that throughput stops improving well before 100; that curve is why bulkheads exist.
5. **Fake versus mock.** Write a test that mocks your repository with `unittest.mock` and asserts `add()` was called. Then change the repository's semantics so the code is *wrong* — and watch the mock test still pass. Rewrite with an in-memory fake; it fails, as it should.
6. **Test-order dependence.** Introduce module-level mutable state, then run the suite with `pytest -p no:randomly` and with random ordering. Reproduce a test that passes alone and fails in company.
7. **Profile the parser.** `cProfile` your Milestone 1 analysis on a large log. Find the top cost. Change one thing. Measure again. Record both numbers even if the change made it worse — especially then.

## 4. Project Implementation

### Step 1 — Make the ports async, keep the domain sync

`IncidentRepository`, and later the artifact reader and LLM provider, become `async def`. Pure domain logic — rules, state transitions, validation — stays synchronous. Domain logic does no I/O, so making it async would add ceremony and no benefit.

| Concept | Engineering problem | Implementation | Testing |
|---|---|---|---|
| Async ports | The database and the LLM are both network I/O and will block a request | `async` protocol methods on repository and artifact-store ports | Async fakes, `pytest.mark.asyncio` |
| Sync domain core | Async in pure logic buys nothing and costs readability | Rules and entities stay plain functions | Domain tests need no event loop |
| Offloaded CPU work | Parsing a 200 MB log freezes every concurrent request | Run parsing via `asyncio.to_thread` behind an async facade | Prove the loop stays responsive while parsing |
| Bounded fan-out | Analysing many artifacts at once exhausts handles | `Semaphore`-limited analysis, limit from config | Test at the bound, above it, and on cancellation |
| Deadlines | Work with no time limit runs forever | An `asyncio.timeout` around every port call, translated to a domain error | Assert the domain error, not `TimeoutError` leaking upward |

### Step 2 — Typed configuration

One `Settings` class: log level, max artifact bytes, max line length, analysis concurrency, and a placeholder for the future LLM key. Loaded from environment with `pydantic-settings`, validated at startup, and never read via `os.environ` scattered through the code.

Rules to enforce now, because they get expensive later: secrets have no defaults; `Settings` is constructed once and injected, never imported as a global singleton by business code; and `repr` of settings must never print a secret. Test that last one.

### Step 3 — Structured logging

Replace every `print` with a logger. Log as key–value structured records (JSON in production, human-readable locally). Add a `correlation_id` field now, even though nothing sets it yet — Milestone 3 fills it from a request header and Milestone 11 ties it to a trace. Retrofitting a correlation ID across a codebase is miserable; adding the field early is free.

### Step 4 — Test architecture

```
tests/
├── unit/        # pure logic, no I/O, milliseconds
├── component/   # a use case wired to fakes
└── contract/    # any adapter must satisfy the port's contract
```

Register markers in `pyproject.toml`. `pytest -m unit` must run in under a second, because a slow inner loop is a suite you stop running.

### Step 5 — Continuous integration

A GitHub Actions workflow on push and pull request: install with the lockfile, run the linter, run the type checker, run unit and component tests, and upload coverage as an artifact. Add branch protection so the pipeline must pass.

Then **prove it works**: push a branch that fails the type checker and confirm the merge is blocked. A pipeline you have never seen fail is a decoration.

## 5. Expected Project State

**Before:** a synchronous package with tests.

**After:** the same user-visible behaviour, but every I/O boundary is async and deadline-bounded, CPU work is offloaded, configuration is typed and validated at startup, logs are structured with a correlation field, tests are layered and fast, and CI enforces all of it.

**Architecturally:** nothing became distributed. The system became *ready* to talk to a network without concurrency concerns leaking into business logic. This is the difference between adding async later as a rescue and having it as a boundary from the start.

## 6. Testing and Validation

- Async tests for timeout translation, cancellation cleanup, and bounded concurrency.
- Contract tests now run against both the sync and async in-memory repositories, proving the port abstraction holds.
- A determinism check: run the unit suite ten times in random order and require identical results.
- A memory profile of the analysis path against a large generated log, with the numbers committed to `architecture/notes/`.
- A deliberate mutation: delete a `finally` cleanup and confirm a test fails.
- Confirm no test touches the real clock, the network, or a developer-specific path.

## 7. Failure Scenarios to Reproduce Deliberately

1. **Coroutine never awaited** — call an async function without `await`. Note that Python only warns, and the work silently never happens. This is the most common async bug in real code.
2. **Blocked event loop** — a synchronous parse inside a coroutine stalls concurrent work. Prove it with timing, then fix it.
3. **Unbounded `gather`** — memory climbs until the process dies. Add the semaphore.
4. **Cancellation leaves debris** — partial file, unreleased semaphore, or open handle.
5. **Timeout that doesn't stop work** — wrap a call in a timeout but let the underlying task keep running; observe the resource still held. Understand the difference between "I stopped waiting" and "it stopped working".
6. **Flaky test from shared state** — reproduce, then fix by isolating fixtures.
7. **Missing environment variable** — the app must fail at startup with a message naming the variable, not with a `KeyError` deep in a request.
8. **Secret in a log line** — log the settings object, spot the leak, add a redacting repr, and add the test that would have caught it.

## 8. Skills Gained

- Reasoning about concurrency: choosing between async, threads, and processes from workload characteristics rather than habit
- Deadlines, cancellation, and bounded concurrency — the exact primitives that keep LLM calls from becoming outages in Milestone 7
- A test architecture that stays fast and trustworthy as the system grows
- Twelve-factor configuration and structured logging: the prerequisites for running anything in a container
- Evidence-based performance work

**Job-description mapping:** this is the "Python" requirement at a level distinguishable from tutorial familiarity. Async I/O, resource lifecycle, and test design are precisely what separates someone who can *write* Python from someone who can *operate* a Python backend.

## 9. Milestone Completion Checklist

- [ ] I can explain the event loop, and what "blocking it" actually means, without notes.
- [ ] I have measured numbers for sync vs async vs threads on both an I/O task and a CPU task, and can explain both results.
- [ ] I can choose async, threads, or processes for a described workload and defend the choice.
- [ ] Every port call has a deadline, and timeout errors surface as domain errors.
- [ ] I have a test that fails if cancellation cleanup is removed.
- [ ] Configuration is typed, validated at startup, injected rather than imported, and never logs a secret.
- [ ] `pytest -m unit` finishes in under a second.
- [ ] I replaced an over-mocked test with a fake and can explain what the fake catches that the mock did not.
- [ ] CI runs lint, types, and tests on every push — and I have watched it block a bad branch.
- [ ] One optimisation is justified by a profile, with before and after numbers recorded.
- [ ] **ADR-002** records the async-boundary decision: which layers are async, which stay sync, and why.

---

**Next:** Milestone 3 puts this behind HTTP — a versioned REST contract, Pydantic validation at the boundary, a consistent error format, and the first tests that exercise the system the way a real client will.
