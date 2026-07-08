# TLS Communication & System Configuration

Covers: TLS enforcement, certificate validation, character encoding, HTTP redirect,
system hardening, server headers, robots.txt, HTTP method restriction, env isolation.

---

## 7. TLS — Communication Security

```typescript
// ✅ Good — enforce TLS for all connections
// Node.js HTTPS server setup
import https from 'https';
import fs    from 'fs';

const tlsOptions = {
  cert:                fs.readFileSync('/etc/ssl/certs/server.crt'),
  key:                 fs.readFileSync('/etc/ssl/private/server.key'),
  ca:                  fs.readFileSync('/etc/ssl/certs/ca-bundle.crt'), // intermediates
  minVersion:          'TLSv1.2',   // TLS 1.0 and 1.1 are deprecated
  ciphers: [
    'TLS_AES_256_GCM_SHA384',
    'TLS_CHACHA20_POLY1305_SHA256',
    'TLS_AES_128_GCM_SHA256',
  ].join(':'),
};

https.createServer(tlsOptions, app).listen(443);

// ✅ Good — HTTPS redirect with HSTS
app.use((req, res, next) => {
  if (!req.secure && process.env.NODE_ENV === 'production') {
    return res.redirect(301, `https://${req.hostname}${req.url}`);
  }
  next();
});

// ✅ Good — enforce TLS on outbound HTTP client calls
import axios from 'axios';
import https from 'https';

// Never allow self-signed certs or TLS fallback in production
const secureHttpClient = axios.create({
  httpsAgent: new https.Agent({
    rejectUnauthorized: true,  // reject invalid/expired/self-signed certs
    minVersion:         'TLSv1.2',
  }),
  timeout: 10_000,
});

// ❌ Bad — TLS verification disabled (common mistake in dev that leaks to prod)
const insecureClient = axios.create({
  httpsAgent: new https.Agent({ rejectUnauthorized: false }), // NEVER in production
});
```

---

## 8. System Configuration Security

```typescript
// ✅ Good — remove framework/server information from response headers
import helmet from 'helmet';

app.use(helmet());
app.disable('x-powered-by'); // removes "X-Powered-By: Express"

// ✅ Good — restrict allowed HTTP methods
const ALLOWED_METHODS = ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'];

app.use((req, res, next) => {
  if (!ALLOWED_METHODS.includes(req.method.toUpperCase())) {
    logger.warn('Disallowed HTTP method attempted', { method: req.method, path: req.path });
    return res.status(405).json({ error: { code: 'METHOD_NOT_ALLOWED' } });
  }
  next();
});

// ✅ Good — robots.txt prevents indexing of sensitive directories
// public/robots.txt
/*
User-agent: *
Disallow: /admin/        # block all indexing under one parent
Disallow: /api/
Disallow: /internal/
Allow: /public/
*/

// ✅ Good — turn off directory listings in Nginx
/*
server {
    autoindex off;            # never list directory contents
    server_tokens off;        # hide Nginx version from headers
    add_header X-Powered-By ""; # remove framework info
}
*/
```

---


---

## 8. System Configuration Security

```typescript
// ✅ Good — remove framework/server information from response headers
import helmet from 'helmet';

app.use(helmet());
app.disable('x-powered-by'); // removes "X-Powered-By: Express"

// ✅ Good — restrict allowed HTTP methods
const ALLOWED_METHODS = ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'];

app.use((req, res, next) => {
  if (!ALLOWED_METHODS.includes(req.method.toUpperCase())) {
    logger.warn('Disallowed HTTP method attempted', { method: req.method, path: req.path });
    return res.status(405).json({ error: { code: 'METHOD_NOT_ALLOWED' } });
  }
  next();
});

// ✅ Good — robots.txt prevents indexing of sensitive directories
// public/robots.txt
/*
User-agent: *
Disallow: /admin/        # block all indexing under one parent
Disallow: /api/
Disallow: /internal/
Allow: /public/
*/

// ✅ Good — turn off directory listings in Nginx
/*
server {
    autoindex off;            # never list directory contents
    server_tokens off;        # hide Nginx version from headers
    add_header X-Powered-By ""; # remove framework info
}
*/
```

---


---

## 12. TLS Certificate Validation & Character Encoding

```typescript
// ✅ Good — validate TLS certificate properties before trusting connection
import tls  from 'tls';
import https from 'https';

function validateTlsCertificate(socket: tls.TLSSocket, expectedHostname: string): void {
  const cert = socket.getPeerCertificate(true);

  // 1. Verify certificate is present
  if (!cert || !Object.keys(cert).length) {
    throw new SecurityError('No TLS certificate presented by server');
  }

  // 2. Verify certificate is not expired
  const now = new Date();
  if (new Date(cert.valid_from) > now || new Date(cert.valid_to) < now) {
    throw new SecurityError(
      `TLS certificate expired or not yet valid: ${cert.valid_from} – ${cert.valid_to}`
    );
  }

  // 3. Verify hostname matches certificate CN/SAN
  const authorized = tls.checkServerIdentity(expectedHostname, cert);
  if (authorized) {
    throw new SecurityError(`Certificate hostname mismatch: ${authorized.message}`);
  }

  logger.debug('TLS certificate validated', {
    subject:  cert.subject?.CN,
    issuer:   cert.issuer?.O,
    validTo:  cert.valid_to,
    hostname: expectedHostname,
  });
}

// ✅ Good — outbound HTTPS client with full TLS validation
const secureClient = https.request({
  hostname:           'api.payment-provider.com',
  port:               443,
  method:             'POST',
  rejectUnauthorized: true,     // reject invalid/self-signed/expired certs
  minVersion:         'TLSv1.2', // reject TLS 1.0 and 1.1
  checkServerIdentity: (host, cert) => {
    // Additional custom validation beyond default hostname check
    if (!cert.subjectaltname?.includes(host)) {
      return new Error(`Certificate does not cover host: ${host}`);
    }
    return undefined; // undefined = valid
  },
});

// ✅ Good — specify character encoding on ALL connections
// HTTP: Content-Type header must include charset
res.setHeader('Content-Type', 'application/json; charset=utf-8');

// Database: specify encoding at connection level
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  // PostgreSQL: set client_encoding at connection time
  options: '-c client_encoding=UTF8',
});

// ✅ Good — filter sensitive parameters from Referer header
// When linking to external sites, use rel="noreferrer" to suppress Referer
// In HTML: <a href="https://external.com" rel="noreferrer noopener">Link</a>
// In server-side redirects — strip sensitive params before external redirect:
function buildSafeExternalRedirect(externalUrl: string, internalUrl: string): string {
  const internal = new URL(internalUrl);
  // Remove any params that should not be in Referer
  internal.searchParams.delete('token');
  internal.searchParams.delete('sessionId');
  internal.searchParams.delete('apiKey');
  // Use meta referrer policy to control what browsers send
  return `<meta name="referrer" content="no-referrer">
          <a href="${externalUrl}" rel="noreferrer noopener">Continue</a>`;
}
```

---


---

## 13. Environment Isolation & System Configuration

```typescript
// ✅ Good — environment isolation enforced at config level
// config/environment.config.ts
const ENV = process.env.NODE_ENV ?? 'development';

// Strict production-only rules — fail startup if misconfigured
if (ENV === 'production') {
  const required = [
    'DATABASE_URL', 'JWT_SECRET', 'ENCRYPTION_KEY',
    'SENTRY_DSN', 'REDIS_URL',
  ];
  const missing = required.filter(key => !process.env[key]);
  if (missing.length) {
    throw new Error(`Missing required production config: ${missing.join(', ')}`);
  }

  // ✅ Refuse to start if debug mode is on in production
  if (process.env.DEBUG === 'true') {
    throw new Error('DEBUG must not be enabled in production');
  }
}

// ✅ Good — remove test/debug code before production
// Use build-time flags or environment checks — not commented-out code
export function isDebugMode(): boolean {
  return process.env.NODE_ENV === 'development' && process.env.DEBUG === 'true';
}

// ✅ Good — asset management: track all components and their versions
// Maintain a software bill of materials (SBOM)
// package.json lock file + npm audit serves this purpose for Node
// For infrastructure: Terraform state tracks all cloud components
```

```nginx
# ✅ Good — Nginx production hardening
server {
    listen 443 ssl http2;
    server_name api.yourdomain.com;

    # TLS hardening
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_session_cache   shared:SSL:10m;

    # Disable directory listing
    autoindex off;

    # Remove server version info
    server_tokens off;
    more_clear_headers Server;         # nginx_headers_more module
    more_clear_headers X-Powered-By;

    # Only allow necessary HTTP methods
    if ($request_method !~ ^(GET|POST|PUT|PATCH|DELETE|OPTIONS)$) {
        return 405;
    }

    # Disable WebDAV methods
    dav_methods off;

    # Protect against clickjacking, MIME sniff, etc.
    add_header X-Frame-Options           "DENY"              always;
    add_header X-Content-Type-Options    "nosniff"           always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header Referrer-Policy           "strict-origin-when-cross-origin" always;

    # Static file uploads: no execution privileges
    location /uploads/ {
        root          /var/www/storage;
        add_header    Content-Disposition "attachment";  # force download, not execute
        default_type  application/octet-stream;
        # Disable PHP/CGI execution in this directory
        location ~ \.php$ { deny all; }
    }
}

# ✅ Good — robots.txt: disallow under one parent directory
# /public/robots.txt
# User-agent: *
# Disallow: /private/        ← one parent, not individual paths
# Allow: /public/
```

---



---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| TLS verification disabled | `rejectUnauthorized: false` on any HTTPS client | 🔴 |
| HTTP without HTTPS redirect | Port 80 with no redirect to 443 | 🟠 |
| TLS version < 1.2 | `minVersion: 'TLSv1'` in any config | 🔴 |
| Certificate expiry not monitored | No alert before cert expires | 🟠 |
| Character encoding not specified | `Content-Type` missing `; charset=utf-8` | 🟡 |
| Server version in headers | `Server: nginx/1.18.0` in responses | 🟡 |
| Directory listing enabled | Web server returns directory index | 🟠 |
| Test/debug code in production | `DEBUG=true`, test routes in prod | 🔴 |
| Dev environment touches prod resources | Dev config pointing at prod DB | 🔴 |
| WebDAV/TRACE methods enabled | `TRACE`, `PROPFIND` not blocked | 🟡 |
