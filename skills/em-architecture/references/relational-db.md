# Relational DBs (MySQL, PostgreSQL, MS SQL) — Gap Detection

## Query Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `SELECT *` | `SELECT *` in application queries | 🟠 |
| String interpolation in SQL | `f"SELECT ... WHERE id = {id}"`, `"SELECT ... WHERE id = " + id` | 🔴 |
| Missing `LIMIT` | Query with no result size constraint on a table that can grow | 🟠 |
| Offset pagination on large table | `OFFSET 10000 LIMIT 20` — degrades linearly | 🟡 |
| Function on indexed column | `WHERE LOWER(email) = ?`, `WHERE DATE(created_at) = ?` — index unused | 🟠 |
| `COUNT(*)` for existence check | `SELECT COUNT(*) ... WHERE ...` then `if count > 0` | 🟡 |
| Loop with single-row insert | `for item in items: db.execute("INSERT ...")` | 🟠 |
| `SELECT *` in ORM | `.findAll()`, `.find({})` with no field selection | 🟠 |

## Schema Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No index on foreign key | FK column with no index — slow joins and cascades | 🔴 |
| Missing index on filter column | Columns in `WHERE` / `ORDER BY` / `GROUP BY` with no index | 🔴 |
| `TIMESTAMP` without timezone (PG) | `TIMESTAMP` instead of `TIMESTAMPTZ` in PostgreSQL | 🟡 |
| `DATETIME` not UTC-enforced (MySQL) | `DATETIME` column without app-level UTC enforcement | 🟡 |
| Nullable FK with no meaning | FK column that is `NULL` but nullability isn't semantically meaningful | 🟡 |
| Unbounded array in document | JSONB column containing an array that grows without bound | 🟠 |

## Migration Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Manual `ALTER TABLE` in production | Schema change outside a versioned migration file | 🔴 |
| Migration with no rollback | `up` migration with no `down` counterpart | 🟠 |
| Non-idempotent migration | Migration not safe to run twice | 🟠 |
| `NOT NULL` without default on large table | Blocks table during migration (MySQL: full table lock) | 🟠 |

## Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| App user with DDL permissions | DB user can `DROP`, `CREATE`, `ALTER` — app user shouldn't | 🔴 |
| DB port publicly accessible | Port 3306 / 5432 / 1433 open to internet | 🔴 |
| PII stored unencrypted | Sensitive fields (SSN, card number) in plain columns | 🔴 |
| No query timeout set | No `statement_timeout` (PG) or `MAX_EXECUTION_TIME` (MySQL) | 🟠 |

## Generation Checklist
- [ ] Parameterized queries only — no string interpolation
- [ ] All FK columns indexed; filtered/sorted columns indexed
- [ ] `TIMESTAMPTZ` (PG) / UTC-enforced `DATETIME` (MySQL) for all timestamps  
- [ ] `LIMIT` on every query that could return many rows
- [ ] Cursor-based pagination: `WHERE id > last_seen_id ORDER BY id LIMIT N`
- [ ] Migrations: idempotent, with rollback, reviewed via `EXPLAIN ANALYZE`

---

## Release-Time Database Checklist

| Gap | What to Look For | Severity |
|---|---|---|
| New queries not reviewed for optimisation and indexes | Slow queries under production load | 🔴 |
| Database changes in release not explained (schema, indexes, queries) | Unreviewed changes break production | 🟠 |
| Migration scripts not up to date and tested in staging | Migration fails or corrupts data in production | 🔴 |
| Migration scripts not reviewed for data loss risk | Irreversible data loss | 🔴 |
| Database caching strategy not reviewed (Redis, query cache) | Repeated expensive queries hit DB unnecessarily | 🟠 |
| Database backups not automated and verified for recovery | Data loss with no recovery | 🔴 |
| Database load not monitored after release | Undetected DB saturation post-deploy | 🟠 |
| Database sharding/partitioning not considered if scale requires | DB becomes single bottleneck at scale | 🟡 |
