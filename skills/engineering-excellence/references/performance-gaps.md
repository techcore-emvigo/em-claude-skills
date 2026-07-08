# Performance Gap Checklist

Use this file during every review. Each gap below is a concrete pattern to scan for.
Performance gaps are often silent — they don't throw errors, they just slow everything down.

---

## 1. Database Query Performance

| Gap | What to Look For | Severity |
|---|---|---|
| N+1 queries | Loop containing `await db.find()`, `await repo.get()`, or any DB call per iteration | 🔴 |
| SELECT * | `SELECT *`, `find({})`, `.findAll()` with no field projection | 🟠 |
| Missing index | `WHERE`, `ORDER BY`, `JOIN ON` columns with no corresponding index | 🔴 |
| Unbounded query | `findAll()`, `SELECT * FROM table` with no `LIMIT` / pagination | 🟠 |
| Offset pagination on large tables | `OFFSET 50000 LIMIT 20` — degrades linearly | 🟡 |
| Function on indexed column | `WHERE LOWER(email) = ?`, `WHERE YEAR(created_at) = ?` — index not used | 🟠 |
| Lazy loading in a loop | ORM lazy-loading a relation inside a loop | 🔴 |
| Missing read transaction flag | Read-only queries without `readOnly: true` / `AsNoTracking()` | 🟡 |
| Large transaction holding lock | DB transaction wrapping a network/HTTP call | 🟠 |
| Bulk ops as single-row loops | `for item in items: db.insert(item)` — 1000 items = 1000 round trips | 🟠 |

**Fix patterns:**
- N+1: use `JOIN FETCH`, `include`, `DataLoader`, or batch query
- Missing index: add index on filtered/sorted/joined columns; verify with `EXPLAIN ANALYZE`
- Unbounded: always `LIMIT` + cursor-based pagination (`WHERE id > last_id`)
- Bulk: `INSERT INTO ... VALUES (...),(...)` or `bulkCreate` / `insertMany`

---

## 2. Caching Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Repeated identical DB query | Same query called on every request with no cache layer | 🟠 |
| No TTL on cache entry | `cache.set(key, value)` with no expiry — stale data or memory leak | 🟠 |
| Cache stampede risk | Multiple requests miss cache simultaneously, all hit DB at once | 🟡 |
| No cache on expensive computation | CPU-intensive or slow I/O operation called per request with static output | 🟠 |
| Missing HTTP cache headers | `ETag`, `Cache-Control`, `Last-Modified` absent on cacheable GET endpoints | 🟡 |
| Redis `KEYS *` in production | `redis.keys('*')` or `redis.keys('user:*')` — blocks event loop | 🔴 |

**Fix patterns:**
- Cache-aside: check cache → miss → load from DB → write to cache with TTL
- Use `SCAN` not `KEYS` in Redis
- Probabilistic early expiry to prevent stampede

---

## 3. Connection & Resource Management

| Gap | What to Look For | Severity |
|---|---|---|
| New DB connection per request | `new Pool()`, `createConnection()`, `MongoClient.connect()` inside a handler | 🔴 |
| New HTTP client per request | `new HttpService()`, `axios.create()`, `new HttpClient()` inside a handler | 🟠 |
| No connection pool limits | Pool with no `max` connections set — can exhaust DB connections under load | 🟠 |
| Unclosed connections / cursors | DB cursor, file handle, or stream opened but not closed in all code paths | 🔴 |
| No timeout on external call | HTTP/gRPC/DB call with no timeout — hangs indefinitely under failure | 🟠 |
| Socket exhaustion | `new HttpClient()` per call in a high-frequency path — OS port exhaustion | 🔴 |

**Fix patterns:**
- Initialise connections at module/service startup, reuse across requests
- Always set pool `min`, `max`, `idleTimeoutMillis`
- Always set HTTP `timeout`, DB `statement_timeout`, gRPC `deadline`

---

## 4. Async & Concurrency

| Gap | What to Look For | Severity |
|---|---|---|
| Sequential awaits for independent work | `await a(); await b();` when a and b don't depend on each other | 🟠 |
| Blocking sync I/O in async context | `fs.readFileSync`, `JSON.parse` on large payload, `execSync` in Node handler | 🟠 |
| Missing retry on transient failure | External call with no retry — one flaky response = user-facing error | 🟠 |
| Retry without backoff | Tight retry loop (`while retries < 3: call()`) — hammers the failing service | 🟠 |
| No jitter on retry | Fixed backoff intervals — causes thundering herd when many clients retry together | 🟡 |
| Promise not awaited | `asyncFn()` called without `await` — result ignored, errors swallowed | 🟠 |
| Unhandled promise rejection | `.then()` chain with no `.catch()`, or `async` function with no `try/catch` | 🟠 |
| CPU work blocking event loop | Heavy computation (sort, parse, transform) on main thread without worker | 🟠 |

**Fix patterns:**
- `Promise.all([a(), b()])` for independent async work
- `await queue.enqueue(task)` for heavy CPU work
- Backoff: `delay = min(base * 2^attempt + jitter, max_delay)`

---

## 5. Frontend Performance

| Gap | What to Look For | Severity |
|---|---|---|
| Unnecessary re-renders | State updates in parent re-rendering unchanged children | 🟠 |
| Missing `key` stability | `key={index}` on dynamic lists — React/Vue reconciliation breaks | 🟠 |
| Heavy computation in render | Expensive calculation inside JSX / template without memoization | 🟠 |
| Missing `useMemo` / `useCallback` | Function or value recreated every render, passed to child or dep array | 🟡 |
| Unvirtualized long list | Rendering 100+ items directly in DOM with no virtualization | 🟠 |
| Missing code splitting | All routes bundled in one JS file — slow initial load | 🟡 |
| Missing lazy image loading | Images below the fold without `loading="lazy"` | 🟡 |
| Full library import | `import _ from 'lodash'` — loads entire library for one function | 🟡 |
| useEffect with missing cleanup | Subscriptions, timers, event listeners not removed on unmount | 🟠 |
| No `srcset` / responsive images | Single large image served to all screen sizes | 🟡 |

**Fix patterns:**
- `React.memo`, `useMemo`, `useCallback` for memoization
- `React.lazy` + `Suspense` for route code splitting
- `react-virtual` or `react-window` for long lists

---

## 6. API & Payload Performance

| Gap | What to Look For | Severity |
|---|---|---|
| No pagination on list endpoint | API returns all records with no page/cursor limit | 🔴 |
| Returning full entity when subset needed | User object with 40 fields returned when only 3 are used | 🟡 |
| Missing response compression | Large JSON responses with no `gzip`/`brotli` compression | 🟡 |
| Synchronous call chain across services | Service A calls B calls C calls D in serial — latency adds up | 🟠 |
| No `ETag` / conditional GET | Cacheable resource fetched in full every time — no cache validation | 🟡 |
| GraphQL N+1 without DataLoader | Resolver fetching related entity per parent without batching | 🔴 |
| gRPC no streaming for large result | Large result set returned as single unary response — blocks until complete | 🟡 |

---

## 7. Memory & Resource Leaks

| Gap | What to Look For | Severity |
|---|---|---|
| Accumulating in-memory store | Array/Map growing without eviction: `cache.push(item)` in a loop | 🟠 |
| Event listener not removed | `addEventListener` / `emitter.on` without corresponding removal | 🟠 |
| Timer not cleared | `setInterval` without a `clearInterval` path | 🟠 |
| Stream not destroyed | Readable/writable stream left open on error path | 🟠 |
| Large buffer held in scope | Entire file read into memory: `fs.readFileSync` for large files | 🟠 |
| Unbounded queue / channel | In-memory queue with no max size — OOMs under backpressure | 🟠 |

**Fix patterns:**
- Use streams for large files: `fs.createReadStream()`
- Always clean up in `finally` blocks, `useEffect` return, or `defer`
- Use TTL-based eviction on in-memory caches (`node-cache`, `lru-cache`)

---

## 8. Infrastructure Performance

| Gap | What to Look For | Severity |
|---|---|---|
| Single-replica service | No redundancy — one crash = downtime | 🟠 |
| No HPA / auto-scaling | K8s deployment with no HorizontalPodAutoscaler | 🟡 |
| No resource limits on containers | K8s pod with no `resources.requests` / `resources.limits` | 🟠 |
| Synchronous queue consumer, single instance | One consumer process with no horizontal scaling path | 🟡 |
| No CDN for static assets | Static JS/CSS/images served directly from origin server | 🟡 |
| Cold start not mitigated (serverless) | Latency-sensitive Lambda/Cloud Function with no provisioned concurrency | 🟡 |

---

## 9. Load Testing & Continuous Performance Monitoring

| Gap | What to Look For | Severity |
|---|---|---|
| No load tests for pages/APIs in release | Pages/APIs not tested under realistic traffic before go-live | 🔴 |
| Load tests not simulating real-world traffic patterns | Synthetic tests miss real usage behaviour | 🟠 |
| No performance benchmarks defined (response time SLA) | No baseline to compare against; regressions go unnoticed | 🟠 |
| Page/API response time outside SLA limits | Slow endpoints shipped to production | 🔴 |
| No memory leak detection during load testing | Memory leaks cause production crashes under sustained load | 🔴 |
| No stress testing to find breaking point | System capacity unknown | 🟠 |
| Load test results not documented | No baseline for future regression comparison | 🟡 |
| No soak tests for resilience | Time-bomb issues (connection leaks, heap growth) undetected | 🟠 |
| Google Lighthouse not run on release pages | Performance, accessibility, SEO issues not caught pre-release | 🟠 |
| Lighthouse score below threshold (Performance < 80, Accessibility < 90) | Poor user experience shipped | 🟠 |
| Network waterfall not reviewed (bundle sizes, TTFB, render-blocking resources) | Root cause of slow loads unknown | 🟠 |
| Mobile Memory Profiler / MemLab not run | Memory leaks on device undetected | 🔴 |

**Tools:** k6, JMeter, Gatling, Locust, Artillery (load); Google Lighthouse (page perf); MemLab, Android Profiler, Xcode Instruments (mobile memory)
