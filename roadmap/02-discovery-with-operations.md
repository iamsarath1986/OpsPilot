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
- Can I have roughly 20 closed tickets with their eventual resolutions, anonymised, to test against?

**That last question matters more than it looks.** Those closed tickets with known resolutions are your evaluation dataset (Milestone 9). Without them, you can only claim the answers look plausible. With them, you can measure how often the system is right, how often it correctly says "I don't know", and whether a change made things better or worse.

Ask for those tickets after the first synthetic-data demo, as planned. They are **not** required to choose the provisional demo model: the bootstrap selection in [ADR-000](01-the-binding-constraint.md) uses 12–20 synthetic, ops-reviewed cases. The approved tickets are required to confirm or replace that model before a pilot or production claim.

## Parallel workstream: author the runbook corpus now

Discovery has not yet established whether a usable written runbook corpus exists; the ops reply described the support workflow but said nothing about documentation. As a deliberately conservative planning assumption, treat operational knowledge as tribal until ops confirms otherwise — assuming a corpus that turns out not to exist costs months, while starting authoring that turns out to be partly unnecessary costs weeks. Do not wait for Milestone 8.

1. Ask the operations manager to name one corpus owner and 2–3 reviewers.
2. Reserve a recurring 45-minute authoring/review session from Milestone 1 onward.
3. Prioritise 20–30 runbooks by ticket volume × repetitiveness × risk; begin with the top five categories used by the deterministic baseline.
4. Use one template: scope, symptoms, evidence to collect, diagnosis branches, safe next steps, escalation conditions, owner, review date, and source references.
5. Track each runbook through `identified → drafted → technical review → approved → versioned`, including reviewer and next-review date.
6. Keep customer data out of drafts. Examples are synthetic unless the approved data-handling process says otherwise.

This is the roadmap's current **longest pole and critical-path risk**. The corpus-readiness gate for Milestone 8 is: 20–30 runbooks approved by ops, stored in a versioned source location, owned, dated, and broad enough to support at least 40 labelled retrieval questions. Escalate missed reviews as a schedule risk; adding engineers does not remove this dependency.

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
3. Whether written runbooks exist or knowledge is tribal. This sizes Milestone 8 — retrieve an existing corpus, or author one with the team.
4. The precise scope of the confirmed data boundary: what counts as company infrastructure, and whether runbook IP is covered as well as customer logs. ADR-000 already blocks hosted inference; these details refine Milestones 5, 8, and 12 without reopening that decision.
5. Whether any diagnostic tooling exists that the reply did not mention.
6. Cost of a confidently wrong suggestion. Sets the abstention policy.

**The channel that opened.** Operations offered to fold questions into an onboarding training they are building, and asked for more. This is a better channel than a meeting request: written, invited, and answered by the people who own the workflow. Open questions 1–4 go back through it, framed as training topics rather than requirements gathering.

**Deliberately not yet asked:** real tickets or log samples. That is a data-protection conversation, it goes through the manager, and it belongs after someone has seen a demo — not bundled into a training request, where it would make an easy ask look like a hard one.

**Start now despite that deferral:** request a corpus owner and recurring runbook-review time. This asks ops to document its own approved procedures, not to release customer tickets or logs, so it can proceed before the demo.

**Also start now: GPU access.** Ask, through your manager, what GPU hardware exists, who owns it, and whether a modest allocation of target-class hardware can be made available before Milestone 6. No customer data is involved, and hardware allocation usually has a longer lead time than the code that needs it. Milestone 6's capacity measurements and model selection are blocked without it.

**Scheduled, not ad hoc: the demo and the ticket request.** The synthetic-data demo is Step 8 of [Milestone 7](milestone-07-reliable-ai-orchestration.md), and the closed-ticket request goes through your manager the same week. Record the date, attendees, feedback, and request status here.
