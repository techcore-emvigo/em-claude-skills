# Security Gap Checklist — Extended Part 2

Covers: data protection, communication security, file upload, error/logging,
database security, system config, dynamic include, memory/resource management.

---

## 15. Access Control Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No global auth guard | Routes with `@UseGuards` per-route only — easy to forget | 🔴 |
| Role from request body/query | `req.body.role`, `req.query.isAdmin` used for auth decisions | 🔴 |
| IDOR — no ownership check | `repo.findById(req.params.id)` with no `userId === user.id` check | 🔴 |
| Returns 403 on IDOR (confirms existence) | `throw new ForbiddenException()` — use 404 instead | 🟠 |
| Auth fails open on exception | `catch(e) { return true; }` in canActivate/canAccess | 🔴 |
| Admin logic in regular service | Privilege-escalating code mixed with normal service logic | 🟠 |
| `Referer` header as sole auth check | `if (req.headers.referer.includes('admin'))` | 🔴 |
| No per-user rate limit | High-value actions with no per-user transaction counter | 🟠 |
| No session re-validation | Long-lived sessions never re-check if user is still active/authorized | 🟠 |
| Account disable doesn't revoke sessions | User can keep using active sessions after account is disabled | 🔴 |
| No inactive account cleanup | No job to disable accounts unused for 30+ days | 🟠 |
| DB user over-privileged | App DB user has `ALTER`, `DROP`, or `SUPERUSER` privileges | 🔴 |
| Single DB user for all roles | Same credentials for app queries and migrations | 🟠 |
| IAM wildcard permissions | `Action: "*"` or `Resource: "*"` on a service account | 🔴 |
| Client-side state not integrity-checked | Role/permissions in localStorage or unverified client-side state | 🔴 |

---

## 16. Data Protection & Communication Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| PII in HTTP GET params | `?email=`, `?ssn=`, `?token=` in URL | 🔴 |
| PII columns unencrypted at rest | `ssn`, `dob`, `card_number` stored as plain text in DB | 🔴 |
| Connection string in source | `postgresql://user:pass@host` committed to repo | 🔴 |
| Sensitive page cached | No `Cache-Control: no-store` on authenticated pages | 🟠 |
| Autocomplete on sensitive form | Password/card fields without `autocomplete="off"` | 🟠 |
| Revealing source comments | `// HACK`, `// backdoor`, `// DB password is` in production code | 🟠 |
| No data erasure support | GDPR erasure request cannot be fulfilled | 🟠 |
| Temp files not purged | Uploaded/generated files with no expiry or cleanup | 🟠 |
| TLS verification disabled | `rejectUnauthorized: false` on any HTTPS client | 🔴 |
| HTTP without HTTPS redirect | Port 80 accessible with no redirect to 443 | 🟠 |
| TLS version < 1.2 | `minVersion: 'TLSv1'` or `TLSv1.1` in any config | 🔴 |
| Server version in headers | `Server: nginx/1.18.0` or `X-Powered-By: Express` in responses | 🟡 |
| Directory listing enabled | Web server returns directory index for a path | 🟠 |
| Unnecessary HTTP methods | `TRACE`, `WebDAV PROPFIND`, `CONNECT` not blocked | 🟡 |

---

## 17. File Upload & File Management Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| File type check by extension only | `path.extname(file.name) === '.jpg'` — no magic byte check | 🔴 |
| No file size limit | Multer/busboy with no `limits.fileSize` | 🟠 |
| User-supplied filename used directly | `fs.writeFile(req.file.originalname, ...)` | 🔴 |
| File stored in web server root | `public/uploads/` — directly HTTP-accessible | 🔴 |
| No malware scan on upload | Uploaded files not scanned before storage | 🟠 |
| Absolute path returned to client | Response includes `/var/www/uploads/real.jpg` | 🟠 |
| Executable file types not blocked | `.php`, `.exe`, `.sh`, `.bat` allowed in upload MIME list | 🔴 |
| No auth on upload endpoint | File upload handler missing `@UseGuards(JwtAuthGuard)` | 🔴 |
| File served from app server | Static uploads served by Node/Python — not CDN/S3 | 🟡 |
| Directory path in API response | API returns folder structure or relative paths | 🟠 |

---

## 18. Error Handling & Logging Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Stack trace in API response | `res.json({ error: err.stack })` or `message: error.toString()` | 🔴 |
| System details in error message | DB table names, file paths, framework version in error body | 🟠 |
| Session ID in error response | Error object contains `sessionId` or `token` field | 🔴 |
| Input validation failures not logged | Zod/Joi validation error with no log entry | 🟠 |
| Auth failures not logged | Failed login with no `logger.warn` entry | 🟠 |
| Access control failures not logged | 403/404 IDOR with no audit log entry | 🟠 |
| Admin actions not logged | Role change, account disable with no audit entry | 🔴 |
| TLS connection failure not logged | Outbound HTTPS error caught and swallowed silently | 🟠 |
| Log injection possible | User input written to logs without newline stripping | 🟠 |
| No log integrity mechanism | Logs stored without tamper-detection (no hash chaining) | 🟡 |
| Logs accessible to all users | Log files with world-readable permissions | 🟠 |

---

## 19. Database Security Gaps (Extended)

| Gap | What to Look For | Severity |
|---|---|---|
| SQL string concatenation | `"SELECT ... WHERE id = " + userId` or f-string SQL | 🔴 |
| Meta-characters not handled | SQL with `<>"\'%&+` in user input without parameterisation | 🔴 |
| DB connection string in source | Any `postgresql://`, `mysql://`, `mongodb://` literal in source | 🔴 |
| Default admin password unchanged | DB `postgres`, `root`, `sa` using default or weak password | 🔴 |
| Default vendor accounts active | Unused vendor-supplied DB accounts still enabled | 🔴 |
| DB port open to internet | Port 3306/5432/1433 not restricted to VPC/private network | 🔴 |
| Unnecessary DB features enabled | `xp_cmdshell`, `pg_read_file`, unneeded stored procs active | 🟠 |
| Sample schemas in production | Vendor sample databases/schemas present | 🟡 |
| Single DB user for all purposes | One credential for app queries, migrations, and reporting | 🟠 |
| DB connection not closed promptly | Connection held open across async gaps or after use | 🟠 |
| No stored procedure abstraction | App user has direct table INSERT/DELETE — no proc boundary | 🟡 |
| Variables not strongly typed in DB | Dynamic SQL built from weakly typed inputs | 🟠 |

---

## 20. System Configuration & Environment Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Test/debug code in production build | `DEBUG=true`, test-only routes, mock data in prod | 🔴 |
| Dev environment connected to prod resources | Dev `.env` points at production DB or services | 🔴 |
| Outdated framework or server version | `express@3`, `node@14`, `nginx/1.10` — no security patches | 🔴 |
| Unpatched CVEs in server components | Known vulnerability in server-side framework version | 🔴 |
| Directory listing enabled | Web server returns `Index of /uploads/` for a directory | 🟠 |
| WebDAV / TRACE / CONNECT enabled | HTTP methods that aren't used by the app left active | 🟡 |
| Server version in response headers | `Server: nginx/1.18.0` or `X-Powered-By: Express` | 🟡 |
| Directory structure in robots.txt | Individual paths disallowed instead of parent directory | 🟡 |
| No software change control | Changes deployed without PR, review, or change log | 🟠 |
| Dev environment on production network | Dev instances can reach production services | 🔴 |
| No asset/component inventory | No SBOM or tracked list of deployed software versions | 🟡 |
| Security config not human-readable | Config stored in binary or encrypted-only format — can't audit | 🟡 |

---

## 21. Dynamic Include & Code Generation Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| User-controlled module path | `require(req.query.mod)`, `import(req.body.path)` | 🔴 |
| User-controlled template structure | `engine.render(req.body.template, data)` | 🔴 |
| `res.render` / `res.sendFile` from user param | Direct path from request used in render or file serve | 🔴 |
| Template injection vector | User data in Nunjucks/Jinja2/Pebble without escaping | 🔴 |
| User can write to source directories | Upload or write endpoint that can reach source code path | 🔴 |
| Update downloaded over HTTP | Auto-update fetching from `http://` URL | 🔴 |
| No signature verification on update | Package applied without RSA/ECDSA signature check | 🔴 |
| No checksum on downloaded dependency | File applied without SHA-256 hash verification | 🔴 |

---

## 22. Memory & Resource Management Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| DB connection not released in finally | `conn.query()` with no `finally { conn.release() }` | 🔴 |
| File stream not closed on error | `fs.createReadStream` with no `.destroy()` in finally | 🟠 |
| Unbounded user input to buffer/string op | User string concatenated without `.max()` / `.substring()` truncation | 🟠 |
| Uninitialised variable used | Variable declared without default, used inside conditional path | 🟠 |
| TOCTOU race — check then act | `findById` + update with no DB-level atomic operation or lock | 🔴 |
| Shared state mutated without lock | Module-level variable written by concurrent async functions | 🟠 |
| Float arithmetic on money values | `price * quantity` producing `19.9999...` without rounding | 🟠 |
| `parseInt` result not validated | `parseInt(req.query.x)` used without `isNaN` / `isFinite` check | 🟠 |
| `eval()` or `new Function()` with user input | Dynamic code execution from user-supplied string | 🔴 |
| OS exec with user input | `exec(\`cmd ${req.query.param}\`)` — command injection | 🔴 |
| `require()` / `import()` from user value | Dynamic module load controlled by request data | 🔴 |
