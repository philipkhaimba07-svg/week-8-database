# NoSQL Layer — Redis (and why not MongoDB)

**Principle:** PostgreSQL stays the system of record for everything that must be correct (orders, stock, money). A second store is added only where its access pattern is clearly a bad fit for relational tables. Code: `app/shop.py`. Measured output: `docs/evidence/redis_demo_run.txt`.

## Use 1 — Shopping cart → Redis hash with TTL
| | |
|---|---|
| Key | `cart:{tenant_id}:{customer_id}` → hash `{product_id: qty}` |
| Expiry | `EXPIRE` 7 days, refreshed on each add |

**Why Redis, what it solves better than PostgreSQL**
- A cart is *temporary, hot, per-user state* with many writes (every +/− click) and almost no value once abandoned. In PostgreSQL each click is a WAL-logged row write that leaves dead tuples for autovacuum to clean, and abandoned carts need a cleanup job. Redis does `HINCRBY` in memory in microseconds and **expiry is built in**.
- Carts are not business records: losing one on a Redis restart is an annoyance, not a financial or audit event. The *order* is created in PostgreSQL.
- Correctness is preserved at the boundary: `checkout()` **ignores any price in the cart**, re-reads prices from PostgreSQL, locks the stock rows (`FOR UPDATE`), and creates order + items + payment + stock decrement in **one ACID transaction**. Only after COMMIT is the cart deleted. Demonstrated: a 10,000-unit cart is rejected (oversell blocked).

## Use 2 — Cache-aside for read-heavy pages → Redis strings with TTL
| Cached item | Key | TTL | Invalidation |
|---|---|---|---|
| Product detail | `product:{tenant}:{id}` | 5 min | deleted on price update |
| Top-10 sellers (Q1) | `top10:{tenant}` | 5 min | deleted on checkout |

**Why:** Q1 is an aggregate over ~47k order lines and, even after optimisation, takes ~110–120 ms — fine once, wasteful at 1,000 dashboard loads/minute for data that only needs to be minutes-fresh. Measured: **106 ms (DB) → 2.7 ms (cache)** for top sellers, **14 ms → 0.13 ms** for a product page. Indexes make a query cheaper; a cache removes the repeat work entirely. Tenant id is part of every key, so one tenant can never be served another's cached data.

**Trade-off accepted:** cached data can be up to 5 minutes stale (explicitly invalidated on the writes we know about). Acceptable for a dashboard and product page, **not** acceptable for stock-at-checkout or order totals, which is why those never use the cache.

## Why not MongoDB?
Considered and rejected — I'm choosing not to add a store just to tick a box.
- *Product catalogue with flexible attributes* is the classic Mongo pitch, but PostgreSQL `jsonb` (indexable with GIN) covers it without a second system, and products need joins/FKs to categories, inventory and order lines.
- *Event/ingestion buffer*: no high-volume ingestion requirement in this system's scope.
- Each extra datastore adds backup, security, monitoring and consistency work. If a clickstream/audit-analytics requirement appeared, Redis Streams or MongoDB would then be justified.

## Failure behaviour (design note)
If Redis is down: cache calls must fall back to PostgreSQL (the code paths are cache-aside, so this is a straightforward `try/except` to add); cart operations would fail visibly. Not implemented in the demo; listed under future work.
