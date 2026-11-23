# What rag.py does

`backend/rag.py` turns audited invoices into a searchable Qdrant index and answers natural-language questions about them. It has four parts that work together: an indexer that builds vectors from text already stored in PostgreSQL, a SQL shortcut for simple count/list/total questions, a reflection evaluator that scores answers, and a LangGraph retrieve, augment, generate, reflect loop.

Main parts:

1. Chunking and report text (`chunk_text`, `report_text`, `human_text`)
2. `VectorIndex`: Qdrant collection, indexing, rebuild and search
3. Structured questions answered with SQL (`try_structured_answer`)
4. `TriadEvaluator`: RAG triad scoring (ragas, with an LLM-judge fallback)
5. `RagPipeline`: the LangGraph question-answering loop and its caches

---

## Overall flow

```
 INDEXING                                           QUESTION ANSWERING
 workflow `index` node                              main.py / query.py ──► RagPipeline.ask / ask_open
   │                                                   │
   ▼                                                   ▼ (ask: no filters) try_structured_answer ─ SQL ─► PostgreSQL
 VectorIndex.index_invoice                             │ not structured
   repository.indexing_source (PostgreSQL)             ▼
   chunk invoice text + report_text                  Redis rag:... hit? ──yes──► cached answer
   gateway.embed ─► Qdrant `invoice_chunks`            │ no
   (delete old points, upsert new)                     ▼
   mark_indexed                                     retrieve ─► augment ─► generate ─► reflect ──avg > 0.7──► END
                                                      ▲  (embed query,   (rerank,    (RAG model,  (TriadEvaluator)
                                                      │   Qdrant search)  context)     Redis llm:)        │ otherwise, up to 3 retries
                                                      └──────────────── rewrite ◄─────────────────────────┘
```

Qdrant is a derived store: every vector is produced from text already saved in the audit schema, so losing the Qdrant volume means re-embedding, never data loss or another OCR pass. Concurrency: `VectorIndex` serialises collection changes with a `threading.Lock`. Caching: generated answers are cached under `llm:<sha256 of the prompt>`; accepted final answers under `rag:...` (see `cache_key`).

---

## 1. Chunking and retrieval text

| Function | Input | Output |
|---|---|---|
| `chunk_text(text, *, size=900, overlap=150)` | Text | Chunks from `RecursiveCharacterTextSplitter` (splits at paragraph, then line, then word). Blank lines are removed first |
| `human_text(human)` | Result of `AuditRepository.human_outcome` | Prose lines: workflow status and review attempts, system or reviewer rejection recommendation with reason, final decision with override/confirm and comment. `[]` if nothing |
| `report_text(report, human=None)` | Report dict | Plain prose: header line, recommendation and status, total, summary, each discrepancy, missing fields, notes, auditor corrections, reviewer notes, accepted exceptions, `human_text`, language line |
| `Chunk` | frozen dataclass: `text`, `source_type` (`invoice` or `report`), `index` | |

Why prose: embeddings of sentences retrieve better than embeddings of JSON, and the human outcome (who rejected, who overrode) becomes searchable.

## 2. `VectorIndex`

`VectorIndex(client, collection, gateway, repository)`. The collection name comes from settings (`invoice_chunks`).

| Method | Purpose |
|---|---|
| `ping()` | `get_collections()` |
| `point_count()` | Exact point count; 0 if the collection does not exist. Used in the `rag:` cache key, so a changed index invalidates cached answers |
| `_ensure_collection(dimension)` | Creates a cosine collection and keyword payload indexes (`invoice_id`, `source_type`, `vendor`, `language`). If it exists with a different vector size, raises `RuntimeError` telling the user to rebuild after changing the embedding model |
| `index_invoice(invoice_id)` | Loads `indexing_source`; returns 0 if the invoice or its run is missing. Chunks the translated text (or raw text) as `invoice` chunks and the report prose as `report` chunks. Prefixes each chunk with `Invoice <number> \| Vendor <name> \| PO <po>`, embeds all, then under the lock deletes the invoice's old points and upserts the new ones, and calls `mark_indexed`. Returns the chunk count |
| `drop()` | Deletes the collection; returns whether it existed |
| `rebuild()` | Deletes the collection and re-indexes every validated invoice; a failure for one invoice is logged and counted, not fatal. Returns `{"indexed", "failed"}` |
| `_chunks(source)` | Builds the `Chunk` list |
| `search(vector, *, limit, invoice_ids, vendors, language, source_type)` | Similarity search with payload filters; `[]` if the collection does not exist. Returns payloads with `score` |

Helpers: `payload_filter(...)` builds a Qdrant filter (one value uses `MatchValue`, several use `MatchAny`); `_invoice_filter(invoice_id)` is the single-invoice filter.

Point ids are `uuid5(namespace, "<invoice_id>:<source_type>:<index>")`, so re-indexing the same invoice overwrites deterministically. Payload: `invoice_id`, `invoice_number`, `chunk_text`, `source_type`, `vendor`, `language`, `run_number`, `chunk_index`.

## 3. Structured answers with SQL

`try_structured_answer(question, repository)` answers plain counts, lists and totals from `audit.invoice_overview` with no model and no Qdrant. It returns `StructuredAnswer(answer, query_name, rows)` or `None` to fall through to retrieval.

| Step | Logic |
|---|---|
| Open-ended | Words like why, explain, reason, what happened, describe: returns `None` |
| Intent | Regexes `_COUNT` ("how many ... invoices", "number of invoices"), `_LIST` ("list/show ... invoices", "which invoices"), `_TOTAL` ("total/sum ... amount/value/spend/..."). None matched: `None`. The patterns are deliberately narrow |
| Vendor | `_vendor()` extracts a name after from/by/for/of (words like "approved" or "all" are ignored) and looks for a distinct `vendor_name ILIKE %phrase%`. A mentioned vendor that is unknown or ambiguous returns `None` so retrieval handles it |
| Recommendation | `_recommendation()` picks `reject`, `approve` or `review` from the wording |
| Filters | Always `latest_run_number IS NOT NULL`, plus vendor and recommendation if present. Values go in as SQL parameters, never into the SQL text |
| Output | `total_by_currency` (grouped by currency), `count`, or `list` (invoice number, vendor, amount, recommendation). Empty results say there are none |

`_singular(scope)` turns "invoices" into "invoice" for a count of one.

## 4. `TriadEvaluator`

The RAG triad: answer relevance, groundedness and context relevance, each 0 to 1 (`TRIAD`).

| Item | Purpose |
|---|---|
| `Evaluation` | `scores` (three values plus `average`), `feedback`, `evaluator` (`ragas`, `llm-judge` or `none`) |
| `evaluate(question, contexts, answer)` | Tries ragas first; on any exception logs and uses the LLM judge; if that fails too, returns all-zero scores with evaluator `none` |
| `_metrics()` | Lazily builds ragas `ContextRelevance`, `ResponseGroundedness` and `ResponseRelevancy` (strictness 1) with a separate (non-shared) reasoning model and the embedding model. Run config: timeout `timeout_seconds`, 1 retry |
| `_evaluate_with_ragas(...)` | Scores each metric; a missing or NaN score raises `ModelCallError` (which triggers the fallback). Feedback names the weakest score |
| `_evaluate_with_judge(...)` | One `judge` / `rag_judge` prompt returning JSON scores and feedback |
| `_clamp`, `_with_average` | Keep scores in 0..1 and add the rounded average |

`os.environ.setdefault("RAGAS_DO_NOT_TRACK", "true")` disables ragas telemetry.

## 5. `RagPipeline`

Constants: `ACCEPT_THRESHOLD` 0.7, `MAX_RETRIES` 3, `INITIAL_TOP_K` 5, `TOP_K_STEP` 3, `CANDIDATE_MULTIPLIER` 3, `MAX_CONTEXT_CHARS` 9000.

Public methods:

| Method | Purpose |
|---|---|
| `ask(question, *, invoice_id=None, vendor=None)` | Without filters, tries SQL first (route `sql`). Otherwise calls `ask_open` |
| `ask_open(question, *, invoice_ids, vendors, language, source_type)` | Raises `ModelNotConfiguredError` listing missing RAG, reasoning or embedding models. Checks Redis (`rag` key from question lowercased, filters and `point_count()`); on a miss, runs the graph. Only accepted answers are cached (TTL `rag_answer_cache_ttl_seconds`) |

Graph nodes:

| Node | What it does |
|---|---|
| `retrieve` | Embeds `search_query`; searches `top_k x 3` candidates with filters |
| `augment` | `rerank()` then `build_context()`. With no chunks and no earlier answer, stores a "No processed invoices are indexed" answer as `best`, and the graph ends |
| `generate` | Builds the `rag_assistant` / `rag_answer` prompt. Redis key `llm:` plus the SHA-256 of the prompt; on a hit reuses the answer. Otherwise calls the RAG model (700 tokens) and caches it |
| `reflect` | Evaluates, marks `accepted` if the average is above 0.7, builds sources (excerpt of 400 chars), records the attempt, and keeps the highest-average answer as `best` |
| `_route_after_reflection` | `done` if accepted or `attempt > MAX_RETRIES`, else `retry` |
| `rewrite` | Reasoning model rewrites the search query using the evaluator's feedback (80 tokens); on failure reuses the question. Increments `attempt` and raises `top_k` by 3 (a wider net) |

Helpers:

| Function | Purpose |
|---|---|
| `_terms(text)` | Lower-case words, minus stopwords |
| `rerank(question, candidates, top_k)` | Drops duplicate chunk text. Score = `0.7 x vector score + 0.3 x query-term overlap`, plus 0.1 for report chunks when the question mentions why, flagged, rejected, discrepancy, mismatch and similar words. Returns the top `top_k` with `rerank_score` |
| `build_context(chunks)` | Numbered blocks (`[n] <source_type> text, invoice <number>:`), stops before exceeding 9000 characters |

Result dict from `ask_open`: `answer`, `route` (`rag`), `accepted`, `cached`, `scores`, `sources`, `attempts`, `filters`.

Why the loop: the evaluator's feedback is used to rewrite the query and widen retrieval, so a weak answer gets another chance. If no attempt passes, the best-scoring answer is still returned with `accepted=False` and is not cached.

---

## End-to-end example

Question: "Why was INV-1002 flagged?"

1. `ask()` has no filters, so `try_structured_answer()` runs. "Why" matches `_OPEN_ENDED`, so it returns `None`.
2. `ask_open()` checks that the RAG, reasoning and embedding models are configured, then looks up the `rag:` key. Miss.
3. `retrieve` embeds the question and gets 15 candidates from Qdrant (`top_k` 5 x 3). `augment` reranks them; report chunks get the +0.1 boost because "why" and "flagged" appear. The top 5 become the numbered context.
4. `generate` has no `llm:` cache entry, so the RAG model answers: the unit price on line 1 differs from the PO.
5. `reflect`: ragas gives answer relevance 0.9, groundedness 0.85, context relevance 0.4, so the average is 0.717. That is above 0.7, so the answer is accepted.
6. `ask_open()` stores the result in Redis and returns the answer with sources (invoice id, number, excerpt) and the one recorded attempt.
7. Asking the same question again returns the cached result with `cached: True`, and no model is called.
