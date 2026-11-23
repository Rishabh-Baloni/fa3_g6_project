# What page_review.py does

`frontend/page_review.py` renders step 2 of the dashboard, "Review Queue". It turns the invoices in the `needs_attention` stage into a queue and, for one opened invoice, into a review workspace where a person can fix values, accept minor exceptions or recommend rejection. Every change is sent to the agent; the page only reads from the database.

Main parts:

1. The queue: `review_page`, `violation_summary`, `close_workspace`
2. The workspace: `review_workspace` and its four tabs
3. The fix form: `_fix_tab`, `acceptable_exceptions`, `item_code_warnings`, `_submit_fix`, `_recommend_reject`, `_revalidate_action`
4. Read-only tabs: `_compare_tab`, `_document_tab`, `_history_tab`
5. Helpers: `earlier_notes`, `_po_line_for`, `_similar`, `_violations_panel`, `_accepted_panel`

---

## Overall flow

```
 review_page()
   data.db_invoices() ─► rows with workflow_stage == "needs_attention"  ─► table (single-row select), data.db_violations()
        │ select a row ─► session["review_open"] = invoice_id   (URL query ?id=<invoice_id> is also honoured)
        ▼
 review_workspace(row)    data.db_invoice, db_runs, db_discrepancies
   ├─ header + _violations_panel + _accepted_panel
   └─ tabs: Fix & Revalidate | Invoice vs purchase order | Document | History
 Fix & Revalidate
   form ─► ui.changed_fields + ticked exceptions ─► item_code_warnings (ask first)
        ├─ [Fix & Revalidate]  ─► data.api_submit_batch  ─► POST /invoices/{id}/corrections (revalidate=true)
        ├─ [Recommend Reject]  ─► data.api_recommend_reject ─► POST /invoices/{id}/recommend-reject
        └─ [Re-validate without changes] ─► data.api_revalidate ─► POST /invoices/{id}/revalidate
 after an action: data.db_clear_cache → ui.flash → close_workspace (if the invoice left the queue) → st.rerun
```

State: `review_open` (the open invoice), `review_nonce` (part of the table key, so closing the workspace drops the old selection), `pending_fix` (a fix waiting for the "Submit anyway" confirmation), `reviewer` (name field shared with Reports). Widget keys include the run number (`run_key`), so the form shows freshly validated values after every run. The attempt limit is enforced by the agent; the page only displays `ui.MAX_REVIEW_ITERATIONS`.

---

## 1. The queue

| Function | Input | Output |
|---|---|---|
| `close_workspace()` | none | Clears `review_open`, removes the `id` query parameter, increments `review_nonce` |
| `violation_summary(codes)` | Rule codes | "—" if none; the first rule label, plus " (+n more)" |
| `review_page()` | none | Shows header and counts (Needs attention, On their final attempt (`review_iteration >= max - 1`), Already moved to Reports). If an invoice id is open and still `needs_attention`, shows its workspace; if it has left the queue, shows an info message pointing to Reports and closes. Empty queue: empty state with links to Intake and Reports. Otherwise a search box (substring on number or vendor) and a single-row table: Attempt (red on the final attempt, otherwise amber), Invoice, Vendor, Total, Rule exceptions, Missing fields, File |

## 2. The workspace

`review_workspace(row)` loads `db_invoice`, `db_runs` and the latest run's discrepancies. If there is no record or run it shows "This invoice has not finished processing yet." The header uses the values validated in the latest run (`report["invoice"]`, corrections included), with status and attempt badges. `_violations_panel` lists the findings and missing fields that put the invoice in the queue (escaped; an info message if there are none). `_accepted_panel` lists exceptions accepted earlier, taken from `report["accepted_exceptions"]`.

| Tab | Content |
|---|---|
| Fix & Revalidate | `_fix_tab` |
| Invoice vs purchase order | `_compare_tab` |
| Document | `_document_tab` |
| History | `_history_tab` |

## 3. Fix & Revalidate

`_fix_tab`:
- Progress bar of attempts used and an explanation of what happens after the last attempt (the system recommends rejection and a person decides in Reports).
- Earlier reviewer notes (`earlier_notes`).
- Fields flagged by the last run (finding reasons and missing fields) get a `flag_` widget key (`flag_missing_` for missing ones) and a warning sign in the label; the stylesheet colours them amber or red, and the help text carries the reason.
- The form: nine header fields (`HEADER_FIELDS`) and five fields per line (`LINE_FIELDS`), prefilled with the validated values.
- "Accept as is" checkboxes from `acceptable_exceptions`.
- A reviewer note, plus two submit buttons.

| Function | Input | Output |
|---|---|---|
| `acceptable_exceptions(latest, found)` | Latest run, findings | `(acceptable, blocked)`: missing fields and medium or low findings can be accepted (keys built with `ui.exception_key`); high-severity messages are blocked and listed in a caption |
| `item_code_warnings(changes, shown, po, threshold=0.6)` | Changed fields, shown values, the PO from the report | Warnings for each typed item code that is not on the PO, with the PO item whose description is at least 0.6 similar (`difflib.SequenceMatcher`) when there is one. Empty if there is no PO. It never blocks |
| `_similar(left, right)` | Two strings | Case-insensitive ratio from 0 to 1 |
| `_submit_fix(...)` | Invoice, reviewer, changes, note, number, limit, accepted | Spinner, `api_submit_batch`; `AgentError` shows "Corrections not saved: ...". Success flashes one of: rejection recommended automatically (`review_outcome == "recommended_reject"`, workspace closes); passed on attempt n of limit (`recommendation == "approve"`, workspace closes); still flagged, attempt n used and n+1 is next (workspace stays open). Then rerun |
| `_recommend_reject(invoice_id, name, note, number)` | | Needs a non-empty note; calls `api_recommend_reject`, flashes, closes the workspace |
| `_revalidate_action(invoice_id)` | | Expander "Re-run the checks": `api_revalidate`, toast with the run number and recommendation label; no attempt is used |
| `earlier_notes(feedback)` | Feedback rows | Notes from earlier attempts, oldest first, de-duplicated by attempt, author and text |

Checks on submit, in order: reviewer name required; Recommend Reject needs a note; Fix needs at least one changed value or ticked exception ("no attempt was used" otherwise); accepting needs a note (recorded as the reason); `item_code_warnings` store a `pending_fix` and ask "Submit anyway" or "Edit the values", because a typo should not cost one of the attempts. The agent validates again (duplicates, field values, attempt limit).

## 4. Read-only tabs

| Function | Output |
|---|---|
| `_po_line_for(item, po_lines, notes)` | The PO line for an invoice line: by item code (case-insensitive), or, when the code is missing, from a validation note matching `Line <n> matched PO item <code> by` |
| `_compare_tab(latest, shown, found)` | Table of vendor and currency against the PO, and a line table of quantity and unit price, invoiced against ordered. Cells for business-rule findings are highlighted; lines not on the PO appear in red as "Not on the purchase order". If the report has no PO: an info message |
| `_document_tab(record)` | The original as an HTML preview next to the extracted text (original plus English translation if one exists). Caption: amounts are always taken from the original text, never from the translation |
| `_history_tab(invoice_id, history, shown, report)` | Every run (first audit or re-validation, time, recommendation, summary), corrections and notes, accepted exceptions, and download buttons for the latest report as HTML (from the agent) and JSON (from the stored report) |

---

## End-to-end example

1. The queue shows `INV-1001` with "Attempt 1 of 3" and "Unit price differs from the PO". The user selects the row; `review_open` is set and the workspace opens.
2. The Fix tab shows the unit price box flagged amber. The reviewer enters the name "maria", changes line 1's unit price from `10.00` to `7.00` and clicks "Fix & Revalidate (Attempt 1 of 3)".
3. `changed_fields` finds one change; no exceptions are ticked; `item_code_warnings` returns nothing, so `api_submit_batch` is called.
4. The agent records feedback, re-validates and returns `recommendation: "approve"` with iteration 1. The page flashes "INV-1001 passed validation on attempt 1 of 3 → moved to Reports as Validated - Pending Sign-off.", closes the workspace and reruns. The queue no longer lists it.
5. Alternative: the reviewer types an item code that is not on the PO. `item_code_warnings` shows "Line 1: 'X9' is not an item on purchase order PO-1. The description '...' matches PO item ...", and nothing is sent until "Submit anyway" is clicked.
