# What extraction.py does

`backend/extraction.py` (the "Extractor Agent") turns an invoice file (PDF, DOCX or image) into raw text, turns that text into a structured `ParsedInvoice` with deterministic rules, and renders HTML previews of the original document for the browser. It uses no LLM: the same text always produces the same values.

Main parts:

1. File type support and the `extract_text()` entry point
2. PDF extraction with an OCR fallback, and image OCR
3. DOCX extraction (standard library only)
4. HTML previews of the original file
5. Number and date parsing
6. `InvoiceParser`: header fields, amounts, currency and line items

---

## Overall flow

```
 file in incoming/ ──► is_supported(path)?
                              │ yes
                              ▼
                    extract_text(path, ocr_languages)
        ┌─────────────────────┼──────────────────────────┐
        ▼ .pdf                ▼ .docx                    ▼ .png .jpg .jpeg .tif .tiff
   _extract_pdf          _extract_docx               _ocr_image (Tesseract)
   text layer? ─yes─► method "pymupdf"      method "docx"   method "tesseract"
        │ no (< 25 alphanumeric chars)
        └─► render pages at 300 dpi ─► _ocr_image ─► method "tesseract"
                              │
                              ▼
                    ExtractedText(text, method, ocr_confidence, page_count)
                              │  (stored raw in the database, then translated)
                              ▼
   InvoiceParser.parse(raw text, language=…) ──► ParsedInvoice (+ extras: field_sources, parse_notes)

 main.py ──► render_docx_html / render_pages_html ──► HTML preview of the original document
```

No state is kept in this module. Parsing runs on the original text, not on the translation, so a translation mistake can never change an amount.

---

## 1. File support and `extract_text`

| Name | Input | Output |
|---|---|---|
| `SUPPORTED_SUFFIXES` | | `.pdf`, `.docx`, `.png`, `.jpg`, `.jpeg`, `.tif`, `.tiff` |
| `UnsupportedFileError` (`ValueError`), `ExtractionError` (`RuntimeError`) | | Raised for unsupported types and unreadable or empty files |
| `ExtractedText` | frozen dataclass: `text`, `method` (`pymupdf`, `docx` or `tesseract`), `ocr_confidence` (or `None`), `page_count` (default 1) | |
| `is_supported(path)` | Path | True if the lower-cased suffix is supported |
| `extract_text(path, ocr_languages)` | Path and a Tesseract language string such as `eng+deu+spa` | `ExtractedText`. Raises `ExtractionError` when an image cannot be opened or no text at all was found |

## 2. PDF and image OCR

| Function | Input | Output |
|---|---|---|
| `_extract_pdf(path, ocr_languages)` | PDF path | Reads each page with `page.get_text("text", sort=True)` (reading order). If the pages contain at least `MIN_TEXT_LAYER_CHARS` (25) alphanumeric characters, returns `method="pymupdf"`. Otherwise logs a fallback and renders each page at `OCR_RENDER_DPI` (300), OCRs it, and returns `method="tesseract"` with the mean page confidence |
| `_ocr_image(image, languages)` | PIL image | `(text, confidence)`. Converts to grayscale after applying EXIF orientation. Confidence is the mean confidence of non-empty words with confidence of 0 or more, rounded to 2 decimals, or `None` |

- A `TesseractError` or `TesseractNotFoundError` becomes `ExtractionError`.
- Why the threshold: a scanned PDF often has an empty or junk text layer. Counting alphanumeric characters decides whether the layer is usable without needing OCR first.
- `ocr_confidence` is later compared with `rules.quality.min_ocr_confidence` in `validation.py`.

## 3. DOCX extraction

| Function | Purpose |
|---|---|
| `_paragraph_text(paragraph)` | Concatenates text runs, with `\t` for tabs and `\n` for line breaks |
| `_extract_docx(path)` | Opens the DOCX as a ZIP, parses `word/document.xml` with `ElementTree`, and walks the body in order. Paragraphs become lines; each table row becomes one line with cells joined by `" | "`. Bad ZIP, missing part or bad XML raises `ExtractionError` |

Why the pipe format: it keeps table structure so `InvoiceParser._pipe_table` can read line items from DOCX tables.

## 4. HTML previews

These serve the browser a view of the original file. A PDF viewer cannot run inside the dashboard's sandboxed iframe, but images and HTML always show.

| Function | Input | Output |
|---|---|---|
| `render_docx_html(path)` | DOCX path | Self-contained HTML with paragraphs and tables in order. Every text value goes through `html.escape` because it comes from untrusted documents. No formatting beyond structure is kept |
| `_content_bounds(page)` | PyMuPDF page | The union of text blocks, images and drawings, or `None` for a blank page |
| `render_pages_html(path, *, max_pages=8, target_px=1400)` | PDF or image path | Self-contained HTML with each page as a base64 PNG. Crops empty left and right margins (same edges on every page, 24 pt margin, never narrower than 75% of the page width), picks a dpi between 72 and 300 so the page is about `target_px` wide, and shows a note if pages were cut off. Any error becomes `ExtractionError` |

## 5. Number and date parsing

| Function | Input | Output |
|---|---|---|
| `parse_number(text, *, language=None)` | Text such as `1,617.00`, `1.617,00`, `25.00€`, `$1.25`, `40000` | `Decimal` or `None` |
| `_last_amount(line, *, language)` | A line | The last number on it that is not followed by `%` |
| `parse_date(text, formats=())` | Text and `strptime` formats | `date` or `None` |

`parse_number` rules:
- Both `,` and `.` present: whichever comes last is the decimal separator.
- Only `,`: thousands separator if it matches `\d{1,3}(,\d{3})+`, otherwise a decimal comma.
- Several `.`: thousands separators.
- A single `.` followed by exactly three digits is one thousand only when `language` is `de`, `es`, `it`, `fr`, `nl` or `pt`.
- A leading minus is kept.

`parse_date` rules:
1. The configured `strptime` formats from `rules.yml > date_validation` are tried first.
2. Then ISO `YYYY-MM-DD`, day-month-name-year (English, German, Spanish month names, optional "de"), month-name-day-year, and numeric `D/M/YYYY` (also `.` and `-`).
3. Numeric dates are day-first unless the second number is above 12, in which case month-first is used. This is for international invoices.
4. An impossible date returns `None`.

## 6. `InvoiceParser`

`InvoiceParser(currency_symbols, date_formats=())` is built in `AuditWorkflow` from `rules.currency_symbols` and `rules.dates.accepted_formats`.

| Method | Purpose |
|---|---|
| `parse(text, *, language=None)` | Returns a `ParsedInvoice`. `extras` gets `field_sources` (the line or rule each value came from) and `parse_notes` (for example "No line items could be identified in the document.") |
| `_labelled(lines, pattern, sources, field)` | First line matching a label regex; value is stripped of trailing punctuation |
| `_vendor(lines, sources)` | A `Vendor:`, `Supplier:`, `From:`, `Lieferant:` or `Proveedor:` line, else the first line with letters that does not look like a title (`Invoice`, `Factura`, `Rechnung`, ...) |
| `_amounts(lines, invoice, sources, language)` | Matches subtotal, tax and total line patterns (English, German, Spanish). The last occurrence wins, because grand totals come last. A `%` on a tax line sets `tax_rate_pct` |
| `_currency(text, lines, sources)` | Order: explicit label (`Currency:`, `Währung:`, `Moneda:`), then the single currency on the last total line, then the most frequent symbol or ISO code in the text. A tie returns `None` and records "ambiguous" in the sources |
| `_line_items(lines, language)` | Tries three strategies in order and returns the first that finds items, with the strategy name |
| `_pipe_table(lines, language)` | Rows with at least 3 pipes. The last three cells are quantity, unit price and line total; earlier cells are item code and description. Header and decoration rows are skipped |
| `_whitespace_table(lines, language)` | Rows split by whitespace (optional code, description, qty, price, total). Stops at a subtotal, tax or total line. Only rows where `qty x price` is within 2% of the total are trusted, because there are no column separators |
| `_key_value_item(lines, language)` | A single item from labelled lines (`Description:`, `Quantity:`, `Unit price:`, `Item code:`) |

Header patterns:
- Invoice number: English, German and Spanish labels; fallback pattern `INV-...`.
- PO number: English, German and Spanish labels; fallback `PO-123`.
- Date: the first labelled line that parses to a date.
- Missing values stay `None`; nothing is guessed. The validation step then reports them as missing fields.

Why deterministic regex rules: every value can be traced to its source line through `field_sources`, results are reproducible, and no model call is needed for extraction.

---

## End-to-end example

A scanned German invoice `rechnung.pdf` arrives:

1. `is_supported()` accepts `.pdf`. `extract_text()` calls `_extract_pdf()`.
2. The text layer has only a few characters (less than 25), so each page is rendered at 300 dpi and OCR'd with `eng+deu+spa`. Result: `method="tesseract"`, `ocr_confidence=86.4`, `page_count=1`.
3. `workflow.py` stores the raw text, detects German and translates it.
4. `InvoiceParser.parse(raw_text, language="de")` runs on the German text. `Rechnungsnummer: RE-2025-014` becomes `invoice_number`; `Rechnungsdatum: 10. April 2025` becomes `date(2025, 4, 10)`.
5. The line `Gesamtbetrag 1.617,00 EUR` is read as `Decimal("1617.00")`, because both separators are present and the comma comes last. `EUR` is the only currency on the total line, so `currency="EUR"`.
6. A pipe table gives two `LineItem` rows. `extras["field_sources"]` records which line each value came from.
7. The browser later requests the original; `render_pages_html()` returns the page as an embedded PNG with margins cropped.
