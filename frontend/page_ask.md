# What page_ask.py does

`frontend/page_ask.py` renders step 4 of the dashboard, "Ask AI". It turns a typed question (optionally scoped to one invoice) into a request to the agent's `/query` endpoint and renders the answer as a chat: which route answered (SQL or RAG), the answer text, and the evidence (executed SQL and rows, or relevance scores and sources).

Main parts:

1. `route_badge`: shows which route answered and how it was chosen
2. `render_answer`: draws one answer, SQL or RAG
3. `ask_page`: scope selector, chat history, input box

---

## Overall flow

```
 ask_page()
   ├─ data.db_invoices() ──► "Search in" selectbox: None = all invoices, or one invoice_id
   ├─ st.session_state["conversation"] ──► past turns ──► render_answer(result)
   └─ st.chat_input(question) ──► data.api_ask(question, scope) ──► agent POST /query
                                                                       │ (backend/query.py)
                                      result: route = "sql" ◄──────────┤ read-only SQL on audit tables
                                              route = "rag" ◄──────────┘ vector search + generation + triad scores
        append {question, scope label, result} to the conversation ─► st.rerun()
```

State: the whole conversation lives in `st.session_state["conversation"]` (a list of `{question, scope, result}`), so it survives reruns but not a new browser session. The "Start over" reset on the Intake page removes it. The page does no routing itself; the agent decides the route.

---

## 1. `route_badge(result)`

| Input | Output |
|---|---|
| Result dict with `route` and optional `router` | HTML: the route name (`SQL` or `RAG` from `ROUTE_LABELS`, otherwise the upper-cased value) in a pill, plus "route chosen by the router model" if `router.source == "llm"`, else "route chosen by keyword rules", plus ` · <reason>` when `router.reason` exists. All values are escaped with `ui.esc` |

## 2. `render_answer(result)`

| Route | What is shown |
|---|---|
| `sql` | If `accepted` is false the answer is a warning, otherwise Markdown. Then: `note` as a caption; the executed SQL in an expander "Executed SQL"; the rows in a DataFrame (columns from `result["columns"]`); a caption "`n` row(s)", plus "more rows exist, only the first are shown" when `truncated`, plus "Answered from the audit tables (read-only query)." |
| `rag`, not accepted | If scores exist: a warning that no answer passed the quality check after several retrieval attempts and this is the best attempt, so the sources should be checked, and the answer is still shown. If there are no scores, only the answer text as a warning and nothing more |
| `rag`, accepted or best attempt | The answer; search filters (empty values dropped, `_ids` removed from names); a caption with the triad scores (answer relevance, groundedness, context relevance, two decimals), the number of retrieval attempts and "repeated question, served from cache" if `cached`; an expander "Sources (n)" with, per source, the invoice number (or "(no number)"), "audit report" or "invoice text", and an escaped excerpt |

Why show the scores and the SQL: a reader can check why an answer is trusted, and SQL answers can be verified by reading the executed statement.

## 3. `ask_page()`

| Step | Behaviour |
|---|---|
| Header | `ui.page_header("Ask AI", 4, ui.ASK_BANNER)` |
| Invoices | `data.db_invoices()`; on an exception shows "Could not load invoices: ..." and `st.stop()` |
| Scope | "Search in" select box with "All processed invoices" and one entry per invoice, labelled `<invoice number or (no number)> — <vendor>`; the chosen value is the invoice id or `None` |
| History | For each stored turn, a user message (escaped question and the scope label) and an assistant message. A turn whose result has `error` but no `route` is shown with `st.error`; others go through `render_answer` |
| Hint | With no conversation, a caption with example questions |
| Input | `st.chat_input`. A question runs `data.api_ask(question, scope)` inside a spinner; an `AgentError` is stored as `{"error": message}`. The turn is appended and the page reruns |

Edge cases: a 503 (model not configured) or 502 (model call failed) from the agent arrives as an `AgentError` and is displayed as an error turn; the conversation continues. Everything that goes into HTML is escaped; the answer text is rendered as Markdown without `unsafe_allow_html`.

---

## End-to-end example

1. The user leaves the scope at "All processed invoices" and types "How many invoices were rejected?".
2. `api_ask` posts `{"question": ..., "invoice_id": null}`. The agent routes it to SQL and returns `route: "sql"`, the answer, the executed SQL, rows, `row_count`.
3. The turn is stored and the page reruns. The badge reads "SQL route chosen by keyword rules" (or by the router model), the answer is shown, "Executed SQL" is expandable, and the caption reads "1 row(s). Answered from the audit tables (read-only query)."
4. The user selects one invoice and asks "Why was it flagged?". The agent answers by RAG; the page shows the answer, the filter caption, the triad scores, and a "Sources (2)" expander listing the audit report and invoice-text excerpts.
