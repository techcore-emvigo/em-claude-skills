# em-web — Rules Reference

## 1. HTML & CSS
| Rule | Description | Severity |
|---|---|---|
| HTML-01 | Semantic elements: `<header>`, `<nav>`, `<main>`, `<section>`, `<article>`, `<footer>` — never `<div>` for everything | 🟠 |
| HTML-02 | Every `<img>` has meaningful `alt` attribute; decorative images use `alt=""` | 🟠 |
| HTML-03 | Every form input has associated `<label for="id">` or `aria-label` | 🟠 |
| HTML-04 | No `<div onClick>` for actions — use `<button>` | 🟠 |
| HTML-05 | Focus styles never removed without a visible replacement (`outline: none` blocked) | 🟠 |
| HTML-06 | `lang` attribute on `<html>` element; `<meta charset="UTF-8">` and viewport meta present | 🟡 |
| HTML-07 | Touch targets minimum 44×44px | 🟡 |
| CSS-01 | No inline styles `style={{ ... }}` — use CSS classes or `StyleSheet.create()` | 🟡 |
| CSS-02 | No `!important` except overriding third-party styles | 🟡 |
| CSS-03 | CSS custom properties for all design tokens — colors, spacing, typography | 🟡 |
| CSS-04 | Mobile-first with `min-width` media queries | 🟠 |
| CSS-05 | No magic numbers — use variables or calculated values | 🟡 |

## 2. JavaScript & TypeScript
| Rule | Description | Severity |
|---|---|---|
| JS-01 | `const` by default; `let` when reassignment needed; `var` never | 🟡 |
| JS-02 | `===` always — `==` never | 🟡 |
| JS-03 | `strict: true` in every `tsconfig.json` | 🔴 |
| JS-04 | No `any` type — use `unknown` with type guard or proper type | 🟠 |
| JS-05 | `async/await` with `try/catch` — no unhandled promise rejections | 🟠 |
| JS-06 | `Promise.all([...])` for independent async operations — never sequential awaits | 🟠 |
| JS-07 | No `console.log` in production — structured logger only | 🟡 |
| JS-08 | No `eval(userInput)` or `new Function(userInput)` | 🔴 |
| JS-09 | No string first argument to `setTimeout`/`setInterval` | 🔴 |
| JS-10 | No PII (email, phone, password) in any log statement | 🔴 |

## 3. React & Next.js
| Rule | Description | Severity |
|---|---|---|
| RX-01 | `key` props use stable unique IDs — never array index on dynamic lists | 🟠 |
| RX-02 | All `useEffect` dependency arrays complete; cleanup returned where needed | 🟠 |
| RX-03 | No `dangerouslySetInnerHTML` with unsanitised user content | 🔴 |
| RX-04 | Routes code-split with `React.lazy` + `Suspense` | 🟡 |
| RX-05 | Skeleton screens per component — no full-page spinners | 🟡 |
| RX-06 | `next/image` for all images; `next/link` for all internal navigation | 🟡 |
| RX-07 | No `NEXT_PUBLIC_` prefix on secret values | 🔴 |
| RX-08 | Auth check on every API route handler | 🔴 |
| RX-09 | Server Components by default in App Router; `"use client"` only when hooks/events needed | 🟡 |
| RX-10 | Token or session ID never stored in `localStorage` | 🔴 |

## 4. Angular & Vue
| Rule | Description | Severity |
|---|---|---|
| AV-01 | `ChangeDetectionStrategy.OnPush` on all components receiving immutable data | 🟡 |
| AV-02 | `trackBy` on all `*ngFor` with stable unique ID | 🟠 |
| AV-03 | `async` pipe in templates over manual subscribe/unsubscribe | 🟠 |
| AV-04 | Subscriptions unsubscribed — `takeUntilDestroyed()` or `async` pipe | 🟠 |
| AV-05 | No `bypassSecurityTrustHtml` / `v-html` with user content | 🔴 |
| AV-06 | Vue 3: `<script setup lang="ts">` with typed `defineProps` and `defineEmits` | 🟡 |
| AV-07 | Pinia: `storeToRefs()` when destructuring store state | 🟠 |

## 5. Performance
| Rule | Description | Severity |
|---|---|---|
| PF-01 | Long lists virtualised — `react-window`, `react-virtual`, or `FlatList` | 🟠 |
| PF-02 | Images lazy-loaded below fold — `loading="lazy"` or `next/image` | 🟡 |
| PF-03 | No N+1 API calls — combine requests or use `Promise.all` | 🟠 |
| PF-04 | `useMemo` for expensive computations; `useCallback` for stable function refs | 🟡 |
| PF-05 | Lighthouse Performance ≥ 80, Accessibility ≥ 90 before release | 🟠 |
| PF-06 | Static assets served from CDN — not origin server | 🟡 |
| PF-07 | CSS and JS minified and compressed in production build | 🟡 |

## 6. General Quality
| Rule | Description | Severity |
|---|---|---|
| GQ-01 | Components ≤ 200 lines; functions ≤ 30 lines | 🟠 |
| GQ-02 | Named constants and enums for all magic numbers and strings | 🟠 |
| GQ-03 | No dead code — unused variables, imports, commented blocks removed | 🟡 |
| GQ-04 | Unit tests cover all business logic — happy path + error + edge cases | 🟠 |
| GQ-05 | Sentry integrated in every frontend application | 🔴 |

## 7. Instant Escalation — 🔴
| # | Violation |
|---|---|
| ESC-01 | `dangerouslySetInnerHTML` / `v-html` / `innerHTML` with unsanitised user content |
| ESC-02 | Token or session ID in `localStorage` |
| ESC-03 | PII (email, phone, password) in any log statement |
| ESC-04 | `eval(userInput)` or `new Function(userInput)` |
| ESC-05 | `key={index}` on dynamically reordered list |
| ESC-06 | `NEXT_PUBLIC_` on any secret value |
| ESC-07 | No auth check on a state-changing API route |
