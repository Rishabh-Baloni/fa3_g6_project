# What test_integration.py does

`tests/test_integration.py` is the end-to-end test suite against a running stack (Docker Compose, or the services run natively). It turns real uploaded DOCX invoices, built in the test with the standard library, into checks on the whole path: agent HTTP API, folder monitor, extraction, validation against the mock ERP, PostgreSQL audit tables, Qdrant and the human review and sign-off workflow.

Run with `pytest tests/test_integration.py -v` after `pip install -r requirements.txt -c constraints.txt`. Each test creates new invoices with a unique number (`INV-IT-<8 hex chars>`), so runs can be repeated.

Main parts:

1. Connection set-up: environment, `db` and `agent` fixtures, `agent_db` fixture
2. DOCX builders and wait helpers
3. Lifecycle and review-limit tests
4. Sign-off tests (decisions, batches, recommendations)
5. Accepted-exception tests
6. Security and database-permission tests
7. Reset tests

---

## Overall flow

```
 pytest ─► ENV = .env values overlaid by os.environ
        ─► fixtures
             agent    httpx.Client(AGENT_URL, default http://localhost:8000), requires GET /health == 200
             db       psycopg as the dashboard role (UI_DB_*, read-only)  → verification reads
             agent_db psycopg as the agent's own role (AGENT_DB_*)        → only to prove append-only triggers
        ─► _invoice_docx(number, price, symbol) ─► build_docx(...)  (zip with [Content_Types].xml, rels, word/document.xml)
        ─► POST /invoices/upload ─► monitor ─► audit  ─► _wait_for(db, SQL) polls every 2 s (timeout 120 s)
        ─► act through the API (corrections, decisions, queries) ─► assert in PostgreSQL, Qdrant, ERP or HTTP responses
```

Ports and paths (from environment): `AGENT_URL` (8000), `QDRANT_URL` (6333), `ERP_URL` (8001), `TEST_INCOMING_DIR` (defaults to `incoming/`), `TEST_DB_HOST`, `TEST_DB_PORT`, `POSTGRES_DB`. Database credentials come from `.env` or the environment.

---

## 1. Helpers

| Function | Purpose |
|---|---|
| `_env_file()` | Reads `KEY=value` lines from `.env` (comments skipped, quotes stripped) |
| `_paragraph`, `build_docx(before, table, after)` | Minimal DOCX bytes (python-docx is not a dependency) |
| `_invoice_docx(number, glove_price, symbol="$")` | An invoice for `PO-1001` from "Global Logistics Ltd" with `SKU-001` (50 x 12.00) and `SKU-002` (120 x glove price), tax 10%. Price `3.00` matches the PO; `3.50` is above tolerance. An empty `symbol` leaves no currency on the document |
| `_wait_for(db, sql, params, timeout=120)` | Polls until the query returns a row, otherwise fails |
| `_upload_and_wait(agent, db, number, content)` | Uploads, asserts 201, waits until the invoice has a validation run |
| `_new_invoice(agent, db, price, symbol)` | Returns `(invoice_id, content, recommendation)` |
| `_stage(db, invoice_id)` | Current `workflow_stage` from `audit.invoice_overview` |
| `_corrections(agent, invoice_id, **body)` | Posts a correction batch as reviewer "ann" |
| `_indexed_text(invoice_id)` | Report chunks stored in Qdrant (`invoice_chunks`) for the invoice |

## 2. Lifecycle and review limit

| Test | What it proves |
|---|---|
| `test_price_above_po_tolerance_goes_to_review` | Price 3.50 against PO 3.00 gives recommendation `review` with exactly one non-low finding: `unit_price_mismatch`, 3.5 against 3, 16.6667% |
| `test_invoice_lifecycle` | Upload, extraction (`docx`, 2 line items, 64-character checksum, missing `currency`), correction to `usd` becomes `USD` and re-validates to `approve`, the original extraction stays unchanged, runs are `initial` then `revalidation`, the feedback row records the original and corrected values, the same bytes under another name are a `duplicate` by checksum, and the HTML and JSON reports include the reviewer note |
| `test_three_failed_attempts_make_the_system_recommend_rejection_not_reject` | Three batches that leave the price wrong: iterations 1 and 2 stay `needs_attention`, iteration 3 gives `recommended_reject`; the AI recommendation stays `review`; a `system` reject recommendation is stored; no human decision exists; extraction is intact; further corrections and recommendations give 409 |
| `test_workflow_stages_follow_the_invoice_lifecycle` | Clean goes to `ready_signoff`, flagged to `needs_attention`, a passing fix goes to `ready_signoff` with iteration 1, three failures go to `recommended_reject` |

## 3. Sign-off

| Test | What it proves |
|---|---|
| `test_a_person_signs_off_a_system_recommendation_like_any_other` | Batch approve skips a recommended rejection; batch reject finalizes it and records `source = batch`, `is_override = False` |
| `test_exceptions_cannot_be_signed_off_from_reports_before_review_is_done` | 409 for a single decision; batch skips it with a "Review Queue" reason |
| `test_final_sign_off_is_final_and_leaves_the_extraction_alone` | Decision recorded (name trimmed, `source = single`), extraction unchanged, stage `finalized`; a second decision or correction gives 409; blank name or invalid decision gives 422 |
| `test_recommend_reject_skips_the_remaining_attempts_and_goes_to_final_sign_off` | Note required (422); stage `recommended_reject` with no attempt used; further corrections and recommendations give 409; the recommendation row is append-only even for the owner role; final reject works |
| `test_a_decision_either_confirms_the_recommendation_or_overrides_it_with_a_reason` | Override without a reason gives 422 and records nothing; with a reason `is_override` is true; confirming a recommended rejection needs no reason |
| `test_the_ai_recommendation_is_never_changed_by_a_human_decision` | `recommendation` stays `approve` and status `passed` after a human reject |
| `test_a_recommendation_needs_an_invoice_that_is_still_in_review` | A clean invoice gives 409 "waiting for final sign-off" |
| `test_final_batch_decisions_only_touch_invoices_at_the_right_stage` | Batch approve decides only ready invoices, batch reject only recommended ones; others (including unknown ids) come back in `skipped`; invalid or empty id lists give 422 |

## 4. Accepted exceptions

| Test | What it proves |
|---|---|
| `test_a_reviewer_can_accept_a_missing_field_as_is_and_the_invoice_moves_on` | Without a note, with a key that is not an open exception, or with nothing at all: 422 and no attempt used. With a note: 201, outcome `resolved`, stage `ready_signoff`, `accepted_exceptions` row (`missing:currency`) and no feedback row, report and HTML show it, and the row is append-only |
| `test_a_medium_finding_can_be_accepted_as_is` | `finding:unit_price_mismatch:2:unit_price` is accepted and no longer in the discrepancies |
| `test_a_high_severity_finding_cannot_be_accepted` | `line_not_on_po` is high; accepting it gives 422 "cannot be accepted" and records nothing |

## 5. Serving, Ask AI, permissions

| Test | What it proves |
|---|---|
| `test_report_and_original_are_served_by_the_agent` | `/original` returns the exact bytes, DOCX media type, inline disposition with no container path; `report.html` has `default-src 'none'`; unknown or malformed ids give 404 |
| `test_original_preview_is_browser_renderable` | DOCX preview is HTML with the invoice number, line items and a table; PDFs preview as `data:image/png;base64,` pages |
| `test_ask_ai_can_see_the_reviewers_recommendation_and_the_final_decision` | Skipped without an embedding model. After a recommend-reject and a decision, Qdrant text is updated in the background and contains the reviewer's text and "Final decision: rejected by dana" |
| `test_structured_question_is_answered_from_sql` | `POST /query` with "How many invoices were approved?" returns `route == "sql"` |
| `test_rejects_unsupported_upload` | A `.txt` upload gives 415 |
| `test_human_feedback_is_append_only_even_for_the_owner_role`, `test_dashboard_role_cannot_write_decisions`, `test_dashboard_role_cannot_write_or_read_erp` | UPDATE and DELETE on feedback fail with "append-only"; the dashboard role cannot insert decisions, delete feedback or read `erp.purchase_orders` |

## 6. Reset

| Test | What it proves |
|---|---|
| `test_reset_refuses_without_the_confirmation_phrase_and_deletes_nothing` | Wrong phrases give 422 and the invoice count is unchanged |
| `test_reset_starts_over_and_the_system_works_again` | Runs only when `AUDITOR_ALLOW_DESTRUCTIVE_TESTS=1`, because it wipes the whole stack under test. After `POST /admin/reset`: all audit tables and `langgraph` checkpoint tables are empty, the incoming folder has no files, the Qdrant collection returns 404, the report is 404, the ERP data (`/po/PO-1001`) remains, and the same file is audited again as new (not a duplicate), with `file_events` ids starting at 1 |

Why polling: processing is asynchronous. The monitor picks up the file and the pipeline runs in the background, so the tests wait on database state instead of sleeping for a fixed time.

Related: `tests/verify.sh` is a shell script (not Python) that runs the six infrastructure checks (clean start, end-to-end processing, persistence across `down`, data loss with `down -v`, no secrets in image layers, service-name resolution). It uses `docker compose`, `curl` and `psql`, and complements this file.

---

## End-to-end example

`test_invoice_lifecycle`:

1. `_invoice_docx(number, "3.00", symbol="")` builds a DOCX with matching prices and no currency; `POST /invoices/upload` returns 201.
2. `_wait_for` polls `audit.invoice_overview` until the invoice has a run; recommendation is `review` and `missing_fields == ["currency"]`.
3. `POST /invoices/<id>/corrections` with `currency = usd`: the response has `revalidation.recommendation == "approve"`; `invoice_audit.currency` is still `NULL` (the extraction is untouched), `invoice_overview.currency` is `USD`, and the feedback row has `corrected_value = "USD"`, `review_iteration = 1`.
4. The same bytes are uploaded as `renamed-<number>.docx`; the file event ends as `duplicate` pointing at the same invoice.
5. `GET /invoices/<id>/report.html` and `report.json` include the reviewer's note.
