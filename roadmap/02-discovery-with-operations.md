# Discovery: Talking to the Operations Team

*The goal of discovery is not to collect feature requests. It is to find the requirements and constraints that decide the architecture — before you build it.*

## Why this comes before more code

Three answers from ops will change your design:

| Question | If the answer is… | Consequence for the build |
|---|---|---|
| May customer log data leave our infrastructure? | No | No public LLM API. You need a self-hosted model, or a contractual/regional arrangement (e.g. Azure OpenAI in a specific region with no-training terms). Redaction stops being a nice-to-have and becomes the product. |
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

## The email to send

Adapt the recipient's name and your team's tone.

---

**Subject:** 25 minutes on support tickets — building an internal diagnosis assistant

Hi [Name],

I'm building an internal tool that could take some of the repetitive diagnosis work off the support queue, and I'd rather design it around what actually slows you down than guess.

The idea in one line: an engineer pastes a ticket and a log excerpt, and the tool returns a likely cause with the exact log lines it's based on, plus the relevant runbook steps. It's read-only — it never touches a server, never runs commands, and never changes a customer account. When it isn't confident, it says so and hands the ticket back rather than guessing.

Could I borrow 25 minutes with you or someone on the team? The things I most want to understand:

1. Which ticket types come up most often, and which are most repetitive to diagnose?
2. Where does time actually go on a typical ticket — finding the right log, knowing what the error means, or knowing the fix?
3. Where does the team's knowledge live today: written runbooks, the ticket history, or people's heads?
4. What would a wrong or overconfident suggestion cost us? I want to set the "I don't know" threshold conservatively.
5. Are there rules about customer log data — what can be copied, and whether it may be processed outside our infrastructure?

If it's useful, I'd also like around 20 already-closed tickets (anonymised is fine) to test against, so I can measure whether the tool is actually right rather than just plausible.

Happy to fit around a quiet slot in the shift — and happy to show you what I've built so far first if that's more useful than a discussion in the abstract.

Thanks,
[Your name]

---

### Variants

**Writing to the ops manager rather than the team:** add a line on measurement — "if it works, the thing I'd measure is handle time on the top three ticket types, and how often tier-1 escalates for information rather than access." Managers back things that arrive with a number attached.

**If the project isn't formally approved yet:** remove the request for closed tickets from the first email. Ask once there's interest, and route it through your manager. Requesting customer data — even anonymised — lands better after buy-in than before it.

**If you already have something running:** lead with the demo offer instead of the questions. Five minutes of a working tool produces better requirements than thirty minutes of hypotheticals.
