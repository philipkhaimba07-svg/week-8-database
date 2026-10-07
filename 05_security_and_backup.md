# Security & Backup

Everything below is **verified by script**, not asserted: `scripts/verify_security.sh` (output in `docs/evidence/security_checklist_run.txt`, 20/20 checks pass) and `scripts/backup.sh` (`docs/evidence/backup_restore_run.txt`).

## Checklist
| # | Item | Status | How / evidence |
|---|---|---|---|
| 1 | Least-privilege roles, no app superuser | ✅ | `app_user` (DML on business tables only; no DDL, no DELETE on orders/payments/customers, no audit_log access) and `app_readonly` (SELECT only, **no `customers`** – PII). Both `NOSUPERUSER NOBYPASSRLS`. Tested: DROP, DELETE, audit read and PII read all denied. |
| 2 | RLS on every multi-tenant/sensitive table | ✅ | V4 enables RLS + `tenant_isolation` policy (`USING` and `WITH CHECK`) on all 8 tenant tables; a check fails the build if any table with `tenant_id` lacks RLS. Unset tenant → 0 rows (fail closed). Cross-tenant INSERT rejected; cross-tenant UPDATE affects 0 rows. |
| 3 | Sensitive fields hashed/encrypted | ✅ | Passwords: bcrypt (`crypt(..., gen_salt('bf',10))`), verified inside the DB. Phone: `pgp_sym_encrypt`. Audit snapshots strip both columns (tested). |
| 4 | Audit log on critical tables | ✅ | Triggers on customers, products, orders, payments → `audit_log` (table, op, pk, tenant, actor, old/new JSON). Actor comes from `app.user_id` (e.g. `customer:1`). Append-only: UPDATE/DELETE blocked by trigger. `SECURITY DEFINER` so the app role needs no access to the log. |
| 5 | Parameterised queries only | ✅ | All SQL in `app/shop.py` uses `%s` placeholders; tenant context via `set_config(%s)`. Script greps for f-string/`.format`/`+` inside `execute()` → 0 hits. (A grep is a smoke test, not a proof — code review still applies.) |
| 6 | Backup taken AND test restore performed | ✅ | `pg_dump -Fc` → 17 MB dump; restored into scratch DB `capstone_restore_test`; row counts of 6 tables identical to source; RLS present on restored DB (8 tables); scratch DB dropped. |

## Backup procedure
```bash
mkdir -p backups
pg_dump -Fc -f backups/capstone_$(date +%F).dump capstone      # the required command
./scripts/backup.sh                                            # same + automated test-restore & comparison
```
Schedule nightly with cron (`0 2 * * * /path/scripts/backup.sh`) and copy dumps off-host. A backup that has never been restored is a hope, not a backup — hence the restore step is part of the script.

## Known limitations / honest gaps
- **Demo secrets in repo:** role passwords (`change_me_*`) and the PGP key (`demo-key-change-me`) are placeholders. In production: roles created out-of-band, passwords from a secrets manager, encryption key held by the application/KMS — a key in SQL seed files is acceptable for a demo only.
- **Migrations run as the postgres superuser in this lab.** In production, use a dedicated non-superuser *owner* role for migrations; `SECURITY DEFINER` functions then run as that owner.
- **Cross-tenant FK integrity:** RLS prevents `app_user` creating cross-tenant references, but the schema itself doesn't forbid them (single-column FKs). Composite `(tenant_id, id)` FKs would.
- **Not covered:** TLS on connections, connection pooling (RLS + `set_config(..., true)` is safe with transaction-level pooling, *not* with session-level state), PITR/WAL archiving (pg_dump is a logical snapshot, RPO = time since last dump).
