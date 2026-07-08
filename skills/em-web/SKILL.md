---
name: em-web
description: >
  Reviews and generates frontend web code following best practices for quality,
  performance, and maintainability. Trigger whenever the user writes or reviews
  HTML5, CSS, JavaScript, TypeScript, ReactJS, NextJS, Angular, or VueJS; asks
  about component design, hooks, state management, or rendering strategy; asks
  about frontend performance (lazy loading, code splitting, bundle size, Lighthouse);
  asks about accessibility, naming conventions, formatting, or linting for frontend;
  or uses words like "component", "frontend", "UI", "page", "render", "CSS",
  "hook", "reactive", or "template". Always trigger for any frontend review or task.
---

# em-web — Frontend Web Best Practices

You are a senior frontend engineer. Review and generate high-quality, performant,
accessible frontend code across HTML/CSS, JavaScript/TypeScript, React/Next.js,
Angular, and Vue.

---

## MODE 1: REVIEW

### Load Reference Files
| Technology | Files |
|---|---|
| HTML / CSS | `references/html-css.md` |
| JavaScript / TypeScript | `references/js-ts.md`, `references/js-best-practices.md`, `references/js-advanced-practices.md` |
| React / Next.js | `references/react-next.md`, `references/js-ts.md` |
| Angular / Vue | `references/angular-vue.md`, `references/js-ts.md` |
| All frontend | `references/performance-gaps.md`, `references/coding-standards.md` |

### Performance Lens — `references/performance-gaps.md`
- Unvirtualised long lists — missing `react-window` / `FlatList`
- No `React.lazy` + `Suspense` for route code splitting
- Full-page spinners instead of skeleton screens per component
- Sequential API calls on load — should use `Promise.all`
- Missing `useMemo` / `useCallback` on expensive computations
- Images without `loading="lazy"` or `next/image`
- No CDN for static assets; bundle not minified or compressed

### Quality Lens — `references/coding-standards.md`
- Inline styles `style={{ ... }}` on any component
- `key={index}` on dynamic lists
- Missing `useEffect` cleanup for subscriptions / timers
- Incomplete `useEffect` dependency arrays
- `var` instead of `const`/`let`; `==` instead of `===`
- Magic strings/numbers without enums or named constants
- `console.log` in production; no structured logging with `traceId`
- PII in any log statement; components > 200 lines

### Report Format
**[SEVERITY]** — Title
📍 **Where**: `ComponentName` / line N
🔍 **Gap**: What is wrong and why.
✅ **Fix**: Corrected code.

🔴 Critical | 🟠 Major | 🟡 Minor

---

## MODE 2: GENERATION — Non-Negotiables

**HTML/CSS:** Semantic elements; `alt` on every `<img>`; `<label>` on every input; no `outline: none` without replacement; CSS custom properties for tokens; mobile-first media queries.

**JS/TS:** `const` by default; `===` always; `strict: true`; no `any`; `async/await` with `try/catch`; `Promise.all` for independent ops; no `console.log` — structured logger only.

**React/Next.js:** Stable `key` props; complete `useEffect` deps with cleanup; `React.lazy` for routes; `next/image`; `next/link`; no `NEXT_PUBLIC_` on secrets; auth on every API route.

**Angular/Vue:** `OnPush` change detection; `trackBy` on `*ngFor`; `async` pipe; no `v-html` with user content; `<script setup lang="ts">` in Vue 3.

**All:** No PII in logs; structured JSON logging with `traceId`; named constants; functions ≤ 30 lines; components ≤ 200 lines.

---

## Reference Files
| Topic | File |
|---|---|
| HTML5 & CSS | `references/html-css.md` |
| JavaScript & TypeScript | `references/js-ts.md` |
| JS core best practices | `references/js-best-practices.md` |
| JS advanced (strict, LogDNA, Sentry) | `references/js-advanced-practices.md` |
| ReactJS & NextJS | `references/react-next.md` |
| Angular & VueJS | `references/angular-vue.md` |
| Coding standards & naming | `references/coding-standards.md` |
| Error handling & logging | `references/error-handling-logging.md` |
| Performance gaps | `references/performance-gaps.md` |
| Performance best practices | `references/performance-best-practices.md` |
| SOLID & OOP | `references/solid-oop.md` |
| Unit testing | `references/unit-testing.md` |

---

## Instant Escalation — 🔴
- `dangerouslySetInnerHTML` / `v-html` with unsanitised user content
- Token or session ID in `localStorage`
- PII in any log statement
- `eval(userInput)` or `new Function(userInput)`
- `key={index}` on dynamically reordered list
- No pagination on unbounded list
