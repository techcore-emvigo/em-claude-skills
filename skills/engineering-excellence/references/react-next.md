# ReactJS & NextJS — Gap Detection

## React Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `dangerouslySetInnerHTML` with user content | `dangerouslySetInnerHTML={{ __html: userInput }}` without DOMPurify | 🔴 |
| `key={index}` on dynamic list | `items.map((item, index) => <div key={index}>` — breaks reconciliation | 🟠 |
| Missing useEffect cleanup | `useEffect` with subscription/timer/listener but no `return () => cleanup()` | 🟠 |
| Incomplete dependency array | `useEffect` or `useCallback` with missing deps — stale closure bugs | 🟠 |
| State update in unmounted component | Async operation calls `setState` after component may be unmounted | 🟠 |
| Heavy computation in render | Expensive calc directly in JSX/return with no `useMemo` | 🟠 |
| Prop drilling > 2 levels | State passed through 3+ component layers via props | 🟡 |
| Token in localStorage | `localStorage.setItem("token", ...)` — XSS-vulnerable | 🔴 |
| Class component in new code | `class MyComponent extends React.Component` in new feature code | 🟡 |
| Missing error boundary | No `<ErrorBoundary>` around async or dynamic components | 🟡 |

## NextJS Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `NEXT_PUBLIC_` on secret value | `NEXT_PUBLIC_API_SECRET` — exposes secret to browser bundle | 🔴 |
| `getServerSideProps` for static content | Using SSR where SSG or ISR would suffice — unnecessary server load | 🟠 |
| `<img>` instead of `next/image` | Raw `<img src=...>` — no lazy load, no WebP, causes layout shift | 🟡 |
| `<a href>` instead of `next/link` | Raw anchor tags for internal navigation — no prefetching | 🟡 |
| No auth on API route handler | `pages/api/` or `app/api/` route with no auth check | 🔴 |
| Client component for data-only work | `"use client"` on a component that only fetches and renders, no hooks/events | 🟡 |
| Missing `loading.tsx` / `error.tsx` | Route segment with no loading or error boundary | 🟡 |

## Generation Checklist
- [ ] `key` props use stable unique IDs, never array index
- [ ] All `useEffect` deps complete; cleanup returned where needed
- [ ] `next/image` for all images, `next/link` for all internal links
- [ ] No `NEXT_PUBLIC_` prefix on secret values
- [ ] Auth middleware or per-route guard on every API route
- [ ] Server Components by default; `"use client"` only when hooks/events needed
