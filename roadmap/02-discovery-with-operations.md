# Discovery: Talking to the Operations Team

*The goal of discovery is not to collect feature requests. It is to find the requirements and constraints that decide the architecture — before you build it.*

## Why this comes before more code

Three answers from ops will change your design:

| Question | If the answer is… | Consequence for the build |
|---|---|---|
| May customer log data leave our infrastructure? | **Answered: no** — see [ADR-000](01-the-binding-constraint.md) | No hosted LLM API. Inference is self-hosted. This was decided by the CEO on 2026-08-21 and constrains every later decision. |
| Do written runbooks exist? | No, it's tribal knowledge | Milestone 8 (RAG) has nothing to retrieve. You must first *create* a small runbook corpus — which is itself valuable to the employer, and a legitimate deliverable. |
| What does a wrong answer cost? | An engineer runs a destructive fix on a customer account | Abstention threshold goes up, escalation becomes the default, and "confidence" must be calibrated, not decorative. |

Everything else refines the product. These three define it.

## What shared hosting operations teams typically struggle with

Go in with hypotheses and ask them to correct you — it is far more productive than an open "what are your problems?" Expect some subset of:

**Ticket volume and repetition**
- The same 10–15 issue types make up the large majority of tickets: site down / 500 error, email delivery and deliverability, SSL renewal, disk or inode quota exceeded, database connection limits, "my site is slow", compromised/defaced WordPress, outbound spam and IP blacklisting, DNS propagation confusion, migration requests, PHP version incompatibility, `.htaccess` mistakes, cron jobs not firing, backup/restore requests, FTP and file-permission problems.
- Tier-1 engineers re-derive the same diagnosis repeatedly because the reasoning was never written down.

**Knowledge and consistency**
- Runbooks are missing, outdated, or scattered across a wiki, Slack threads, and old tickets.
- Two engineers give two different answers to the same ticket.
- New hires take months to become productive; onboarding is shadowing, not documentation.

**Diagnosis friction**
- Finding the right log is half the work: per-account logs, server-level logs, mail logs, and control-panel logs all live in different places, and often need SSH or WHM access that tier-1 may not have.
- Logs are noisy; the one meaningful line is buried in thousands of routine ones.

**Shared-hosting-specific pressures**
- Noisy-neighbour resource contention (CPU, entry processes, I/O limits) where the cause is another account entirely.
- Compromised accounts and malware — high urgency, high skill requirement, and a customer who is upset.
- Outbound spam causing IP or range blacklisting, which affects every customer on the server.
- Customers with no technical background describing symptoms inaccurately.

**Process and people**
- SLA pressure on first response, especially on night and weekend shifts when the experienced engineers are asleep.
- Escalation to tier-2/tier-3 for things tier-1 could handle with better information.
- Repeat contacts because the first answer didn't actually resolve it.
- Multilingual customers and machine-translated tickets.

**Constraints you must respect**
- Customer data in logs: IPs, email addresses, file paths containing names, occasionally credentials in query strings.
- GDPR or regional data-residency obligations; possibly PCI scope for billing systems.
- Existing tooling that anything new must fit alongside rather than replace: WHM/cPanel, Plesk or DirectAdmin, CloudLinux LVE, Imunify360, LiteSpeed, JetBackup, and a ticketing system such as WHMCS, Zendesk, or Freshdesk.

## The question bank

Use in a conversation, not an email. Aim to cover sections 1–5; the rest can follow.

### 1. Volume and mix
- Which ticket categories are the highest volume? Which are the most *repetitive* to diagnose (as opposed to merely frequent)?
- What does the queue look like on a bad day versus an average one? Any seasonality — renewals, migrations, campaign traffic?
- Which tickets do you personally get tired of seeing?

### 2. Where the time goes
- On a typical 500-error ticket, roughly how long until you know the cause? What is the slow part — locating the log, interpreting it, or knowing the fix?
- Which tickets take the longest overall, and why?
- What percentage of tickets does tier-1 resolve without escalating? What triggers an escalation — missing access, missing knowledge, or missing authority?
- How often does a ticket come back because the first fix didn't hold?

### 3. Current workflow and tooling
- Walk me through a ticket end to end, from arrival to closure. Who touches it, and what do they open?
- Which systems do you have to log into during one ticket?
- Does tier-1 have log access directly, or does that require escalation?
- What is already automated, and what has been tried and abandoned? (Ask this one explicitly — it prevents you rebuilding something that failed for a non-technical reason.)

### 4. Knowledge
- Where do runbooks live today? How current are they?
- When someone learns something useful on a ticket, where does it go?
- What do you find yourself re-explaining to newer colleagues every week?
- If a new engineer joined on Monday, what would they get wrong on their first shift?

### 5. Data and compliance — ask before assuming
- May log excerpts be copied out of the production environment for tooling? Under what conditions?
- What is in our logs that must never be exposed: customer IPs, email addresses, file paths, tokens in URLs?
- Are we permitted to send sanitised log text to an external AI service, or must processing stay inside our own infrastructure or a specific region?
- Who signs off on that decision — you, security, or legal?
- Is there an existing anonymisation or retention standard I should follow rather than inventing one?

### 6. Quality tolerance and risk
- What would a confidently wrong suggestion cost us in the worst realistic case?
- Would you rather the tool answer 60% of tickets very reliably, or 90% with occasional confident mistakes?
- Which ticket categories should it never attempt — compromised accounts, billing, anything involving deleting data?
- Who is accountable when a suggestion is followed and turns out to be wrong? (The honest answer shapes how the output is worded and presented.)

### 7. What "good" looks like
- What would make you actually use this on a live shift instead of ignoring it?
- What would make you stop using it after one bad experience?
- If it worked, what would you want it to do next — and what should it never do?
- What number would convince your manager it was worth continuing: handle time, escalation rate, first-response time, repeat-contact rate?

### 8. Integration and access
- Which ticketing system, and does it have a usable API?
- Would you want this inside the ticket view, or as a separate tool to paste into? (Separate is far cheaper to build first, and answers "would anyone use it" before you invest in integration.)
- Is there SSO we should authenticate against?
- Would read-only API access to account metadata — service status, PHP version, disk usage — be feasible and permitted?

### 9. The pilot
- Could we trial it on one ticket category for two weeks?
- Who on the team would be willing to give feedback on the answers?
- Could you describe, from memory and without customer details, a few cases you solved recently — the symptom, what misled you, and the actual cause? (These become `scenario` evaluation cases. Nothing is exported.)
- Roughly what share of technical tickets falls into each category? (Numbers only. They weight the evaluation slices by real frequency.)

**Do not ask for real tickets or log samples.** They will not be released, and the project does not need them: the evaluation dataset and the runbook corpus are both authored by us — see [Data we create ourselves](01-the-binding-constraint.md#data-we-create-ourselves) in ADR-000. What ops can usefully contribute costs them minutes and involves no customer data: remembered scenarios, the ticket mix as percentages, and reviews of runbooks we have written.

## Workstream: write the runbook corpus ourselves, starting now

Written runbooks may or may not exist internally; it does not matter for the schedule, because we author our own. If ops later shares existing material, it improves the corpus rather than unblocking it. Start early anyway: 20–30 careful runbooks take calendar time, and Milestone 8 needs them.

1. Prioritise by ticket volume × repetitiveness × risk. Begin with the categories behind the Milestone 1 deterministic rules, then the rest of the top ticket types.
2. Use one template: scope, symptoms, evidence to collect, diagnosis branches, safe next steps, escalation conditions, author, review status, version, and source references. Write them into **`corpus/runbooks/`** — the retrieval corpus — never into the top-level `runbooks/`, which is for operating OpsPilot itself.
3. Track each runbook through `drafted → unreviewed → ops-reviewed (optional) → versioned`. Record author, reviewer, and review date.
4. Write diagnosis branches and safe steps yourself. A model may help with formatting only — a model-written runbook retrieved by a model is the model citing itself.
5. Keep customer data out entirely. Examples are synthetic or come from reproduced incidents.
6. Offer ops the review, not the writing: "is this how we handle it?" takes minutes. It fits the training-questions channel they already opened.

The corpus-readiness gate for Milestone 8 is: 20–30 runbooks written to the template, versioned, with review status recorded, and broad enough to support at least 40 labelled retrieval questions. Ops review is **not** required to pass the gate; it changes what the answers may claim, not whether engineering can proceed.

## Workstream: build the evaluation cases ourselves

Evaluation cases are written in parallel with, but independently from, the runbooks. Follow the six rules in [ADR-000](01-the-binding-constraint.md#the-risk-grading-your-own-homework) from the first case. The most important habits:

- **Reproduce before you write.** Script a fault on a local stack, capture the logs it really produces, and record the injected fault as the ground truth.
- **Write the ticket text as a customer would**, never in runbook vocabulary.
- **Freeze a held-out split early** and do not look at its results while tuning.
- **Keep a fault library** in the repository (`evals/faults/`): one script per failure, so every reproduced case can be regenerated after a stack or log-format change.

## Running the conversation

- 25 minutes, not an hour. Bring five questions; let their answers redirect you.
- Ask for the last ticket they closed and walk through it together. Concrete beats abstract every time.
- Ask "what did you have to know that isn't written down anywhere?" — that sentence is the specification for your runbook corpus.
- Take notes on *their* vocabulary. If they say "entry processes" or "LVE limits", those words belong in your domain model and your prompts.
- Do not promise automation of fixes. Say read-only, repeatedly. Ops teams have well-earned scar tissue about tools that change things.
- Close by asking who else you should talk to, and whether they'd want to see a demo in two weeks.

## After the conversation

Write up, in this order:

1. The top 5 ticket types by volume × repetitiveness — these become your Milestone 1 deterministic rules and your first prompt scope.
2. The data-handling constraint, in writing, with who approved it — this becomes an ADR and drives your Milestone 12 hosting/region decision.
3. What runbook content exists versus what you must write — this sizes Milestone 8.
4. The cost of a wrong answer — this sets the abstention and escalation policy in Milestone 9.
5. The one metric ops said would convince them — this becomes the headline of your final report, and of the project section on your CV.

## Findings so far — reply from operations, 2026-08-21

First contact was a short written question about how a "500 Internal Server Error" ticket is handled. The operations manager's reply established the following.

**Confirmed**

- Intake is by **phone and email**. Tickets are then triaged **by a human** into Accounting, Legal, Tariff, or Technical. Categorisation happens before anyone opens a log.
- First-level support diagnoses using **"FTP analysis or available logs"**. The manual log-hunting hypothesis is correct, and it is the documented normal path, not a fallback for hard cases.
- First-level has two possible outcomes: **give the customer a hint about where the error might be**, or fix it directly.
- Escalation ladder: **first-level support → next support level → Technology / Data Centre / Programmers.**
- **No internal diagnostic application was mentioned.** The question was asked directly and the answer described only people, FTP, and log files. Silence is not proof — confirm explicitly — but nothing suggests tooling sits between the engineer and the log file.

**What this changes**

- **The user is confirmed:** first-level support, mid-ticket, with log text in front of them. Not the customer, not the programmers.
- **The value hypothesis sharpens.** Not "answer the ticket", but two measurable things: (a) more technical tickets resolved at first level, and (b) when escalation does happen, it carries a complete picture instead of "customer says 500". (b) is the escalation packet in Milestone 9, it is the safer win, and it has value even when the diagnosis is wrong — the human above still gets a head start.
- **"Give the customer a hint about where the error might be" is the output specification**, phrased by the people who do the job. A located, evidenced pointer — not a fix, not a guess. That sentence belongs in the Milestone 6 prompt and constrains the Milestone 1 domain model.
- **The headline metric is the first-level resolution rate.** That is the number a manager funds a project against, and the number the final report should lead with.

**Still open, highest value first**

1. First-level resolution rate, and what actually triggers escalation — access, knowledge, or authority.
2. Repeat-cause distribution: is there a top five? (Milestone 1 rules, Milestone 9 eval slices.)
3. Whether written runbooks exist internally. No longer blocking — we author our own corpus — but existing material would improve it.
4. The precise scope of the confirmed data boundary: what counts as company infrastructure, and whether runbook IP is covered as well as customer logs. ADR-000 already blocks hosted inference; these details refine Milestones 5, 8, and 12 without reopening that decision.
5. Whether any diagnostic tooling exists that the reply did not mention.
6. Cost of a confidently wrong suggestion. Sets the abstention policy.

**The channel that opened.** Operations offered to fold questions into an onboarding training they are building, and asked for more. This is a better channel than a meeting request: written, invited, and answered by the people who own the workflow. Open questions 1–4 go back through it, framed as training topics rather than requirements gathering.

**Decided: no real tickets.** Real tickets and log samples will not be released, and we will not ask for them. The runbook corpus and the evaluation dataset are both authored by us ([ADR-000](01-the-binding-constraint.md#data-we-create-ourselves)). The consequence is a limit on claims, not a blocker: results describe authored and reproduced cases, never real-ticket accuracy.

**Hardware: resolved for development.** OpsPilot is developed on a local machine with an integrated GPU, using llama.cpp. Server GPUs are needed only for the company deployment in Milestone 12; ask what hardware exists and who owns it when that deployment becomes a real plan, not before.

**Scheduled, not ad hoc: the demo.** The demo is Step 8 of [Milestone 7](milestone-07-reliable-ai-orchestration.md). Its ask is review, not data: ops checks a handful of our runbooks and scenarios, and says which ticket categories matter most. Record the date, attendees, and feedback here.
