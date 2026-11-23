# What settings.py does

`backend/settings.py` turns two configuration sources into typed Python objects: **environment variables** become a `Settings` object, and the supplied **`config/rules.yml`** becomes a `Rules` object. Every other backend module receives these objects instead of reading the environment or the YAML file itself.

Main parts:

1. Runtime configuration: the `Settings` model and `get_settings()`
2. `.env` loading for runs outside Docker
3. The rules model: `Rules` and its sub-models
4. Mapping `rules.yml` onto the engine's field names: `from_rules_yml()`
5. Loaders and path helpers: `load_rules()`, `get_rules()`, `reports_dir()`

---

## Overall flow

```
 environment variables ──┐          .env file (local runs only)
 (compose env_file)      │                 │
                         ▼                 ▼
                 _load_local_env_file()  (override=False)
                         │
                         ▼
                 _from_environment()  ──►  Settings.model_validate()  ──► get_settings()  (cached)
                                                   │ invalid/missing
                                                   ▼
                                           ConfigurationError

 config/rules.yml ──► load_rules() ──► from_rules_yml(raw, ocr_min_confidence)
                                           │  field-name mapping + ENGINE_DEFAULTS
                                           ▼
                                         Rules  ──► get_rules()  (cached)
```

`get_settings()` and `get_rules()` use `@lru_cache`, so the environment and the YAML file are each read and validated once per process. A changed `.env` or `rules.yml` needs a restart.

---

## 1. Runtime configuration: `Settings`

`Settings` is a Pydantic model. Field names are the lower-case form of the environment variable name (`agent_db_host` is read from `AGENT_DB_HOST`).

| Group | Fields (defaults) | Notes |
|---|---|---|
| PostgreSQL | `agent_db_host` (`postgres`), `agent_db_port` (5432), `agent_db_name`, `agent_db_user`, `agent_db_password` | The last three have `min_length=1` and no default, so they are required |
| Other services | `erp_base_url`, `qdrant_url`, `qdrant_collection` (`invoice_chunks`), `redis_url` | Default to compose service names |
| Filesystem | `incoming_dir`, `config_dir`, `prompts_dir`, `reports_dir` (optional) | `reports_dir=None` means "use `rules.yml` > `reporting.output_dir`" |
| Monitor and OCR | `monitor_enabled`, `allow_reset`, `monitor_poll_seconds` (5.0), `ocr_languages` (`eng+deu+spa`), `ocr_min_confidence` (70.0) | `allow_reset=false` disables `POST /admin/reset` |
| Chat models | `llm_provider`, `llm_translation_model`, `llm_rag_model`, `llm_reasoning_model`, `llm_api_base`, `llm_api_key`, `aws_region` | Empty string means "not configured" |
| Embeddings | `embedding_provider`, `embedding_model`, `embedding_api_base`, `embedding_api_key` | |
| Tuning | `llm_timeout_seconds` (60), `llm_max_retries` (2), `embedding_batch_size` (16), `llm_cache_ttl_seconds` (7 days), `rag_answer_cache_ttl_seconds` (1 hour), `log_level` | |
| Review limit | `max_review_iterations` (3, allowed 1 to 10) | How many correction batches an auditor may submit per invoice |

Two computed properties build SQLAlchemy URLs:

| Property | Output |
|---|---|
| `db_url` | `postgresql+psycopg://user:password@host:port/db` as a `sqlalchemy.engine.URL` |
| `checkpoint_db_url` | The same URL plus `options=-c search_path=langgraph`, so the LangGraph checkpointer works in its own schema |

Constants: `CHAT_PROVIDERS = ("bedrock", "groq")` and `EMBEDDING_PROVIDERS = ("bedrock", "huggingface")`.

## 2. Reading the environment

| Function | Input | Output |
|---|---|---|
| `_from_environment()` | Process environment | `dict[str, str]` of only the settings that are set |
| `_load_local_env_file()` | none | Loads the nearest `.env` found above the file, with `override=False`, then stops |
| `get_settings()` | none | A validated, cached `Settings` |

How it works:
- An empty or whitespace-only variable counts as unset. That is how "model not configured" works: leave `LLM_TRANSLATION_MODEL` empty and the default (empty string) applies.
- `aws_region` falls back to `AWS_REGION`, then `AWS_DEFAULT_REGION`, because Bedrock uses the standard AWS variables.
- On a Pydantic `ValidationError`, `get_settings()` raises `ConfigurationError("Invalid or missing settings: AGENT_DB_NAME, ...")`, listing the upper-cased variable names.
- A non-empty `LLM_PROVIDER` or `EMBEDDING_PROVIDER` outside the allowed tuples also raises `ConfigurationError`.

Why: secrets arrive through the compose `env_file` at run time, so they are never in source or image layers. The `.env` loader exists only for local runs; `override=False` guarantees real environment variables win.

## 3. The rules model

`Rules` is what the validation and reporting code reads. Sub-models:

| Class | Fields | Meaning |
|---|---|---|
| `RequiredFields` | `header`, `line_item` | Engine field names that must be present |
| `ArithmeticRules` | `line_total_tolerance_pct`, `subtotal_tolerance_pct`, `tax_tolerance_pct`, `total_tolerance_pct` | Allowed percentage difference in recomputed amounts |
| `ErpTolerances` | `unit_price_pct`, `quantity_pct`, `tax_rate_pct`, `vendor_name_similarity`, `description_match_similarity`, `high_severity_price_deviation_pct` | Allowed difference from the ERP purchase order |
| `DateRules` | `accepted_formats`, `max_future_days`, `stale_after_days` | Date parsing and plausibility |
| `QualityRules` | `min_ocr_confidence`, `min_translation_confidence` | Quality gates |
| `Policies` | `missing_field`, `total_mismatch`, `invalid_currency` | Each is an `Action`: `approve`, `flag`, `manual_review` or `reject` |
| `ReportingRules` | `output_dir`, `include_translation_confidence`, `include_discrepancy_summary` | Report switches |
| `Rules` | the above plus `field_types`, `accepted_currencies`, `currency_symbols` | Top-level container |

## 4. `from_rules_yml(raw, *, ocr_min_confidence)`

Input: the parsed YAML dict and the OCR threshold from `Settings`. Output: a `Rules`.

Mapping details:
- **Field names.** `rules.yml` uses invoice vocabulary; the engine uses its own. `HEADER_FIELD_MAP` maps for example `invoice_no` to `invoice_number`, `vendor_id` to `vendor_name`, `subtotal` to `subtotal_amount`. `LINE_FIELD_MAP` maps `qty` to `quantity` and `total` to `line_total`. `_map_fields()` raises `ValueError` for an unknown name, so a typo in `rules.yml` fails loudly.
- **Tolerances.** `line_item_math_tolerance_percent` feeds line total, subtotal and total tolerances. `tax_difference_percent` feeds both the arithmetic tax tolerance and the ERP tax-rate tolerance. `price_difference_percent` and `quantity_difference_percent` feed the ERP tolerances.
- **Defaults when the file is silent:** `max_future_days` 7, `stale_invoice_days` 365, `auto_approve_confidence_threshold` 0.95 (becomes `min_translation_confidence`), policies `missing_field_action` = `flag`, `total_mismatch_action` = `manual_review`, `invalid_currency_action` = `reject`, reporting `output_dir` = `./outputs/reports`.
- **`ENGINE_DEFAULTS`** covers values `rules.yml` does not define: `vendor_name_similarity` 0.85, `description_match_similarity` 0.6, `high_severity_price_deviation_pct` 25.
- Currency codes are upper-cased. Percentages go through `Decimal(str(value))` (`_pct`) to avoid float rounding.
- `min_ocr_confidence` is not in `rules.yml`; it comes from `Settings.ocr_min_confidence`.

## 5. Loaders and path helper

| Function | Input | Output |
|---|---|---|
| `load_rules(path, *, ocr_min_confidence=70.0)` | Path to a YAML file | `Rules` (uses `yaml.safe_load`) |
| `get_rules()` | none | Cached `Rules` from `config_dir / "rules.yml"` |
| `reports_dir()` | none | `Settings.reports_dir` if set, otherwise `rules.yml` `reporting.output_dir` resolved against the working directory |

---

## End-to-end example

1. The environment (for example through the compose `env_file`) contains `AGENT_DB_NAME`, `AGENT_DB_USER`, `AGENT_DB_PASSWORD`, `LLM_PROVIDER=groq`, `LLM_API_KEY=...` and an empty `LLM_REASONING_MODEL`.
2. Backend code calls `get_settings()` (`main.py` and `llm.py` both import it). Only the non-empty variables enter the dict, so `llm_reasoning_model` keeps its default `""`.
3. Validation passes. Provider `groq` is in `CHAT_PROVIDERS`. A `Settings` is cached.
4. `get_rules()` opens `/app/config/rules.yml`. The `required_fields.header` entry `invoice_no` becomes `invoice_number`.
5. `reports_dir()` returns the resolved `./outputs/reports` path, since `REPORTS_DIR` is unset.
6. If `AGENT_DB_PASSWORD` had been missing, startup would raise `ConfigurationError: Invalid or missing settings: AGENT_DB_PASSWORD`.
