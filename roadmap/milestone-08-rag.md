# Milestone 8 — RAG: Ingestion, Retrieval, Ranking, Grounding, and Citations

**Goal:** OpsPilot answers from *your* runbooks rather than from the model's general knowledge. A full pipeline — documents → chunking → embeddings → `pgvector` storage → hybrid retrieval → reranking → context construction → cited answer — with measured evidence that it beats the non-RAG baseline.

**Why now:** Milestone 7 made the output trustworthy in *form*. It is still ungrounded in *substance*: the model knows generic Linux troubleshooting, not that your company's policy is to restore permissions with a specific script, or that you run LiteSpeed rather than Apache. RAG is what makes the answer specific to your business.

**Why after reliability, not before:** retrieval adds a second failure surface. Debugging "was that the retrieval or the model?" is only tractable when the model path is already validated and observable.

**Time:** 5–6 focused engineering weeks; writing the runbooks is a separate workstream that began early and is not included. **Prerequisites:** Milestone 7 checklist green, and the corpus-readiness gate below green.

### Corpus-readiness gate

Do not start the ingestion implementation while source material is still hypothetical. 20–30 runbooks must be written to the template, versioned, with author and review status recorded, covering enough of the prioritised ticket categories to label at least 40 retrieval questions. Ops review is **not** required: unreviewed runbooks are a limit on what answers may claim, not on engineering. If this gate is red, finish writing before building; do not fill the gap with model-generated operational policy (see [Data we create ourselves](01-the-binding-constraint.md#data-we-create-ourselves)).

---

## 1. Topics to Learn

| Topic | The engineering problem it solves |
|---|---|
| Why RAG at all: grounding, freshness, access control, cost | Compared with fine-tuning: RAG updates in seconds, cites sources, and respects per-tenant permissions. Fine-tuning does none of those. |
| RAG versus long-context prompting | If you can fit everything in the context window, do you need retrieval? Sometimes not — and knowing when is a real design judgement about cost, latency, and precision. |
| Document loading and normalisation | Runbooks arrive as Markdown, wiki exports, PDFs, and help-centre pages. Garbage in, garbage retrieved. |
| Chunking strategies: fixed, recursive, structural, semantic | A chunk is the unit of retrieval. Split a five-step fix across two chunks and you retrieve half a procedure. |
| Chunk overlap | Recovers context lost at boundaries — at the cost of duplication and storage. |
| Embeddings and vector space | What "semantic similarity" actually measures, and why it is not the same as relevance. |
| Embedding model choice, dimensionality, compute, multilingual support | Directly affects recall, storage, latency, and self-hosted capacity. Your hosting customers may not all write in English. |
| Similarity metrics: cosine, inner product, L2 | Must match how the embedding model was trained, or your results quietly degrade. |
| `pgvector` and index types (HNSW, IVFFlat) | Exact search is fine at 10k chunks and unacceptable at 10M. Index parameters trade recall for speed. |
| Lexical search (PostgreSQL full-text, BM25) | Vector search is poor at exact tokens — error codes, file paths, `0555`, `ERR_CONNECTION_REFUSED`. Lexical search excels there. Hosting support is full of exact tokens. |
| Hybrid retrieval and Reciprocal Rank Fusion | Combines both strengths without hand-tuned score weighting. |
| Metadata filtering | Filter by product, OS, control panel, and — critically — tenant visibility. |
| Authorization-aware retrieval | The retriever must never surface a document the caller may not read. A leak here is a data breach with a citation attached. |
| Top-K and context budget | More chunks means more compute, more latency, and often *worse* answers as noise crowds out signal. |
| Reranking (self-hosted cross-encoder or local model) | Retrieve broadly, then rank precisely. Adds latency and capacity demand, so it must earn its place with numbers. |
| Corpus versioning | Answers change when documents change. Reproducibility requires knowing which corpus produced an answer. |
| Retrieval metrics: recall@K, precision@K, MRR, nDCG | You cannot improve retrieval by looking at answers. You must measure the retrieval step in isolation. |
| Groundedness and citation accuracy | Whether each claim is supported by retrieved text — the practical definition of "not hallucinating". |

## 2. Learning Depth

| Topic | Target depth | You have reached it when… |
|---|---|---|
| Chunking | **Design** | Your chunk size and overlap are chosen from a measured sweep, not a blog default |
| Embeddings | **Implement** | You have compared two models on your own labelled queries |
| `pgvector` and indexes | **Debug** | You can explain a slow vector query and the recall cost of the index parameters you set |
| Hybrid retrieval | **Design** | You can show, per query type, where lexical beats vector and vice versa |
| Authorization in retrieval | **Productionize** | Filtering happens in the query, not after — and tests prove no unauthorised chunk is ever fetched |
| Reranking | **Design** | You can state its measured benefit against its measured latency and capacity demand |
| Retrieval evaluation | **Implement** | You can report recall@5 with a labelled set and a confidence caveat |

## 3. Hands-On Learning Exercises

Build a small labelled set first: 40 support questions, each with the runbook chunk IDs that *should* be retrieved. Write the questions **in the customer's voice** and, where possible, before rereading the target runbook; record each question's provenance and author. Compute a vocabulary-overlap score between each question and its target chunk, and report every metric below separately for high- and low-overlap questions — a retriever that only succeeds when the question repeats the runbook's wording has not learned anything useful. Everything below depends on this set.

1. **Chunk sweep.** Sizes 200 / 500 / 1000 tokens × overlap 0% / 10% / 25%. Measure recall@5 for each of the nine combinations. Pick from the table. Keep the table — it is one of the most persuasive artifacts you will produce.
2. **Embedding comparison.** Two self-hosted models on the same corpus and queries. Compare recall@5, index size, embedding time, memory demand, and query latency. Note that the better model is not automatically the right one.
3. **The exact-token test.** Query `0555 permission` and `ERR_TOO_MANY_REDIRECTS`. Compare vector-only, lexical-only, and hybrid retrieval. Vector search often loses badly on exact strings — this experiment is why your system will be hybrid.
4. **Top-K curve.** K = 1, 3, 5, 10, 20. Measure recall, answer groundedness, latency, and generation capacity consumed. Find the point where more context stops helping and starts hurting.
5. **Adversarial document.** Add an outdated runbook contradicting a current one. Does retrieval prefer it? Add a `deprecated` metadata flag and filter it. Now you understand why corpus hygiene is an engineering concern, not a documentation concern.
6. **Boundary split.** Deliberately chunk so a five-step procedure splits after step three. Ask a question needing step four. Watch a confident, incomplete, dangerous answer appear.
7. **Scale.** Load 50,000 synthetic chunks. Time queries with no index, with IVFFlat, and with HNSW. Vary HNSW parameters and measure the recall/latency trade-off.
8. **The ablation that matters.** Run the same 40 questions with RAG and without. Score groundedness and correctness for both. If RAG does not win, do not ship it — find out why first. This experiment is what distinguishes engineering from assuming.

## 4. Project Implementation

### Step 1 — Accept and freeze the first corpus release

Import the 20–30 runbooks we have written since the start of the project, which live in **`corpus/runbooks/`**: 500 error after deploy, permission problems, disk and inode quota, mail delivery failures, SSL renewal, database connection limits, compromised WordPress, outbound spam, DNS propagation, slow site triage. Verify author, review status, version, and absence of customer data; freeze the release used by the retrieval experiments.

**Ingest from `corpus/runbooks/` and nowhere else.** The top-level `runbooks/` directory holds *operational* procedures for the people running OpsPilot — the Milestone 4 restore drill, the Milestone 11 alert runbooks — and must never enter the corpus. A path-scoped loader plus a test asserting that no chunk originates outside `corpus/` keeps an internal operations document from being retrieved and cited as customer-facing hosting guidance.

The runbooks are genuinely useful to your employer independent of the software, and a reviewed corpus is worth offering to ops on its own. Writing them is not part of the 5–6-week engineering estimate. Whenever ops reviews or corrects a runbook, publish a new corpus version. Clear headings in the authoring template become chunk boundaries here.

### Step 2 — Ingestion pipeline

| Concept | Engineering problem | Implementation | Testing |
|---|---|---|---|
| Loader and normaliser | Mixed formats produce inconsistent text | Markdown/HTML/PDF loaders producing normalised text plus metadata | Golden-file tests per format |
| Structural chunking | Naive splitting severs procedures | Split on headings first, then size-bound with overlap; never split mid-list | Assert no chunk contains a partial numbered procedure |
| Chunk and vector metadata | Filtering, citation, and vector compatibility need provenance | `document_id`, heading path, product, OS, visibility, `deprecated`, `corpus_version`, source URL, `embedding_model`, `embedding_revision`, `embedding_dimensions`, and `embedding_config_version` | Metadata round-trip tests; query rejects a mismatched embedding identity |
| Embedding with batching | One call per chunk wastes local inference capacity | Batched calls to the self-hosted embedding service with bounded concurrency, reusing the Milestone 7 resilience policy | Compute time and latency measured against sequential |
| Idempotent re-ingestion | Re-running must not duplicate rows or waste compute | Content hash per chunk; skip unchanged | Re-ingest twice, assert stable row count and near-zero new embedding work |
| Corpus and embedding versioning | Answers must be reproducible and incompatible vector spaces must never mix | Record `corpus_version` and `embedding_config_version` on chunks, retrieval runs, diagnoses, and cache keys; an embedding change creates a new index namespace and requires re-embedding | Either version bump invalidates the Milestone 7 cache; mixed identities fail closed |

### Step 3 — Storage and indexing

`documents` and `chunks` tables with a `vector` column, plus a `tsvector` column for lexical search. An HNSW index with parameters chosen from your measurements. Metadata columns indexed for filtering. Keep it in PostgreSQL: one backup, one authorization model, one transaction boundary. Record in the ADR what would justify a dedicated vector service later — corpus scale, query throughput, or ranking features you cannot express in SQL.

### Step 4 — Retrieval

Hybrid: vector search and lexical search in parallel, fused with Reciprocal Rank Fusion, then optional reranking of the top 20 down to the top 5.

**Authorization is part of the query, never a post-filter.** A `WHERE tenant_id = ... AND visibility = ...` clause in the retrieval SQL. Post-filtering means unauthorised content was fetched, ranked, and possibly logged before you dropped it.

### Step 5 — Context construction

Assemble the prompt within a token budget: deduplicate near-identical chunks, order by rank, attach a stable `doc_id#chunk_id` to each passage, and record exactly what was included. The Milestone 7 evidence-validation layer now checks citations against *both* log evidence IDs and retrieved chunk IDs — so a fabricated source is rejected automatically. That connection between two milestones is the system working as designed.

### Step 6 — The abstention path

When retrieval returns nothing above a relevance threshold, the correct behaviour is `abstained`, not "answer from general knowledge". Wire this to the Milestone 7 state machine. In a hosting-support context, a confident generic answer about Apache when you run LiteSpeed is worse than no answer.

### Step 7 — The retrieval evaluation harness

`evals/retrieval/`: run the labelled query set, report recall@K, precision@K, MRR, and nDCG overall and per slice, with results written to a versioned report. This is the foundation Milestone 9 builds on.

## 5. Expected Project State

**Before:** validated answers from the model's general knowledge.

**After:** answers grounded in your own runbooks, with citations that are verified to exist and to be authorised for the caller; hybrid retrieval that handles both natural-language questions and exact error strings; corpus versioning; and measured proof that RAG beats the non-RAG baseline.

**Architecturally:** you now have two AI subsystems that fail independently — retrieval and generation — and the telemetry to attribute a bad answer to one of them. That attribution capability is what makes the system debuggable.

## 6. Testing and Validation

- Ingestion tests: format handling, chunk-boundary invariants, idempotent re-ingestion, metadata integrity.
- Retrieval tests on the labelled set with a minimum recall@5 threshold enforced in CI.
- **Authorization tests: no query may return another tenant's or an unauthorised document, under any filter combination.** Treat this as the same severity as the Milestone 5 IDOR tests.
- Citation-integrity tests: every returned citation resolves to a real, authorised chunk.
- The RAG-versus-no-RAG ablation, recorded with per-slice results.
- Latency budget: measure retrieval, reranking, and generation separately so you know which to optimise.
- A corpus-version bump invalidating cached answers.
- Extend the standing `egress-policy` CI job from M6 to exercise ingestion, query embedding, and reranking; only the allowlisted self-hosted embedding and reranker endpoints may receive marker content.

## 7. Failure Scenarios to Reproduce Deliberately

1. **Retrieval returns nothing relevant** and the model answers anyway from general knowledge. This is the most dangerous RAG failure and the reason abstention exists.
2. An outdated runbook outranks the current one.
3. A chunk boundary severs a procedure, producing a confident half-answer.
4. **Cross-tenant retrieval** — a document from another customer appears in context. Fail closed and add the regression test.
5. Changing the embedding model without re-embedding: old and new vectors are incomparable and quality collapses silently. This is why corpus and embedding versions must be stored.
6. Context overflow from too many chunks, truncating the highest-ranked passage.
7. A vector query without an index at scale, and the same query with one.
8. Near-duplicate chunks consuming the whole context budget.
9. An exact error code that vector search cannot find, rescued by the lexical half of the hybrid.
10. Ingestion crashing halfway, leaving a partially embedded document — is your corpus now inconsistent? Make ingestion transactional or resumable.

## 8. Skills Gained

- The complete RAG pipeline, built rather than assembled from a framework, with each stage measurable
- Chunking, embedding, and index parameters chosen from experiments you ran
- Hybrid retrieval and the specific reason it exists in a technical-support domain
- Authorization-aware retrieval — a requirement most RAG tutorials ignore entirely
- Retrieval evaluation separate from answer evaluation, enabling failure attribution
- Grounding, citation integrity, and principled abstention

**Job-description mapping:** *AI applications* at the depth employers actually mean. Also *collaboration with data scientists* — retrieval metrics, ablations, and labelled datasets are the shared vocabulary you will use with them.

## 9. Milestone Completion Checklist

- [ ] I can draw the full pipeline from document to cited answer and explain each stage's failure mode.
- [ ] My chunk size and overlap come from a measured sweep I can show.
- [ ] I compared two embedding models on my own labelled queries.
- [ ] Every stored vector records its embedding model, revision, dimensions, and configuration version; mismatched query and corpus embeddings fail closed.
- [ ] I can show a query where lexical search beats vector search and explain why.
- [ ] Authorization filtering happens inside the retrieval query, and tests prove no cross-tenant leak.
- [ ] Every citation is validated against real, authorised chunks before the response is returned.
- [ ] Empty or low-relevance retrieval produces `abstained`, never a generic answer.
- [ ] I ran the RAG-versus-no-RAG ablation and can state where RAG wins and where it does not.
- [ ] I can state recall@5 on my labelled set, with the set size and its limitations.
- [ ] I can explain what would justify moving from `pgvector` to a dedicated vector database.
- [ ] I know what breaks if I swap embedding models without re-embedding — because I did it.
- [ ] The standing `egress-policy` CI job proves embedding and reranking content can reach only approved self-hosted endpoints.
- [ ] **ADR-008** records chunking, embedding, hybrid retrieval, and the `pgvector` decision.

---

**Next:** Milestone 9 replaces "it seems better" with measurement — evaluation datasets, judges, guardrails, regression gates, and human escalation.
