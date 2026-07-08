# Performance Best Practices — Code Reference

## 1. API Response Time Standard
- **Target:** ≤ 800ms for all API responses (Lambda and GraphQL)
- **Page load:** Google PageSpeed Insight score ≥ 80 (mobile and web)
- **Lighthouse:** Performance ≥ 80, Accessibility ≥ 90

---

## 2. Frontend Performance

### Minification & Asset Optimisation

```javascript
// webpack.config.js / vite.config.ts
// ✅ Good — minify JS/CSS in production build
export default defineConfig({
  build: {
    minify:          'terser',
    cssMinify:       true,
    rollupOptions: {
      output: {
        // Code splitting — each route gets its own chunk
        manualChunks: (id) => {
          if (id.includes('node_modules')) return 'vendor';
        },
      },
    },
  },
});
```

### Lazy Loading — Routes and Components

```typescript
// ❌ Bad — entire app loaded upfront
import { AdminDashboard }   from './pages/AdminDashboard';
import { UserProfile }      from './pages/UserProfile';
import { PaymentHistory }   from './pages/PaymentHistory';

// ✅ Good — each route loaded on demand
const AdminDashboard  = React.lazy(() => import('./pages/AdminDashboard'));
const UserProfile     = React.lazy(() => import('./pages/UserProfile'));
const PaymentHistory  = React.lazy(() => import('./pages/PaymentHistory'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/admin"    element={<AdminDashboard />} />
        <Route path="/profile"  element={<UserProfile />} />
        <Route path="/payments" element={<PaymentHistory />} />
      </Routes>
    </Suspense>
  );
}
```

### Lazy Loading Images

```html
<!-- ❌ Bad — all images load immediately -->
<img src="/hero-banner.jpg" />

<!-- ✅ Good — browser native lazy loading -->
<img src="/hero-banner.jpg" loading="lazy" alt="Hero banner" width="1200" height="600" />

<!-- ✅ Better — responsive images with WebP -->
<picture>
  <source srcset="/hero-banner.webp" type="image/webp" />
  <img src="/hero-banner.jpg" loading="lazy" alt="Hero banner" width="1200" height="600" />
</picture>
```

### Component Skeleton / Placeholder (No Full-Page Loaders)

```typescript
// ❌ Bad — full page spinner blocks all content
function Dashboard() {
  if (loading) return <FullPageSpinner />;
  return <DashboardContent data={data} />;
}

// ✅ Good — skeleton screens per component; only payment gets full-page loader
function Dashboard() {
  return (
    <div>
      <UserHeader />                              {/* always visible */}
      <Suspense fallback={<StatsSkeleton />}>
        <StatsWidget />                           {/* loads independently */}
      </Suspense>
      <Suspense fallback={<OrderListSkeleton />}>
        <RecentOrders />                          {/* loads independently */}
      </Suspense>
    </div>
  );
}
```

### Avoid Multiple API Calls — Combine or Lazy Load

```typescript
// ❌ Bad — 5 separate API calls on page load
useEffect(() => {
  fetchUser();
  fetchOrders();
  fetchNotifications();
  fetchStats();
  fetchRecentActivity();
}, []);

// ✅ Good — single aggregated call (BFF pattern)
useEffect(() => {
  // One API returns everything needed for this page
  fetchDashboardData(); // /api/dashboard — aggregated on server
}, []);

// ✅ Alternative — lazy load non-critical sections
useEffect(() => {
  fetchCriticalData(); // load above-the-fold data immediately
}, []);

// Load non-critical sections only when they scroll into view
const { ref, inView } = useInView({ triggerOnce: true });
useEffect(() => {
  if (inView) fetchRecentActivity();
}, [inView]);
```

---

## 3. Caching Strategy

### Frontend — Browser Caching

```typescript
// ✅ Good — cache-control headers via server/CDN
// For static assets (CSS, JS, fonts, images):
// Cache-Control: public, max-age=31536000, immutable
// (hashed filenames ensure cache-busting on deploy: main.a1b2c3.js)

// For API responses that are user-specific:
// Cache-Control: private, no-store
// (never cache auth-required responses in shared CDN)

// For public, slowly-changing data:
// Cache-Control: public, s-maxage=300, stale-while-revalidate=60
```

### Backend — Redis Caching Pattern

```typescript
// cache/cache.service.ts
@Injectable()
export class CacheService {
  constructor(@Inject(REDIS_CLIENT) private redis: Redis) {}

  async getOrSet<T>(
    key:     string,
    ttlSecs: number,
    factory: () => Promise<T>,
  ): Promise<T> {
    const cached = await this.redis.get(key);
    if (cached) return JSON.parse(cached) as T;

    const value = await factory();
    await this.redis.setex(key, ttlSecs, JSON.stringify(value));
    return value;
  }

  async invalidate(key: string): Promise<void> {
    await this.redis.del(key);
  }
}

// ✅ Usage — wrap expensive DB queries
async getProductCatalog(): Promise<Product[]> {
  return this.cache.getOrSet(
    'product:catalog',
    300,                                  // 5 minute TTL
    () => this.productRepo.findAllActive(), // only called on cache miss
  );
}
```

### Pre-compute Expensive Analytics

```typescript
// ❌ Bad — complex aggregation on every API call
app.get('/analytics/summary', async (req, res) => {
  // This takes 3 seconds to compute on 1M rows
  const summary = await db.query(`
    SELECT category, SUM(revenue), COUNT(*) 
    FROM orders 
    WHERE created_at > NOW() - INTERVAL 30 DAY
    GROUP BY category
  `);
  res.json(summary);
});

// ✅ Good — pre-computed by a scheduled job, served from cache
// Cron job runs every 15 minutes:
async function precomputeAnalyticsSummary() {
  const summary = await db.query(heavyAnalyticsQuery);
  await redis.setex('analytics:summary:30d', 900, JSON.stringify(summary));
}

// API serves pre-computed result instantly
app.get('/analytics/summary', async (req, res) => {
  const cached = await redis.get('analytics:summary:30d');
  if (cached) return res.json(JSON.parse(cached));
  // Fallback if cache miss (first run)
  const summary = await computeAnalyticsSummary();
  res.json(summary);
});
```

---

## 4. Database Performance

### Pagination — Always Cursor-Based for Large Tables

```typescript
// ❌ Bad — offset pagination degrades as offset grows
app.get('/orders', async (req, res) => {
  const { page = 1, limit = 20 } = req.query;
  const orders = await db.orders.findAll({
    offset: (page - 1) * limit,  // OFFSET 50000 = full table scan
    limit,
  });
  res.json(orders);
});

// ✅ Good — cursor-based pagination stays fast regardless of position
app.get('/orders', async (req, res) => {
  const { cursor, limit = 20 } = req.query;
  const orders = await db.orders.findMany({
    where:   cursor ? { id: { gt: cursor } } : {},
    orderBy: { id: 'asc' },
    take:    Number(limit) + 1,   // fetch one extra to determine hasMore
  });
  const hasMore = orders.length > limit;
  res.json({
    data:       hasMore ? orders.slice(0, -1) : orders,
    nextCursor: hasMore ? orders[orders.length - 2].id : null,
    hasMore,
  });
});
```

### Fetch Only What You Need

```typescript
// ❌ Bad — fetches entire entity when only 3 fields are needed
const users = await prisma.user.findMany(); // SELECT * — 40 columns

// ✅ Good — project only needed fields
const users = await prisma.user.findMany({
  select: { id: true, name: true, email: true }, // only 3 columns
  where:  { isActive: true },
});
```

### Index Strategy

```sql
-- ❌ Bad — querying on unindexed column on a large table
SELECT * FROM orders WHERE user_id = ? AND status = 'pending';
-- No index on (user_id, status) = full table scan

-- ✅ Good — composite index matches query pattern
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

-- ✅ Good — partial index for common filtered queries
CREATE INDEX idx_orders_pending ON orders (user_id)
WHERE status = 'pending';  -- PostgreSQL only
```

---

## 5. API Design for Performance

### Separate Mobile and Web Responses

```typescript
// ❌ Bad — one fat response for all clients
app.get('/products/:id', async (req, res) => {
  const product = await productRepo.findById(req.params.id);
  res.json(product); // 50 fields, images, reviews, specs — too much for mobile
});

// ✅ Good — client specifies what it needs, or separate endpoints
app.get('/products/:id', async (req, res) => {
  const { client = 'web' } = req.query;
  const product = await productRepo.findById(req.params.id);

  const webResponse    = new ProductWebDto(product);    // full detail
  const mobileResponse = new ProductMobileDto(product); // minimal fields

  res.json(client === 'mobile' ? mobileResponse : webResponse);
});
```

### Response Size Optimisation

```typescript
// ✅ Good — always enable gzip/brotli compression
import compression from 'compression';
app.use(compression()); // reduces response size 60–80% for JSON

// ✅ Good — return only changed data on update operations
@Patch('orders/:id')
async updateOrder(@Param('id') id: string, @Body() dto: UpdateOrderDto) {
  const updated = await this.orderService.update(id, dto);
  // Return only the fields that changed, not the full order
  return { id: updated.id, status: updated.status, updatedAt: updated.updatedAt };
}
```

---

## 6. CDN for Static Assets

```html
<!-- ✅ Good — serve static assets from CDN, not origin server -->
<!-- Set in your deployment pipeline / terraform / serverless.yml -->

<!-- Images -->
<img src="https://cdn.yourdomain.com/images/logo.webp" alt="Logo" />

<!-- JS/CSS — with hashed filename for cache-busting -->
<link href="https://cdn.yourdomain.com/assets/main.a1b2c3.css" rel="stylesheet" />
<script src="https://cdn.yourdomain.com/assets/app.d4e5f6.js" defer></script>
```

---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| Full-page spinner blocking all content | `if (loading) return <Spinner />` wrapping the whole page | 🟠 |
| No lazy loading on route components | All pages imported at top of router file | 🟡 |
| Images without `loading="lazy"` | `<img src=...>` below the fold with no lazy attribute | 🟡 |
| Multiple API calls on page load | 5+ `useEffect` fetches firing simultaneously | 🟠 |
| No pagination on list API | Endpoint returns all records with no cursor/page | 🔴 |
| Offset pagination on large table | `OFFSET N` on table that can have millions of rows | 🟡 |
| No backend caching on repeated queries | Same expensive query executed on every request | 🟠 |
| On-the-fly analytics computation | Complex aggregation query in a user-facing API | 🟠 |
| `SELECT *` in queries | No field projection — fetching unnecessary columns | 🟠 |
| No CDN for static assets | Images/JS/CSS served from origin app server | 🟡 |
| No response compression | Large JSON responses without gzip/brotli | 🟡 |
| API response not optimised per client | Same heavy response for mobile and web | 🟡 |
| Missing API response time monitoring | No SLA defined or measured for API endpoints | 🟠 |
