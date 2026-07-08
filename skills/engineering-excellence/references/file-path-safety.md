# File Path Safety & Dynamic Include Prevention

Covers: index mapping instead of user-supplied paths, blocking dynamic includes,
redirect validation, whitelist for referenced files.

---

## 15. File Path Safety — Index Mapping Instead of User Paths

Never let user input control file paths. Map user-supplied identifiers to pre-defined paths.

```typescript
// ❌ Bad — user controls file path
app.get('/documents/:filename', (req, res) => {
  const filePath = path.join('/var/docs', req.params.filename);
  res.sendFile(filePath); // ../../etc/passwd traversal possible!
});

// ❌ Bad — dynamic require/include from user input
const template = require(`./templates/${req.query.template}`); // arbitrary module load!

// ✅ Good — index map: user provides an opaque key, not a path
const DOCUMENT_MAP: Record<string, string> = {
  'terms':     '/var/docs/legal/terms-of-service.pdf',
  'privacy':   '/var/docs/legal/privacy-policy.pdf',
  'user-guide':'/var/docs/help/user-guide.pdf',
};

app.get('/documents/:docKey', (req, res) => {
  const filePath = DOCUMENT_MAP[req.params.docKey];

  if (!filePath) {
    // Unknown key — use hard-coded default, not user value
    return res.status(404).json({ error: 'Document not found' });
  }

  // Path comes from our map — safe, no traversal possible
  res.sendFile(filePath);
});

// ✅ Good — template selection via whitelist index map
const TEMPLATE_MAP: Record<string, () => Promise<unknown>> = {
  'welcome':       () => import('./templates/welcome'),
  'order-confirm': () => import('./templates/order-confirmation'),
  'reset':         () => import('./templates/password-reset'),
};

async function loadTemplate(key: string) {
  const loader = TEMPLATE_MAP[key];
  if (!loader) throw new ValidationError('template', `Unknown template: ${key}`);
  return loader(); // imports from our map — user cannot control the path
}

// ✅ Good — validate redirect is relative-path only (no absolute URL)
function safeRelativeRedirect(res: Response, returnPath: string): void {
  // Only allow relative paths — block http://, //, data:, javascript: etc.
  const SAFE_RELATIVE = /^\/[a-zA-Z0-9\-_/?=&#%]+$/;

  if (!SAFE_RELATIVE.test(returnPath)) {
    logger.warn('Unsafe redirect path blocked', { path: returnPath });
    return res.redirect('/');  // fallback to home
  }
  res.redirect(returnPath);
}
```

---

## Gap Detection Table (Additions)


---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| SQL concatenation | f-string or `+` building a SQL string | 🔴 |
| DB connection string in source | `postgresql://user:pass@host` in code | 🔴 |
| Default DB admin password | DB using `postgres`/`root` default credentials | 🔴 |
| DB port open to internet | Port 3306/5432 not restricted to VPC | 🔴 |
| File type by extension only | No magic byte check on uploads | 🔴 |
| No file size limit | Upload with no `limits.fileSize` | 🟠 |
| User filename used in storage path | `fs.writeFile(req.file.originalname)` | 🔴 |
| File stored in web root | `public/uploads/` directly accessible | 🔴 |
| No malware scan on upload | Files stored without antivirus check | 🟠 |
| Absolute path returned to client | API returns `/var/www/...` path | 🟠 |
| User input in dynamic include | `require(userInput)` | 🔴 |
| File path from user input | `path.join(base, req.params.file)` without index map | 🔴 |
