# What workflow.py does

This module is the audit pipeline. It turns an invoice file dropped into incoming into a validated, explained audit report that is saved to the database, written to disk and indexed for search.

It has 6 parts:

1. Language detection (offline)
2. Translation agent (LLM)
3. Field labels (readable names)
4. HTML report rendering
5. Reporting agent (decision, summary, report)
6. The LangGraph workflow and the folder monitor

---

## Overall flow

```
incoming/ folder
   │  (polled every N seconds)
   ▼
InvoiceMonitor.scan_once ─► _handle ─► AuditWorkflow.ingest
                                           │
   START ─(mode=ingest)─► extract ─► translate ─► parse ─┐
   START ─(mode=revalidate)─► load_extracted ────────────┤
                                                          ▼
              apply_corrections ─► validate_data ─► validate_business ─► report ─► index ─► END
```

The graph saves a checkpoint to PostgreSQL after every node. If the process crashes, the run can continue from the last completed step.

---

## 1. Language detection

| Function | Input | Output |
|---|---|---|
| `_scores(text)` | Raw text | `Counter` of how many known function words (`"the"`, `"und"`, `"factura"`…) belong to each language |
| `_exclusive_scores(text)` | Raw text | Same count, but only for words that belong to exactly one language (`"de"` and `"total"` are ignored because several languages use them) |
| `detect_language(text)` | Raw text | `DetectedLanguage(code, confidence)` |

How `detect_language` works:
- If fewer than 3 words match (`MIN_HITS`), it returns `code="unknown"`.
- Otherwise the language with the most hits wins.
- Confidence is that language's share of the exclusive hits. If there are too few exclusive hits, it uses the share of all hits instead.

Example: `"Rechnung ... Gesamtbetrag ... und"` gives `DetectedLanguage("de", 0.9)`.

**Translation quality check:**
- `_english_and_foreign_hits(text)` returns `(english_count, foreign_count)`. It does not count capitalised names or words next to numbers as foreign.
- `english_share(text)` returns a score from 0 to 1 for how English the text is. It returns 1.0 if 25% or less of the recognisable words are still foreign.

---

## 2. Translation agent: `Translator`

`translate(text, language: DetectedLanguage)` returns a `TranslationResult(status, text, confidence, note)`.

Steps:
1. If the language is `en`, it returns `status="not_required"`.
2. If the language is `unknown` and no translation model is configured, it returns `not_required` with a note.
3. It checks the Redis cache. The key is a hash of `"translation"`, `"v2"`, the language and the text. On a hit, it returns the cached result.
4. It splits the text into chunks of up to 2500 characters (`_chunks`) and translates each one with `_translate_chunk`:
   - Sends the `translator/translate` prompt to the LLM and expects JSON `{"translation": ..., "confidence": ...}`.
   - If the reply is not valid JSON, it keeps the raw text and sets confidence to 0.5.
   - Chunk confidence is `min(model confidence, english_share(translation))`, so the model cannot claim a good translation that is still half German.
5. Final confidence is the lowest chunk confidence. The result is cached for 30 days.
6. Error statuses: `unavailable` when no model is configured, `failed` when the model call fails.

---

## 3. Field labels

`field_label("line_items[1].item_code")` returns `"Line 1: Item code"`, and `field_label("total_amount")` returns `"Total amount"`. These labels are used only for text people read. The JSON report keeps the machine names.

---

## 4. HTML report: `render_html(report, reporting)`

- **Input:** the report dict built by `ReportWriter.build`, plus the `ReportingRules` switches from `rules.yml`.
- **Output:** one self-contained HTML string.

Sections, in order:
1. Header and verdict
2. Invoice facts
3. Line items
4. Discrepancies (optional)
5. Missing fields
6. Notes
7. Human review
8. Accepted exceptions
9. Reviewer notes
10. Corrections
11. Footer

Every invoice value goes through `_e()`, which HTML-escapes it to prevent XSS from untrusted vendor documents.

---

## 5. Reporting logic

### Turning findings into a decision
- `_action_for(finding, rules)` maps one finding to an action:
  - Arithmetic rules use `policies.total_mismatch`.
  - Currency rules use `policies.invalid_currency`.
  - Any other medium or high finding gives `manual_review`.
  - Low findings give `approve`.
- `decide(findings, missing_fields, business_completed, rules)` returns `(Recommendation, ValidationStatus)`:
  - The worst action wins. Action ranks: approve = 0, flag or manual_review = 1, reject = 2.
  - Missing fields add the `missing_field` policy's action. An ERP check that could not complete adds `manual_review`.
  - Status is `FAILED` if the recommendation is reject or any finding is high severity. It is `PASSED` if the recommendation is approve, and `PARTIAL` otherwise.

### Human review limit: `apply_review_limit(...)`
It returns the same recommendation and status plus an outcome:

| Outcome | When |
|---|---|
| `None` | This is not a review pass |
| `"resolved"` | The recommendation is approve |
| `"open"` | Attempts remain |
| `"recommended_reject"` | The last allowed attempt (`max_iterations`) still has issues |

The system never rejects by itself. It only records a recommendation, and a person makes the final decision.

### Accepted exceptions
A reviewer can accept specific open issues so they stop blocking approval:
- `missing_key` and `finding_key` build stable IDs such as `"missing:po_number"` or `"finding:tax_mismatch:0:tax_amount"`.
- `current_exceptions(report)` returns every open exception, each marked with `acceptable` (true only for missing fields and medium/low findings).
- `resolve_accepted(keys, report)` checks the keys a reviewer ticked. It raises `ValueError` if a key is unknown or belongs to a high-severity finding.

### Summary text
- `rule_based_summary(...)` produces fixed text such as `"Manual review required. <top 3 reasons>"`.
- `escalation_note(...)` produces the reason recorded when the system recommends rejection.

### `ReportWriter`
| Method | Input | Output |
|---|---|---|
| `summarise(...)` | Invoice, recommendation, findings, missing fields, notes | `(summary_text, "llm" or "rules")`. Uses the LLM `auditor/report_summary` prompt, and falls back to the rule-based summary if the model is missing or fails |
| `build(...)` | All results of the run | The full report dict: file, language, invoice, recommendation, discrepancies, ERP snapshot, corrections, review, etc. |
| `write_files(report)` | Report dict | `(json_path, html_path)` written to `reports/<invoice_id>/run-NNN.json` and `.html`. Returns `(None, None)` if the write fails; the database copy is the system of record |

---

## 6. The workflow: `AuditWorkflow`

### State (`AuditState`)
A dict that is passed between nodes. Each node returns only the keys it adds, and LangGraph merges them into the state.

| Key | Set by |
|---|---|
| `mode`, `invoice_id`, `file_*`, `checksum` | Entry method |
| `extraction` | `extract` or `load_extracted` |
| `language` | `translate` or `load_extracted` |
| `invoice` (as extracted) | `parse` or `load_extracted` |
| `effective_invoice`, `corrections_applied` | `apply_corrections` |
| `data_result` | `validate_data` |
| `business_result` | `validate_business` |
| `run_number`, `recommendation`, `validation_status`, `review_outcome` | `report` |
| `index_status` | `index` |

### Entry points
| Method | Input | What it does |
|---|---|---|
| `ingest(invoice_id, file_path, checksum)` | A new file | Runs the full pipeline on thread `invoice-<id>` |
| `resume(invoice_id)` | — | Calls `graph.invoke(None, …)`, which continues from the last checkpoint |
| `revalidate(invoice_id)` | — | Runs a fresh pass in `revalidate` mode on thread `invoice-<id>-run-<n>`. If a batch of corrections is pending, this pass counts as that batch's review iteration |
| `has_pending_steps(invoice_id)` | — | True if an ingest run stopped part-way |

Each invoice has its own `threading.Lock`, so two runs for the same invoice never overlap.

### Nodes in order

1. **`_extract`**: Reads the file with `extract_text` (PDF text or OCR), inserts the invoice row with the raw text into the database, and returns `extraction` (method, OCR confidence, page count).
2. **`_translate`**: Loads the raw text, runs `detect_language` and then `Translator.translate`, saves the result, and returns the `language` summary.
3. **`_parse`**: Runs `InvoiceParser.parse` on the raw text to get structured fields (number, date, PO, totals, line items), saves them, and returns `invoice`.
4. **`_load_extracted`** (revalidate only): Reloads the stored record and parsed fields instead of re-extracting. If the earlier translation was `unavailable` or `failed`, it retries the translation.
5. **`_apply_corrections`**: Overlays the auditor's manual corrections from the database on the extracted values. Returns `effective_invoice` and the list of applied corrections.
6. **`_validate_data`**: Runs `validate_invoice_data`, which covers required fields, arithmetic, dates, currency, duplicates, OCR confidence and translation confidence. Returns findings, missing fields and notes.
7. **`_validate_business`**: Runs `validate_against_erp`, which compares the invoice with the ERP purchase order and uses an LLM to match line items. Returns `completed`, findings, notes and an ERP snapshot.
8. **`_report`**, the core step:
   - Combines all findings.
   - Removes the exceptions a reviewer accepted (high severity can never be removed) and adds notes about them.
   - Runs `decide` and then `apply_review_limit`.
   - Generates the summary.
   - Builds the report, writes the JSON and HTML files, and saves the run to the database with `save_run`, including an escalation note if rejection is recommended.
   - Returns the run number, recommendation, status and review outcome.
9. **`_index_invoice`**: Embeds the invoice into the vector index (RAG) for the "Ask" page. If this step fails, it records the error and does not fail the audit.

---

## 7. `InvoiceMonitor`: watching the folder

| Method | Purpose |
|---|---|
| `start()` / `stop()` | Runs `_run` in a daemon thread |
| `_run()` | Calls `recover_interrupted()` once, then `scan_once()` every `poll_seconds` |
| `scan_once()` | For each file in incoming, skips hidden or temporary files, files already seen (same path, modification time and size) and files modified less than 2 seconds ago (still being copied). Calls `_handle` on the rest and returns the count |
| `_handle(path, size)` | Computes the SHA-256 checksum. Then: already recorded → skip; unsupported type → record `unsupported`; same content already audited → record `duplicate`; otherwise creates a new UUID, records `processing` and calls `workflow.ingest` |
| `_process(event_id, run)` | Runs the workflow and marks the file event `processed` or `failed` with the error |
| `retry(event_id)` | For a failed event only: resumes from the checkpoint if one exists, re-ingests if the file still exists, otherwise fails with "file gone" |
| `recover_interrupted()` | On startup, marks events left in `processing` by a crash as failed and retries them |
| `paused()` / `clear_incoming()` | Used for resets: blocks scanning, deletes the incoming files and forgets which files were seen |

Polling is used instead of filesystem events because filesystem events are unreliable through Docker bind mounts on macOS and Windows hosts.

---

## End-to-end example

1. `rechnung.pdf` is dropped into incoming.
2. The monitor finds it, computes the checksum (new content), records `processing` and calls `ingest`.
3. `extract` produces the text, and `translate` detects German and translates it to English with confidence 0.86.
4. `parse` gives the fields, and `apply_corrections` finds none yet.
5. `validate_data` finds a tax mismatch, and `validate_business` finds that the PO matches the ERP.
6. `report` applies the `total_mismatch` policy, which gives "review" with status `FAILED` or `PARTIAL`, writes `run-001.json` and `.html`, and saves the run.
7. `index` embeds the invoice, and the file event is marked `processed`.
8. Later, an auditor corrects the tax and calls `revalidate`. The graph starts at `load_extracted`, applies the correction, validates again and writes `run-002`. The outcome is `resolved`, `open`, or `recommended_reject` once the attempts run out.
