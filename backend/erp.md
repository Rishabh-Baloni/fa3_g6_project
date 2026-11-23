# What erp.py does

`backend/erp.py` is a separate, small FastAPI service (the "mock ERP", its own container built from `Dockerfile.erp`). It turns rows in the PostgreSQL `erp` schema (purchase orders, vendors, SKU master) into JSON over four read-only HTTP routes. The audit application never reads ERP tables itself; it calls this service through `ErpClient` in `validation.py`.

Main parts:

1. Response models (`Vendor`, `Sku`, `PoLine`, `PurchaseOrder`)
2. Database engine and start-up check (`_engine`, `lifespan`)
3. Query helper (`_rows`)
4. The four routes

---

## Overall flow

```
 validation.ErpClient ──HTTP GET──► /health | /po/{po_number} | /vendor/{vendor_id} | /sku/{sku_code}
                                         │
                                         ▼  _rows(sql, **params)  (bound parameters, one pooled connection)
                                   SQLAlchemy engine  (psycopg, search_path=erp)
                                         │
                                         ▼  role erp_svc: can read schema `erp` only
                                   PostgreSQL tables: purchase_orders, po_line_items, vendors, sku_master
                                         │
                      row found ─► Pydantic model ─► JSON 200      no row ─► HTTP 404      DB down ─► /health 503
```

State: one SQLAlchemy engine stored in `app.state.engine` (pool of 5 plus 5 overflow, `pool_pre_ping`, 5 s connect timeout). It is created at start-up and disposed on shutdown. No caching.

---

## 1. Models

| Model | Fields |
|---|---|
| `Vendor` | `vendor_id`, `vendor_name`, `country` (nullable), `currency` |
| `Sku` | `item_code`, `description`, `category`, `uom` (all nullable except the code), `tax_rate_pct` (`Decimal`) |
| `PoLine` | `line_number`, `item_code`, `description`, `quantity` (`Decimal`), `unit_price` (`Decimal`), `currency` |
| `PurchaseOrder` | `po_number`, `vendor_id`, `po_date` (text, nullable), `currency`, `status`, `line_items: list[PoLine]` |

## 2. Engine and lifespan

| Function | Input | Output |
|---|---|---|
| `_engine()` | Environment variables `ERP_DB_HOST`, `ERP_DB_NAME`, `ERP_DB_USER`, `ERP_DB_PASSWORD` (required; `ERP_DB_PORT` defaults to 5432) | A SQLAlchemy `Engine` with connection option `-c search_path=erp`. Raises `RuntimeError` naming any missing variables |
| `lifespan(app)` | FastAPI app | At start-up runs `SELECT count(*) FROM purchase_orders`. If it fails, logs critical, disposes the engine and re-raises, so the service refuses to start with an unreachable database or missing schema. On success stores the engine in `app.state.engine` and disposes it at shutdown |

Logging goes to stdout at the level in `LOG_LEVEL` (default `INFO`).

## 3. `_rows`

`_rows(sql, **params)` runs a text query with bound parameters on a pooled connection and returns a list of dicts. All values reach the database as bound parameters (`:po`, `:id`, `:code`), never by string formatting.

## 4. Routes

| Route | Input | Output | Notes |
|---|---|---|---|
| `GET /health` | none | `{"status": "ok"}` | Runs `SELECT 1`; on failure HTTP 503 `database unavailable: ...` |
| `GET /po/{po_number}` | PO number | `PurchaseOrder` | Header lookup is case-insensitive (`upper(po_number) = upper(:po)`) on the stripped value. Then loads `po_line_items` for the stored `po_number`, ordered by `line_number`. 404 if the header is not found. `po_date` is cast to text |
| `GET /vendor/{vendor_id}` | Vendor id | `Vendor` | Case-insensitive; 404 `Vendor ... not found` |
| `GET /sku/{sku_code}` | SKU code | `Sku` | Case-insensitive on `item_code`; 404 `SKU ... not found` |

Why a separate service and a restricted role: it keeps ERP master data behind an HTTP boundary like a real ERP, and the `erp_svc` role can read only the `erp` schema (created by `backend/db/01_roles_and_schemas.sh`). A 404 means "does not exist" and an HTTP or connection failure means "unavailable"; `ErpClient` treats those differently.

---

## End-to-end example

`validation.ErpClient.purchase_order("po-1001")` is called:

1. The client requests `GET /po/po-1001` (the value is URL-quoted).
2. `get_purchase_order` strips the value and runs the header query with `upper(po_number) = upper('po-1001')`. It finds `PO-1001`, vendor `V001`, currency `EUR`.
3. It loads that PO's lines (for example two rows ordered by `line_number`) and returns a `PurchaseOrder` as JSON.
4. The client then calls `/vendor/V001` and, for matched item codes, `/sku/<code>`.
5. A request for `PO-9999` returns 404, which `ErpClient` turns into `None`, and `validation.py` records an `unknown_po` finding.
