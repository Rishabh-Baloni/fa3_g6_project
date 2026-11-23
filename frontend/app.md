# What app.py does

`frontend/app.py` is the entry point of the Streamlit dashboard (`streamlit run app.py`). It turns the page modules into one multi-page app: it sets the page configuration, builds the sidebar navigation with live queue counts, shows service problems, shows queued notifications, and runs the selected page. It contains no business logic.

Main parts:

1. `_service_status`: sidebar warnings about the database, agent and models
2. `_queue_sizes` and `_with_count`: counts shown next to page names
3. `main`: configuration, navigation and page execution

---

## Overall flow

```
 streamlit run app.py ─► main()
    ├─ st.set_page_config(layout="wide")
    ├─ ui.apply_style()
    ├─ _queue_sizes() ◄── data.db_invoices() (cached 5 s)
    ├─ ui.NAV.update({overview, intake, review, reports, ask})   st.Page(...) per page module
    ├─ st.navigation(pages)
    ├─ _service_status() ◄── data.db_check_database(), data.api_health()
    ├─ ui.show_flash()            toasts/banners queued by the previous action
    └─ navigation.run() ──► page_overview | page_intake | page_review | page_reports | page_ask
```

State: Streamlit re-runs the whole script on every interaction, so `main()` runs again each time. `ui.NAV` is filled here so pages can link to each other. The `if __name__ == "__main__":` guard lets tests import `app` without starting the app.

---

## 1. Service status

| Function | Input | Output |
|---|---|---|
| `_service_status()` | none | Writes to the sidebar. Database check fails: an error. Agent not reachable (`data.AgentError`): an error saying corrections, uploads and questions are disabled, then stops. Otherwise: a warning listing every dependency whose state is not `ok` (from `/health`), and an info box listing model roles reported as `"not configured"`. Shows nothing when everything is well |

## 2. Queue sizes

| Function | Input | Output |
|---|---|---|
| `_queue_sizes()` | none | `(needs_attention count, sign-off count)` from `data.db_invoices()["workflow_stage"]`. The sign-off count includes `ready_signoff` and `recommended_reject`. Any exception returns `(0, 0)`, because the pages report database problems themselves |
| `_with_count(title, count)` | Title and number | `"Title (n)"`, or just the title when the count is 0 |

## 3. `main`

| Step | What it does |
|---|---|
| Configuration | `st.set_page_config(page_title="Invoice Auditor", layout="wide", initial_sidebar_state="auto")` |
| Style | `ui.apply_style()` |
| Pages | Registers five `st.Page` objects in `ui.NAV`: `overview` (default, `page_overview.overview_page`), `intake` ("1 · Intake", `page_intake.intake_page`), `review` ("2 · Review Queue (n)", `page_review.review_page`), `reports` ("3 · Reports & Sign-Off (n)", `page_reports.reports_page`), `ask` ("4 · Ask AI", `page_ask.ask_page`). Each has an icon and a `url_path` |
| Navigation | `st.navigation(list(ui.NAV.values()))`, sidebar caption "Invoice Auditor", then `_service_status()` |
| Notifications | `ui.show_flash()` shows messages queued by the previous action, such as where an invoice went |
| Run | `navigation.run()` runs the selected page |

Why the counts in the titles: reviewers see at a glance how many invoices wait in the Review Queue and for sign-off without opening the pages.

---

## End-to-end example

Two invoices are in `needs_attention` and one is in `ready_signoff`:

1. `main()` runs; `_queue_sizes()` returns `(2, 1)`.
2. The sidebar lists "Overview", "1 · Intake", "2 · Review Queue (2)", "3 · Reports & Sign-Off (1)" and "4 · Ask AI".
3. If the agent container is down, `_service_status()` shows "The audit service is not reachable ... Corrections, uploads and questions are disabled until it is back." Reads from the database still work.
4. A reviewer fixes an invoice on the Review page. That page calls `ui.flash(...)` and `st.rerun()`. On the next run `main()` calls `ui.show_flash()` first, so the message appears before the page is drawn, and the sidebar counts now read "(1)" and "(2)".
