# Input Validation & Output Encoding

Covers: server-side validation, centralised routine, character sets, whitelist approach,
hazardous characters, output encoding for HTML/URL/SQL contexts.

---

## 1. Input Validation

### Core Rules
- **All validation on a trusted system (server)** — client-side validation is UX only
- **Centralised validation routine** — one place, not scattered per endpoint
- **Whitelist approach** — allow only known-good characters; reject everything else
- **Validate type, range, length, format, and character set**
- **All validation failures result in rejection** — never silently sanitise and continue

### Centralised Validation Module

```typescript
// validation/input-validator.ts — ONE centralised validation entry point
import { z, ZodSchema, ZodError } from 'zod';

export class InputValidator {

  // Enforce UTF-8 encoding and strip null bytes before any validation
  static sanitizeEncoding(value: string): string {
    // Remove null bytes — common in injection attacks
    let sanitized = value.replace(/\x00/g, '');
    // Remove CRLF injection characters from header-bound fields
    sanitized = sanitized.replace(/[\r\n]/g, '');
    return sanitized;
  }

  // Validate path segments — prevent path traversal
  static validatePathSegment(value: string, basePath: string): string {
    const resolved = require('path').resolve(basePath, value);
    if (!resolved.startsWith(basePath)) {
      throw new ValidationError('PATH_TRAVERSAL', 'Invalid path: traversal attempt detected');
    }
    return resolved;
  }

  // Generic schema validation — used at every API boundary
  static validate<T>(schema: ZodSchema<T>, data: unknown): T {
    const result = schema.safeParse(data);
    if (!result.success) {
      throw new ValidationError('VALIDATION_FAILED', formatZodErrors(result.error));
    }
    return result.data;
  }
}

// ✅ Good — centralised schema with all validation rules
const CreateUserSchema = z.object({
  name:  z.string()
           .min(2,   'Name must be at least 2 characters')
           .max(100, 'Name must be at most 100 characters')
           .regex(/^[\p{L}\s\-'.]+$/u, 'Name contains invalid characters')  // unicode-aware whitelist
           .trim(),

  email: z.string()
           .email('Invalid email format')
           .max(254,  'Email exceeds maximum length')   // RFC 5321
           .toLowerCase(),

  phone: z.string()
           .regex(/^\+?[1-9]\d{7,14}$/, 'Invalid phone format (E.164)')  // whitelist digits and +
           .optional(),

  age:   z.number()
           .int('Age must be an integer')
           .min(0,   'Age cannot be negative')
           .max(150, 'Age out of valid range'),

  role:  z.enum(['member', 'moderator']),  // whitelist — never accept 'admin' from input
});
```

### Validate ALL Input Sources

```typescript
// ✅ Good — validate every input source, not just body

// 1. Request body
const body = InputValidator.validate(CreateUserSchema, req.body);

// 2. Query parameters — always typed, always bounded
const QuerySchema = z.object({
  page:   z.coerce.number().int().min(1).max(1000).default(1),
  limit:  z.coerce.number().int().min(1).max(100).default(20),
  status: z.enum(['active', 'inactive', 'pending']).optional(),
  search: z.string().max(200).trim().optional(),
});
const query = InputValidator.validate(QuerySchema, req.query);

// 3. URL path parameters
const ParamSchema = z.object({
  id: z.string().uuid('Invalid ID format'),   // enforce UUID — prevents sequential ID probing
});
const params = InputValidator.validate(ParamSchema, req.params);

// 4. HTTP headers
const HeaderSchema = z.object({
  'x-api-version':  z.string().regex(/^[0-9]+\.[0-9]+$/).optional(),
  'x-request-id':   z.string().max(64).optional(),
  // Only ASCII in headers — reject non-ASCII
}).catchall(z.string().regex(/^[\x20-\x7E]*$/, 'Header contains non-ASCII characters'));

// 5. Cookies
const CookieSchema = z.object({
  sessionId: z.string().min(32).max(128).regex(/^[a-zA-Z0-9_-]+$/),
});

// 6. Redirect parameters — ALWAYS validate redirect URLs
const RedirectSchema = z.object({
  returnUrl: z.string()
               .url()
               .refine(url => {
                 // Whitelist allowed redirect origins — prevent open redirect
                 const allowed = process.env.ALLOWED_REDIRECT_ORIGINS?.split(',') ?? [];
                 return allowed.some(origin => url.startsWith(origin));
               }, 'Redirect URL is not in the allowed list'),
});
```

### Hazardous Character Detection

```typescript
// ✅ Good — detect and reject or escape hazardous characters
const HAZARDOUS_CHARS = /[<>"'%()&+\\]/;

function containsHazardousChars(value: string): boolean {
  return HAZARDOUS_CHARS.test(value);
}

// For fields that MUST allow hazardous chars (e.g. address with & symbol):
// — Validate, then output-encode when rendering (see Output Encoding section)
// — Never allow raw user input to reach HTML, SQL, or OS commands

// ✅ Good — null byte and CRLF injection checks
function detectInjectionChars(value: string): void {
  if (value.includes('\x00')) {
    throw new ValidationError('NULL_BYTE', 'Input contains null byte');
  }
  if (/\r|\n/.test(value)) {
    throw new ValidationError('CRLF_INJECTION', 'Input contains newline characters');
  }
  // Path traversal
  if (value.includes('../') || value.includes('..\\')) {
    throw new ValidationError('PATH_TRAVERSAL', 'Input contains path traversal sequence');
  }
}
```

---


---

## 2. Output Encoding

### Core Rules
- **Encode on the server** — never trust client-side encoding
- **Context-aware encoding** — HTML, URL, JS, CSS, SQL each need different encoding
- **Encode ALL untrusted data** before sending to any interpreter

### HTML Output Encoding

```typescript
// ✅ Good — DOMPurify for server-side HTML sanitisation (when HTML must be allowed)
import DOMPurify from 'isomorphic-dompurify';

function sanitizeHtml(dirty: string): string {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS:  ['b', 'i', 'em', 'strong', 'p', 'ul', 'li', 'ol', 'br'],
    ALLOWED_ATTR:  [],         // no attributes — prevents event handler injection
    FORBID_TAGS:   ['script', 'iframe', 'object', 'embed', 'form'],
    FORBID_ATTR:   ['onerror', 'onload', 'onclick', 'href', 'src'],
  });
}

// ✅ Good — HTML entity encoding for plain text in HTML context
function encodeHtmlEntities(text: string): string {
  return text
    .replace(/&/g,  '&amp;')
    .replace(/</g,  '&lt;')
    .replace(/>/g,  '&gt;')
    .replace(/"/g,  '&quot;')
    .replace(/'/g,  '&#x27;');
}

// ❌ Bad — unescaped user content in HTML template
const html = `<div>${req.query.name}</div>`;               // XSS!

// ✅ Good — always encode
const html = `<div>${encodeHtmlEntities(req.query.name)}</div>`;
```

### URL Encoding

```typescript
// ❌ Bad — raw user input in URL
const url = `https://api.example.com/search?q=${userInput}`;

// ✅ Good — always encode URL components
const url = `https://api.example.com/search?q=${encodeURIComponent(userInput)}`;

// ✅ Good — validate redirect URL against whitelist before redirect
function safeRedirect(res: Response, url: string, allowedOrigins: string[]): void {
  const isAllowed = allowedOrigins.some(origin => url.startsWith(origin));
  if (!isAllowed) {
    logger.warn('Open redirect attempt blocked', { url });
    return res.redirect('/');  // redirect to home, not to arbitrary URL
  }
  res.redirect(url);
}
```

### SQL / Database Output Encoding

```typescript
// ❌ Critical — SQL injection via concatenation
const result = await db.query(`SELECT * FROM users WHERE name = '${req.body.name}'`);

// ✅ Good — parameterised queries always
const result = await db.query('SELECT * FROM users WHERE name = $1', [req.body.name]);

// ✅ Good — ORM with named parameters
const user = await prisma.user.findFirst({
  where: { name: { equals: req.body.name } },  // ORM handles parameterisation
});

// ✅ Good — dynamic ORDER BY (can't use parameters for column names)
const ALLOWED_SORT_COLUMNS = new Set(['name', 'email', 'created_at']);
const sortCol = ALLOWED_SORT_COLUMNS.has(req.query.sort) ? req.query.sort : 'created_at';
// sortCol is from a whitelist — safe to interpolate
const result = await db.query(`SELECT * FROM users ORDER BY ${sortCol}`);
```

---
