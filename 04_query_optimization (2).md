# Query Optimization Evidence

**Environment:** PostgreSQL 16, Ubuntu 24 sandbox, default `shared_buffers`. Data: see row counts below. All plans were captured as **`app_user` with `app.tenant_id = 1`**, so RLS is included in what is measured (RLS adds a tenant predicate to every scan; this matters for Q3).
**Method:** `perf/run_bench.sh` rebuilds the DB to V5, bulk-loads data, runs each query once to warm the cache, captures `EXPLAIN (ANALYZE, BUFFERS)` (**BEFORE**), applies V6, `VACUUM ANALYZE`, warms again, captures **AFTER**. Reproduce with `./perf/run_bench.sh`.
**Honesty notes:** single-run timings on a shared sandbox are noisy — Q1 BEFORE measured 283 ms in one run and 372.362 ms in the next, so Q1 is reported as a range. Ratios are directional; the plan *shape* changes are the firm evidence.

```
t      | count  
-------------+--------
 products    |   2005
 customers   |  50003
 payments    | 213893
 orders      | 300021
 order_items | 750048
(5 rows)
```

## Summary

| Query | Question | Before | After | What changed |
|---|---|---|---|---|
| Q1 | Top 10 products by revenue, last 90 days | 283–372.362 ms | 119.900 ms (~2.4–3×) | Seq Scan → Index Scan on `orders`; Seq Scan → PK Index Scan on `order_items` |
| Q2 | Monthly revenue, last 12 months | 125.039 ms | 49.273 ms (~2.5×) | Seq Scan → **Index Only Scan**, 0 heap fetches |
| Q3 | Login lookup by email | 16.998 ms | 0.051 ms (~330×) | Seq Scan → Index Scan (query rewrite + CHECK constraint) |
| Q4 | A customer's 20 latest orders | 0.144 ms | 0.131 ms | **No change needed** — baseline FK index already optimal |

---

## Q1 — Top products by revenue (last 90 days)
**1. QUERY:** which 10 products earned the most revenue in the last 90 days, excluding cancelled orders?
```sql
-- Q1: Which 10 products earned the most revenue in the last 90 days (tenant 1)?
SELECT p.product_id, p.name, sum(oi.qty * oi.unit_price_cents)/100.0 AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p     ON p.product_id = oi.product_id
WHERE o.placed_at >= now() - interval '90 days'
  AND o.status <> 'cancelled'
GROUP BY p.product_id, p.name
ORDER BY revenue DESC
LIMIT 10;
```
**2. BEFORE PLAN** (`372.362 ms`)
```
 Limit (actual time=372.174..372.181 rows=10 loops=1)
   Buffers: shared hit=8351 read=3566 written=27
   ->  Sort (actual time=372.172..372.177 rows=10 loops=1)
         Sort Key: (((sum((oi.qty * oi.unit_price_cents)))::numeric / 100.0)) DESC
         Sort Method: top-N heapsort  Memory: 26kB
         Buffers: shared hit=8351 read=3566 written=27
         ->  HashAggregate (actual time=371.546..371.816 rows=1003 loops=1)
               Group Key: p.product_id
               Batches: 1  Memory Usage: 193kB
               Buffers: shared hit=8348 read=3566 written=27
               ->  Hash Join (actual time=66.818..359.481 rows=47399 loops=1)
                     Hash Cond: (oi.product_id = p.product_id)
                     Buffers: shared hit=8348 read=3566 written=27
                     ->  Hash Join (actual time=66.110..346.341 rows=47399 loops=1)
                           Hash Cond: (oi.order_id = o.order_id)
                           Buffers: shared hit=8323 read=3566 written=27
                           ->  Seq Scan on order_items oi (actual time=0.040..205.053 rows=450031 loops=1)
                                 Filter: (tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer)
                                 Rows Removed by Filter: 300017
                                 Buffers: shared hit=1954 read=3562 written=23
                           ->  Hash (actual time=65.937..65.939 rows=18884 loops=1)
                                 Buckets: 32768  Batches: 1  Memory Usage: 994kB
                                 Buffers: shared hit=6369 read=4 written=4
                                 ->  Seq Scan on orders o (actual time=0.009..61.784 rows=18884 loops=1)
                                       Filter: ((status <> 'cancelled'::text) AND (placed_at >= (now() - '90 days'::interval)) AND (tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer))
                                       Rows Removed by Filter: 281137
                                       Buffers: shared hit=6369 read=4 written=4
                     ->  Hash (actual time=0.670..0.670 rows=1003 loops=1)
                           Buckets: 1024  Batches: 1  Memory Usage: 62kB
                           Buffers: shared hit=25
                           ->  Seq Scan on products p (actual time=0.010..0.496 rows=1003 loops=1)
                                 Filter: (tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer)
                                 Rows Removed by Filter: 1002
                                 Buffers: shared hit=25
 Planning:
   Buffers: shared hit=331
 Planning Time: 1.141 ms
 Execution Time: 372.362 ms
```
**3. CHANGE (two parts, each tested separately):**
- `CREATE INDEX idx_orders_tenant_placed ON orders (tenant_id, placed_at DESC) INCLUDE (status, total_cents)` — the tenant+time filter scanned all 300k orders and discarded 280k. On its own this cut the `orders` scan from ~52 ms to ~16 ms but total time barely moved (283 → 272 ms) because `order_items` dominated.
- `ALTER DATABASE … SET random_page_cost = 1.1`. The default (4.0) models spinning disks, so the planner thought 19k index lookups into `order_items` were more expensive than reading all 750k rows. With an SSD-realistic value it switches to primary-key lookups (200 → 116 ms in an isolated test).
- **Tested and rejected:** a covering index `order_items(order_id) INCLUDE (product_id, qty, unit_price_cents)` — with it, 252 ms (no gain on its own) and ~102 ms combined with the cost change, versus ~116 ms without it. A ~12% gain doesn't justify the write cost and storage of an index on the largest table.

**4. AFTER PLAN** (`119.900 ms`)
```
 Limit (actual time=119.753..119.758 rows=10 loops=1)
   Buffers: shared hit=98074 read=12
   ->  Sort (actual time=119.752..119.755 rows=10 loops=1)
         Sort Key: (((sum((oi.qty * oi.unit_price_cents)))::numeric / 100.0)) DESC
         Sort Method: top-N heapsort  Memory: 26kB
         Buffers: shared hit=98074 read=12
         ->  HashAggregate (actual time=119.200..119.441 rows=1003 loops=1)
               Group Key: p.product_id
               Batches: 1  Memory Usage: 193kB
               Buffers: shared hit=98071 read=12
               ->  Hash Join (actual time=0.448..109.384 rows=47399 loops=1)
                     Hash Cond: (oi.product_id = p.product_id)
                     Buffers: shared hit=98071 read=12
                     ->  Nested Loop (actual time=0.036..98.366 rows=47399 loops=1)
                           Buffers: shared hit=97877 read=6
                           ->  Index Scan using idx_orders_tenant_placed on orders o (actual time=0.020..18.158 rows=18884 loops=1)
                                 Index Cond: ((tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer) AND (placed_at >= (now() - '90 days'::interval)))
                                 Filter: (status <> 'cancelled'::text)
                                 Rows Removed by Filter: 3132
                                 Buffers: shared hit=22150
                           ->  Index Scan using order_items_pkey on order_items oi (actual time=0.003..0.004 rows=3 loops=18884)
                                 Index Cond: (order_id = o.order_id)
                                 Filter: (tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer)
                                 Buffers: shared hit=75727 read=6
                     ->  Hash (actual time=0.397..0.398 rows=1003 loops=1)
                           Buckets: 1024  Batches: 1  Memory Usage: 62kB
                           Buffers: shared hit=194 read=6
                           ->  Index Scan using products_tenant_id_sku_key on products p (actual time=0.023..0.218 rows=1003 loops=1)
                                 Index Cond: (tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer)
                                 Buffers: shared hit=194 read=6
 Planning:
   Buffers: shared hit=360 read=7
 Planning Time: 1.137 ms
 Execution Time: 119.900 ms
```
**5. RESULT:** `Seq Scan on order_items (450k rows read)` → `Index Scan using order_items_pkey (3 rows × 18.9k loops)`; `Seq Scan on orders` → `Index Scan using idx_orders_tenant_placed`. 283–372.362 ms → 119.900 ms. **Still ~120 ms** — it aggregates ~47k line items, so the next step is not another index but caching: see `03_nosql.md` (cached read: ~3 ms).

---

## Q2 — Monthly revenue
**1. QUERY:** revenue and order count per month for the last 12 months, settled orders only.
```sql
-- Q2: Monthly revenue and order count for the last 12 months (tenant 1, settled orders).
SELECT date_trunc('month', placed_at) AS month, count(*) AS orders, sum(total_cents)/100.0 AS revenue
FROM orders
WHERE status IN ('paid','shipped','delivered')
  AND placed_at >= now() - interval '12 months'
GROUP BY 1 ORDER BY 1;
```
**2. BEFORE PLAN** (`125.039 ms`)
```
 GroupAggregate (actual time=113.838..124.745 rows=13 loops=1)
   Group Key: (date_trunc('month'::text, placed_at))
   Buffers: shared hit=6376
   ->  Sort (actual time=112.675..118.602 rows=63858 loops=1)
         Sort Key: (date_trunc('month'::text, placed_at))
         Sort Method: quicksort  Memory: 4031kB
         Buffers: shared hit=6376
         ->  Seq Scan on orders (actual time=0.017..100.153 rows=63858 loops=1)
               Filter: ((status = ANY ('{paid,shipped,delivered}'::text[])) AND (placed_at >= (now() - '1 year'::interval)) AND (tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer))
               Rows Removed by Filter: 236163
               Buffers: shared hit=6373
 Planning:
   Buffers: shared hit=205
 Planning Time: 0.533 ms
 Execution Time: 125.039 ms
```
**3. CHANGE:** same covering index as Q1 — `INCLUDE (status, total_cents)` means every column the query needs lives in the index.
**4. AFTER PLAN** (`49.273 ms`)
```
 GroupAggregate (actual time=39.726..49.011 rows=13 loops=1)
   Group Key: (date_trunc('month'::text, placed_at))
   Buffers: shared hit=575
   ->  Sort (actual time=38.934..42.544 rows=63858 loops=1)
         Sort Key: (date_trunc('month'::text, placed_at))
         Sort Method: quicksort  Memory: 4031kB
         Buffers: shared hit=575
         ->  Index Only Scan using idx_orders_tenant_placed on orders (actual time=0.043..28.507 rows=63858 loops=1)
               Index Cond: ((tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer) AND (placed_at >= (now() - '1 year'::interval)))
               Filter: (status = ANY ('{paid,shipped,delivered}'::text[]))
               Rows Removed by Filter: 25846
               Heap Fetches: 0
               Buffers: shared hit=572
 Planning:
   Buffers: shared hit=243
 Planning Time: 0.796 ms
 Execution Time: 49.273 ms
```
**5. RESULT:** `Seq Scan` → `Index Only Scan`, `Heap Fetches: 0` (table was vacuumed so the visibility map is current). 125.039 → 49.273 ms. *Caveat:* index-only scans depend on autovacuum keeping the visibility map fresh on a busy table.

---

## Q3 — Login lookup (the interesting one)
**1. QUERY:** find a customer by email, case-insensitively.
```sql
-- Q3: Case-insensitive login lookup by email (tenant 1).
SELECT customer_id, password_hash FROM customers WHERE lower(email) = lower('USER12345@T1.EXAMPLE');
```
**2. BEFORE PLAN** (`16.998 ms`)
```
 Seq Scan on customers (actual time=5.240..16.979 rows=1 loops=1)
   Filter: ((tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer) AND (lower(email) = 'user12345@t1.example'::text))
   Rows Removed by Filter: 50002
   Buffers: shared hit=960
 Planning:
   Buffers: shared hit=136
 Planning Time: 0.448 ms
 Execution Time: 16.998 ms
```
**3. CHANGE:** The obvious fix is a functional index on `lower(email)`. I built it first — **and PostgreSQL ignored it** (same Seq Scan). Running the identical query as a superuser (no RLS) *did* use it (0.05 ms). Root cause: `lower()` is **not LEAKPROOF** (`pg_proc.proleakproof = f`), so under row-level security the planner must not evaluate it before the policy qual and therefore cannot use it as an index condition. Fix: stop calling a function on the column. V6 adds `CHECK (email = lower(email))` (emails are normalised on write) and the query becomes `WHERE email = lower($1)` — `lower()` now applies to the *parameter*, and the existing `UNIQUE (tenant_id, email)` index serves it. No new index.
```sql
-- Q3 (rewritten): emails are stored lower-case (CHECK constraint), so compare the column directly.
SELECT customer_id, password_hash FROM customers WHERE email = lower('USER12345@T1.EXAMPLE');
```
**4. AFTER PLAN** (`0.051 ms`)
```
 Index Scan using customers_tenant_id_email_key on customers (actual time=0.018..0.019 rows=1 loops=1)
   Index Cond: ((tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer) AND (email = 'user12345@t1.example'::text))
   Buffers: shared hit=4
 Planning:
   Buffers: shared hit=139
 Planning Time: 0.542 ms
 Execution Time: 0.051 ms
```
**5. RESULT:** `Seq Scan (50k rows)` → `Index Scan using customers_tenant_id_email_key`; 16.998 → 0.051 ms. Lesson: *test performance as the real application role* — benchmarking as superuser would have hidden the problem.

---

## Q4 — Customer order history (no change)
**1. QUERY:** a customer's 20 most recent orders.
```sql
-- Q4: A customer's 20 most recent orders.
SELECT order_id, status, total_cents, placed_at FROM orders
WHERE customer_id = 12345 ORDER BY placed_at DESC LIMIT 20;
```
**2. BEFORE PLAN** (`0.144 ms`)
```
 Limit (actual time=0.115..0.118 rows=6 loops=1)
   Buffers: shared hit=15
   ->  Sort (actual time=0.114..0.116 rows=6 loops=1)
         Sort Key: placed_at DESC
         Sort Method: quicksort  Memory: 25kB
         Buffers: shared hit=15
         ->  Index Scan using idx_orders_customer on orders (actual time=0.043..0.078 rows=6 loops=1)
               Index Cond: (customer_id = 12345)
               Filter: (tenant_id = (NULLIF(current_setting('app.tenant_id'::text, true), ''::text))::integer)
               Buffers: shared hit=12
 Planning:
   Buffers: shared hit=155
 Planning Time: 0.523 ms
 Execution Time: 0.144 ms
```
**3. CHANGE:** none. A composite `(customer_id, placed_at DESC)` index would remove the tiny sort, but each customer has ~6 orders, so the benefit is unmeasurable (0.14 → 0.13 ms). Not added: every index slows writes and I only add indexes that evidence justifies. Revisit if customers accumulate hundreds of orders.
**4. AFTER PLAN:** unchanged (`0.131 ms` after V6).
**5. RESULT:** baseline V2 foreign-key index was already optimal; the correct optimisation here was to do nothing.
