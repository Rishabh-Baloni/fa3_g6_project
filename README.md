# Code walkthrough

Developer walkthroughs for every Python file in the repository. Each document explains what the file turns into what, how data and state move through it, and one concrete example. All statements come from reading the code.

## Files

| Doc | Source | Role |
|---|---|---|
| [backend/settings.md](backend/settings.md) | `backend/settings.py` | Turns environment variables into `Settings` and `config/rules.yml` into `Rules` |
| [backend/llm.md](backend/llm.md) | `backend/llm.py` | Single gateway for chat and embedding model calls, Redis caches, prompt loading |
| [backend/database.md](backend/database.md) | `backend/database.py` | Domain models, `audit` schema, `AuditRepository` and the PostgreSQL LangGraph checkpoint saver |
| [backend/extraction.md](backend/extraction.md) | `backend/extraction.py` | Invoice file to text (PDF, DOCX, OCR) to `ParsedInvoice` with deterministic rules; HTML previews |
| [backend/validation.md](backend/validation.md) | `backend/validation.py` | Rule checks on the data, comparison with the mock ERP, and correction handling |
| [backend/workflow.md](backend/workflow.md) | `backend/workflow.py` | LangGraph audit pipeline, recommendation policy, reports and the incoming-folder monitor |
| [backend/rag.md](backend/rag.md) | `backend/rag.py` | Qdrant indexing, retrieval-augmented answers and reflection scoring |
| [backend/query.md](backend/query.md) | `backend/query.py` | Routes questions to guarded SQL or RAG |
| [backend/erp.md](backend/erp.md) | `backend/erp.py` | Separate mock-ERP FastAPI service over the `erp` schema |
| [backend/main.md](backend/main.md) | `backend/main.py` | Agent FastAPI app: start-up wiring and all 16 HTTP routes |
| [frontend/app.md](frontend/app.md) | `frontend/app.py` | Streamlit entry point: navigation, queue counts, service status |
| [frontend/data.md](frontend/data.md) | `frontend/data.py` | Read-only database queries and the HTTP client for the agent |
| [frontend/ui.md](frontend/ui.md) | `frontend/ui.py` | Shared vocabulary, styles, formatting and notifications |
| [frontend/page_overview.md](frontend/page_overview.md) | `frontend/page_overview.py` | Landing page with live counts |
| [frontend/page_intake.md](frontend/page_intake.md) | `frontend/page_intake.py` | Step 1: upload, file tracking, retry and reset |
| [frontend/page_review.md](frontend/page_review.md) | `frontend/page_review.py` | Step 2: Review Queue with fix, accept and recommend-reject |
| [frontend/page_reports.md](frontend/page_reports.md) | `frontend/page_reports.py` | Step 3: audit ledger and final sign-off |
| [frontend/page_ask.md](frontend/page_ask.md) | `frontend/page_ask.py` | Step 4: natural-language questions (SQL or RAG) |
| [tests/test_backend.md](tests/test_backend.md) | `tests/test_backend.py` | Unit tests with fakes: parsing, validation, policy, RAG, SQL guard, reset |
| [tests/test_frontend.md](tests/test_frontend.md) | `tests/test_frontend.py` | Headless Streamlit page tests with a fake database and agent |
| [tests/test_integration.md](tests/test_integration.md) | `tests/test_integration.py` | End-to-end tests against a running stack |

Not documented: non-Python files (`compose.yaml`, `config/rules.yml`, `backend/prompts/*.yaml`, `backend/db/*`, `tests/verify.sh`, Dockerfiles), except where a Python file loads or calls them. There are no `__init__.py` files.

## How it fits together

```
 Browser ──► frontend (Streamlit)                                   incoming/  ◄── folder drops
              app ─► page_overview | page_intake | page_review | page_reports | page_ask
                       └─ ui (vocabulary, styles)   └─ data
                             reads: PostgreSQL (audit, read-only role)    changes: HTTP
                                                                            │
                                                                            ▼
 backend/main.py  (agent API, wires everything in build_services)
    ├─ workflow ─ InvoiceMonitor ─► AuditWorkflow (LangGraph, checkpoints in PostgreSQL schema langgraph)
    │      ├─ extraction (text, OCR, parse) ─► validation ─► erp.py (mock ERP over HTTP, schema erp)
    │      ├─ llm (translation, summaries; Redis caches)
    │      └─ rag (index into Qdrant)
    ├─ query ─► SQL guard ─► database (read-only SELECT)   or   rag (retrieve ─► generate ─► reflect)
    └─ database (AuditRepository: PostgreSQL schema audit; the agent is the only writer)
 settings ◄── used by llm, validation, workflow, main  (env vars + config/rules.yml)
```

Import dependencies between backend modules: `settings` and `database` are base modules; `llm` uses `settings`; `extraction` uses `database`; `validation` uses `database`, `settings`, `llm`; `rag` uses `database`, `llm`, `settings`; `query` uses `database`, `llm`, `rag`; `workflow` uses all of those except `query`; `main` uses all of them except `erp`, which is standalone. Every frontend page imports `data` and `ui`; `app` imports `data`, `ui` and all pages.

The invoice lifecycle used by the dashboard and the agent: Intake, Review Queue (up to 3 attempts), Reports and Sign-Off, then Ask AI. Which step an invoice is in is `audit.invoice_overview.workflow_stage`.

## Suggested reading order

1. [backend/settings.md](backend/settings.md) and [backend/database.md](backend/database.md): configuration and data model.
2. [backend/extraction.md](backend/extraction.md), [backend/validation.md](backend/validation.md), [backend/workflow.md](backend/workflow.md): how one invoice is audited.
3. [backend/main.md](backend/main.md): the HTTP surface.
4. [backend/rag.md](backend/rag.md) and [backend/query.md](backend/query.md): questions about audited invoices.
5. [frontend/app.md](frontend/app.md) then the pages in lifecycle order.
6. The tests, to see each behaviour pinned down.
