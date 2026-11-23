# What ui.py does

`frontend/ui.py` is the dashboard's shared presentation layer. It turns database rows (a DataFrame row or a dict) and rule codes into the words, colours, badges, formatted numbers and notifications that every page uses, and it holds the dashboard's CSS. It contains no database or HTTP access; which stage an invoice is in comes from `audit.invoice_overview.workflow_stage`, and this module only turns it into words.

Main parts:

1. Workflow vocabulary: stages, labels, banners, status helpers
2. Rule and field labels
3. Styles (`_CSS`, `WORKSPACE_CSS`, `apply_style`)
4. Formatting helpers (`money`, `number`, `date_text`, `esc`)
5. Correction helpers (`exception_key`, `changed_fields`)
6. HTML building blocks (`page_header`, badges, counts, empty state)
7. Notifications (`flash`, `show_flash`)

---

## Overall flow

```
 data.db_invoices() row / dict ──► status(), workflow_origin(), signoff_text(), can_batch_*()   ──► labels and colours
 backend rule_code / field name ──► rule_label(), field_label()                                      ──► readable text
 values (Decimal, dates, None)  ──► money(), number(), date_text(), esc()                           ──► safe strings

 page_*.py ──► page_header / counts / badges / empty_state ──► st.markdown(unsafe_allow_html=True)
 page_*.py ──► flash(kind, message)  ──► st.session_state["flash"]  ──st.rerun()──► app.main → show_flash → toast + banner
 app.main ──► ui.NAV (the five pages) ──► used by pages to link to each other
```

State: `NAV` is a module-level dict filled by `app.main()`. Notifications live in `st.session_state["flash"]`, so they survive `st.rerun()` (a toast sent just before a rerun is lost).

---

## 1. Workflow vocabulary

Stage constants: `PROCESSING`, `NEEDS_ATTENTION`, `READY_SIGNOFF`, `RECOMMENDED_REJECT`, `FINALIZED` (same strings as the backend `WorkflowStage`). `SYSTEM_REVIEWER = "system"` marks the rejection the system recommends after the last attempt. `MAX_REVIEW_ITERATIONS` is read from the environment (default 3) for display only; the agent enforces the limit.

| Name | Purpose |
|---|---|
| `DECISION_LABELS` | `approve` shows as "Accept", `review` as "Needs Attention", `reject` as "Reject" |
| `AI_COLOURS`, `SEVERITY_LABELS`, `TRANSLATION_LABELS` | Colours and text for recommendations, severities and translation status |
| `REVIEW_BANNER`, `REPORTS_BANNER`, `INTAKE_BANNER`, `ASK_BANNER` | The one-line purpose shown at the top of each page |
| `_CELL_STYLES` | Cell styles for `green`, `amber`, `red`, `blue`, `grey` (tables cannot hold HTML badges) |

| Function | Input | Output |
|---|---|---|
| `_value(row, name, default)` | DataFrame row (namedtuple) or dict | The field, with `None`, NaN, `NaT` and empty text turned into the default |
| `attempts_used(row)` | Row | `review_iteration` as an int |
| `attempt_text(row)` | Row | "Attempt 2 of 3" (the attempt about to be made), or "3 of 3 attempts used" |
| `status(row)` | Row | `(label, colour)`. Finalized: "Approved (Final)" green or "Rejected (Final)" red. Processing blue; Needs Attention amber; Ready for sign-off: "Validated - Pending Sign-off" green if attempts were used, else "Auto-Approved" green; Recommended Reject red |
| `status_text(row)` | Row | Label only |
| `colour_cells(table, column, colours, **more)` | DataFrame and a colour per row | A pandas Styler with the given columns coloured; the style grid is built empty so number columns accept it, and `st.dataframe` can still select rows |
| `workflow_origin(row)` | Row | How the invoice got here: "Being read and validated", "Flagged by rules · Attempt n of 3", "Recommended Reject (Auto, 3/3 Attempts)", "Recommended Reject (Reviewer Note)", "Validated via HITL (Attempt n/3)", "Auto-Approved by Rules" |
| `signoff_text(row)` | Row | "In Review Queue", "Awaiting sign-off", or "Approved/Rejected by <who>" with " (override)" if the decision overrode the recommendation |
| `recommended_outcome(row)` | Row | `approve` at Ready for Sign-off, `reject` at Recommended Reject, else `None` |
| `is_signoff_stage`, `can_batch_approve`, `can_batch_reject` | Row | Booleans: sign-off stage is either of the two; batch approve only at Ready for Sign-off; batch reject only at Recommended Reject. This mirrors the agent's rules for batch decisions |

## 2. Labels

| Function | Purpose |
|---|---|
| `rule_label(code)` | Readable text from `RULE_LABELS` (20 rule codes, for example `unit_price_mismatch` becomes "Unit price differs from the PO"); unknown codes are capitalised with underscores replaced |
| `field_label(name, line_number=None)` | Same behaviour as `field_label` in `workflow.py`: `line_items[1].item_code` becomes "Line 1: Item code" |

## 3. Styles

| Name | Purpose |
|---|---|
| `_CSS` | One `<style>` block: IBM Plex Sans (falls back to the system font offline), colour variables, banner, step trail, counts, badges, decision pills, attempt dots, violation and accepted boxes, overview cards (4 columns reflowing to 2 and 1), empty state, `table.facts`, severity colours, and highlight styles for correction inputs keyed `flag_...` (red for missing fields, amber for differences) |
| `WORKSPACE_CSS` | Widens the page and sizes the iframe for the side-by-side review and report pages only |
| `apply_style()` | Injects `_CSS` with blank lines removed (Markdown would end the HTML block at a blank line) |

## 4. Formatting

| Function | Input | Output |
|---|---|---|
| `esc(value)` | Any | HTML-escaped text; `None` becomes empty. Used for every value placed into HTML |
| `_blank(value)` | Any | True for `None`, empty, NaN, `NaT`, `<NA>` |
| `money(value, currency=None)` | Amount | `1,234.50 EUR` using `Decimal` (no float rounding); an em dash if blank; unparseable text returned as is |
| `number(value)` | Quantity or price | Normalised decimal text without trailing zeros; em dash if blank |
| `date_text(value)` | ISO date | `5 Apr 2025` style; "unknown" if blank; unparseable text returned as is |

## 5. Correction helpers

| Function | Purpose |
|---|---|
| `exception_key(kind, name, rule_code=None, line_number=None)` | Builds the key the agent understands when a reviewer accepts an exception: `missing:<field>` or `finding:<rule>:<line or 0>:<field>`. Same format as `missing_key` and `finding_key` in `workflow.py` |
| `changed_fields(entries, current)` | From the form's `{field: text}` and a function returning the current value, returns `[{"field_name", "corrected_value"}]` for fields that really changed. Blank input counts as unchanged; `"1,000"` vs `1000` counts as unchanged when both parse as `Decimal` |

## 6. Building blocks

| Function | Purpose |
|---|---|
| `STEPS` | `("Intake", "Review Queue", "Reports & Sign-Off", "Ask AI")` |
| `page_header(title, step, banner=None)` | Title, a trail with the current step highlighted, and the banner text (escaped) |
| `status_badge(row)`, `attempt_badge(row)`, `decision_badge(recommendation)` | HTML badges; the attempt badge shows one dot per allowed attempt, filled when used; "Not validated" if there is no recommendation |
| `counts(items)` | A row of big numbers from `(label, number, css class)` tuples |
| `lede(text)`, `empty_state(title, text)` | Intro text and a dashed "nothing here" box |

## 7. Notifications

| Function | Purpose |
|---|---|
| `flash(kind, message, icon=None)` | Queues `(kind, message, icon)` in session state; kind is `success`, `info`, `warning` or `error` |
| `show_flash()` | Pops the queue and shows each as a toast and as a banner (`st.success` etc.) |

---

## End-to-end example

A row for an invoice that went through one review attempt and is now ready for sign-off (`workflow_stage="ready_signoff"`, `review_iteration=1`, `recommendation="approve"`):

1. `status(row)` returns `("Validated - Pending Sign-off", "green")`, because attempts were used.
2. `workflow_origin(row)` returns "Validated via HITL (Attempt 1/3)", and `signoff_text(row)` returns "Awaiting sign-off".
3. `can_batch_approve(row)` is true, so the Reports page lets the invoice be ticked for batch approval; `can_batch_reject(row)` is false.
4. `money("1617.00", "EUR")` shows `1,617.00 EUR`; `date_text("2025-04-10")` shows `10 Apr 2025`.
5. After the reviewer approves, the page calls `flash("success", "...")` and `st.rerun()`; `app.main()` calls `show_flash()` on the next run, which shows the toast and banner.
