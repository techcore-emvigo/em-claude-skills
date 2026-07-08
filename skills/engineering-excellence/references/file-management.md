# File Upload Security

Covers: magic-byte validation, size limits, safe filenames, storage outside web root,
execution permission removal, malware scanning, no absolute path to client.

---

# File Management & Upload Security

Covers: file type validation (magic bytes), size limits, safe storage location,
malware scanning, path traversal prevention, index mapping, signed URLs.

---

## 10. File Upload Security

```typescript
// ✅ Good — comprehensive file upload validation
import path    from 'path';
import crypto  from 'crypto';
import fileType from 'file-type'; // inspects magic bytes, not just extension
import ClamAV  from 'clamav.js';  // antivirus scanning

const ALLOWED_MIME_TYPES = new Set([
  'image/jpeg', 'image/png', 'image/webp', 'image/gif',
  'application/pdf',
]);

const MAX_FILE_SIZE_BYTES = 10 * 1024 * 1024; // 10MB

// ✅ File upload endpoint
@Post('uploads')
@UseGuards(JwtAuthGuard)
@UseInterceptors(FileInterceptor('file', {
  limits: { fileSize: MAX_FILE_SIZE_BYTES },
  fileFilter: (req, file, cb) => {
    // 1. Check declared MIME type (spoofable — we also check magic bytes below)
    if (!ALLOWED_MIME_TYPES.has(file.mimetype)) {
      return cb(new BadRequestException('File type not allowed'), false);
    }
    cb(null, true);
  },
}))
async uploadFile(
  @UploadedFile() file: Express.Multer.File,
  @CurrentUser() user: AuthUser,
): Promise<{ fileId: string }> {

  // 2. Verify actual file type by magic bytes (not just extension or declared MIME)
  const detectedType = await fileType.fromBuffer(file.buffer);
  if (!detectedType || !ALLOWED_MIME_TYPES.has(detectedType.mime)) {
    throw new BadRequestException('File content does not match declared type');
  }

  // 3. Scan for malware
  const scanResult = await this.clamav.scanBuffer(file.buffer);
  if (scanResult.isInfected) {
    logger.warn('Malware detected in upload', { userId: user.id, virus: scanResult.viruses });
    throw new BadRequestException('File failed security scan');
  }

  // 4. Generate safe filename — never use user-supplied filename
  const extension  = detectedType.ext;
  const safeFileId = crypto.randomBytes(16).toString('hex');
  const safeKey    = `uploads/${user.id}/${safeFileId}.${extension}`;
  // ✅ Never reveal the storage path or absolute file path to client
  // ✅ Store outside web context — S3/Cloud Storage, not the web server root

  await this.storageService.upload(safeKey, file.buffer, detectedType.mime);

  return { fileId: safeFileId }; // return opaque ID — not the path
}

// ✅ Good — serve files via signed URL (never absolute path)
@Get('uploads/:fileId')
@UseGuards(JwtAuthGuard)
async getUpload(@Param('fileId') fileId: string, @CurrentUser() user: AuthUser) {
  // Verify ownership
  const upload = await this.uploadRepo.findByFileId(fileId);
  if (!upload || upload.userId !== user.id) {
    throw new NotFoundException();
  }

  // Return signed URL (expires in 15 minutes) — never the absolute path
  const signedUrl = await this.storageService.getSignedUrl(upload.storageKey, 900);
  return { url: signedUrl };
  // ❌ Never: return { path: '/var/www/uploads/real-path.jpg' }
}
```

---


---

## 10. File Upload Security

```typescript
// ✅ Good — comprehensive file upload validation
import path    from 'path';
import crypto  from 'crypto';
import fileType from 'file-type'; // inspects magic bytes, not just extension
import ClamAV  from 'clamav.js';  // antivirus scanning

const ALLOWED_MIME_TYPES = new Set([
  'image/jpeg', 'image/png', 'image/webp', 'image/gif',
  'application/pdf',
]);

const MAX_FILE_SIZE_BYTES = 10 * 1024 * 1024; // 10MB

// ✅ File upload endpoint
@Post('uploads')
@UseGuards(JwtAuthGuard)
@UseInterceptors(FileInterceptor('file', {
  limits: { fileSize: MAX_FILE_SIZE_BYTES },
  fileFilter: (req, file, cb) => {
    // 1. Check declared MIME type (spoofable — we also check magic bytes below)
    if (!ALLOWED_MIME_TYPES.has(file.mimetype)) {
      return cb(new BadRequestException('File type not allowed'), false);
    }
    cb(null, true);
  },
}))
async uploadFile(
  @UploadedFile() file: Express.Multer.File,
  @CurrentUser() user: AuthUser,
): Promise<{ fileId: string }> {

  // 2. Verify actual file type by magic bytes (not just extension or declared MIME)
  const detectedType = await fileType.fromBuffer(file.buffer);
  if (!detectedType || !ALLOWED_MIME_TYPES.has(detectedType.mime)) {
    throw new BadRequestException('File content does not match declared type');
  }

  // 3. Scan for malware
  const scanResult = await this.clamav.scanBuffer(file.buffer);
  if (scanResult.isInfected) {
    logger.warn('Malware detected in upload', { userId: user.id, virus: scanResult.viruses });
    throw new BadRequestException('File failed security scan');
  }

  // 4. Generate safe filename — never use user-supplied filename
  const extension  = detectedType.ext;
  const safeFileId = crypto.randomBytes(16).toString('hex');
  const safeKey    = `uploads/${user.id}/${safeFileId}.${extension}`;
  // ✅ Never reveal the storage path or absolute file path to client
  // ✅ Store outside web context — S3/Cloud Storage, not the web server root

  await this.storageService.upload(safeKey, file.buffer, detectedType.mime);

  return { fileId: safeFileId }; // return opaque ID — not the path
}

// ✅ Good — serve files via signed URL (never absolute path)
@Get('uploads/:fileId')
@UseGuards(JwtAuthGuard)
async getUpload(@Param('fileId') fileId: string, @CurrentUser() user: AuthUser) {
  // Verify ownership
  const upload = await this.uploadRepo.findByFileId(fileId);
  if (!upload || upload.userId !== user.id) {
    throw new NotFoundException();
  }

  // Return signed URL (expires in 15 minutes) — never the absolute path
  const signedUrl = await this.storageService.getSignedUrl(upload.storageKey, 900);
  return { url: signedUrl };
  // ❌ Never: return { path: '/var/www/uploads/real-path.jpg' }
}
```

---

## Gap Detection Table

### Data Protection Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Sensitive data in GET params | `/search?email=x`, `/verify?token=x`, `/reset?password=x` in URLs | 🔴 |
| PII stored unencrypted at rest | `ssn`, `dob`, `cardNumber` columns with no encryption layer | 🔴 |
| Connection string hardcoded | `postgresql://user:pass@host/db` literal in source code | 🔴 |
| No cache-control on auth pages | Authenticated pages missing `no-store` header | 🟠 |
| Autocomplete on sensitive form | Password/SSN/card forms without `autocomplete="off"` | 🟠 |
| Revealing comments in production | `// HACK:`, `// backdoor`, `// DB table:` in committed code | 🟠 |
| No data erasure mechanism | User account deletion that doesn't anonymise PII | 🟠 |
| Temp files not purged | Temporary sensitive files with no expiry/cleanup job | 🟠 |
| Absolute file path returned | API response contains `/var/www/uploads/real.jpg` | 🟠 |

### Communication Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| TLS downgrade allowed | `rejectUnauthorized: false` on any outbound HTTPS client | 🔴 |
| HTTP redirect missing | Service on port 80 with no redirect to 443 | 🟠 |
| `X-Powered-By` header present | `X-Powered-By: Express` in response headers | 🟡 |
| TLS < 1.2 allowed | `minVersion: 'TLSv1'` in TLS configuration | 🔴 |
| Directory listing enabled | Web server returns file list for a directory URL | 🟠 |
| Unnecessary HTTP methods allowed | `TRACE`, `OPTIONS`, `CONNECT`, `WebDAV` not disabled | 🟡 |
| Server version in headers | `Server: nginx/1.18.0` reveals exact version | 🟡 |

### File Upload Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| File type by extension only | `if (file.ext === 'jpg')` without magic byte check | 🔴 |
| No file size limit | Upload with no `limits: { fileSize }` configuration | 🟠 |
| User-supplied filename used | `fs.writeFile(file.originalname, ...)` — path traversal risk | 🔴 |
| File stored in web root | Uploads saved to `public/uploads/` — directly web-accessible | 🔴 |
| No malware scan | Uploaded files stored without antivirus check | 🟠 |
| Absolute path returned to client | Response includes server filesystem path | 🟠 |
| No auth on file upload | Upload endpoint with no authentication guard | 🔴 |
| Executable files not blocked | `.php`, `.exe`, `.sh` files not in blocked list | 🔴 |

---


---

## 11. Stored Procedures — Abstracted Data Access

Use stored procedures or repository pattern to abstract DB access. This allows permissions
to be granted on procedures only, removing direct table access from the app user entirely.

```sql
-- ✅ Good — stored procedure abstracts data access
-- Create the procedure (in migration):
CREATE OR REPLACE PROCEDURE create_order(
  p_user_id    BIGINT,
  p_items      JSONB,
  p_total      DECIMAL(10,2),
  OUT p_order_id BIGINT
)
LANGUAGE plpgsql AS $$
BEGIN
  INSERT INTO orders (user_id, items, total_amount, status, created_at)
  VALUES (p_user_id, p_items, p_total, 'pending', NOW())
  RETURNING id INTO p_order_id;
END;
$$;

-- Grant EXECUTE on procedure, not INSERT on table
GRANT EXECUTE ON PROCEDURE create_order TO app_user;
-- REVOKE INSERT ON orders FROM app_user; -- optional: app can't write directly
```

```typescript
// ✅ Good — TypeScript repository calling stored procedure
// repositories/order.repository.ts
@Injectable()
export class OrderRepository implements IOrderRepository {

  constructor(@InjectPool() private pool: Pool) {}

  async createOrder(
    userId:  number,
    items:   OrderItemDto[],
    total:   number,
  ): Promise<number> {
    const result = await this.pool.query<{ p_order_id: number }>(
      'CALL create_order($1, $2, $3, NULL)',
      [userId, JSON.stringify(items), total],
    );
    return result.rows[0].p_order_id;
  }

  // ✅ Read-only queries use readonly user — parameterised always
  async findById(id: number): Promise<Order | null> {
    const result = await this.readonlyPool.query<Order>(
      'SELECT id, user_id, status, total_amount, created_at FROM orders WHERE id = $1',
      [id],  // parameterised — never interpolated
    );
    return result.rows[0] ?? null;
  }
}
```

---


---

## 14. Database Surface-Area Reduction

```sql
-- ✅ Good — disable unnecessary database features (PostgreSQL example)

-- 1. Disable dangerous built-in functions if not needed
REVOKE EXECUTE ON FUNCTION pg_read_file(text) FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION pg_ls_dir(text)   FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION pg_write_file(text, bytea) FROM PUBLIC;

-- 2. Remove sample/default schemas and data
DROP SCHEMA IF EXISTS pg_toast_temp_1 CASCADE;  -- temp schemas from old sessions
-- Remove unused vendor-installed extensions
DROP EXTENSION IF EXISTS plpython3u;  -- untrusted PL — code execution risk if unused

-- 3. Disable default accounts not needed for business
-- In PostgreSQL: disable the default 'postgres' superuser remote login
-- pg_hba.conf: host all postgres 0.0.0.0/0 reject

-- 4. Use separate credentials for each trust level
-- app_user:      SELECT, INSERT, UPDATE, DELETE on business tables
-- report_user:   SELECT on reporting views only
-- migration_user: DDL permissions — used only by CI/CD pipeline, never at runtime
-- admin_user:    Full access — only for DBAs via MFA-gated bastion

-- ✅ Good — application connects with app_user (minimum privilege)
-- Migration runs with migration_user (DDL access)
-- Reporting runs with report_user (read-only)
```

```typescript
// ✅ Good — enforce parameterised queries with meta-character protection
// Always use typed parameters — never raw interpolation
// If parameterisation fails, do NOT run the command

function buildUserQuery(
  filters:  { status?: string; role?: string },
  sortCol:  string,
): { sql: string; params: unknown[] } {
  const ALLOWED_STATUS   = new Set(['active', 'inactive', 'pending']);
  const ALLOWED_SORT_COL = new Set(['name', 'email', 'created_at', 'last_login']);

  // ✅ Validate against whitelist before using in query
  if (filters.status && !ALLOWED_STATUS.has(filters.status)) {
    throw new ValidationError('status', 'Invalid status value');
  }
  if (!ALLOWED_SORT_COL.has(sortCol)) {
    throw new ValidationError('sort', 'Invalid sort column');
  }

  const params: unknown[] = [];
  const conditions: string[] = ['is_deleted = false'];

  if (filters.status) {
    params.push(filters.status);
    conditions.push(`status = $${params.length}`);
  }
  if (filters.role) {
    params.push(filters.role);
    conditions.push(`role = $${params.length}`);
  }

  // sortCol validated against whitelist above — safe to interpolate
  const sql = `SELECT id, name, email, status, role
               FROM users
               WHERE ${conditions.join(' AND ')}
               ORDER BY ${sortCol} ASC`;

  return { sql, params };
}
```

---


---

