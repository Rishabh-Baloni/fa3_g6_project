# What test_backend.py does

`tests/test_backend.py` is the backend unit-test suite. It turns hand-written invoice text, in-memory objects and scripted fake models into assertions about extraction, validation, decision policy, corrections, review limits, the RAG pipeline, the SQL guard, the query workflow and the admin reset. No service is needed (no PostgreSQL, Redis, Qdrant, ERP or language model). The one exception is the checkpointer test, which runs only when `CHECKPOINT_TEST_DB_PASSWORD` is set.

Run with `pytest tests/test_backend.py`.

Main parts:

1. Parsing and extraction tests
2. Validation tests (data checks, ERP checks, policy, reports)
3. Corrections, previews, review limit and file serving
4. RAG pipeline tests (routing, retries, reflection)
5. Checkpointer test (needs PostgreSQL)
6. Query workflow tests (SQL guard, router, graph)
7. Reset, accepted exceptions and human-decision text

Set-up (path, environment defaults, imports, the session-scoped `rules` fixture) is described in the flow below.

---

## Overall flow

```
 pytest ─► sys.path += backend/ ; os.environ.setdefault(AGENT_DB_*, CONFIG_DIR, PROMPTS_DIR)
        ─► imports backend modules directly (database, extraction, llm, rag, query, validation, main, workflow)
        ─► `rules` fixture (session) = settings.get_rules() from config/rules.yml
        ─► tests call pure functions or small classes with fakes:
              FakeErp, FakeRepository, ScriptedGateway, ScriptedEvaluator, StubIndex, NullCache,
              WorkflowGateway, WorkflowRepository, RecordingRag, _ResetServices (FastAPI TestClient)
 samples: data/sample_invoices/ and tests/samples/ (read by test_sample_documents and the OCR tests)
```

State: only the real `config/rules.yml` is shared (session fixture). Environment defaults are set with `setdefault`, so the settings object can be built without a real database password. Skipped automatically: tests marked `requires_tesseract` (when `tesseract` is not on `PATH`) and `requires_database`.

---

## 1. Parsing and extraction

| Tests | What they prove |
|---|---|
| `test_parse_number`, `test_parse_date`, `test_dates_use_formats_from_rules` | Number parsing per language and date parsing, with date formats taken from `rules.yml` |
| `test_pipe_table_with_codes`, `test_whitespace_rows_require_consistent_arithmetic`, `test_labelled_single_item` | `InvoiceParser` reads pipe tables, whitespace-separated rows (only when quantity, price and total agree) and labelled single items |
| `test_subtotal_is_not_mistaken_for_total`, `test_ambiguous_currency_is_left_empty`, `test_missing_fields_stay_empty` | The parser leaves a field empty rather than guessing |
| `test_sample_documents` | Each sample file in `SAMPLE_EXPECTATIONS` yields the values read by eye from the documents |
| `test_scanned_png_uses_ocr`, `test_pdf_without_text_layer_falls_back_to_ocr` | OCR path (skipped without Tesseract) |

## 2. Validation

Helpers: `invoice(**overrides)` builds a `ParsedInvoice` (fixed `TODAY = 2026-01-15`); `data_check` runs `validate_invoice_data`; `FakeErp` records calls (`purchase_order`, `vendor`, `sku`) and can simulate an outage by raising `ErpUnavailableError`.

| Tests | What they prove |
|---|---|
| `test_consistent_invoice_has_no_data_findings`, `test_missing_required_fields_are_listed`, `test_line_subtotal_tax_and_total_arithmetic`, `test_unsupported_currency_and_future_date`, `test_quality_gates`, `test_duplicate_invoice_number_is_flagged` | Data-only rules: arithmetic, required fields, currency, dates, OCR and translation confidence gates, duplicates |
| `test_unit_price_deviation_beyond_tolerance`, `test_all_erp_data_comes_through_the_client`, `test_unknown_po_and_currency_mismatch`, `test_over_billed_quantity_is_high_severity`, `test_line_without_code_is_matched_by_description`, `test_unmatched_line_uses_matcher_then_flags`, `test_vendor_names_are_compared_without_legal_suffixes` | ERP rules: tolerances, PO lookup, matching lines without item codes, vendor name normalisation. Everything comes through the ERP client |
| `test_erp_outage_marks_validation_incomplete`, `test_no_po_number_skips_erp` | Outage handling and skipping the ERP when no PO number exists |
| `test_rules_yml_is_mapped_onto_engine_fields`, `test_decisions_follow_rules_yml_policies`, `test_policies_can_be_changed_in_rules_yml` | `decide` follows the policies in `rules.yml`, and changing them changes decisions |
| `test_html_report_escapes_document_content`, `test_html_report_sections_follow_reporting_switches` | `render_html` escapes document text and respects the reporting switches |

## 3. Corrections, previews, review limit, file serving

| Tests | What they prove |
|---|---|
| `test_a_wrong_item_code_points_to_the_po_item_its_description_matches` and two similar tests (unknown code without a matching description; line without a code keeps the description message), `test_suggest_po_line_needs_a_similar_enough_description` | Messages for an item code that is not on the PO, with a suggestion only when the description is similar enough |
| Translation-confidence tests (`test_correct_translations_are_not_marked_down...`, `test_an_untranslated_or_half_translated_text_is_still_marked_down`, `test_language_confidence_ignores_words_that_belong_to_several_languages`) | `detect_language` and `english_share` confidence heuristics |
| `test_field_labels_are_readable`, `test_summary_and_report_use_readable_names` | `field_label` and the summary use readable names |
| `test_docx_preview_*`, `test_pdf_and_image_previews_are_embedded_page_images`, `test_pdf_preview_crops_empty_margins_and_renders_sharp` | DOCX and page previews keep order, render tables, escape markup, embed page images, crop margins; a non-DOCX file is rejected |
| `test_files_are_served_only_from_inside_their_folder` | `_file_inside` from `main.py` refuses paths outside the folder |
| `test_review_limit_policy`, `test_escalation_note_gives_the_attempts_and_readable_reasons`, `test_system_recommendation_text_reaches_the_index`, `test_the_attempt_limit_never_rewrites_the_rules_verdict` | `apply_review_limit`: after the last attempt the system recommends rejection and the rules' verdict is not rewritten |
| `test_valid_corrections`, `test_invalid_corrections`, `test_latest_correction_wins_and_original_is_untouched` | `normalise_correction`, `InvalidCorrectionError`, `apply_corrections` and `current_value` |

## 4. RAG pipeline

| Tests | What they prove |
|---|---|
| `test_json_is_extracted_from_chatty_model_output`, `test_cache_keys_are_stable_and_namespaced` | `parse_json_object` and `cache_key` |
| `test_chunks_overlap_and_cover_all_lines`, `test_report_text_mentions_reasons`, `test_rerank_prefers_reports_for_why_questions` | `chunk_text`, `report_text`, `rerank` |
| `test_structured_routing`, `test_structured_answer_filters_by_vendor_and_decision` | `try_structured_answer` with `FakeRepository` |
| `test_answer_above_threshold_is_accepted_first_time_and_cached`, `test_low_scores_retry_with_rewritten_query_up_to_three_times`, `test_retry_stops_once_threshold_is_exceeded`, `test_threshold_must_be_exceeded_not_just_met` | `RagPipeline` retry loop with `ScriptedGateway`, `ScriptedEvaluator`, `StubIndex`, `NullCache` from `make_pipeline(scores)` |
| `test_reflection_falls_back_to_llm_judge_when_ragas_fails`, `test_reflection_reports_failure_when_no_evaluator_works` | Evaluator fallback and failure reporting |
| `test_payload_filter_covers_invoice_vendor_language_and_source_type`, `test_rag_passes_every_filter_to_qdrant_and_caches_per_filter_set`, `test_generation_agent_checks_redis_first_under_llm_sha256_of_the_prompt` | Qdrant payload filters, per-filter caching, and cache-first generation under the `llm:` key |

## 5. Checkpointer (needs PostgreSQL)

`saver` builds a `Database` with `search_path=langgraph` from `CHECKPOINT_TEST_DB_*` variables and a `PostgresCheckpointSaver`. `test_failed_run_resumes_from_last_completed_step` builds a two-node LangGraph graph whose second node fails once. It asserts the state after the failure (`value` 2, next node `second`), then that resuming with `invoke(None, config)` returns value 20 without running `first` again, that history has at least 3 entries, and that `delete_thread` removes the state.

## 6. Query workflow

| Tests | What they prove |
|---|---|
| `test_sql_guard_accepts_read_only_selects_over_audit_tables`, `test_sql_guard_rejects_everything_else`, `test_sql_comments_cannot_hide_a_forbidden_statement`, `test_default_limit_only_on_non_aggregate_queries` | `validate_select`, `SqlRejectedError`, `ensure_limit`: only read-only SELECTs over allowed audit tables, comments cannot hide a statement, a default LIMIT is added only for non-aggregate queries |
| `test_router_json_is_validated_and_filters_are_sanitised`, `test_keyword_router_fallback` | `parse_route` and `heuristic_route` |
| `test_structured_question_runs_generated_sql_and_returns_sql_rows_and_summary` and following | `QueryWorkflow` with `WorkflowGateway`, `WorkflowRepository` and `RecordingRag`: generated SQL runs and is summarised; a write statement from the model is never executed; a rejected query or database error gets one repair attempt with the error fed back; unusable SQL falls back to the built-in answer for plain counts; a selected-invoice scope must be respected; open-ended questions go to RAG with resolved filters; interface context overrides router filters; RAG does not search everything when the named invoice or vendor does not exist; without a router model, keyword rules still route |

## 7. Reset, accepted exceptions, human text

| Tests | What they prove |
|---|---|
| `test_clear_incoming_deletes_files_but_keeps_folders_and_dot_files`, `test_the_monitor_does_not_scan_while_paused`, `test_cache_cleanup_removes_only_the_given_namespaces` | Monitor clean-up and pause, `Cache.delete_namespaces` |
| `test_reset_*` (4 tests), `test_emptying_a_directory_refuses_the_filesystem_root` | `POST /admin/reset` through FastAPI's `TestClient` with `_ResetServices` (no start-up): wrong phrases give 422 and delete nothing; `allow_reset=False` gives 403; success calls files, tables, vectors and cache in that order and empties the reports folder but keeps it; a failure part way gives 500 with "safe to run it again" and a second run succeeds; `_empty_directory` refuses `/` |
| `test_exception_keys_identify_missing_fields_and_findings`, `test_only_missing_fields_and_medium_or_low_findings_are_acceptable`, `test_resolve_accepted_checks_every_key_against_the_latest_run`, `test_a_correction_request_needs_a_correction_or_an_accepted_exception`, `test_accepted_exceptions_appear_in_the_html_report_and_in_the_index_text`, `test_an_accepted_exception_no_longer_blocks_the_decision` | `missing_key`, `finding_key`, `current_exceptions`, `resolve_accepted`; high-severity findings cannot be accepted; accepted exceptions reach the report and index text and no longer block the decision |
| `test_reviewer_recommendation_and_decision_text_reaches_the_index`, `test_report_text_carries_reviewer_notes_and_the_human_outcome`, `test_html_report_lists_reviewer_notes_escaped` | Human actions and notes appear in the index text and in the escaped HTML report |

Why fakes everywhere: each test pins one rule (a threshold, a guard, a policy) so a failure names the rule; real services are covered by `test_integration.py` and `verify.sh`.

---

## End-to-end example

`test_a_write_statement_from_the_model_is_never_executed`:

1. `WorkflowGateway(sql=("DELETE FROM audit.invoice_audit", "DROP TABLE audit.invoice_audit"))` scripts a model that answers every SQL request with a write statement.
2. `make_workflow(gateway, repository).ask("remove all invoices please, how many are there")` routes the question to SQL.
3. `validate_select` rejects the write statements before they reach the repository (the neighbouring `test_a_rejected_query_is_repaired_once_with_the_error_fed_back` covers the single repair attempt).
4. The assertions: `repository.executed == []` (nothing reached the database), the result is not accepted, and `"SELECT"` appears in the answer or error text.
