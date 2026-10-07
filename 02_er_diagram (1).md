# ER Diagram

`er_diagram.dbml` can be pasted into https://dbdiagram.io for the editable version. Mermaid rendering below (cardinalities are on the relationship lines).

```mermaid
erDiagram
    TENANTS      ||--o{ CUSTOMERS   : "has"
    TENANTS      ||--o{ CATEGORIES  : "has"
    TENANTS      ||--o{ PRODUCTS    : "sells"
    TENANTS      ||--o{ ORDERS      : "receives"
    CATEGORIES   ||--o{ PRODUCTS    : "classifies"
    PRODUCTS     ||--|| INVENTORY   : "has stock row"
    CUSTOMERS    ||--o{ ORDERS      : "places"
    ORDERS       ||--|{ ORDER_ITEMS : "contains 1..n"
    PRODUCTS     ||--o{ ORDER_ITEMS : "appears in"
    ORDERS       ||--o{ PAYMENTS    : "paid by 0..n"

    TENANTS { int tenant_id PK
              text name UK
              timestamptz created_at }
    CUSTOMERS { bigint customer_id PK
                int tenant_id FK
                text email "UK with tenant_id"
                text password_hash "bcrypt"
                text full_name
                bytea phone_enc "pgp encrypted"
                timestamptz created_at
                timestamptz updated_at }
    CATEGORIES { int category_id PK
                 int tenant_id FK
                 text name "UK with tenant_id" }
    PRODUCTS { bigint product_id PK
               int tenant_id FK
               int category_id FK
               text sku "UK with tenant_id"
               text name
               text description
               int price_cents
               boolean is_active
               timestamptz created_at
               timestamptz updated_at }
    INVENTORY { bigint product_id PK
                int tenant_id FK
                int qty_on_hand
                timestamptz updated_at }
    ORDERS { bigint order_id PK
             int tenant_id FK
             bigint customer_id FK
             text status
             bigint total_cents
             timestamptz placed_at
             timestamptz updated_at }
    ORDER_ITEMS { bigint order_id PK
                  int line_no PK
                  int tenant_id FK
                  bigint product_id FK
                  int qty
                  int unit_price_cents "price snapshot" }
    PAYMENTS { bigint payment_id PK
               int tenant_id FK
               bigint order_id FK
               int amount_cents
               text method
               text status
               timestamptz paid_at }
    AUDIT_LOG { bigint audit_id PK
                timestamptz at
                text table_name
                text op
                text row_pk
                int tenant_id
                text actor
                jsonb old_row
                jsonb new_row }
```

`AUDIT_LOG` has no FKs on purpose: audit rows must outlive the rows they describe.

## Cardinalities
| Relationship | Cardinality | Rule |
|---|---|---|
| Tenant → Customer / Category / Product / Order | 1 : N | each row belongs to exactly one tenant |
| Category → Product | 1 : N | a product has exactly one category |
| Product → Inventory | 1 : 1 | PK = FK on `product_id` |
| Customer → Order | 1 : 0..N | a customer may have no orders yet |
| Order → Order item | 1 : 1..N | ≥1 line enforced by the checkout transaction |
| Product → Order item | 1 : 0..N | resolves the M:N between orders and products |
| Order → Payment | 1 : 0..N | retries / split payments |
