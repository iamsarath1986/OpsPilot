# Milestone 4 — PostgreSQL, Transactions, Migrations, and Query Behaviour

**Goal:** OpsPilot gains durable state. Incidents, artifacts, findings, and analyses live in PostgreSQL, behind the same repository contracts written in Milestone 1, with migrations, transactions, indexes, and integration tests against a real database.

**Why now:** the API contract is settled, so the schema can serve the domain rather than dictate it. And every later feature — multi-tenancy, RAG chunks, job records, evaluation results, usage ledgers — is a table. Getting transactional discipline right once, here, means it is right in ten places later.

**Time:** 4–5 focused weeks. **Prerequisite:** Milestone 3 checklist green.

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| Relational modelling: keys, constraints, foreign keys | The database is your last line of defence against invalid data. Application code has bugs; a `CHECK` constraint does not. |
| Normalisation, and when to denormalise | Duplication causes contradictions; excessive normalisation causes ten-table joins. Both are design failures. |
| SQL: joins, aggregates, `GROUP BY`, CTEs | You must be able to answer questions about your data without the ORM. Interviewers ask; incidents demand it. |
| Indexes: B-tree, composite, partial | The difference between a 4 ms and a 4,000 ms list endpoint. Also: every index slows writes, so they are a trade-off, not free speed. |
| `EXPLAIN ANALYZE` | The only way to know why a query is slow. Guessing at indexes is the most common junior mistake. |
| Transactions and ACID | An incident and its first analysis must both exist or neither. Partial writes are how data becomes untrustworthy. |
| Isolation levels and MVCC | Two engineers updating one incident concurrently is normal. What the database does about it is your decision, not luck. |
| Locking and deadlocks | Under concurrency you *will* hit these. Recognising a deadlock in a log saves hours. |
| Optimistic concurrency (version column) | Prevents the lost-update problem without holding locks across a user's thinking time. |
| Connection pooling | Connections are expensive and limited. Milestone 12 will hit a hard connection cap on managed PostgreSQL; the reasoning starts here. |
| The N+1 problem | The single most common ORM performance bug. You should be able to spot it in a log. |
| SQLAlchemy 2.0: sessions, identity map, unit of work | Knowing what the session does explains most "why didn't my change save?" confusion. |
| Alembic migrations, forward and backward | Schema changes must be reviewable, repeatable, and reversible — in CI and in production. |
| Expand/contract migration pattern | How to change a schema without taking the API down. Needed for real in Milestone 12. |
| Docker Compose for local infrastructure | A reproducible PostgreSQL for every developer and for CI. Application-image work comes in Milestone 11. |
| Backup, restore, and PITR concepts | A backup you have never restored is a hypothesis, not a backup. |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Schema design | **Design** | You can defend every table, key, and constraint from a requirement |
| SQL | **Implement** | You write the join and aggregate by hand before letting the ORM generate it |
| Query plans and indexes | **Debug** | You find a slow query, read its plan, add the right index, and measure the improvement |
| Transactions | **Design** | You can state the transaction boundary for each use case and why |
| Concurrency conflicts | **Debug** | You have reproduced a lost update and a deadlock, and fixed both |
| SQLAlchemy | **Implement** | You can explain flush versus commit versus refresh without looking them up |
| Migrations | **Productionize** | You have applied and rolled back a migration against data, and know which changes lock a table |

## 3. Hands-On Learning Exercises

1. **SQL before ORM.** Create the schema by hand in `psql`. Insert data. Write, by hand: incidents per tenant per day, the most common finding category, and incidents with no analysis. Only then let SQLAlchemy generate anything.
2. **The index experiment.** Load 200,000 synthetic incidents. Time `SELECT ... WHERE tenant_id = ? ORDER BY created_at DESC LIMIT 20`. Run `EXPLAIN ANALYZE` — a sequential scan. Add a composite index on `(tenant_id, created_at DESC)`. Measure again, and re-read the plan. Record both numbers; this is the clearest performance story you will own.
3. **Index cost.** Time 10,000 inserts with no indexes, then with five. Now you can explain why "just add an index" is not free.
4. **N+1 in the wild.** Turn on `echo=True`, list 50 incidents with their findings, and count the queries. Fix with `selectinload`. Count again.
5. **Lost update.** Two sessions read the same incident, both change severity, both commit. One change vanishes. Add a `version` column with an optimistic check and watch the second commit fail loudly instead.
6. **Deadlock.** Two sessions update rows A then B, and B then A. Reproduce the deadlock, read the PostgreSQL error, then fix it with consistent lock ordering.
7. **Pool exhaustion.** Set `pool_size=5`, fire 50 concurrent requests holding sessions. Watch requests queue and then time out. This is exactly the Milestone 12 failure mode, rehearsed cheaply.
8. **Restore drill.** `pg_dump`, drop the database, restore, verify row counts. Time it. That number is your RTO for this component.
9. **Migration rollback.** Write a migration that adds a non-nullable column with data backfill. Apply it, then downgrade it. Notice what breaks — and derive the expand/contract rule yourself.

## 4. Project Implementation

### Step 1 — Local infrastructure

A `docker-compose.yml` with PostgreSQL, a named volume, a health check, and a pinned major version matching what you will run on Azure. One command should give any developer a working database.

### Step 2 — Schema

| Table | Purpose | Notable constraints |
|---|---|---|
| `tenants` | The hosting customer or reseller boundary | Slug unique |
| `incidents` | Core entity | FK to tenant; `status`/`severity` as enums or checked values; `version` for optimistic locking |
| `artifacts` | Log excerpts attached to an incident | FK to incident; size and line-count columns; content reference |
| `analyses` | One analysis run | FK to incident; `produced_by` (`rules@v1`); created_at |
| `findings` | Rule output within an analysis | FK to analysis; `rule_id`; category; severity |
| `evidence` | The exact log lines a finding relies on | FK to finding and artifact; line number; text |
| `idempotency_keys` | Deduplicate client retries | Unique on (tenant, key); stored response; expiry |

Design decisions to make consciously and record: surrogate UUID keys versus bigserial; enum type versus a checked varchar; storing artifact content in a column versus a file reference (choose the reference — Milestone 12 moves it to Blob Storage, and a 200 MB `TEXT` column will hurt you before then); and cascade behaviour on delete.

### Step 3 — Mapping and the unit of work

| Concept | Engineering problem | Implementation | Testing |
|---|---|---|---|
| ORM mapping | Hand-written SQL everywhere is unmaintainable; a hidden ORM is unpredictable | SQLAlchemy 2.0 mapped classes in `infrastructure/db/`, converted to and from domain entities explicitly | Round-trip tests: domain → row → domain |
| Repository implementations | The domain must not know about SQL | `PostgresIncidentRepository` implementing the Milestone 1 protocol | **The Milestone 1 contract tests run unchanged against it** |
| Unit of work | Partial writes corrupt state | One session and one transaction per request, committed by the use case, not by the repository | A use case that raises mid-way leaves no rows behind |
| Session lifecycle | Leaked sessions exhaust the pool | A FastAPI dependency that opens, yields, and always closes | Assert pool checkouts return to zero after requests |
| Optimistic concurrency | Concurrent updates silently overwrite | `version` column with a conflict error mapped to HTTP 409 | Concurrent update test asserts one side gets 409 |
| Indexes | Tenant-scoped list queries degrade with volume | Composite indexes matching real query patterns | Plan assertions or timing on a seeded dataset |

### Step 4 — Migrations

`alembic init`, then generate the initial migration and **read every line of it** before committing — autogenerate misses enum changes, index names, and server defaults. Add a CI step that applies all migrations to an empty database and then applies `downgrade` for the newest one.

### Step 5 — Integration tests

A test suite that runs against real PostgreSQL (Compose service or Testcontainers), with per-test transactional rollback for speed. SQLite is not an acceptable substitute: it does not share PostgreSQL's types, constraints, concurrency behaviour, or SQL dialect, so it would let bugs through precisely where they matter.

## 5. Expected Project State

**Before:** a stateless API that forgot everything on restart.

**After:** durable, relational persistence with transactions, migrations, optimistic concurrency, indexes chosen from measured plans, and integration tests against the real engine. Restarting the service loses nothing; two replicas would now agree.

**Architecturally:** the service became genuinely stateless — state moved to a backing service. That is the precondition for horizontal scaling in Milestone 12, and it is worth being able to say why in exactly those terms.

## 6. Testing and Validation

- The Milestone 1 contract tests, unchanged, passing against PostgreSQL. This is the payoff for the port abstraction and worth calling out in your README.
- Integration tests for transaction rollback, constraint violations, cascade behaviour, and optimistic-concurrency conflict.
- Migration tests: fresh database up to head, plus one downgrade.
- Query-performance test on a seeded dataset with a documented threshold.
- Idempotency test: identical requests with the same key produce one row and the same response.
- A restore drill, performed and timed, written up in `runbooks/restore.md`.

## 7. Failure Scenarios to Reproduce Deliberately

1. Unique-constraint violation from a race between two identical creates — map it to 409, not 500.
2. Lost update without optimistic locking, then the 409 with it.
3. Deadlock between two transactions; read the error and fix the ordering.
4. Connection-pool exhaustion under concurrency; observe queuing, then timeouts.
5. A session left open by an exception path, leaking a connection per request until the pool dies.
6. A migration that takes an `ACCESS EXCLUSIVE` lock on a large table — measure the block. This is the one that causes real outages.
7. Database down while the API is up: readiness must fail, and requests must return 503 rather than hanging.
8. A slow query with no index, found via `EXPLAIN ANALYZE` and fixed.
9. Restore from backup and confirm the data is actually complete.

## 8. Skills Gained

- Relational schema design driven by requirements and query patterns
- Reading query plans and choosing indexes from evidence rather than superstition
- Transaction boundaries, isolation, and concurrency-conflict handling
- SQLAlchemy session discipline and repository implementations behind stable contracts
- Migration practice including rollback and lock-awareness
- Integration testing against the real engine, and a rehearsed restore

**Job-description mapping:** *APIs and backend systems* in its fullest sense — a backend engineer is expected to own the data layer, not just the routes. This is also the milestone that most often separates candidates in interviews.

## 9. Milestone Completion Checklist

- [ ] I can write the main queries in SQL without the ORM.
- [ ] I can read `EXPLAIN ANALYZE` output and explain a plan change after adding an index.
- [ ] I have measured index benefit on reads **and** index cost on writes.
- [ ] I can explain flush, commit, refresh, and the identity map from memory.
- [ ] Transaction boundaries are at the use case, and a mid-way failure leaves no partial data.
- [ ] I reproduced and fixed a lost update, a deadlock, and pool exhaustion.
- [ ] The Milestone 1 contract tests pass unchanged against PostgreSQL.
- [ ] Migrations run forward on an empty database in CI, and I have rolled one back against data.
- [ ] I performed a restore drill and recorded how long it took.
- [ ] I can explain why not SQLite for tests, and why not a document database for this domain.
- [ ] **ADR-004** records the PostgreSQL choice, the key strategy, and the artifact-storage decision.

---

**Next:** Milestone 5 adds the people — authentication, tenant isolation, roles, secure uploads, and the redaction pipeline that must exist before a single customer log line goes anywhere near a model.
