# Milestone 9 — Evaluation, Guardrails, Safety, and Human Escalation

**Goal:** replace "the answers look good" with a measurement system — versioned evaluation datasets, calibrated graders, per-slice metrics, guardrails on input and output, and a release gate in CI that blocks a prompt or model change that makes things worse.

**Why now:** you have three things that all affect quality (prompt, retrieval, model) and no way to tell which change helped. Every future improvement is guesswork until this exists. This milestone is also the one most explicitly named in modern AI job descriptions.

**Time:** 5–6 focused weeks. **Prerequisite:** Milestone 8 checklist green, with the retrieval harness working.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| Why AI needs evaluation rather than only tests | Deterministic code has one right answer. A diagnosis has many acceptable phrasings and several unacceptable meanings. `assertEqual` cannot express that. |
| Dataset design: representative, boundary, adversarial | A dataset of easy questions proves nothing. The hard cases are where quality actually lives. |
| Dev / regression / held-out splits | Tuning against one set until it passes is overfitting. The held-out set is what keeps you honest. |
| Slices | An aggregate score hides that you got worse at permissions while getting better at email. Aggregates are how regressions ship. |
| Labelling and provenance | Who decided this answer was correct, when, and against which corpus version. |
| Deterministic graders | When the requirement is exact — a severity level, an abstention, a required citation — assert it. Cheapest and most reliable. |
| Retrieval graders | Already built in Milestone 8; now they join the release gate. |
| Groundedness and citation accuracy | Every claim traceable to retrieved text. The operational definition of "not hallucinating". |
| LLM-as-a-judge: rubrics, pairwise comparison | Scales nuanced judgement — with real caveats you must understand before trusting it. |
| Judge bias: position, verbosity, self-preference | Judges prefer the first option, longer answers, and their own model family. Unmanaged, these bias every decision you make. |
| Judge calibration against humans | A judge you have not compared with human labels is an unvalidated instrument. |
| Inter-rater agreement | Two humans disagreeing about a case means the case is ambiguous, not that one is wrong. |
| Noise floor and sample size | Run the same eval three times; the spread is your noise floor. Reacting to a move smaller than it is superstition. |
| Regression gates in CI | Make quality a merge condition, exactly like tests. |
| Input guardrails | Injection attempts, out-of-scope requests, PII, oversized input, abuse. |
| Output guardrails | Schema and evidence (from Milestone 7), plus secret leakage, unsafe recommendations, policy compliance, tone. |
| Destructive-action detection | An ops assistant recommending `rm -rf`, `DROP DATABASE`, or a permission change to 777 is a business risk, not a quality issue. This is domain-specific and deterministic — write the rule. |
| Abstention calibration | Abstaining too often makes the tool useless; too rarely makes it dangerous. There is a measurable trade-off curve. |
| Hallucination detection strategies | Grounding checks, self-consistency across samples, claim–evidence entailment, and their respective costs. |
| Human escalation design | The escalation packet's quality determines whether engineers trust or bypass the tool. |
| The incident-to-regression-case loop | Every escaped failure becomes a permanent test. This is how quality compounds. |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Dataset design | **Design** | You can defend your slices, your sample size, and what your set does *not* cover |
| Deterministic graders | **Implement** | You reach for these first and use a judge only where they cannot work |
| LLM-as-judge | **Debug** | You have measured your judge against human labels and can state the agreement |
| Judge bias | **Understand** | You have demonstrated position bias in your own setup |
| Statistical honesty | **Understand** | You know your noise floor and refuse to act inside it |
| Guardrails | **Productionize** | Input and output guardrails run in production with their trips recorded as metrics |
| Abstention | **Design** | You chose your threshold from a measured precision/recall curve |
| Escalation | **Implement** | An escalated ticket arrives with everything a human needs and nothing they must re-derive |

## 3. Hands-On Learning Exercises

1. **Build the dataset.** 60–100 cases from the sanitised tickets your ops team gave you. Each with: stable ID, input, corpus version, expected retrieved chunks, reference facts, forbidden claims, whether it is answerable at all, slice tags, and risk level. **Include 15 unanswerable cases** — where correct behaviour is abstention. Systems that cannot abstain fail exactly here, and most datasets omit these entirely.
2. **Measure your noise floor.** Run the same eval three times without changing anything. Record the spread. Any future improvement smaller than this is not an improvement.
3. **Calibrate the judge.** Score 30 answers yourself. Have the judge score the same 30. Compute agreement. Read every disagreement — you will usually find your *rubric* is ambiguous, not the judge.
4. **Prove position bias.** Run pairwise comparisons, then run them again with the options swapped. If the winner changes, you have measured a bias that would otherwise have silently driven your decisions.
5. **Regression drill.** Deliberately degrade a prompt — remove the citation requirement. Run the gate. It must block. If it passes, your gate is decorative.
6. **Red team.** 25 adversarial inputs: injection in the ticket text, injection inside a *retrieved runbook* (the harder one), requests for another tenant's data, requests to produce destructive commands, PII extraction attempts, and out-of-scope questions.
7. **Abstention curve.** Sweep the confidence and relevance thresholds. Plot abstention rate against wrong-answer rate. Choose a point deliberately and write down the reasoning — this is a product decision with a safety dimension, and it should be documented as one.
8. **Self-consistency.** Sample the same question three times; measure whether disagreement between samples predicts wrongness. Then decide whether the tripled cost is worth it. Usually it is not — but now you know rather than assume.

## 4. Project Implementation

### Step 1 — The evaluation package

```
evals/
├── datasets/     # versioned JSONL: dev, regression, held-out
├── graders/      # deterministic, retrieval, judge
├── runners/      # parallel, cached, cost-capped
└── reports/      # versioned markdown + JSON
```

The runner must be cost-capped and cache-aware. An eval run that costs $40 and takes an hour is one you will stop running, and an eval you do not run is not a gate.

### Step 2 — Grading hierarchy

| Layer | Method | Applies to |
|---|---|---|
| Deterministic | Exact assertions | Severity correctness, abstention when unanswerable, citation presence, forbidden-claim absence, schema and evidence validity |
| Retrieval | recall@K, precision@K, MRR, nDCG | The retrieval stage in isolation |
| Judge | Versioned rubric, blinded, pairwise where possible | Groundedness, completeness, actionability, tone |
| Human | Sampled review | High-risk slices, ambiguous cases, and judge calibration |

Use the cheapest layer that can answer the question. A judge for something a deterministic check can decide is slower, costlier, and less reliable.

### Step 3 — Guardrails in the request path

| Guardrail | Stage | Behaviour on trip |
|---|---|---|
| Input size and rate | Before anything | Reject with 413 / 429 |
| Injection heuristics | Before prompt construction | Flag, isolate content as data, record the attempt |
| Out-of-scope classifier | Before retrieval | Politely decline; do not burn a model call |
| PII / secret scan | Before the provider call | Redact (Milestone 5 pipeline) and record what was removed |
| Schema + evidence validation | After the model | Repair once, then abstain (Milestone 7) |
| Secret-leak scan | After the model | Strip and record; never return |
| **Destructive-action detector** | After the model | Block the step, escalate to a human |
| Confidence and grounding threshold | After validation | Abstain or escalate |

Every guardrail trip is a counted metric, not a silent branch. In Milestone 11 those counters become dashboard lines, and a spike in injection attempts becomes an alert.

### Step 4 — Escalation

An escalation packet containing: the incident, deterministic findings, retrieved passages with citations, the model's partial diagnosis, the reason for escalation, and the correlation ID. Plus a review endpoint where an engineer resolves it — and their resolution becomes a **candidate** evaluation case, subject to human approval. Never auto-add production data to the dataset.

### Step 5 — The release gate

A CI job that runs a small suite on every pull request touching prompts, retrieval, or orchestration, and the full suite nightly and pre-release. Gate rules:

- **Hard block**, no override: any cross-tenant leak, secret leak, destructive recommendation, or fabricated citation. One instance fails the build.
- **Block on regression** beyond the noise floor in any critical slice.
- **Report, do not block**, on latency and cost changes — but require an explicit decision in the pull request.

Publish the report with dataset size, per-slice results, and known limitations. Never a single context-free accuracy number.

## 5. Expected Project State

**Before:** a grounded system whose quality was a matter of impression.

**After:** quality is measured per slice, changes are compared against a baseline, unsafe outputs are blocked deterministically, uncertain cases abstain or escalate with a useful packet, and CI refuses to merge a regression. Guardrail trips and abstention rates are recorded as production metrics.

**Architecturally:** the system now has a feedback loop. Escaped failures become regression cases, so quality ratchets rather than drifts.

## 6. Testing and Validation

- Grader unit tests — a wrong grader silently corrupts every decision you make afterwards. Test the graders themselves.
- Guardrail tests: true positives, and false positives on legitimate tickets (a guardrail that blocks real work gets switched off).
- The red-team suite in CI, including injection via retrieved documents.
- Abstention tests: unanswerable cases must abstain; answerable ones must not.
- The regression drill: a deliberately degraded prompt must fail the gate.
- Judge stability: same inputs, repeated runs, spread recorded.
- Escalation packet completeness tests.

## 7. Failure Scenarios to Reproduce Deliberately

1. A confident, fluent, wrong answer that passes schema and evidence checks — the failure mode that makes evaluation necessary.
2. Gate blocking on noise because the change was smaller than your measured spread.
3. Judge and human disagreement traced to an ambiguous rubric.
4. Overfitting: tune against the regression set until it looks excellent, then watch the held-out set disagree.
5. Injection embedded in a *retrieved runbook* rather than user input — a supply-chain attack on your own corpus.
6. Guardrail false positive blocking a legitimate ticket about "deleting a customer's files".
7. Over-abstention: thresholds so cautious the tool is useless. Measure it as a failure, because it is one.
8. Escalation with insufficient context, forcing the human to start over.
9. PII appearing in an evaluation report committed to Git.
10. A destructive recommendation (`chmod 777`, `rm -rf`) reaching a response, before the detector exists.

## 8. Skills Gained

- Designing evaluation datasets with slices, adversarial cases, and unanswerable cases
- Choosing the cheapest sufficient grading method, and calibrating judges against humans
- Statistical honesty: noise floors, sample size, and refusing to act on noise
- Input and output guardrails, including domain-specific safety rules
- Abstention as a tuned, measured product decision
- Release gating on AI quality, and the incident-to-regression-case loop

**Job-description mapping:** *AI applications* and *collaboration with data scientists and product teams* — evaluation is the shared language across all three. This is also the milestone that most distinguishes an AI engineer from a developer who has integrated an API.

## 9. Milestone Completion Checklist

- [ ] My dataset has documented slices, adversarial cases, and unanswerable cases, with provenance for each.
- [ ] I know my noise floor and refuse to act on differences smaller than it.
- [ ] I calibrated my judge against my own labels and can state the agreement level.
- [ ] I demonstrated position bias and describe how my setup mitigates it.
- [ ] Deterministic graders handle everything a judge is not needed for.
- [ ] The release gate blocked a deliberately degraded prompt.
- [ ] Cross-tenant, secret-leak, destructive-action, and fabricated-citation failures are hard blocks with no override.
- [ ] My red-team suite includes injection through retrieved documents, and it runs in CI.
- [ ] My abstention threshold came from a measured curve I can show.
- [ ] An escalation packet contains everything a human needs, verified by an actual human.
- [ ] Guardrail trips and abstentions are counted metrics, not silent branches.
- [ ] I can explain how a production failure becomes a permanent regression case.
- [ ] **ADR-009** records the evaluation strategy, grading hierarchy, and gate policy.

---

## Phase II checkpoint

| Job requirement | Evidence after Milestone 9 |
|---|---|
| AI applications | LLM integration, structured output, four-layer validation, RAG with citations, guardrails, evaluation |
| Real-world AI products | Reliability, degradation, abstention, escalation, safety rules, cost and latency measurement |
| Python | Async orchestration, evaluation tooling, layered testing throughout |
| Collaboration with data scientists | Labelled datasets, retrieval metrics, judge calibration, ablation studies |
| Product collaboration | Abstention trade-offs, scope decisions, measured quality reporting |

**Next:** Milestone 10 moves the expensive work off the request path — queues, workers, idempotency, distributed limits, caching, and the first bounded read-only tools.
