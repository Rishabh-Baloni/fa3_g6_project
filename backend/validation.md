# What validation.py does

`backend/validation.py` turns a `ParsedInvoice` plus the rules from `rules.yml` into a list of `Finding` objects, missing fields and notes. It contains the HTTP client for the mock ERP, the invoice-only data checks, the ERP comparison ("business validation"), and the helpers that validate and apply auditor corrections.

Main parts:

1. `ErpClient`: HTTP access to the mock ERP
2. Data validation: completeness and internal arithmetic, no ERP and no LLM
3. Business validation: comparison with ERP purchase order, vendor and SKU data
4. `LlmLineMatcher`: one narrow LLM job for lines without an item code
5. Corrections: validate submitted values and overlay them in memory

---

## Overall flow

```
                          ParsedInvoice (extracted + corrections)
                                      │
        ┌─────────────────────────────┴──────────────────────────────┐
        ▼                                                            ▼
 validate_invoice_data(invoice, rules, today, duplicate ids,   validate_against_erp(invoice, rules, erp, matcher)
                       ocr_confidence, translation)                    │ no po_number ─► completed=False, note
   completeness ─ currency ─ line arithmetic ─ totals ─                │
   dates ─ quality ─ duplicate                                         ▼ ErpClient ──HTTP──► mock-erp  /po /vendor /sku
        │                                                      PO ─ vendor ─ currency ─ lines ─ tax rate
        ▼                                                            │  ErpUnavailableError ─► completed=False, note
 DataValidationResult(findings, missing_fields, notes)               ▼
                                                    BusinessValidationResult(completed, findings, notes, erp_snapshot)

 corrections:  normalise_correction()  (API validates input)  ──►  apply_corrections()  (each run, in memory)
```

Everything except `LlmLineMatcher` is deterministic, so every finding can be reproduced from stored values. Extracted values in the database are never changed by corrections.

---

## 1. `ErpClient`

The agent never reads ERP tables directly; all ERP data comes over HTTP.

| Method | Input | Output |
|---|---|---|
| `ErpClient(base_url, timeout_seconds=10.0)` | Base URL | Creates an `httpx.Client` |
| `health()` | none | Calls `/health`; raises `ErpUnavailableError` on failure |
| `purchase_order(po_number)` | PO number | JSON dict from `/po/{po}` or `None` |
| `vendor(vendor_id)` | Vendor ID | JSON dict from `/vendor/{id}` or `None` |
| `sku(sku_code)` | SKU code | JSON dict from `/sku/{code}` or `None` |
| `_get(path)` | Path | Parsed JSON; 404 returns `None`; a network error or any other non-200 status raises `ErpUnavailableError` |

- Path parameters are URL-quoted with `quote(value, safe="")`, so a value containing `/` cannot change the request path.
- 404 means "does not exist" (a business finding); other failures mean "ERP unavailable" (validation incomplete). The two cases are kept apart deliberately.

## 2. Data validation

`validate_invoice_data(invoice, rules, *, today, duplicate_invoice_ids, ocr_confidence, translation)` returns a `DataValidationResult` (`findings`, `missing_fields`, `notes`). It calls these checks in order:

| Check | Rule code and severity | Logic |
|---|---|---|
| `_check_completeness` | none (adds to `missing_fields`) | Every name in `rules.required_fields.header` must be non-empty. No line items adds `line_items`. Each required line field that is empty adds `line_items[N].field` |
| `_check_currency` | `unsupported_currency`, high | Currency present but not in `rules.accepted_currencies` |
| `_check_line_arithmetic` | `line_total_mismatch`, medium | For lines with quantity, unit price and line total: `deviation_pct(line_total, qty x price)` above `line_total_tolerance_pct` |
| `_check_totals` | `subtotal_mismatch` medium, `tax_mismatch` medium, `total_mismatch` high | Subtotal vs sum of lines; tax amount vs `subtotal x rate / 100`; total vs `(subtotal or sum of lines) + tax`. Each has its own tolerance from `rules.arithmetic` |
| `_check_dates` | `future_date` high, `stale_date` low | Date more than `max_future_days` ahead, or older than `stale_after_days` |
| `_check_quality` | `low_ocr_confidence` medium, `translation_unavailable` medium, `low_translation_confidence` medium | OCR confidence below `min_ocr_confidence`; translation status `unavailable` or `failed`; translated with confidence below `min_translation_confidence` |
| duplicate check (in `validate_invoice_data`) | `duplicate_invoice_number`, high | The repository found earlier invoices with the same number and vendor |

Helpers:

| Function | Purpose |
|---|---|
| `deviation_pct(actual, expected)` | `abs(actual - expected) / abs(expected) x 100`, rounded to 4 decimals. If `expected == 0`: 0 when `actual == 0`, otherwise 100 |
| `_money(value)` | Two-decimal text |
| `effective_line_total(item)` | The stated line total, else `quantity x unit_price`, else `None` |

All amounts are `Decimal`. Sums are skipped when any line total is unknown, so a partially parsed table does not trigger a false subtotal finding.

## 3. Business validation

`validate_against_erp(invoice, rules, erp, matcher=None)` returns a `BusinessValidationResult(completed, findings, notes, erp_snapshot)`.

- No `po_number`: `completed=False` with a note that the ERP comparison was not possible.
- `ErpUnavailableError`: `completed=False` with a note. `decide()` in `workflow.py` then forces manual review, so an ERP outage never silently approves an invoice.

`_validate` runs these steps:

| Step | Rule code and severity | Logic |
|---|---|---|
| PO lookup | `unknown_po`, high | PO not found; stops here |
| Vendor | `vendor_mismatch`, high | Looks up the PO's vendor; `similarity()` below `vendor_name_similarity` |
| Currency | `currency_mismatch`, high | Invoice currency differs from the PO currency. A missing currency adds a note |
| `_compare_lines` | see below | Matches each invoice line to a PO line |
| `_compare_tax_rate` | `unknown_sku` high, `tax_rate_mismatch` medium | Compares the invoice tax rate with the SKU master |

`_compare_lines` matching order for each invoice line:
1. By item code (case-insensitive).
2. Without a code: best description by `similarity()`, accepted at or above `description_match_similarity`.
3. Otherwise the optional `matcher` (the LLM matcher).
4. Matches by description or LLM add a note ("matched PO item ... by ...") so an auditor can see how.

Findings from `_compare_lines` and `_compare_line_values`:

| Rule code | Severity | When |
|---|---|---|
| `line_not_on_po` | high | No PO line matched. If a wrong item code was given, the message names the PO item the description points to (`_line_not_on_po_message` using `suggest_po_line`) |
| `duplicate_line` | high | A PO line is billed a second time |
| `quantity_mismatch` | high if billed above the PO quantity, else medium | Deviation above `quantity_pct` |
| `unit_price_mismatch` | high if deviation above `high_severity_price_deviation_pct` (default 25%), else medium | Deviation above `unit_price_pct` |

PO lines that were never invoiced add a note ("partial delivery?").

`_compare_tax_rate`:
- The invoice rate is `tax_rate_pct`, or `tax_amount / subtotal x 100` if the rate is missing.
- Skipped if there is no rate or no matched item.
- Looks up each matched SKU; an unknown SKU is a finding. If the matched SKUs have different rates, the comparison is skipped with a note.

Name matching helpers:

| Function | Purpose |
|---|---|
| `normalise_name(name)` | ASCII-fold, lower-case, remove legal suffixes (`ltd`, `gmbh`, `inc`, `s.a.`, ...) and punctuation |
| `similarity(left, right)` | `SequenceMatcher` ratio of the normalised names |

## 4. `LlmLineMatcher`

| Item | Purpose |
|---|---|
| `LineMatcher` | `Protocol` with `match(item, po_lines) -> int | None` |
| `LlmLineMatcher(gateway).match(item, po_lines)` | Returns `None` if no reasoning model is configured. Otherwise sends the `line_matcher` / `match_line` prompt (max 100 tokens) and expects JSON with `line_number`. The answer is accepted only if it is an integer that exists in the PO lines. Model errors are logged and give `None` |

Why: plain string similarity fails when a description is in another language, and the LLM is used only for that one decision. The result is validated against real PO line numbers and recorded in the notes.

## 5. Corrections

| Function | Input | Output |
|---|---|---|
| `InvalidCorrectionError` | | `ValueError` safe to show to a user |
| `normalise_correction(field_name, value)` | Field name and submitted text | Canonical text. Empty value is rejected. Text fields (`invoice_number`, `vendor_name`, `po_number`) pass through. `currency` must be three letters and is upper-cased. `invoice_date` must be `YYYY-MM-DD`. Decimal fields (`subtotal_amount`, `tax_rate_pct`, `tax_amount`, `total_amount`, and line `quantity`, `unit_price`, `line_total`) are parsed as `Decimal` after removing commas. Line `item_code` and `description` pass through. Any other field name is rejected |
| `current_value(invoice, field_name)` | Invoice and a field such as `line_items[2].unit_price` | The current value or `None` |
| `apply_corrections(invoice, corrections)` | Invoice and correction rows, oldest first | `(corrected copy, applied list)`. Works on a deep copy. For each field only the latest correction counts. A correction for a line number that does not exist creates that line. Applied entries keep `field_name`, `original_value`, `corrected_value`, `corrected_by` and `corrected_at` (as text) |

Field names for lines follow the pattern `line_items[N].item_code|description|quantity|unit_price|line_total`.

---

## End-to-end example

An invoice from "Acme GmbH" for PO `PO-1001` is validated:

1. `validate_invoice_data()` finds all required header fields present (in `rules.yml` these are invoice number, date, vendor, currency and total). The line `2 x 10.00 = 25.00` gives `line_total_mismatch` (medium, deviation 25%, above the 1% tolerance), with `expected_value="20.00"`.
2. `validate_against_erp()` calls `/po/PO-1001`, then `/vendor/<id>`. The vendor names match after `normalise_name` strips "GmbH".
3. Line 1 has code `ABC-1`, found on the PO. Its unit price is 10.00 vs PO 7.00, a 42.86% deviation. That is above 25%, so `unit_price_mismatch` is high.
4. Line 2 has no code and the description is in German. Similarity is too low, so `LlmLineMatcher` is asked and returns PO line 2. A note records "matched ... by LLM description match".
5. A reviewer later submits `line_items[1].unit_price = "7.00"`. `normalise_correction()` returns `"7.00"`. Note that it only strips commas (`"7,00"` would become `"700"`), so a decimal comma must not be used here.
6. On re-validation, `apply_corrections()` overlays the correction on a copy of the extracted invoice. The price finding disappears, while the stored extracted values stay unchanged.
