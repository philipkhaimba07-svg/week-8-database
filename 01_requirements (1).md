# Capstone: Multi-Tenant E-Commerce Platform — Requirements Summary

**Track:** E-commerce / marketplace backend (PostgreSQL 16 + Redis)
**Context:** A SaaS platform hosts many independent online stores (*tenants*). Each store has a catalogue, customers, orders and payments. Stores must never see each other's data.

## Business goals
1. Let each tenant manage products, stock and categories.
2. Let customers register, browse, fill a cart, check out and view order history.
3. Give tenant owners analytics: best sellers, monthly revenue.
4. Keep a tamper-evident history of every change to money-related data.

## Functional requirements
| ID | Requirement |
|----|-------------|
| F1 | Tenants own categories, products, customers, orders (strict isolation). |
| F2 | Product has SKU (unique per tenant), price in integer cents, active flag, one category. |
| F3 | Inventory is tracked per product and may never go negative. |
| F4 | Customers register with email (unique per tenant) and a hashed password; phone is stored encrypted. |
| F5 | An order belongs to one customer, has 1..n line items (price snapshot at purchase) and a status lifecycle `pending → paid → shipped → delivered` or `cancelled`. |
| F6 | An order has 0..n payment attempts (method, status, amount). |
| F7 | Shopping cart is fast, temporary, per-customer, and expires if abandoned. |
| F8 | Product detail pages and "top sellers" are served with low latency under read-heavy load. |
| F9 | Inserts/updates/deletes on customers, products, orders, payments are audited (who, when, old/new values, never secrets). |

## Non-functional requirements
- **Security:** least-privilege DB roles; row-level security on every tenant table; hashed passwords; encrypted PII; parameterised queries only.
- **Performance:** dashboard queries over ~300k orders in tens of milliseconds (measured in `04_query_optimization.md`).
- **Recoverability:** `pg_dump -Fc` backups with a verified test restore.
- **Repeatability:** database builds from empty through ordered migrations; no manual DDL.

## Scale assumptions (benchmark data)
2 tenants, ~50k customers, 2k products, 300k orders, ~750k order lines.

## Out of scope
Shipping carriers, tax engines, real payment gateway, search relevance.

## Key design decisions
- Money as `integer` cents (no floating-point error).
- `tenant_id` on every business table (denormalised onto `order_items`, `payments`, `inventory`) so RLS is a cheap, index-friendly equality with no joins.
- Tenant context is passed per transaction via `set_config('app.tenant_id', …, true)`; if unset, RLS returns **zero rows** (fail closed).
- Redis for cart + cache; MongoDB deliberately **not** used (see `03_nosql.md`).
- Known trade-off: single-column FKs do not by themselves prevent a row pointing at another tenant's parent. RLS `WITH CHECK` blocks the app role from doing it; composite `(tenant_id, id)` FKs would make it structurally impossible and are listed as future work.
