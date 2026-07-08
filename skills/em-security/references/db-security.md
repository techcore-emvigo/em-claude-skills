# Database Security

Covers: parameterised queries, DB user least privilege, connection string management,
stored procedures, default credential removal, surface-area reduction.

---

# Database Security & File Management

Covers: parameterised queries, DB user privileges, connection strings, stored
procedures, file upload validation, malware scanning, path safety, index mapping.

---

## 9. Database Security

```typescript
// ✅ Good — application DB user has minimum privileges
// In PostgreSQL migration (run as superuser once):
/*
  CREATE ROLE app_user LOGIN PASSWORD '...'; -- strong password from secrets manager
  GRANT CONNECT ON DATABASE myapp TO app_user;
  GRANT USAGE ON SCHEMA public TO app_user;
  GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
  -- NOT: GRANT ALL, ALTER, DROP, CREATE
*/

// ✅ Good — connection string from environment, never hardcoded
const dbClient = new Pool({
  connectionString: process.env.DATABASE_URL,  // from secrets manager
  max:              20,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 2_000,
  ssl: process.env.NODE_ENV === 'production'
    ? { rejectUnauthorized: true }  // enforce TLS on DB connection
    : false,
});

// ✅ Good — close connections promptly; use connection pool
// Connection pool manages connections — never hold open beyond request lifetime
app.use(async (req, res, next) => {
  req.db = await dbPool.connect(); // acquire from pool
  res.on('finish', () => req.db?.release()); // always release back to pool
  next();
});

// ✅ Good — different DB users per trust level
const connections = {
  app:       new Pool({ connectionString: process.env.DB_APP_URL }),       // DML only
  reporting: new Pool({ connectionString: process.env.DB_REPORTING_URL }), // SELECT only
  migration: new Pool({ connectionString: process.env.DB_MIGRATION_URL }), // DDL — CI only
};
```

---

