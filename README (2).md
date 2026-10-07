# Capstone — Multi-Tenant E-Commerce (PostgreSQL 16 + Redis)

## Layout
```
docs/01_requirements.md          Day 1  requirements summary (1 page)
docs/02_er_diagram.md + .dbml    Day 1  ER diagram (Mermaid + dbdiagram.io source)
migrations/V1..V6__*.sql         Day 2  versioned schema (V6 = measured perf changes)
scripts/migrate.sh               Day 2  `clean migrate` runner (Flyway-style); flyway.conf for real Flyway
app/shop.py, app/demo.py         Day 3  Redis cart/cache + parameterised PG access
docs/03_nosql.md                 Day 3  justification
perf/*.sql, perf/run_bench.sh    Day 4  seed + queries + before/after harness; perf/results/ raw plans
docs/04_query_optimization.md    Day 4  QUERY/BEFORE/CHANGE/AFTER/RESULT for each query
scripts/backup.sh                Day 5  pg_dump -Fc + verified test restore
scripts/verify_security.sh       Day 5  20 security checks
docs/05_security_and_backup.md   Day 5  checklist + honest gaps
docs/06_presentation.md          Day 5  7-slide walkthrough with speaker notes
docs/evidence/                   captured output of every run
```

## Reproduce
```bash
createdb capstone
./scripts/migrate.sh clean migrate     # or: flyway clean migrate
./perf/run_bench.sh                    # rebuilds, loads ~1M rows, captures plans (~1 min)
cd app && pip install "psycopg[binary]" redis && python demo.py   # needs Redis on localhost:6379
./scripts/verify_security.sh && ./scripts/backup.sh
```
(Scripts that use `su postgres`-style local auth may need `PGHOST/PGUSER` set on your machine.)

## What was actually run vs. what you must do
- **Run & measured here:** all migrations from empty, bulk load, all EXPLAIN ANALYZE plans, Redis demo, 20 security checks, backup + restore.
- **Not run:** real Flyway (jar not downloadable in my sandbox) — `migrate.sh` replicates the behaviour; run `flyway clean migrate` locally to confirm. Timings come from a shared sandbox: expect different absolute numbers, same plan shapes.
- **Needs you:** *Day 1 design feedback.* Ask a peer/instructor to review the ER diagram **before** relying on the schema, using: (1) Is `tenant_id` denormalisation on child tables justified? (2) Should `inventory` be merged into `products`? (3) Is one payment-per-attempt row enough, or do refunds need their own table? (4) Any missing entity (addresses, shipments)? I made reasonable decisions on these, but they are unreviewed.
