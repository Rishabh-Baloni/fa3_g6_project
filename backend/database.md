# What database.py does

`backend/database.py` is the data layer. It defines the shared domain models (`ParsedInvoice`, `LineItem`, `Finding` and the enums), the PostgreSQL schema for the `audit` tables and the `invoice_overview` view, every read and write the application makes (`AuditRepository`), and a PostgreSQL-backed LangGraph checkpoint saver (`PostgresCheckpointSaver`). The agent is the only writer of the `audit` schema.

Main parts:

1. Domain models and enums
2. Exceptions for refused actions
3. `Database`: pooled connections
4. `AUDIT_SCHEMA_SQL`: tables, triggers and the `invoice_overview` view
5. `AuditRepository`: all audit reads and writes
6. `PostgresCheckpointSaver`: LangGraph state in the `langgraph` schema

---

## Overall flow

```
 workflow / main / rag / query
        │  AuditRepository methods (plain parameterised SQL)
        ▼
   Database.connection()  ── SQLAlchemy pool (pre-ping) ──► psycopg connection (dict rows)
        │                                                        commit on success / rollback on error
        ▼
   PostgreSQL
     schema audit     file_events, invoice_audit, invoice_line_items, validation_runs, discrepancies,
                      human_feedback*, human_decisions*, reject_recommendations*, accepted_exceptions*
                      view invoice_overview  (computes workflow_stage)
     schema langgraph checkpoints, checkpoint_blobs, checkpoint_writes   ◄── PostgresCheckpointSaver
                      (* = append-only, protected by trigger audit.reject_row_change)
```

Important behaviours: extracted values are never overwritten (corrections are new rows); several tables reject `UPDATE` and `DELETE` through a trigger; rows that must not race are locked with `SELECT ... FOR UPDATE` inside a transaction.

---

## 1. Domain models and enums

Money and quantities are `Decimal` end to end. `model_dump(mode="json")` turns them into strings for the LangGraph state, so no value passes through a float.

| Class | Fields | Purpose |
|---|---|---|
| `LineItem` | `line_number`, `item_code`, `description`, `quantity`, `unit_price`, `line_total` | One invoice line |
| `ParsedInvoice` | `invoice_number`, `invoice_date`, `vendor_name`, `po_number`, `currency`, `subtotal_amount`, `tax_rate_pct`, `tax_amount`, `total_amount`, `line_items`, `extras` | The extracted invoice. `extras` holds parsing notes and leftovers (stored as JSONB). Class constants `HEADER_FIELDS` and `LINE_FIELDS` list the editable fields |
| `Finding` | `check_source`, `rule_code`, `field_name`, `severity`, `message`, `line_number`, `invoice_value`, `erp_value`, `expected_value`, `deviation_pct` | One discrepancy; becomes one row in `audit.discrepancies` |
| `Severity` | `low`, `medium`, `high` | |
| `CheckSource` | `data`, `business` | Whether a finding came from invoice-only checks or the ERP checks |
| `Recommendation` | `approve`, `review`, `reject` | The rules' verdict |
| `ValidationStatus` | `passed`, `partial`, `failed` | |
| `WorkflowStage` | `processing`, `needs_attention`, `ready_signoff`, `recommended_reject`, `finalized` | Lifecycle stage. It is computed once, in the SQL view, and used by both the dashboard and the sign-off rules so they always agree |

## 2. Exceptions

| Exception | Raised when |
|---|---|
| `DecisionNotAllowedError` | A final decision cannot be recorded (already decided, still in review, not validated, not eligible) |
| `OverrideReasonRequired` (subclass) | The decision goes against the recommendation and no reason was given |
| `ReviewNotAllowedError` | Another review batch is not allowed (limit reached, approved, escalated, decided, or the previous batch is not re-validated) |

## 3. `Database`

| Method | Purpose |
|---|---|
| `Database(url, *, pool_size=5, autocommit=False)` | Builds a SQLAlchemy engine with `max_overflow=pool_size`, `pool_pre_ping=True` and a 5 s connect timeout. `autocommit=True` sets isolation `AUTOCOMMIT` |
| `connection()` | Context manager that yields the raw psycopg connection with `dict_row` rows. Commits on success (unless autocommit), rolls back on any `BaseException`, and always returns the connection to the pool |
| `ping()` | Runs `SELECT 1` |
| `close()` | Disposes the engine |

Why SQLAlchemy only for the pool: callers write plain SQL against psycopg, so psycopg types such as `Jsonb` work directly. `_jsonb(value)` wraps a value as `Jsonb` after a JSON round-trip with `default=str`, so dates and decimals serialise.

## 4. The audit schema (`AUDIT_SCHEMA_SQL`)

Applied idempotently at start-up by `AuditRepository.apply_schema()`. The `audit` schema itself is created by `backend/db/01_roles_and_schemas.sh`.

| Table or view | Contents | Notes |
|---|---|---|
| `file_events` | Every file the monitor saw | `status` is one of `processing`, `processed`, `duplicate`, `failed`, `unsupported` |
| `invoice_audit` | One row per invoice: file reference and checksum, raw text, language, translation, extracted header values, latest validation status, recommendation, review counters, `thread_id`, `indexed_at` | `file_checksum` is `UNIQUE`; the file itself is never stored, only its path and checksum |
| `invoice_line_items` | Extracted line items | `UNIQUE (invoice_id, line_number)` |
| `validation_runs` | One row per run (`initial` or `revalidation`) with report JSONB, summary, effective values, report paths and the `review_iteration` | `UNIQUE (invoice_id, run_number)` |
| `discrepancies` | Findings per run | Linked to `validation_runs.id` |
| `human_feedback` | Corrections and notes | Append-only |
| `human_decisions` | Final approve or reject | `invoice_id` is `UNIQUE`, so exactly one per invoice; append-only |
| `reject_recommendations` | Reviewer or system rejection recommendation | One per invoice; non-empty note required; append-only |
| `accepted_exceptions` | Exceptions a reviewer accepted as is | `UNIQUE (invoice_id, exception_key)`, reason required; append-only |
| `invoice_overview` (view) | Extracted values overlaid with the latest run's effective values, counts, decision data and `workflow_stage` | Used by the dashboard and structured questions |

Design points worth knowing:
- **Append-only:** the function `audit.reject_row_change()` raises on `UPDATE` or `DELETE`, and triggers attach it to `human_feedback`, `human_decisions`, `reject_recommendations` and `accepted_exceptions`.
- **`workflow_stage` rule** (the view's `CASE`): no run yet gives `processing`; a decision row gives `finalized`; a rejection recommendation gives `recommended_reject`; recommendation `approve` gives `ready_signoff`; anything else gives `needs_attention`.
- **Migration blocks** in the SQL (`ADD COLUMN IF NOT EXISTS`, `DROP COLUMN IF EXISTS`, a `DO $$` block) upgrade databases created by earlier versions. Old `review_locked` invoices become system rejection recommendations.
- "Was this batch re-validated" is derived, not stored: a batch counts as re-validated when a validation run carries its `review_iteration`. That keeps `human_feedback` immutable.

Module constants:

| Constant | Purpose |
|---|---|
| `SYSTEM_REVIEWER = "system"` | Recorded as `recommended_by` when the system recommends rejection after the last attempt |
| `_STAGE_MESSAGES` | Human-readable refusal text per `WorkflowStage` |
| `PENDING_ITERATIONS` | SQL that finds correction or accepted-exception batches not yet followed by a run with the same `review_iteration` |

## 5. `AuditRepository`

Constructed with a `Database`. Methods by area:

### Lifecycle and file events

| Method | Input | Output |
|---|---|---|
| `apply_schema()` | none | Runs `AUDIT_SCHEMA_SQL` |
| `ping()` | none | `SELECT 1` |
| `invoice_id_for_checksum(checksum)` | SHA-256 | Existing `invoice_id` as string, or `None` |
| `file_event_seen(file_path, checksum)` | Path and checksum | True if that exact pair was already recorded |
| `record_file_event(...)` | name, path, checksum, size, status, optional invoice id and error | New event id. `finished_at` is set unless status is `processing` |
| `finish_file_event(event_id, *, status, invoice_id, error)` | | Updates status and sets `finished_at`; keeps an existing `invoice_id` via `COALESCE` |
| `file_event(event_id)` | | Row dict or `None` |

### Invoices

| Method | Purpose |
|---|---|
| `insert_invoice(...)` | Inserts the invoice with raw text. `ON CONFLICT (invoice_id) DO NOTHING`, so a resumed workflow can repeat the extract step. Sets `thread_id = "invoice-<id>"` and stores `page_count` in `extracted_raw` |
| `save_translation(invoice_id, *, source_language, language_confidence, translation)` | Writes language and translation columns |
| `save_parsed_fields(invoice_id, invoice)` | In one transaction: updates header values and merges `extras` into `extracted_raw`. Line items are inserted only if none exist yet, so the original extraction is stored once |
| `invoice_record(invoice_id)` | Full `invoice_audit` row |
| `extracted_invoice(invoice_id)` | Rebuilds the originally extracted `ParsedInvoice` (before corrections). Raises `KeyError` if missing |
| `duplicate_invoice_ids(invoice_number, vendor_name, exclude_id)` | IDs of earlier invoices with the same number (case-insensitive) and vendor; empty if there is no number |

### Review and feedback

| Method | Purpose |
|---|---|
| `corrections(invoice_id)` | All correction rows, oldest first |
| `reviewer_notes(invoice_id)` | Distinct notes per attempt and author from `human_feedback` comments and accepted-exception reasons |
| `accepted_exceptions(invoice_id)` | Accepted exceptions, in insertion order |
| `record_review_batch(invoice_id, *, corrected_by, comment, items, max_iterations, accepted)` | Atomically records one review attempt. Locks the invoice row with `FOR UPDATE`. Refuses (`ReviewNotAllowedError`) if the limit is used, the invoice is approved, a decision or rejection recommendation exists, or the previous batch is not re-validated. Increments `review_iteration`, inserts one `human_feedback` row per item and the accepted exceptions (with the note as the reason). Returns `(iteration, feedback ids)` |
| `review_state(invoice_id)` | `review_iteration`, `review_outcome`, `recommendation` plus `pending_iteration` (highest batch awaiting re-validation) |
| `record_reject_recommendation(invoice_id, *, recommended_by, note)` | Allowed only in stage `needs_attention` (else `DecisionNotAllowedError`) |

### Final decisions

| Method | Purpose |
|---|---|
| `_stage(conn, invoice_id)` | Reads `workflow_stage` from the view |
| `_decide(conn, invoice_id, *, decision, decided_by, note, source, allowed_stages)` | Locks the row, checks the stage, works out `recommended` (`approve` for `ready_signoff`, otherwise `reject`), sets `override = decision != recommended`, requires a note for an override (`OverrideReasonRequired`), inserts into `human_decisions` |
| `record_decision(...)` | Single decision (`source="single"`) in one transaction |
| `record_batch_decision(invoice_ids, *, decision, decided_by, note)` | Batch decision. Approve is allowed only from `ready_signoff`; reject only from `recommended_reject`. Invoices are processed in sorted order so concurrent batches cannot deadlock. Returns `{"decision", "decided", "skipped": [{invoice_id, reason}]}` |

### Validation runs and reports

| Method | Purpose |
|---|---|
| `next_run_number(invoice_id)` | `max(run_number) + 1` |
| `save_run(...)` | In one transaction: inserts the run, records `review_outcome` for `resolved` or `recommended_reject`, inserts a system rejection recommendation if the last attempt failed (`ON CONFLICT DO NOTHING`), inserts all findings, and updates `validation_status`, `recommendation` and `latest_run_number` on `invoice_audit`. Returns the run id |
| `latest_report(invoice_id)` | Newest run's `report` and file paths |

### Indexing, queries and administration

| Method | Purpose |
|---|---|
| `mark_indexed(invoice_id)` | Sets `indexed_at` |
| `invoices_to_index(*, only_unindexed)` | IDs of validated invoices, oldest first |
| `indexing_source(invoice_id)` | Everything the RAG index is built from, all from PostgreSQL: invoice text, latest run, and `human_outcome()` |
| `human_outcome(invoice_id)` | Workflow stage, rejection recommendation, final decision, decision note, override flag |
| `query(sql, params)` | Generic parameterised read |
| `run_readonly_select(sql, *, max_rows, timeout_ms=5000)` | Runs a validated SELECT in a `READ ONLY` transaction with `SET LOCAL statement_timeout`. Returns `(columns, rows, truncated)` by fetching `max_rows + 1` rows. Defence in depth behind the SQL guard in `query.py` |
| `reset_all()` | `TRUNCATE ... RESTART IDENTITY` on all audit and checkpoint tables. `TRUNCATE` does not fire the append-only triggers, so this is the one deliberate way to start over. Does not touch the ERP schema. Returns counts of invoices and files deleted |
| `counts()` | Number of invoices and number indexed |

## 6. `PostgresCheckpointSaver`

Why it exists: the project's requirements include LangGraph but not its PostgreSQL saver package, so this class implements the `BaseCheckpointSaver` interface directly. It follows the official storage layout and uses LangGraph's own serializer (`self.serde`).

Tables (`CHECKPOINT_SCHEMA_SQL`, created without schema prefix, so the connection's `search_path` must be `langgraph`; see `Settings.checkpoint_db_url`):

| Table | Holds |
|---|---|
| `checkpoints` | One row per checkpoint, minus channel values |
| `checkpoint_blobs` | One row per channel value version |
| `checkpoint_writes` | Pending writes of each task |

| Method | Purpose |
|---|---|
| `setup()` | Creates the three tables |
| `put(config, checkpoint, metadata, new_versions)` | Stores new channel blobs (`ON CONFLICT DO NOTHING`) and upserts the checkpoint row. Returns the config with the new `checkpoint_id` |
| `put_writes(config, writes, task_id, task_path)` | Regular writes keep the first value (`DO NOTHING`); special writes (errors, interrupts, index below 0) are replaced |
| `delete_thread(thread_id)` | Deletes a thread from all three tables |
| `get_tuple(config)` | Latest matching checkpoint, or `None` |
| `list(config, *, filter, before, limit)` | Iterates checkpoints newest first, with an optional metadata filter |
| `_select(...)` | Builds the query; results are built eagerly so the pooled connection is returned before the caller iterates |
| `_tuple(conn, row)` | Deserialises one row into a `CheckpointTuple`, loading blobs and pending writes |
| `get_next_version(current, channel)` | Returns `"{n+1:032}.{random:016}"` |

---

## End-to-end example

A German invoice moves through the repository:

1. The monitor calls `file_event_seen()` (no), `invoice_id_for_checksum()` (no), then `record_file_event(status="processing")`.
2. `insert_invoice()` stores the raw text; `save_translation()` and `save_parsed_fields()` store language and fields plus line items.
3. LangGraph writes a checkpoint after each node through `PostgresCheckpointSaver.put()`.
4. `save_run()` writes run 1 (recommendation `review`), its findings, and updates `invoice_audit`. The view now reports `needs_attention`.
5. A reviewer submits a fix. `record_review_batch()` locks the row, sets `review_iteration = 1` and inserts the `human_feedback` rows. `review_state()` reports `pending_iteration = 1`.
6. Re-validation runs; `save_run(review_iteration=1, review_outcome="resolved")` stores run 2 with recommendation `approve`. The view now reports `ready_signoff`.
7. `record_decision(decision="approve")` confirms the recommendation (no override) and inserts the single `human_decisions` row. The stage becomes `finalized`; further decisions are refused with `DecisionNotAllowedError`.
