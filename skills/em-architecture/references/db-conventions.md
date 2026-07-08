# Database Architecture & Naming Conventions

## 1. Naming Conventions

### Tables
- `snake_case`, lowercase, **plural** nouns
- Descriptive and domain-aligned — no abbreviations
- Junction/pivot tables: `entity_a_entity_b` alphabetically ordered

```sql
-- ❌ Bad
CREATE TABLE User         ();   -- singular, PascalCase
CREATE TABLE userOrders   ();   -- camelCase
CREATE TABLE tbl_prd      ();   -- prefix, abbreviation
CREATE TABLE OrderDetails ();   -- PascalCase

-- ✅ Good
CREATE TABLE users              ();
CREATE TABLE orders             ();
CREATE TABLE order_items        ();
CREATE TABLE product_categories ();
CREATE TABLE user_roles         ();   -- junction table: user <-> role
```

### Columns
- `snake_case`, lowercase, descriptive
- Foreign keys: `referenced_table_singular_id` → `user_id`, `order_id`
- Boolean columns: `is_*` or `has_*` prefix → `is_active`, `has_verified_email`

```sql
-- ❌ Bad
userId, createAt, UpdatedDate, active, DelFlag

-- ✅ Good
user_id, created_at, updated_at, is_active, is_deleted
```

### Mandatory Audit Columns
Every entity table must include these columns:

```sql
CREATE TABLE orders (
  id           BIGSERIAL      PRIMARY KEY,          -- or UUID
  -- ... business columns ...
  created_by   BIGINT         NOT NULL,              -- user ID who created
  created_at   TIMESTAMPTZ    NOT NULL DEFAULT NOW(),
  updated_by   BIGINT,                               -- null until first update
  updated_at   TIMESTAMPTZ,
  is_active    BOOLEAN        NOT NULL DEFAULT TRUE,  -- soft enable/disable
  is_deleted   BOOLEAN        NOT NULL DEFAULT FALSE, -- soft delete flag
  deleted_at   TIMESTAMPTZ,                          -- timestamp of deletion
  deleted_by   BIGINT                                -- who deleted
);
```

### Indexes Naming

```sql
-- Convention: idx_{table}_{columns}
CREATE INDEX idx_orders_user_id           ON orders (user_id);
CREATE INDEX idx_orders_status_created    ON orders (status, created_at);
CREATE INDEX idx_users_email              ON users (email);
CREATE UNIQUE INDEX uidx_users_email      ON users (email);   -- unique index

-- FK constraint convention: fk_{table}_{referenced_table}
ALTER TABLE orders ADD CONSTRAINT fk_orders_users FOREIGN KEY (user_id) REFERENCES users(id);
```

---

## 2. Relationships & Foreign Keys

```sql
-- ✅ Good — explicit FK with proper naming and action
CREATE TABLE order_items (
  id          BIGSERIAL PRIMARY KEY,
  order_id    BIGINT       NOT NULL,
  product_id  BIGINT       NOT NULL,
  quantity    INTEGER      NOT NULL CHECK (quantity > 0),
  unit_price  DECIMAL(10,2) NOT NULL,

  CONSTRAINT fk_order_items_orders
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  CONSTRAINT fk_order_items_products
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT
);

-- Always index the FK columns
CREATE INDEX idx_order_items_order_id   ON order_items (order_id);
CREATE INDEX idx_order_items_product_id ON order_items (product_id);
```

---

## 3. Column Type Standards

```sql
-- ✅ Standards for common column types

-- IDs: prefer BIGSERIAL (sequence) or UUID
id          BIGSERIAL PRIMARY KEY
id          UUID DEFAULT gen_random_uuid() PRIMARY KEY  -- PostgreSQL 13+

-- Timestamps: always with timezone
created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()

-- Money / Currency: DECIMAL not FLOAT (floating point errors)
amount      DECIMAL(15, 4) NOT NULL   -- 15 digits total, 4 decimal places

-- Status / Type: VARCHAR with CHECK constraint or lookup table
status      VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'confirmed', 'shipped'))

-- Boolean: explicit DEFAULT
is_active   BOOLEAN NOT NULL DEFAULT TRUE

-- Long text: TEXT not VARCHAR with large max
description TEXT

-- Phone: VARCHAR(20) — allows international format
phone       VARCHAR(20)

-- Email: VARCHAR(254) — RFC 5321 max
email       VARCHAR(254) NOT NULL
```

---

## 4. ER Diagram & Documentation Standard

Every database must have:
1. **ER Diagram** — maintained in draw.io, dbdiagram.io, or equivalent
2. **Data Dictionary** — table and column descriptions
3. **Migration history** — tracked via Flyway, Liquibase, or ORM migrations

```
# dbdiagram.io syntax example (commit this to your repo)
Table users {
  id         bigint    [pk, increment]
  email      varchar   [unique, not null]
  name       varchar   [not null]
  is_active  boolean   [default: true]
  created_at timestamp [default: `now()`]
}

Table orders {
  id          bigint    [pk, increment]
  user_id     bigint    [ref: > users.id]
  status      varchar   [not null]
  total_amount decimal
  created_at  timestamp [default: `now()`]
}
```

---

## 5. MongoDB Naming Conventions

```javascript
// Collections: camelCase, plural
// ❌ Bad
db.User, db.order_item, db.ProductCategory

// ✅ Good
db.users, db.orderItems, db.productCategories

// Fields: camelCase (JS convention for Mongo)
// ❌ Bad
{ user_id: ..., created_at: ..., isActive: ... } // mixed conventions

// ✅ Good — consistent camelCase throughout
{
  userId:    ObjectId('...'),
  createdAt: ISODate('...'),   // not created_at
  isActive:  true,
  isDeleted: false,
}
```

---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| PascalCase or camelCase table names | `UserProfile`, `orderItems` as table names | 🟡 |
| Singular table names | `user`, `order` instead of `users`, `orders` | 🟡 |
| Inconsistent column naming | Mix of `created_at` and `createdAt` in same DB | 🟠 |
| Missing audit columns | Table without `created_at`, `updated_at`, `is_deleted` | 🟠 |
| FK column without index | `order_id` column with no `CREATE INDEX` | 🔴 |
| FK with no constraint | Column referencing another table with no FK constraint | 🟠 |
| `FLOAT` for money | `price FLOAT` or `amount DOUBLE` — floating point errors | 🔴 |
| `TIMESTAMP` without timezone (PG) | `TIMESTAMP` instead of `TIMESTAMPTZ` | 🟡 |
| No ER diagram maintained | No visual representation of DB structure | 🟡 |
| No migration history | Manual `ALTER TABLE` with no tracking | 🔴 |
| DB accessible publicly | Port open to internet | 🔴 |
| Firestore rules allow all | `allow read, write: if true` | 🔴 |
| MongoDB column convention mixed | `user_id` and `userId` both appear in same collection | 🟡 |

---

## 6. Consistent Column Naming — MySQL vs MongoDB

A common gap is inconsistent naming between MySQL and MongoDB collections in the same project,
or even within the same DB. Enforce one convention per technology — never mix.

```sql
-- ❌ Bad — mixed conventions in the same MySQL schema
-- Table 1: snake_case
CREATE TABLE users (
  user_id     INT,
  created_at  DATETIME,
  isActive    BOOLEAN   -- camelCase mixed in!
);

-- Table 2: camelCase (different convention!)
CREATE TABLE orders (
  orderId     INT,
  createdAt   DATETIME,
  is_deleted  BOOLEAN   -- snake_case mixed in!
);

-- ✅ Good — pure snake_case throughout MySQL
CREATE TABLE users (
  id          BIGINT PRIMARY KEY,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ,
  is_active   BOOLEAN NOT NULL DEFAULT TRUE,
  is_deleted  BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE TABLE orders (
  id          BIGINT PRIMARY KEY,
  user_id     BIGINT NOT NULL,       -- FK: always snake_case
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  is_deleted  BOOLEAN NOT NULL DEFAULT FALSE
);
```

```javascript
// ❌ Bad — mixed conventions in MongoDB documents
db.users.insertOne({
  userId:      ObjectId('...'),   // camelCase
  created_at:  new Date(),        // snake_case
  isActive:    true,              // camelCase
  delete_YN:   'N',              // mixed — 'YN' flag pattern
});

// ✅ Good — pure camelCase throughout MongoDB
db.users.insertOne({
  _id:         ObjectId('...'),   // MongoDB default — always _id
  userId:      'usr_abc123',
  createdAt:   new Date(),        // camelCase — matches JS convention
  updatedAt:   new Date(),
  isActive:    true,
  isDeleted:   false,             // boolean, not 'Y'/'N' string
  createdBy:   'usr_xyz',
  updatedBy:   null,
});
```

---

## 7. Foreign Key Relations — Complete Pattern

```sql
-- ✅ Good — complete FK setup: constraint + index + meaningful name
CREATE TABLE order_items (
  id           BIGSERIAL      PRIMARY KEY,
  order_id     BIGINT         NOT NULL,
  product_id   BIGINT         NOT NULL,
  quantity     INTEGER        NOT NULL CHECK (quantity > 0),
  unit_price   DECIMAL(10,2)  NOT NULL,

  created_at   TIMESTAMPTZ    NOT NULL DEFAULT NOW(),
  created_by   BIGINT,
  is_deleted   BOOLEAN        NOT NULL DEFAULT FALSE,

  -- Constraints with meaningful names
  CONSTRAINT fk_order_items_orders
    FOREIGN KEY (order_id)
    REFERENCES orders(id)
    ON DELETE CASCADE       -- delete items when order is deleted
    ON UPDATE RESTRICT,

  CONSTRAINT fk_order_items_products
    FOREIGN KEY (product_id)
    REFERENCES products(id)
    ON DELETE RESTRICT      -- prevent product deletion if items reference it
    ON UPDATE RESTRICT
);

-- ✅ Always index FK columns — these are join columns
CREATE INDEX idx_order_items_order_id   ON order_items (order_id);
CREATE INDEX idx_order_items_product_id ON order_items (product_id);

-- ✅ Composite index when you always filter by both
CREATE INDEX idx_order_items_order_product ON order_items (order_id, product_id);
```

---

## Gap Detection Table (Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| `active_YN` / `delete_YN` pattern | String `'Y'/'N'` flags instead of `BOOLEAN` | 🟡 |
| `created_date` instead of `created_at` | Column named `created_date`, `createDate`, `createdDate` | 🟡 |
| `update_by` instead of `updated_by` | Inconsistent past-tense naming on audit columns | 🟡 |
| camelCase columns in MySQL | `userId`, `createdAt` as MySQL column names | 🟠 |
| snake_case fields in MongoDB | `user_id`, `created_at` as MongoDB document keys | 🟡 |
| FK without `ON DELETE` action specified | FK constraint with no cascade/restrict policy | 🟡 |
| Missing `created_by` / `updated_by` | Audit trail impossible — can't trace who made changes | 🟠 |
