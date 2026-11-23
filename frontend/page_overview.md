# What page_overview.py does

`frontend/page_overview.py` renders the dashboard's landing page ("Overview"). It turns static descriptive text plus live invoice counts into a hero card, four step cards and a row of status tiles with links to the page where each kind of work happens. It does not change any data.

Main parts:

1. `_cards`: builds the HTML for the four step cards
2. `_hero`: title, subtitle and two navigation buttons
3. `_status_strip`: live counts with page links
4. `overview_page`: the page function registered in `app.py`

---

## Overall flow

```
 app.py ──► overview_page()
              ├─ st.title("Overview")
              ├─ _hero()          buttons ──st.switch_page──► ui.NAV["intake"] / ui.NAV["ask"]
              ├─ _cards(PIPELINE) 4 cards (Step 1..4) as one HTML grid
              └─ _status_strip()  data.db_invoices()["workflow_stage"] ──► 4 counts + st.page_link to Intake / Review / Reports
```

State: none of its own. The counts come from `data.db_invoices()` (cached for 5 seconds). If the database cannot be read, the page still renders and shows a caption instead of the counts.

---

## 1. Content constants

| Name | Content |
|---|---|
| `HERO_TITLE`, `HERO_SUBTITLE` | "AI Invoice Auditor" and a one-sentence description (extraction, policy validation, human-in-the-loop exception handling, dual-path audit intelligence) |
| `PIPELINE` | Four `(label, title, text)` tuples: Step 1 Intake, Step 2 Review Queue (up to 3 review attempts), Step 3 Reports & Sign-Off, Step 4 Ask AI |

## 2. Functions

| Function | Input | Output |
|---|---|---|
| `_cards(items)` | List of `(label, title, text)` | HTML string of a `cardgrid` with one `card` per item; every text is escaped with `ui.esc` |
| `_hero()` | none | A bordered container (key `hero`) with the title, subtitle and two buttons: "Start Auditing → Upload Invoices" (primary) goes to the Intake page, "Explore Audit Assistant" goes to Ask AI |
| `_status_strip()` | none | Four bordered tiles, each with a count and a `st.page_link`: "Total processed" (all rows, links to Intake), "Pending in Review Queue" (`needs_attention`, amber, links to Review Queue), "Ready for sign-off" (`ready_signoff` plus `recommended_reject`, green, links to Reports), "Finalized" (`finalized`, links to Reports). Adds "Nothing has been processed yet. Start with Intake." when the total is 0 |
| `overview_page()` | none | Draws the title, `_hero()`, the cards, `_status_strip()` |

Notes:
- Page links and buttons use `ui.NAV`, which `app.main()` fills before running a page.
- Any exception from `data.db_invoices()` is swallowed and shown as "The live counts are unavailable right now (the audit database cannot be reached)." The Overview is a landing page and must open even when the database is down.
- Counts use the same `workflow_stage` values as the other pages, so the numbers match the sidebar counts.

---

## End-to-end example

The database holds 5 invoices: 2 `needs_attention`, 1 `ready_signoff`, 1 `recommended_reject`, 1 `finalized`.

1. The user opens the dashboard; the Overview is the default page. `overview_page()` draws the hero and the four step cards.
2. `_status_strip()` reads the stages and shows: Total processed 5, Pending in Review Queue 2, Ready for sign-off 2, Finalized 1.
3. The user clicks the link under "Pending in Review Queue" and `st.page_link` opens the Review Queue page.
4. On a fresh install with an empty database all four tiles show 0, plus the caption telling the user to start with Intake.
