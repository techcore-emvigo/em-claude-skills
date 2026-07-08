# NoSQL & Search DBs — Gap Detection

## MongoDB Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `collection.find(req.body)` | Unvalidated object passed as query filter — NoSQL injection | 🔴 |
| No field projection | `.find({})` or `.findOne({})` with no `{ field: 1 }` projection | 🟠 |
| Missing index on filter field | Fields used in `find({ status: ... })` without an index | 🔴 |
| Unbounded array in document | Array field that grows forever — 16MB doc limit | 🟠 |
| `COLLSCAN` on large collection | `explain()` shows `COLLSCAN` — no index used | 🔴 |
| No auth enabled | `--noauth` or no authentication configured | 🔴 |
| Port publicly accessible | Port 27017 open to internet | 🔴 |

## Firestore Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `allow read, write: if true` in rules | Security rules allow all access | 🔴 |
| No `request.auth != null` check | Rules without authentication requirement | 🔴 |
| No `.limit(n)` on query | Query without a result limit | 🟠 |
| Deep subcollection nesting | More than 2 levels of subcollections | 🟡 |
| Listener not unsubscribed | `onSnapshot` without calling the returned unsubscriber on cleanup | 🟠 |

## Redis Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No TTL on cached key | `redis.set(key, value)` without `EX` / `PX` — unbounded memory growth | 🟠 |
| `KEYS *` in production code | `redis.keys("*")` or `redis.keys("prefix:*")` — blocks event loop | 🔴 |
| New connection per request | `createClient().connect()` inside a handler — connection pool exhaustion | 🔴 |
| No `requirepass` / ACL | Redis accessible without authentication | 🔴 |
| Redis port publicly accessible | Port 6379 open to internet | 🔴 |
| No `maxmemory-policy` | No eviction policy set — OOM when memory fills | 🟠 |

## Elasticsearch Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Dynamic mapping in production | No explicit index mapping — field types guessed, can cause mapping explosion | 🟠 |
| `from/size` deep pagination | `from: 50000, size: 20` — degrades and has 10,000 result limit by default | 🟠 |
| Wildcard at start of pattern | `{ "wildcard": { "name": "*smith" } }` — very expensive | 🟠 |
| No auth / X-Pack security disabled | Elasticsearch accessible without credentials | 🔴 |
| Port publicly accessible | Port 9200 open to internet | 🔴 |

## Vector DB Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No tenant filter on query | Vector search query without `user_id`/`tenant_id` metadata filter — cross-tenant data leak | 🔴 |
| Storing PII in vector metadata | Email, name, or sensitive data in embedding metadata sent to third-party | 🔴 |
| No index on vector column (pgvector) | `vector` column with no `ivfflat` or `hnsw` index — exact search = slow | 🔴 |
| Dimension mismatch | Embedding model changed but vector column dimension not updated | 🔴 |

## Generation Checklist
- [ ] MongoDB: validate input before use as query filter; project only needed fields
- [ ] Firestore: security rules deny-by-default; `request.auth != null` on all rules
- [ ] Redis: always set TTL; use `SCAN` not `KEYS`; reuse connections
- [ ] All stores: auth enabled; ports not exposed publicly
- [ ] Vector DBs: tenant filter on every query; no PII in metadata
