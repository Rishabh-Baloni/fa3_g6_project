# What page_intake.py does

`frontend/page_intake.py` renders step 1 of the dashboard, "Intake". It turns uploaded files into calls to the agent and shows a self-refreshing table of every file the monitor detected, with its outcome and where it was routed (Review Queue or Reports). It also offers retrying failed files and a guarded "Start over" reset.

Main parts:

1. Routing helpers: `outcome_and_route`, `pipeline_counts`
2. Notifications: `announce_transitions`
3. The live fragment: `live_files`
4. The reset panel: `reset_panel`
5. The page: `intake_page`

---

## Overall flow

```
 intake_page()
   ├─ file_uploader ──[Upload and process]──► data.api_upload(name, bytes) ──► agent POST /invoices/upload
   │                                                                           (saved in incoming/, monitor picks it up)
   └─ live_files()   @st.fragment(run_every=3 s)
        ├─ data.db_intake()   (file_events joined to invoice_overview, cached 2 s)
        ├─ data.db_invoices() (cached 5 s)
        ├─ announce_transitions → toasts   ├─ pipeline_counts → four counts
        ├─ table: File | Outcome | Routed to | Details | Detected  (colour_cells)
        ├─ failed files → selectbox + [Retry processing] ──► data.api_retry_file(id) ──► POST /files/{id}/retry
        └─ [Start over] → reset_panel ──► data.api_reset() ──► POST /admin/reset
```

State: `st.session_state` keys `intake_seen` (last known state per file id), `intake_ready` (False on the first render) and `reset_open`. The fragment re-runs itself every 3 seconds without re-running the whole page, so the uploader is not reset.

---

## 1. Routing helpers

| Function | Input | Output |
|---|---|---|
| `outcome_and_route(row)` | A row from `db_intake()` | `(text, colour, routed_to)`. If the file is not `processed`: the text and colour from `FILE_STATUS` (`processing` Processing blue, `duplicate` "Duplicate (skipped)" grey, `failed` Failed red, `unsupported` "Unsupported type" grey; unknown statuses are shown as they are, grey) and `"—"`. If processed but `workflow_stage` is blank: "Validating…", blue, "Intake". Otherwise the label and colour from `ui.status(row)` and "Review Queue" for `needs_attention`, else "Reports & Sign-Off" |
| `pipeline_counts(events, frame)` | File events and invoices | Draws four counts: Processing (events with status `processing`), Needs attention, Ready for sign-off (`ready_signoff` plus `recommended_reject`), Finalized |

## 2. Notifications

| Function | Purpose |
|---|---|
| `announce_transitions(events)` | Compares each file's `(status, workflow_stage)` with the last one seen and shows a toast when it changed: failed ("failed to process. See Details below."), duplicate ("is a duplicate of an audited invoice and was skipped"), processed and needs attention ("moved to the Review Queue"), processed otherwise ("was auto-approved → moved to Reports (ready for sign-off)"). The first render only records the current state, so files from before are not announced again |

## 3. `live_files`

| Part | Behaviour |
|---|---|
| Load | `data.db_intake()` and `data.db_invoices()`; an exception shows "Could not load the file log: ..." and stops |
| Controls | "Refresh now" (clears caches, reruns the fragment) and "Start over" (opens the reset panel) |
| Empty | `ui.empty_state("No files yet", ...)` when there are no events |
| Table | `st.dataframe` of file, outcome, routed to, error message and detection time, with the outcome column coloured by `ui.colour_cells` |
| Links | "Go to Review Queue" and "Go to Reports" |
| Retry | Only when failed files exist: a select box of failed files, then "Retry processing" calls `data.api_retry_file`, clears caches and toasts the result; `AgentError` becomes "Retry failed: ..." |

## 4. `reset_panel`

- Shows what will be deleted (files, invoices and reports, review history, recommendations and decisions, the search index and cached answers) and that ERP master data stays.
- The "Reset everything" button stays disabled until the typed text equals `RESET_PHRASE` ("RESET") after stripping. The agent checks the phrase again.
- On success: `data.db_clear_cache()`, removes the session keys in `_STATE_AFTER_RESET` (`intake_seen`, `intake_ready`, `review_open`, `reports_open`, `pending_fix`, `review_nonce`, `reports_nonce`, `conversation`, `reset_open`), queues a success message with `ui.flash` and calls `st.rerun()`. On `AgentError` it shows "Reset failed: ...".

Why clear those keys: other pages keep selections and conversation state in the session, which would point at invoices that no longer exist.

## 5. `intake_page`

Shows the header (step 1 with `INTAKE_BANNER`), then a multi-file uploader for PDF, DOCX, PNG, JPG, JPEG, TIF and TIFF. On "Upload and process" each file goes to `data.api_upload`: success toasts the saved name (the agent may rename it, for example with `_2`), `AgentError` shows "<name> was not added: ...". Caches are cleared afterwards. A caption explains that files are identified by content, so a renamed copy of an audited invoice is recorded as a duplicate. Then it calls `live_files()`.

Why a polling fragment: the monitor works in the background and writes `file_events` rows; the page polls the database every few seconds instead of receiving push events.

---

## End-to-end example

1. The user selects `acme_0412.pdf` and clicks "Upload and process". `api_upload` posts it; the agent saves it and replies 201 with `file_name`. A toast says it is being processed.
2. Within 3 seconds `live_files` reruns. `db_intake()` shows the file with status `processing`: the table row reads "Processing", routed to "—"; the Processing count is 1.
3. The monitor finishes. The row now has status `processed` and `workflow_stage = needs_attention`. The outcome reads "Needs Attention" (amber), routed to "Review Queue", and a toast says "<invoice number> needs attention → moved to the Review Queue."
4. If extraction had crashed, the status would be `failed`, with the error in Details. The user picks the file under "Retry a failed file" and clicks "Retry processing".
