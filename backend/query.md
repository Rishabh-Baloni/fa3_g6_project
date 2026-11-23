# What query.py does

`backend/query.py` answers natural-language questions by choosing between two paths: a model-written SQL query over the audit tables (counts, totals, lists, averages) or the open-ended RAG pipeline in `rag.py`. Its main job beyond routing is the SQL guard: model-written SQL is never trusted and must pass `validate_select` before it is run in a read-only transaction.

Main parts:

1. SQL guard: `validate_select`, `ensure_limit`, `json_safe`
2. Router: `parse_route`, `heuristic_route`, `RouteDecision`
3. `QueryWorkflow`: the LangGraph state machine
4. Entity resolution: invoice number or vendor name to ids

---

## Overall flow

```
 question (+ optional invoice_id / vendor selected in the UI)
        │
        ▼
      route ── reasoning model JSON ── parse_route ──(bad / no model)──► heuristic_route (keywords)
        │
        ├─ RAG ──► rag node ─► RagPipeline.ask_open(filters)  (retrieve → augment → generate → reflect, see rag.md) ─► END
        │
        └─ SQL ─► sql_generate ──► sql_execute ──ok──► sql_summarize ─► END
                      ▲   (model writes SQL)  │ validate_select → ensure_limit → scope check
                      │                        │ → AuditRepository.run_readonly_select (READ ONLY, 5 s timeout, max 200 rows)
                      └──── retry once ────────┘ rejected / DB error
                                               └─ still failing ─► sql_fallback ─► try_structured_answer (canned SQL) or an apology ─► END
```

The router and SQL generator use the reasoning model; the RAG path uses the RAG model. All prompts live in `prompts/prompts.yaml`. The graph keeps no state between questions; each `ask()` builds a fresh `QueryState`.

---

## 1. SQL guard

| Name | Value / purpose |
|---|---|
| `ALLOWED_TABLES` | `invoice_audit`, `invoice_line_items`, `discrepancies`, `human_feedback`, `reject_recommendations`, `human_decisions`, `accepted_exceptions` (all in schema `audit`) |
| `FORBIDDEN_SCHEMAS` | `erp`, `langgraph`, `public`, `information_schema` (and any schema starting `pg_`) |
| `MAX_ROWS` / `DEFAULT_LIMIT` | 200 rows returned at most; `LIMIT 50` added to queries without one |
| `SqlRejectedError` | A `ValueError` with a message that is safe to show and to feed back to the model |

| Function | Input | Output |
|---|---|---|
| `validate_select(sql)` | Model SQL | The cleaned statement, or `SqlRejectedError` |
| `_check_tables(body)` | SQL body | Scans tokens; checks forbidden schemas and every table after `FROM`/`JOIN` |
| `_check_from_list(tokens, index, ctes)` | Tokens | Validates a comma list of tables with optional `AS` aliases; a `(` means a subquery and is scanned by the caller. CTE names are allowed |
| `ensure_limit(sql, default=50)` | SQL | Adds `LIMIT` unless one exists or the query is a pure aggregate (aggregate function without `GROUP BY`) |
| `json_safe(value)` | Any DB value | `Decimal` to string (money stays exact), dates to ISO text, UUID to string, dict/list round-tripped through JSON |

`validate_select` checks, in order:
1. Remove comments (`--`, `/* */`); empty is rejected.
2. Blank out string literals (keywords inside strings do not count); an unbalanced quote is rejected.
3. Strip one trailing `;`; any other `;` is rejected (single statement only).
4. Must start with `SELECT` or `WITH`.
5. Reject write and admin keywords: `insert`, `update`, `delete`, `merge`, `drop`, `alter`, `truncate`, `create`, `grant`, `revoke`, `copy`, `call`, `execute`, `vacuum`, `into`.
6. Reject double quotes and `$` (quoted identifiers, dollar quoting).
7. Reject function calls such as `pg_*`, `lo_*`, `dblink*`, `set_config`, `current_setting`, `query_to_xml*`, `xpath*`.
8. Table check: every table must be written `audit.<table>` from the allowed list.

Why so strict: this is defence in depth. Even if the guard missed something, `run_readonly_select` executes in a `READ ONLY` transaction with a statement timeout. `FROM` inside `EXTRACT(...)`, `SUBSTRING(...)`, `TRIM(...)`, `OVERLAY(...)`, `POSITION(...)` and `IS DISTINCT FROM` is not treated as a table list.

## 2. Router

| Item | Purpose |
|---|---|
| `RouteDecision(route, filters, reason, source)` | `route` is `SQL` or `RAG`; `filters` has `invoice_id`, `vendor`, `language`, `source_type`; `source` is `llm` or `heuristic`; `as_dict()` for the state |
| `parse_route(payload)` | Validates the router model's JSON. Unknown route raises `ValueError`. `language` must be two lowercase letters; `source_type` must be `invoice` or `report`; `"null"`, `"none"` and `"n/a"` count as empty (`_text`) |
| `heuristic_route(question)` | Keyword fallback. SQL if the question is not open-ended and matches the count/list/total patterns from `rag.py` or `_SQL_WORDS` (average, sum, top, highest, between, per vendor, ...). RAG otherwise. Sets `source_type="report"` when the wording mentions flags, rejects, discrepancies |

The private regex constants `_COUNT`, `_LIST`, `_OPEN_ENDED`, `_REPORT_INTENT` and `_TOTAL` are imported from `rag.py`, so both modules classify questions the same way.

## 3. `QueryWorkflow`

`QueryWorkflow(gateway, repository, rag)`. `ask(question, *, invoice_id=None, vendor=None)` runs the graph and returns `final["result"]`.

| Node | What it does |
|---|---|
| `route` | If a reasoning model is configured, asks the `query_router` / `router_classify` prompt (300 tokens) and parses it. A model error or a `ValueError` falls back to `heuristic_route` |
| `sql_generate` | Builds a scope hint from the invoice or vendor (UI selection wins over the router's filter): "Restrict the query to invoice_id IN (...)" and "vendor_name IN (...)" with quotes doubled. If a previous attempt failed, adds the error and the old query as feedback. Calls `sql_analyst` / `sql_generate` (700 tokens). An empty answer or a model error jumps straight to fallback (`sql_attempt` is set to the maximum) |
| `sql_execute` | `ensure_limit(validate_select(sql))`. If the question was scoped to specific invoices, the SQL must mention one of their ids. Then `run_readonly_select(sql, max_rows=200)`. `SqlRejectedError` and any database error are stored as `sql_error` so the model can repair them. Rows go through `json_safe` |
| `_after_execute` | `ok` if no error; `retry` while `sql_attempt < MAX_SQL_ATTEMPTS` (2); otherwise `fallback` |
| `sql_summarize` | Plain summary first (`_fallback_summary`: "No records matched.", a single value, or "N row(s) returned"). If a reasoning model is configured, it writes the summary from at most 25 rows (250 tokens); a failure keeps the plain one |
| `sql_fallback` | Tries `try_structured_answer` from `rag.py` (a canned query); returns it with a note explaining why the model's SQL was not used. If that does not apply, returns an apology with the reason and `accepted=False` |
| `rag` | Resolves invoice and vendor names to ids. If a named invoice or vendor does not exist, returns "No processed invoice/vendor matches ..." rather than silently searching everything. Otherwise calls `RagPipeline.ask_open` and attaches the router decision |

Result shape for SQL: `route: "sql"`, `router`, `answer`, `accepted`, `cached: False`, `sql`, `columns`, `rows`, `row_count`, `truncated`, `scores: None`, `sources: []`, `attempts: []`.

## 4. Entity resolution

| Method | Input | Output |
|---|---|---|
| `_resolve_invoice_ids(identifier)` | A UUID or an invoice number as typed | A valid UUID is returned as is. Otherwise looks up `invoice_number` case-insensitively and exactly, then with `ILIKE %...%`. Parameterised queries |
| `_resolve_vendors(name)` | Vendor name fragment | Distinct vendor names matching `ILIKE %name%` |
| `_scope(state)` | State | `(invoice ids, prompt hint)` |

---

## End-to-end example

Question: "What is the average invoice total per vendor?"

1. `route` asks the router model. It returns `{"route": "SQL", ...}`. If the model were not configured, `heuristic_route` would match "average" and "per vendor" and also choose SQL.
2. `sql_generate` calls the SQL prompt and gets `SELECT vendor_name, avg(total_amount) FROM audit.invoice_audit GROUP BY vendor_name`.
3. `sql_execute`: `validate_select` accepts it (single `SELECT`, table `audit.invoice_audit` allowed). `ensure_limit` sees an aggregate with `GROUP BY` and no `LIMIT`, so it appends `LIMIT 50`. `run_readonly_select` runs it read-only.
4. Decimals are returned as strings. `sql_summarize` asks the model for a one-or-two sentence summary of the rows.
5. If the model had written `FROM vendors` instead, `_check_from_list` would reject it ("'vendors' is not an allowed table"). That message goes back to `sql_generate` for one repair attempt, and if it fails again, `sql_fallback` tries the canned answer or apologises.
