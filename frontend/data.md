# What data.py does

`frontend/data.py` is the dashboard's data layer. It turns database rows and HTTP calls into the DataFrames, dicts and strings the Streamlit pages display. Reads go straight to PostgreSQL through a read-only role; every change goes through the agent's HTTP API, so the dashboard itself never writes to the database.

Main parts:

1. Read-only access to the `audit` schema (`db_*` functions)
2. HTTP client for the agent (`api_*` functions)
3. Cached wrappers for documents (`report_html`, `original`, `original_preview`)

---

## Overall flow

```
 page_*.py / app.py
     │ reads                                         │ changes
     ▼                                               ▼
 db_* functions  (st.cache_data, TTL 5 s; intake 2 s)      api_* functions ──► _api_request / _fetch_original
     │ psycopg, dict rows                                       │ httpx to AGENT_BASE_URL (default http://agent:8000)
     ▼                                                          ▼
 PostgreSQL schema audit                                 agent (backend/main.py)
 role ui_svc (SELECT only,                               HTTP errors ≥ 400 → AgentError(detail)
 default_transaction_read_only=on)                       network errors → AgentError("not reachable")
```

Caching: `st.cache_data` with a TTL of `REFRESH_SECONDS` (5 s) keeps the dashboard responsive while still showing near-live data; the file-intake query uses 2 s. `db_clear_cache()` clears every cache after an action so the next render reads fresh data.

---

## 1. Database reads

| Function | Input | Output |
|---|---|---|
| `_conninfo()` | Environment `UI_DB_HOST`, `UI_DB_NAME`, `UI_DB_USER`, `UI_DB_PASSWORD` (and `UI_DB_PORT`, default 5432) | Connection string with a 5 s timeout, application name `invoice-dashboard` and `default_transaction_read_only=on`. `RuntimeError` listing any missing variable |
| `_rows(sql, params=())` | SQL and parameters | List of dict rows; opens a new connection each call |
| `db_check_database()` | none | Runs `SELECT 1` (used for the connection check) |
| `db_invoices()` | none | DataFrame from `audit.invoice_overview` where `latest_run_number IS NOT NULL`: number, vendor, date, total, currency, recommendation, status, counts, language, file name, review and decision columns, `workflow_stage`. Ordered by stage (`needs_attention`, `recommended_reject`, `ready_signoff`, then the rest), then newest validation first |
| `db_violations()` | none | `{invoice_id: [rule_code, ...]}` for invoices in `needs_attention`, from the latest run's discrepancies, most severe first, without duplicates |
| `db_invoice(invoice_id)` | UUID text | The `audit.invoice_audit` row, or `None` |
| `db_runs(invoice_id)` | UUID text | Validation runs, newest first (including the report JSON) |
| `db_discrepancies(run_id)` | Run id | Findings, ordered high, medium, low, then by line number |
| `db_feedback(invoice_id)` | UUID text | Corrections and notes, newest first, with `revalidated` computed by checking whether a run exists with the same `review_iteration` |
| `db_accepted(invoice_id)` | UUID text | Accepted exceptions |
| `db_intake(limit=200)` | Row limit | DataFrame of file events (newest first) joined to the overview for stage, iteration, decision and rejection recommender |
| `db_clear_cache()` | none | `st.cache_data.clear()` |

Why direct reads: the pages need many small queries and joins; a read-only role (`ui_svc`) makes this safe. Everything that changes state is a request to the agent, which enforces the rules (review limit, decision eligibility).

## 2. Agent HTTP client

`AGENT_BASE_URL` comes from the environment (default `http://agent:8000`). `AgentError` is the one exception pages catch and show.

| Function | Endpoint | Timeout |
|---|---|---|
| `_api_request(method, path, *, timeout=30, **kwargs)` | any | Returns JSON if the content type is JSON, else text. For status 400 or above, raises `AgentError` with the response's `detail` (or the text). Network errors become "The audit service is not reachable (ErrorName)." |
| `api_health()` | `GET /health` | 5 s |
| `api_upload(file_name, content)` | `POST /invoices/upload` | 30 s |
| `api_submit_batch(invoice_id, corrected_by, corrections, comment, accepted_exceptions=None)` | `POST /invoices/{id}/corrections` with `revalidate: True` | 180 s |
| `api_recommend_reject(invoice_id, recommended_by, note)` | `POST /invoices/{id}/recommend-reject` | 30 s |
| `api_decide(invoice_id, decided_by, decision, note)` | `POST /invoices/{id}/decision` | 30 s |
| `api_batch_decision(invoice_ids, decided_by, decision, note)` | `POST /invoices/batch-decision` | 120 s |
| `api_revalidate(invoice_id)` | `POST /invoices/{id}/revalidate` | 180 s |
| `api_reset()` | `POST /admin/reset` with `{"confirm": "RESET"}` (the user has already typed the phrase in the UI) | 120 s |
| `api_retry_file(event_id)` | `POST /files/{id}/retry` | 300 s |
| `api_report_html(invoice_id)` | `GET /invoices/{id}/report.html` | 30 s |
| `api_ask(question, invoice_id)` | `POST /query` | 300 s |
| `_fetch_original(path)` | `GET` of a document path | 30 s; returns `(bytes, content type)` |
| `api_original(invoice_id)` | `/invoices/{id}/original` | The file as received, for download |
| `api_original_preview(invoice_id)` | `/invoices/{id}/original/preview` | A self-contained HTML page for the side-by-side view |

Empty comments and notes are sent as `None`. The long timeouts match calls that run a workflow (re-validation, retries, questions that call models).

## 3. Cached wrappers

`report_html(invoice_id)`, `original(invoice_id)` and `original_preview(invoice_id)` wrap the matching `api_*` function with `st.cache_data(ttl=5)`, so repeated Streamlit reruns do not refetch the same document.

---

## End-to-end example

The Review page shows an invoice that needs attention:

1. `db_invoices()` returns the invoice with `workflow_stage = "needs_attention"`; `db_violations()` lists its rule codes, for example `["unit_price_mismatch", "line_total_mismatch"]`.
2. The page loads details with `db_invoice()`, `db_runs()` and `db_discrepancies(run_id)`, and shows `original_preview()` beside `report_html()`.
3. The reviewer fixes a price and clicks Fix & Revalidate. The page calls `api_submit_batch(invoice_id, "maria", [{"field_name": "line_items[1].unit_price", "corrected_value": "7.00"}], "Price confirmed with vendor")`.
4. The agent records the batch, re-validates and returns the run summary. If it responds with 409 or 422, `_api_request` raises `AgentError` with the agent's message, which the page shows.
5. The page calls `db_clear_cache()`, so the next render reads the new run and the new stage.
