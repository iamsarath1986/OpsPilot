# Milestone 5 — Identity, Multi-Tenancy, Secure Uploads, and Redaction

**Goal:** real authentication and authorization, hard tenant isolation, safe file upload, an audit trail, and a redaction pipeline that strips secrets and personal data out of log content before it is stored — let alone sent to a model.

**Why now:** the `X-Tenant-Id` header from Milestone 3 is a security hole with a comment on it. More importantly, self-hosting does not make raw customer logs safe to copy: Milestones 4–11 place derived content in databases, caches, evaluation datasets, developer machines, and telemetry with wider internal access than the source log. Redaction limits that blast radius and supports GDPR least-privilege handling before any of those copies exist.

**Time:** 4–5 focused weeks. **Prerequisite:** Milestone 4 checklist green.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| Authentication versus authorization | Distinct problems, distinct failures. "Who are you?" and "may you do this?" get conflated constantly, and the conflation is what causes data leaks. |
| Password hashing (Argon2/bcrypt) | Fast hashes are crackable at scale. Also: consider whether you should store passwords at all, versus delegating to an identity provider. |
| JWT structure, signing, and validation | Widely used, widely misused. You must know what a JWT does *not* prove. |
| JWT pitfalls: `alg=none`, missing `aud`/`iss` checks, expiry, clock skew, revocation | Each is a real CVE-class mistake. Validating a token badly is worse than not validating it, because it looks safe. |
| Sessions versus stateless tokens | Revocation, logout, and scaling behave differently. There is a genuine trade-off. |
| API keys for machine clients | Your ticketing system integration is not a human and should not use a human's credentials. |
| RBAC versus ABAC | Roles are simple and get you far; attribute rules handle "only incidents for accounts you manage". |
| Tenant isolation strategies | Row-level `tenant_id`, PostgreSQL row-level security, schema-per-tenant, database-per-tenant — a real architecture decision with cost, isolation, and operational trade-offs. |
| Broken Object Level Authorization (OWASP API #1) | The most common serious API vulnerability in the world: a valid user reading someone else's object by changing an ID. |
| Secure file upload | Size limits, declared versus actual type, filename sanitisation, path traversal, decompression bombs, and where the bytes are stored. |
| PII and secret redaction | Hosting logs contain IPs, email addresses, customer names in paths, and sometimes tokens in query strings. |
| Audit logging | Who viewed which customer's incident, and when. Needed for trust, and often for compliance. |
| Rate limiting basics | Brute-force protection on login, and abuse control on expensive endpoints. |
| Secrets handling | Environment variables now; Azure Key Vault and the approved internal secret store in Milestone 12. Never in code, never in logs, never in Git history. |
| Threat modelling (STRIDE) | A structured way to find the attacks you didn't think of, before an attacker does. |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Authn/authz distinction | **Design** | You can point at each layer of your code and say which question it answers |
| JWT validation | **Debug** | You have forged and rejected bad tokens yourself |
| Tenant isolation | **Design** | You can compare four strategies and defend yours on isolation, cost, and operations |
| Object-level authorization | **Productionize** | Cross-tenant access is blocked at a chokepoint, not per-route, and tests prove it |
| Upload security | **Implement** | Your handler survives every abusive file you throw at it |
| Redaction | **Design** | You can state your false-positive and false-negative rates on a test corpus, and what you do about the gap |
| Threat modelling | **Understand** | You have a written model with assets, entry points, abuse cases, and accepted risks |

## 3. Hands-On Learning Exercises

1. **Forge tokens.** Craft a token with `alg: none`. Craft one signed with the wrong key. Take a valid token and change the tenant claim. Take an expired one. Confirm your validator rejects all four — and watch what happens if you disable each check in turn.
2. **IDOR hunt.** Log in as tenant A, request an incident belonging to tenant B by its ID. If you get data, you have found the most common serious API bug in production systems. Fix it at a single chokepoint, then re-run against *every* endpoint.
3. **Where to enforce.** Try enforcing tenant scope in the route, then in the use case, then in the repository. Argue which is safest against a *forgotten* new endpoint. (The answer shapes your architecture: the deepest enforcement is the one a careless future you cannot bypass.)
4. **Abusive uploads.** Send a 2 GB file, a file whose name is `../../etc/passwd`, a `.php` renamed to `.log`, a zip bomb, a file with a UTF-16 BOM, and one with a 50 MB single line. Fix what breaks.
5. **Redaction accuracy.** Build a synthetic corpus of 200 log lines, half containing planted secrets (emails, IPv4/IPv6, bearer tokens, API keys, DB connection strings, customer paths). Run your redactor. Count misses and false positives. Tune, and record the final numbers — you will cite them in Milestone 6 and in interviews.
6. **Timing.** Compare response times for a nonexistent user versus a wrong password. If they differ measurably, you have a user-enumeration oracle.
7. **Threat model.** Spend one hour with STRIDE on the diagnosis flow. Write down the three risks you had not considered before starting.

## 4. Project Implementation

### Step 1 — Decide the identity model, and write it down

Two credible paths: implement local accounts with hashed passwords, or delegate to an OIDC provider (Microsoft Entra ID, which you will meet again in Milestone 12). Local accounts teach you more about the mechanics; delegation is what most employers actually run.

Recommendation: implement local login **first** for the learning, keep token validation as a swappable dependency, then switch to Entra in Milestone 12 and keep both in the repo. The swap itself becomes portfolio evidence that you understand the boundary.

### Step 2 — Core implementation

| Concept | Engineering problem | Implementation | Testing |
|---|---|---|---|
| Users, roles, memberships | Access must be attributable to a person | `users`, `roles`, `tenant_memberships` tables; Argon2 hashing | Hash verification; no password or hash ever logged or returned |
| Token issuance and validation | Every request must prove identity | `POST /v1/auth/token`; a FastAPI dependency validating signature, `exp`, `iss`, `aud` | Forged, expired, wrong-audience, and wrong-signature tokens all rejected |
| Authenticated principal | Downstream code needs identity, not a header | A `Principal` (user, tenant, roles) injected into use cases; the `X-Tenant-Id` header is deleted | Test that the header is now ignored entirely |
| Tenant scoping at the chokepoint | A future endpoint will forget to filter | Repositories require the tenant from the principal; no query path can omit it | A test that enumerates routes and asserts cross-tenant access fails on each |
| Role checks | Not everyone may close an incident or view audit logs | A permission policy module, e.g. `viewer`, `engineer`, `admin` | A full authorization matrix test: role × endpoint × expected status |
| Secure upload | Files are the classic attack surface | Streamed upload with a hard byte cap, extension and sniffed-type check, generated storage name, quarantine path; stored via the storage port | Every abusive file from exercise 4 |
| Redaction pipeline | Secrets must not be stored or sent onward | Redact on ingest, before persistence: emails, IPs, tokens, keys, connection strings, home paths. Keep a redaction report per artifact | Corpus test with measured miss rate; a test asserting the raw original is never persisted |
| Audit log | Access to customer data must be traceable | Append-only `audit_events`: actor, action, resource, tenant, timestamp, request ID | Reads and writes on customer data create events; the log is not modifiable via the API |
| Login rate limiting | Credential stuffing is constant and automated | Per-account and per-IP attempt limits with lockout and `Retry-After` | Boundary and lockout-expiry tests |

### Step 3 — Choose and record the isolation strategy

Start with `tenant_id` on every table plus enforcement in the repository layer. Then evaluate PostgreSQL row-level security as defence in depth, and write the ADR comparing all four strategies. Note what would push you to a stronger one: a compliance requirement, a very large customer, or an incident.

### Step 4 — Threat model document

`architecture/threat-model.md`: assets (customer log content, credentials, tenant metadata), entry points, trust boundaries, abuse cases, mitigations, and **accepted risks with reasons**. The accepted-risks section is what makes it credible rather than performative.

## 5. Expected Project State

**Before:** an unauthenticated API that trusted a header.

**After:** every request is authenticated; every data access is tenant-scoped at a chokepoint and role-checked; uploads are bounded and sanitised; log content is redacted before storage; sensitive actions are audited; and there is a written threat model with a known residual-risk list.

**Architecturally:** OpsPilot is now safe enough to hold approved customer data — which is the precondition for any later in-company pilot on live tickets, and for Milestone 6 sending redacted content to the self-hosted inference service without widening internal access unnecessarily.

## 6. Testing and Validation

- **Authorization matrix tests**: every role against every endpoint, asserting the expected status. This is the single highest-value test suite in the project.
- Negative auth tests: missing, malformed, expired, forged, and wrong-audience tokens.
- Cross-tenant tests for every resource-returning endpoint, driven by route enumeration so new endpoints are covered automatically.
- Upload abuse suite.
- Redaction corpus test with an asserted maximum miss rate, run in CI.
- An assertion that no log line, error body, or audit record contains a raw secret or password.
- Restore the Milestone 4 drill with the new tables included.

## 7. Failure Scenarios to Reproduce Deliberately

1. Cross-tenant read via a guessed ID (IDOR) — before and after the chokepoint fix.
2. `alg=none` and wrong-signature tokens accepted, by disabling a check, then re-enabled.
3. An expired token accepted because of unbounded clock skew.
4. A new endpoint added *without* tenant scoping — confirm your enumeration test catches it. If it doesn't, the test is the bug.
5. Path-traversal filename writing outside the storage directory.
6. Upload that fills the disk; then the same request with the byte cap in place.
7. A secret in a log line that redaction missed — trace how it reached the log, and add both a redaction rule and a test.
8. Privilege escalation: a `viewer` calling an `admin` endpoint.
9. Brute-force login until lockout, then confirm legitimate access recovers.
10. A JWT signing key rotation with old tokens still in flight — decide your grace behaviour deliberately.

## 8. Skills Gained

- Authentication and authorization implemented and, more importantly, *attacked*
- Multi-tenant isolation as an architectural decision with defended trade-offs
- Defence against the most common real API vulnerability class
- Secure file handling and a measured PII/secret redaction pipeline
- Audit logging and threat modelling

**Job-description mapping:** *APIs and backend systems* and *real-world AI products*. Security is rarely listed explicitly in AI job descriptions and is always expected. The defensible interview point is that self-hosting satisfied the residency boundary, while redaction separately limited internal exposure across persistence, caches, telemetry, and portable evaluation data.

## 9. Milestone Completion Checklist

- [ ] I can explain authentication versus authorization and point to where each lives in my code.
- [ ] I forged four kinds of invalid token and watched my validator reject each.
- [ ] Cross-tenant access is blocked at a chokepoint, and a route-enumerating test proves it for every endpoint.
- [ ] The authorization matrix test covers every role and endpoint.
- [ ] My upload handler survives oversized files, traversal filenames, wrong types, and bombs.
- [ ] I can state my redaction miss rate and false-positive rate on a corpus, and what I do about residual risk.
- [ ] Sensitive actions produce audit events that the API cannot alter.
- [ ] `X-Tenant-Id` is gone, and a test fails if anything reintroduces trust in client-supplied tenancy.
- [ ] I can compare four tenant-isolation strategies and defend mine.
- [ ] The threat model lists assets, abuse cases, mitigations, and accepted risks.
- [ ] **ADR-005** records the identity and tenant-isolation decisions and what would change them.

---

## Phase I checkpoint — what you can already claim

You have a secure, tested, multi-tenant Python REST backend with a relational database and CI. **There is still no AI in it.** That ordering is deliberate and worth saying out loud in an interview: most AI demos fail in production for backend reasons, not model reasons.

| Job requirement | Evidence so far |
|---|---|
| Python | Typed modular package, async boundaries, profiling, layered tests |
| APIs and backend development | Versioned REST contract, validation, error contract, pagination, idempotency |
| Backend systems | PostgreSQL schema, transactions, migrations, indexes, connection pooling |
| Real-world products | Multi-tenancy, authorization, audit, secure uploads, redaction, threat model |
| Collaboration | ADRs, contract tests, CI gates, reviewable history |

**Next:** Milestone 6 finally calls a model — and treats it, correctly, as an unreliable network dependency rather than a magic box.
