# What workflow.py does

`backend/workflow.py` takes an invoice file that appeared in the incoming folder and turns it into a validated audit record, a JSON and HTML report, and a vector-index entry. It also decides the recommendation (approve, review or reject) and runs the folder monitor that feeds the workflow. It is the orchestrator: it calls `extraction`, `validation`, `rag`, `llm` and `database`, and is called by `main.py`.

Main parts:

1. Offline language detection (`detect_language`)
2. `Translator`: translate to English with a Redis cache
3. Field labels (`field_label`)
4. `render_html`: the HTML audit report
5. Recommendation logic, accepted exceptions and `ReportWriter`
6. `AuditState` and `AuditWorkflow`: the LangGraph state machine
7. `InvoiceMonitor`: polls the incoming folder and recovers failed files

---

## Overall flow

```
 incoming/ ──poll──► InvoiceMonitor.scan_once ─► _handle
                        │ unsupported / duplicate (SHA-256) ─► audit.file_events, stop
                        ▼ new file: uuid invoice_id, file event "processing"
                  AuditWorkflow.ingest            thread_id = invoice-<id>
 START ─ mode=ingest ─► extract ─► translate ─► parse ─┐
 START ─ mode=revalidate ─► load_extracted ────────────┴─► apply_corrections ─► validate_data
                                                           ─► validate_business ─► report ─► index ─► END
   extract            extraction.extract_text            → audit.invoice_audit
   translate          detect_language + Translator       → Redis translation:*   → audit.invoice_audit
   parse              InvoiceParser                      → header fields, line items
   apply_corrections  human_feedback overlay (in memory)
   validate_data      validation.validate_invoice_data
   validate_business  validation.validate_against_erp ──HTTP──► mock-erp
   report             accepted exceptions → decide → review limit → summary → files + save_run
   index              VectorIndex.index_invoice  → Qdrant
 every node's result is checkpointed to PostgreSQL schema `langgraph` (PostgresCheckpointSaver)
```

State and concurrency:
- The graph state holds only JSON-safe values (`model_dump(mode="json")`). Large text stays in the audit tables.
- Because of the checkpoints, a run that fails part-way can be resumed from its last completed step.
- One `threading.Lock` per invoice id serialises `ingest`, `resume` and `revalidate` for the same invoice.
- Ingest uses thread id `invoice-<id>`; each re-validation uses `invoice-<id>-run-<n>`, so a new pass never resumes an old checkpoint.

---

## 1. Language detection

The requirements contain no language-detection package, so the detector counts common function words per language (`FUNCTION_WORDS`: `en`, `de`, `es`, `fr`, `it`, `pt`, `nl`).

| Function | Input | Output |
|---|---|---|
| `_scores(text)` | Text | Count of vocabulary hits per language (alphabetic words of 2+ letters only, because invoices are full of numbers and codes) |
| `_exclusive_scores(text)` | Text | Same, but a word counts only if exactly one language owns it ("total", "date", "de" are shared and say nothing) |
| `detect_language(text)` | Text | `DetectedLanguage(code, confidence)` |
| `_english_and_foreign_hits(text)` | Translated text | Count of English words and of words that are only foreign. Capitalised words inside a line and words next to a digit are not counted as foreign (names, places, dates such as "10 de abril de 2025") |
| `english_share(text)` | Translated text | 0 to 1 |

- Fewer than `MIN_HITS` (3) hits: code `"unknown"`, confidence 0.
- Confidence is the chosen language's share of the exclusive hits. With fewer than 3 exclusive hits it is the share of all hits.
- `english_share` returns 1.0 when fewer than 3 recognisable words exist or the foreign share is within `FOREIGN_TOLERANCE` (0.25). Otherwise it is `english / (english + foreign)`.
- `"unknown"` is translated to be safe when a translation model is configured.

## 2. `Translator`

`Translator(gateway, cache)`. `TranslationResult(status, text, confidence, note)` where status is `not_required`, `translated`, `unavailable` or `failed`; `as_dict()` is the stored form.

| Method | Purpose |
|---|---|
| `translate(text, language)` | English gives `not_required` (confidence 1.0). `unknown` with no translation model gives `not_required` with the note "Language could not be identified". Otherwise looks up Redis key `cache_key("translation", "v2", language.code, text)`; on a miss, translates chunk by chunk and stores the result for `TRANSLATION_CACHE_TTL_SECONDS` (30 days). `ModelNotConfiguredError` gives `unavailable`; `ModelCallError` gives `failed`. Failures are not cached |
| `_translate_chunk(chunk, source_name)` | Calls the `translator` / `translate` prompt (max 4000 tokens), expects JSON with `translation` and `confidence`. If the model ignores the JSON format, keeps its text with confidence 0.5. Returned confidence is `min(model confidence clamped to 0..1, english_share(translation))` |
| `_chunks(text)` | Splits on line boundaries into pieces of about `MAX_CHUNK_CHARS` (2500) |

Why the minimum of two signals: a translation that is still partly German must not score highly just because the model claimed it did well. The overall confidence is the minimum over all chunks. The cache key carries `v2` because the scoring changed and older cached confidences would be stale.

## 3. Field labels

| Name | Purpose |
|---|---|
| `FIELD_LABELS` | Readable names for header and line fields |
| `field_label(name, line_number=None)` | `line_items[1].item_code` becomes "Line 1: Item code"; `total_amount` becomes "Total amount"; unknown names are capitalised with underscores replaced |

Stored values and the JSON report keep the machine names; labels are for people.

## 4. HTML report

`render_html(report, reporting)` builds a self-contained page from the report dict using only the standard library.

| Helper | Purpose |
|---|---|
| `_e(value, empty="—")` | `html.escape` of any value; `None` or empty becomes a dash |
| `_table(headers, rows)`, `_list(items)` | Build table and list markup (numeric columns right-aligned) |
| `DECISIONS` | Display text for `approve`, `review`, `reject` |

Sections: verdict and summary; invoice facts; line items; discrepancies (reference column shows the ERP value or the computed value); missing fields; notes; human review; accepted exceptions; reviewer notes; auditor corrections; footer with invoice id, checksum prefix and summary source. The `rules.yml > reporting` switches `include_discrepancy_summary` and `include_translation_confidence` control two of them.

Security: every value is escaped, because report content comes from untrusted vendor documents.

## 5. Recommendation, accepted exceptions and reports

| Function | Input | Output |
|---|---|---|
| `_action_for(finding, rules)` | Finding | Policy action: arithmetic rules use `policies.total_mismatch`; `unsupported_currency` uses `policies.invalid_currency`; any other medium or high finding is `manual_review`; low is `approve` |
| `decide(findings, missing_fields, business_completed, rules)` | | `(Recommendation, ValidationStatus)`. Takes the worst action by rank (`approve` 0; `flag` and `manual_review` 1; `reject` 2). Missing fields add `policies.missing_field`; incomplete business validation adds `manual_review`. Status is `failed` if the recommendation is reject or any finding is high, `passed` if approve, else `partial` |
| `apply_review_limit(recommendation, status, *, review_iteration, max_iterations)` | | `(recommendation, status, outcome)`. Outcome is `None` for a non-review pass, `resolved` if approve, `recommended_reject` if the attempt number reached the limit, else `open`. The recommendation and status are never changed |
| `escalation_note(findings, missing_fields, attempts)` | | Text stored with the system's rejection recommendation (up to 3 messages, high severity first, plus missing fields) |
| `rule_based_summary(...)` | | Fallback summary text |
| `missing_key(field)`, `finding_key(rule, line, field)` | | Keys `missing:<field>` and `finding:<rule>:<line or 0>:<field>` |
| `current_exceptions(report)` | Report dict | Open exceptions by key, each with `acceptable` |
| `resolve_accepted(keys, report)` | Keys ticked by a reviewer | Exceptions, or a user-safe `ValueError` if a key is not open or not acceptable |

Rules to know:
- Only `medium` and `low` findings and missing fields can be accepted (`ACCEPTABLE_SEVERITIES`). High-severity findings must be fixed or the invoice recommended for rejection.
- Running out of review attempts is a workflow event, not a new verdict. The system writes a rejection recommendation and a person makes the final decision; it never rejects by itself.
- The limit is `Settings.max_review_iterations`, not a threshold from `rules.yml`.

`ReportWriter(gateway, reports_dir, reporting)`:

| Method | Purpose |
|---|---|
| `summarise(invoice, recommendation, status, findings, missing_fields, notes)` | Returns `(text, source)`. Uses the reasoning model (`auditor` / `report_summary`, 300 tokens) if configured; falls back to `rule_based_summary` with source `"rules"` if it is not configured or fails. The LLM only writes the wording; the recommendation is computed by rules |
| `build(...)` | The report dict (`report_version` 1): run, file, language, invoice, recommendation, summary, discrepancy counts, discrepancies, missing fields, notes, ERP snapshot, corrections, reviewer notes, accepted exceptions, review |
| `write_files(report)` | Writes `<reports_dir>/<invoice_id>/run-NNN.json` and `.html`. An `OSError` is logged and returns `(None, None)`, because the database copy is the system of record |

## 6. `AuditWorkflow`

`AuditState` is a `TypedDict` with: `mode`, `invoice_id`, `file_name`, `file_path`, `checksum`, `extraction`, `language`, `invoice` (as extracted), `effective_invoice` (extracted plus corrections), `corrections_applied`, `data_result`, `business_result`, `run_number`, `recommendation`, `validation_status`, `index_status`, `review_iteration`, `review_outcome`.

Public methods:

| Method | Purpose |
|---|---|
| `ingest(*, invoice_id, file_path, checksum)` | Runs the whole graph for a new file |
| `resume(invoice_id)` | `graph.invoke(None, ...)` continues from the last checkpoint |
| `revalidate(invoice_id)` | Reads `review_state`; the pending review iteration (if any) becomes this pass's `review_iteration`, so a failed pass can be repeated. Without a pending batch it is a plain re-run that cannot unlock a rejected invoice |
| `max_review_iterations` | Property from settings |
| `has_pending_steps(invoice_id)` | True if a checkpoint exists and has a next step |

Graph nodes:

| Node | What it does |
|---|---|
| `extract` | `extract_text`, then `insert_invoice` (idempotent per invoice id). Returns method, OCR confidence and page count |
| `translate` | Detects language from the stored raw text, translates, saves the translation |
| `parse` | `InvoiceParser.parse(raw text, language=...)` on the original text; saves fields and line items |
| `load_extracted` | Re-validation entry: loads the stored invoice. If the earlier translation was `unavailable` or `failed`, tries again and saves it if it now works |
| `apply_corrections` | Loads corrections and overlays them in memory; the extracted values stay unchanged |
| `validate_data` | `validate_invoice_data` with today's date, duplicate ids, OCR confidence and translation info |
| `validate_business` | `validate_against_erp` with an `LlmLineMatcher` |
| `report` | Removes accepted exceptions from the findings, then `decide`, `apply_review_limit`, summary, report build, file write and `save_run` |
| `index` | If embeddings are configured, `VectorIndex.index_invoice`. Failures are logged and returned as `index_status`; the audit record is already safe |

Accepted exceptions in `report`: an accepted medium or low finding, or a missing field, no longer blocks approval but stays in the report with the reviewer's reason and a note. A high finding is never removed this way.

Thread-id naming: `_config(thread_id)` and `_lock_for(invoice_id)` are the helper functions; `_language_summary` and `_float` shape state values.

## 7. `InvoiceMonitor`

`InvoiceMonitor(incoming_dir, repository, workflow, poll_seconds)`.

| Method | Purpose |
|---|---|
| `start()`, `stop()`, `alive` | Daemon thread named `invoice-monitor`; `stop` sets the event and joins for up to 30 s |
| `_run()` | Calls `recover_interrupted()` once, then loops `scan_once()` every `poll_seconds`; exceptions are logged and the loop continues |
| `scan_once()` | Under `_scan_lock`, goes through sorted files. Skips directories, names starting with `.` or `~$`, names ending `.part`, `.tmp` or `.crdownload`, files already seen with the same `(path, mtime_ns, size)`, and files modified less than `SETTLE_SECONDS` (2 s) ago. Returns how many were handled |
| `_handle(path, size)` | Computes `sha256_of(path)`. Skips if `file_event_seen`. Unsupported suffix gives `unsupported`. Same checksum as an existing invoice gives `duplicate` (a renamed copy is never audited twice). Otherwise creates a UUID, records `processing`, and runs `ingest` |
| `_process(event_id, run)` | Runs the callback; success gives `processed`, any exception gives `failed` with `TypeName: message` |
| `retry(event_id)` | For `failed` events only (else `ValueError`; unknown id gives `KeyError`). Resumes the checkpointed workflow if steps are pending, otherwise re-ingests the file if it still exists, otherwise marks the new event failed |
| `recover_interrupted()` | Marks every event left in `processing` as failed ("Interrupted by a restart; retrying.") and retries it |
| `paused()` | Context manager holding `_scan_lock`, so no scan runs during a reset |
| `clear_incoming()` | Deletes files in the incoming folder (not sub-folders or dot-files) and clears the seen set. Call it inside `paused()` |
| `sha256_of(path)` | Streams the file in 1 MiB blocks |

Why polling: filesystem events (inotify) are not delivered reliably through Docker bind mounts on macOS and Windows hosts. Why the settle time: a file may still be copying in.

---

## End-to-end example

`rechnung.pdf` (German, PO `PO-1001`, one wrong unit price) is dropped into `incoming/`:

1. `scan_once()` sees the file after it has been quiet for 2 s. `_handle()` hashes it; no event or invoice has that checksum, so it records a `processing` event and calls `ingest`.
2. `extract` reads the text layer with PyMuPDF and stores the raw text. `translate` detects `de` (for example confidence 0.93), translates through the model, checks that `english_share` is high, and caches it in Redis.
3. `parse` reads the invoice number, date, totals and line items from the German original.
4. `validate_data` finds nothing. `validate_business` calls the ERP and finds `unit_price_mismatch` (high).
5. `report`: `decide()` ranks the finding as `manual_review`, so the recommendation is `review` and the status is `failed` because a high finding exists. The model writes a summary (or the rule-based one is used), `run-001.json` and `run-001.html` are written, and `save_run` stores run 1.
6. `index` stores the invoice chunks in Qdrant. The file event becomes `processed`.
7. A reviewer corrects the price. `revalidate()` runs `load_extracted`, applies the correction, finds no discrepancies and gets `approve`. With review iteration 1, `apply_review_limit` returns outcome `resolved`, and run 2 is saved.
