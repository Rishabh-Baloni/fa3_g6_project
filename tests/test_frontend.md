# What test_frontend.py does

`tests/test_frontend.py` tests the Streamlit dashboard without a browser, a database or an agent. It turns fake database rows and fake agent responses into rendered pages (using Streamlit's `AppTest`) and asserts what is shown and what is sent to the agent. It checks page logic and wiring, not the services themselves.

Main parts:

1. Fixtures and fake data (`ROWS`, `REPORT`, `DISCREPANCY`, the autouse `fakes` fixture)
2. Test helpers (`run_page`, `all_markdown`, `_select`, `_idx`, `_button`, ...)
3. Test groups by page: navigation, overview, intake, review, reports, ask, shared helpers (67 tests)

---

## Overall flow

```
 pytest ─► autouse fixture `fakes`
             monkeypatches data.db_* (in-memory rows) and data.api_* (record calls into a dict)
             also st.page_link / st.switch_page, and clears the st.cache_data wrappers
        ─► run_page("review") ─► AppTest.from_function(script) ─► ui.apply_style(); ui.NAV stand-ins; page function
        ─► click / select / type ─► app.run() ─► assert on app.markdown, app.button, app.toast, app.warning, ...
        ─► assert on the recorded calls: calls["fix"], calls["decisions"], calls["batches"], ...
```

State: none persists between tests. The fixture returns the `calls` dict, which collects what each fake `api_*` function received (`fix`, `recommend`, `decisions`, `batches`, `questions`, `previews`, `uploads`, `resets`, `switched`, plus `accepted`).

---

## 1. Fake data and fixtures

| Name | Purpose |
|---|---|
| `REVIEW_ID`, `READY_1`, `READY_2`, `HITL_ID`, `REC_ID`, `SYS_ID`, `DECIDED_ID` | Fixed invoice UUIDs, one per situation (needs attention; auto-approved twice; fixed in review on attempt 2; reviewer-recommended rejection; system-recommended rejection after three attempts; final decision recorded) |
| `REPORT`, `DISCREPANCY` | A stored report for `INV-1001` with a PO (`PO-1001`, SKU-002 at 3.00) and one medium `unit_price_mismatch` finding (3.5 against 3) |
| `_row(...)` | Builds one `invoice_overview`-shaped row |
| `ROWS` | Seven invoices: 1 needs attention, 3 ready for sign-off, 2 recommended rejections, 1 finalized |
| `fakes` (autouse fixture) | Replaces the `data` functions listed above, `st.page_link` and `st.switch_page`; the fake `submit` returns iteration 2 and an outcome dict `calls["outcome"]` that tests change to simulate a pass or a failed last attempt |

## 2. Helpers

| Function | Purpose |
|---|---|
| `run_page(page, **state)` | Runs one page function in an `AppTest` (with styling and stand-in navigation names), optionally preloading session state; asserts there was no exception |
| `all_markdown(app)` | All Markdown output as one string |
| `_pick`, `_select(app, table, *rows)` | Inject a row selection for the table `<table>_table_<nonce>`, then run |
| `_idx(app, *numbers)` | Row positions of invoices as they are shown (the ledger sorts actionable invoices first) |
| `_button`, `_submit`, `_text`, `_area` | Find a button, form button, text input or text area by label |

## 3. Test groups

| Group | What the tests prove |
|---|---|
| Navigation (1 test) | `app.main()` registers pages in order: Overview, "1 · Intake", "2 · Review Queue (1)", "3 · Reports & Sign-Off (5)", "4 · Ask AI"; the Overview is the default; the sidebar shows "Language models not configured" |
| Overview (6) | Hero, four steps and four live counts (7 total, 1 pending, 5 ready, 1 finalized); the text matches reality (mentions `config/rules.yml` and recommending rejection, not auto-reject); start buttons call `st.switch_page` for Intake and Ask AI; empty database caption; page still renders when the database raises |
| Intake (8) | File outcomes and routing in the table; uploads go to `api_upload`; toasts announce where a file went but not on the first render; "Start over" asks before deleting; reset needs the phrase typed; Cancel deletes nothing; reset calls the agent once, forgets screen state and confirms; a failed reset is reported and keeps the screen |
| Review queue (about 25) | Only `needs_attention` invoices are listed; empty state; opening a row shows the attempt counter and violations; missing fields have readable names; flagged boxes are highlighted with reasons; Fix & Revalidate sends every changed field with the note; a pass moves to Reports; a failed attempt stays and names the next attempt; failing the last attempt recommends rejection; name and change are required and use no attempt otherwise; item-code warnings (not on the PO, with or without a similar description, "Edit the values" sends nothing, a code on the PO needs no confirmation, no warnings without a PO); accepting exceptions (missing and medium findings only, note required as the reason, nothing changed and nothing accepted uses no attempt, high severity cannot be accepted); accepted exceptions shown with who and why, also on the sign-off page; Recommend Reject needs a note and moves to Reports; one reviewer note, earlier notes shown on the next attempt; an invoice that left the queue is not opened for editing |
| Reports (about 20) | Ledger shows every invoice with its workflow origin; stage pills; AI-recommendation filter alone and combined with the pills; filter empty states; batch confirms only ready invoices for approval and recommended rejections for rejection; batch needs a reviewer and something eligible; no batch override; one selected row opens the workspace with the reviewer's recommendation and corrections; confirming sends the matching decision with no reason; overriding needs a reason and sends the opposite decision; button meaning follows the recommendation; invoices in review or final have no sign-off buttons; a system recommendation after three attempts gets a person to sign off; `embed_report` adds pane styles while the download stays standalone |
| Ask AI (4) | Question sent with scope; SQL answer shows route, SQL and table; RAG answer shows route, filters and triad scores; a failed SQL result is a warning without a table |
| Shared helpers (6) | `ui.status`, `workflow_origin`, `attempt_text` for every stage; `colour_cells` works with integer, float and datetime columns (regression: assigning `""` into an int column failed); AI vocabulary, `recommended_outcome`, `signoff_text` with override; banners; `field_label` and `rule_label` |

Why `AppTest` with fakes: the dashboard's behaviour depends on what it sends to the agent (for example one batched request per attempt), and that can be asserted without running PostgreSQL or the agent.

---

## End-to-end example

`test_fix_and_revalidate_sends_every_changed_field_with_the_note_and_moves_a_pass_to_reports`:

1. `run_page("review")` renders the queue from `ROWS` (one invoice, `INV-1001`).
2. `_select` picks its row; the workspace opens with the flagged unit-price box.
3. The test types a reviewer name, changes the field, adds a note and clicks "Fix & Revalidate".
4. The fake `submit` records `(invoice_id, by, corrections, comment)` in `calls["fix"]`; its outcome says `approve`.
5. The assertions check the recorded request (all changed fields in one call, with the note) and that the page flashes the "moved to Reports" message.
