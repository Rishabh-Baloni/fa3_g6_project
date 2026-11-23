# What page_reports.py does

`frontend/page_reports.py` renders step 3 of the dashboard, "Reports & Sign-Off". It turns the list of validated invoices into an audit ledger with filters, and into a sign-off workspace where an approver sees the original document and the generated audit report side by side and records the final decision (one invoice, or a batch) by calling the agent.

Main parts:

1. Filtering and ordering: `PILLS`, `AI_FILTERS`, `filter_frame`
2. The ledger table: `reports_page`
3. Batch sign-off: `batch_panel`
4. The single-invoice workspace: `signoff_workspace`, `_audit_trail`, `embed_report`
5. The decision bar: `_decision_bar`
6. Navigation state: `close_workspace`

---

## Overall flow

```
 reports_page()
   data.db_invoices() ─► counts ─► filters (stage pill, AI recommendation, text search) ─► st.dataframe (multi-row select)
        │ one row selected ──► session["reports_open"] = invoice_id ─► signoff_workspace(row)
        │ several rows       ──► batch_panel ──► data.api_batch_decision ──► POST /invoices/batch-decision
        ▼
 signoff_workspace(row)
   ├─ _audit_trail: origin, reviewer/system rejection note, accepted exceptions, corrections (data.db_accepted, db_feedback)
   ├─ summary + top findings (data.db_runs, db_discrepancies)
   ├─ left pane:  data.original_preview(id) ─► iframe     [Download original: data.original(id)]
   ├─ right pane: data.report_html(id) ─► embed_report ─► iframe  [Download report]
   └─ _decision_bar ─► data.api_decide ─► POST /invoices/{id}/decision
 after any decision: data.db_clear_cache, ui.flash, close_workspace, st.rerun
```

State: `reports_open` (invoice shown in the workspace), `reports_nonce` (part of the table key; changing it drops the old row selection so the same row does not reopen), `reviewer` (name field shared with the Review page), `batch_note`, and per-invoice `comment_<id>`. Final decisions are enforced by the agent, not by this page.

---

## 1. Filtering and ordering

| Name | Purpose |
|---|---|
| `PILLS` | "All", "Ready for Sign-off", "Recommended Rejections", "Finalized"; `_PILL_STAGES` maps the last three to workflow stages |
| `AI_FILTERS` | "Any AI recommendation", "AI: Accept" (`approve`), "AI: Needs Attention" (`review`), "AI: Reject" (`reject`) |
| `_EMPTY` | Empty-state title and text per pill |
| `_LEDGER_ORDER` | Sort order: ready for sign-off, recommended reject, needs attention, processing, finalized |

| Function | Input | Output |
|---|---|---|
| `filter_frame(frame, pill, ai=None)` | Invoices, pill, AI value | Frame filtered by stage (unless "All") and `recommendation`, with a stable sort by `_LEDGER_ORDER`, so actionable invoices come first |

## 2. `reports_page`

- Header with step 3 and `REPORTS_BANNER`; database errors show "Could not load invoices: ..." and stop.
- If `reports_open` is set and that invoice still exists, shows `signoff_workspace` and returns.
- Empty database: empty state and a link to Intake.
- Counts: Ready for sign-off, Recommended rejections, Still in review, Finalized.
- Controls: segmented pill, AI-recommendation select, search box (case-insensitive substring match on invoice number or vendor, not a regex).
- Table columns: Status, AI recommendation, Invoice, Vendor, Total, Workflow origin, Reviewer note, Sign-off. Status and AI cells are coloured. Rows are selectable (multi-row, reruns on selection).
- One selected row opens the workspace (sets `reports_open`, reruns). Several selected rows show a divider, a reviewer name field and `batch_panel`.

The AI column explains in its help text that the AI recommendation is never changed by a person's decision.

## 3. `batch_panel(selected, reviewer)`

| Behaviour | Detail |
|---|---|
| Eligibility | `ui.can_batch_approve` (Ready for Sign-off) and `ui.can_batch_reject` (Recommended Reject). If neither applies, a warning explains why |
| Summary | Counts to approve and to reject, plus how many selected rows are left unchanged |
| Buttons | "Confirm recommendation: approve (n)" (primary) and "Confirm recommendation: reject (n)", disabled when n is 0. An optional note |
| Validation | A missing reviewer name shows "Enter the reviewer name." |
| Call | `data.api_batch_decision(ids, reviewer, decision, note)`. `AgentError` shows "Batch sign-off failed: ...". Success flashes the number confirmed and up to three skipped reasons (warning if none were decided) |

Why confirm only: a batch can only agree with recommendations. An override needs its own reason, so it is done one invoice at a time.

## 4. Single-invoice workspace

| Function | Input | Output |
|---|---|---|
| `embed_report(report_html)` | Report HTML | Same HTML with `_REPORT_EMBED_CSS` inserted before `</head>` (or prepended if there is none), so it fits the pane; the downloadable report is the unchanged original |
| `signoff_workspace(row)` | Invoice row | "← Back to Reports"; heading with number, status badge and AI recommendation badge; vendor, total and sign-off text; `_audit_trail`; the summary and up to 3 findings plus up to 3 missing fields from the latest run (skipped if run data cannot be read); the two panes; the decision bar |
| `_audit_trail(row)` | Invoice row | Workflow origin; a red box for a rejection recommendation ("Rejection recommended automatically (all review attempts used)" when by `system`, else "Reviewer recommended rejection") with the note, author and date; a green box listing accepted exceptions with who, attempt and reason; an expander table of corrections and notes |

Panes: the left pane is the original as an HTML preview with a "Download original" button; the right pane is the report with "Download report" (file name `<invoice number>.html`). If either cannot be fetched, a warning replaces it and the other pane still works.

## 5. `_decision_bar(row, number)`

| Stage | Behaviour |
|---|---|
| `needs_attention` | Info: the invoice is still in the Review Queue (with the attempt text), plus a link to the Review Queue |
| Not a sign-off stage (processing, finalized) | Info: "No sign-off needed: ..." |
| `ready_signoff` or `recommended_reject` | Explains the recommendation (`ui.recommended_outcome`), a reviewer name, a comment box, and two buttons: "Confirm recommendation: ..." (primary) and "Override recommendation: <opposite>" |

Rules enforced on the page: reviewer name required; an override requires a non-empty comment. Then `data.api_decide(...)`; `AgentError` shows "Decision not saved: ...". Success flashes "<number> approved for export/payment (or rejected) by <reviewer>: recommendation confirmed (or overridden). Finalized." as success for a confirmation and warning for an override. The agent re-checks both rules (409 if the invoice is not eligible, 422 if an override has no reason).

## 6. `close_workspace`

Removes `reports_open` and increments `reports_nonce`, giving the table a new key so the previous selection is cleared.

---

## End-to-end example

1. Three invoices are Ready for Sign-off with AI recommendation Accept. The user filters "Ready for Sign-off", selects all three rows (header checkbox), types a reviewer name and clicks "Confirm recommendation: approve (3)".
2. `api_batch_decision` posts the three ids. The agent returns `decided` with three ids and no `skipped`. The page flashes "3 recommendation(s) confirmed: approved for export/payment by ... Finalized.", closes the workspace and reruns. The three rows now show Approved (Final).
3. Another invoice was recommended for rejection by the system after three attempts. The user clicks that single row; the workspace shows the red "Rejection recommended automatically" box, the original on the left and the report on the right.
4. The approver finds the invoice is actually fine, types a comment and clicks "Override recommendation: approve". `api_decide` posts `decision: "approve"`; the agent records it as an override and the page flashes a warning "... recommendation overridden. Finalized."
