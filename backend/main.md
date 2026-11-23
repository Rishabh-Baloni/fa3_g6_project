# What main.py does

`backend/main.py` is the agent service: a FastAPI app on port 8000 that wires together every backend component at start-up and exposes the audit workflow over HTTP. It turns HTTP requests (uploads, corrections, decisions, questions, admin actions) into calls on `AuditWorkflow`, `InvoiceMonitor`, `AuditRepository`, `QueryWorkflow` and `VectorIndex`, and serves reports and original documents back.

Main parts:

1. Request bodies (Pydantic models)
2. `Services`, `build_services`, start-up checks and background work
3. `lifespan` and logging
4. Helpers (`_invoice_or_404`, `_file_inside`, `_reindex`, `_empty_directory`)
5. The 16 HTTP routes

---

## Overall flow

```
 uvicorn start ─► lifespan ─► get_settings ─► build_services  (fail fast: StartupError / ConfigurationError)
                                  │  rules.yml, prompts, incoming + reports folders, PostgreSQL, audit schema,
                                  │  LangGraph checkpoint tables, Redis, mock-erp, Qdrant, models
                                  ▼
                           app.state.services ─► start_background_work
                                                   ├─ thread "index-sync": sync_vector_index
                                                   └─ InvoiceMonitor.start (if monitor_enabled)

 Streamlit frontend / API client
        │ HTTP
        ▼
   FastAPI routes ──► AuditRepository (PostgreSQL)      workflow.revalidate (LangGraph)
                  ├─► InvoiceMonitor (incoming/)         QueryWorkflow ─► SQL or RagPipeline (Redis, Qdrant, models)
                  └─► files: reports/, incoming/ (served only if inside the folder)
 blocking work runs in a thread pool (`run_in_threadpool`); slow follow-ups (re-indexing) run as BackgroundTasks after the response
```

Important behaviours: start-up refuses to run in a broken state; a single `Services` object lives in `app.state`; blocking calls never run on the event loop; after human actions the invoice's index entries are rebuilt in the background.

---

## 1. Request bodies

| Model | Fields and validation |
|---|---|
| `CorrectionItem` | `field_name`, `corrected_value` |
| `CorrectionRequest` | `corrected_by` (1 to 128 chars), `corrections`, `accepted_exceptions` (keys, max 100), `comment` (max 2000), `revalidate` (default true). A validator requires at least one correction or accepted exception |
| `DecisionRequest` | `decided_by` (stripped, 1 to 128), `decision` (`approve` or `reject`), `note` (max 2000) |
| `BatchDecisionRequest` | Same plus `invoice_ids` (1 to 200) |
| `RejectRecommendationRequest` | `recommended_by`, non-empty stripped `note` (max 2000) |
| `ResetRequest` | `confirm` (must equal `RESET_PHRASE = "RESET"`) |
| `QuestionRequest` | `question` (3 to 1000 chars), optional `invoice_id`, `vendor` |

## 2. `Services` and start-up

`Services` is a dataclass holding: `settings`, `rules`, `database`, `checkpoint_database`, `repository`, `cache`, `erp`, `gateway`, `index`, `workflow`, `monitor`, `rag`, `query`, `reports_dir`. `close()` stops the monitor and closes both databases.

| Function | Purpose |
|---|---|
| `_check(name, action)` | Runs a start-up check; any exception becomes `StartupError("<name> is not reachable or not usable: ...")`, otherwise logs "ok" |
| `build_services(settings)` | In order: `rules.yml`, prompts and personas, incoming folder (readable), reports folder (writable), PostgreSQL (pool 8), audit schema (`apply_schema`), checkpoint database (pool 4, autocommit) and its tables, Redis, mock-erp health, Qdrant client, `LlmGateway`, `VectorIndex` and Qdrant ping. Logs the configured model per role. Then builds `AuditWorkflow`, `InvoiceMonitor`, `RagPipeline` (with a `TriadEvaluator`) and `QueryWorkflow` |
| `_require_dir(path, *, writable)` | Raises if the path is not a directory or lacks read, execute (and write) permission |
| `sync_vector_index(services)` | Skipped if no embedding model. If audited invoices exist but Qdrant is empty, rebuilds from PostgreSQL. Otherwise indexes invoices that were never indexed. Errors are logged; `POST /index/rebuild` retries |
| `start_background_work(services)` | Starts the daemon thread `index-sync`, and the monitor if `monitor_enabled` |

Why fail fast: an agent that starts without PostgreSQL, Redis, Qdrant or the ERP would accept files it cannot process.

## 3. Lifespan and logging

| Function | Purpose |
|---|---|
| `_configure_logging(level)` | Logs to stdout; quiets `httpx`, `httpcore`, `botocore`, `urllib3`, `groq`; a filter drops `GET /health` lines from uvicorn's access log |
| `lifespan(app)` | Configures logging, loads settings, builds services (a `StartupError` or `ConfigurationError` is logged as critical and re-raised), stores them in `app.state.services`, starts background work, yields, then `services.close()` |

## 4. Helpers

| Function | Purpose |
|---|---|
| `_services()` | The `Services` in `app.state` |
| `_invoice_or_404(invoice_id)` | Returns the record or raises 404. A malformed UUID is treated as not found |
| `_file_inside(path, root)` | Returns the path only if it resolves (symlinks and `..` followed) to a regular file inside `root`. A bad database row can therefore never make the agent serve a file from outside the incoming or reports folder |
| `_reindex(invoice_ids)` | Background task: re-indexes invoices whose story changed (a recommendation or decision). Does nothing without embeddings; errors are logged |
| `_empty_directory(folder)` | Removes everything inside a folder, not the folder. Refuses to empty a filesystem root |
| `_revalidate(invoice_id)` | Runs `workflow.revalidate` in a thread and returns `run_number`, `recommendation`, `validation_status`, `index_status`, `review_iteration`, `review_outcome` |

Constants: `MAX_UPLOAD_BYTES` 20 MB; `_SAFE_NAME` for file-name cleaning; `_REPORT_HEADERS` (Content-Security-Policy `default-src 'none'; style-src 'unsafe-inline'`, `nosniff`); `_PREVIEW_HEADERS` (same plus `img-src data:`); `_ORIGINAL_TYPES` (suffix to media type).

## 5. Routes

| Method and path | Purpose | Responses and key logic |
|---|---|---|
| `GET /health` | Dependency status | Probes postgres, redis, qdrant, mock_erp. Status `ok`; `degraded` if only Redis fails; `unavailable` (HTTP 503) if postgres, qdrant or the ERP fail. Also returns model status and monitor state |
| `POST /invoices/upload` | Save an upload to the incoming folder | 415 unsupported type, 413 over 20 MB, 400 empty. Name is cleaned and made unique (`_2`, `_3`). Written as `.part` and renamed, so the monitor never sees a half-written file. 201 |
| `POST /monitor/scan` | Trigger a scan now | `{"files_handled": n}` |
| `POST /files/{event_id}/retry` | Retry a failed file | 404 unknown event, 409 if not failed |
| `POST /invoices/{invoice_id}/corrections` | One human review iteration | 422 for duplicate fields, invalid values, accepted exceptions without a comment, or exceptions that cannot be accepted; 409 if `ReviewNotAllowedError`. Records the batch, then re-validates unless `revalidate` is false. 201 |
| `POST /invoices/{invoice_id}/revalidate` | Re-run validation | Returns the run summary |
| `GET /invoices/{invoice_id}/report.html` | Latest HTML report | Serves the stored file if it is inside the reports folder, otherwise re-renders from the database copy. CSP headers. 404 if no report |
| `GET /invoices/{invoice_id}/original` | The file as received | Only from inside the incoming folder, inline, `nosniff`; 404 if gone |
| `GET /invoices/{invoice_id}/original/preview` | HTML preview of the original | `render_docx_html` for DOCX, otherwise `render_pages_html`; 422 on `ExtractionError` |
| `POST /invoices/batch-decision` | Final batch sign-off | IDs must be UUIDs (422). Returns `{decision, decided, skipped}`; decided invoices are re-indexed in the background |
| `POST /invoices/{invoice_id}/recommend-reject` | Reviewer escalates to rejection | 409 if not allowed; re-index in the background. 201 |
| `POST /invoices/{invoice_id}/decision` | Final sign-off of one invoice | 422 if an override has no reason, 409 if the invoice is not eligible; re-index in the background. 201 |
| `GET /invoices/{invoice_id}/report.json` | Latest report as JSON | 404 if no report |
| `POST /admin/reset` | Delete everything the application wrote | 403 if `allow_reset` is false, 422 unless `confirm == "RESET"`. With the monitor paused: clears incoming files, truncates the audit and checkpoint tables, drops the vector collection, deletes the `llm`, `rag`, `translation` cache namespaces, empties the reports folder. ERP data stays. A failure returns 500 and says it is safe to run again |
| `POST /query` | Natural-language question | Checks the invoice exists if given; runs `QueryWorkflow.ask` in a thread. 503 when a model is not configured, 502 on a model call error |
| `POST /index/rebuild` | Rebuild the index from PostgreSQL | 503 without an embedding model; returns `{"indexed", "failed"}` |

Notes:
- Decision, recommend-reject and corrections endpoints return stored rows with values converted to strings (`None` stays `None`).
- Corrections: each field's original value comes from `extracted_invoice` (not from earlier corrections), through `current_value`, and the submitted value is normalised by `normalise_correction`.
- The review limit is `settings.max_review_iterations`; the repository enforces it under a row lock, so two clients cannot exceed it.

---

## End-to-end example

A reviewer fixes a price in the dashboard:

1. The frontend calls `POST /invoices/<id>/corrections` with `corrected_by="maria"`, `corrections=[{"field_name": "line_items[1].unit_price", "corrected_value": "7.00"}]`.
2. `_invoice_or_404` confirms the invoice. The field appears once; `normalise_correction` returns `"7.00"`; `current_value` reads the extracted value `10.00` as the original.
3. No accepted exceptions are sent, so the comment is not required. `record_review_batch` locks the invoice, sets iteration 1 of the configured maximum and inserts the feedback row.
4. `_revalidate` runs `workflow.revalidate` in a thread. The pass becomes review iteration 1; the price finding disappears and the recommendation is `approve` with outcome `resolved`.
5. The response (201) contains `feedback_ids`, `review: {iteration: 1, max_iterations: ...}` and the `revalidation` summary.
6. The invoice is now Ready for Sign-off. Later, `POST /invoices/<id>/decision` with `approve` records the final decision and triggers a background `_reindex`.
