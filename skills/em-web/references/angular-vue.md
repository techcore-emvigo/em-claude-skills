# Angular & VueJS — Gap Detection

## Angular Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Missing `OnPush` change detection | Component with `ChangeDetectionStrategy.Default` receiving immutable inputs | 🟡 |
| Subscription not unsubscribed | `this.service.obs$.subscribe(...)` without `takeUntilDestroyed()` or `async` pipe | 🟠 |
| Logic in component | HTTP calls, business rules, or DB logic inside `@Component` class | 🟠 |
| `bypassSecurityTrustHtml` | `domSanitizer.bypassSecurityTrustHtml(userInput)` — XSS | 🔴 |
| No `trackBy` on `*ngFor` | `*ngFor="let item of items"` without `trackBy` on a list that can change | 🟡 |
| Nested subscriptions | `outer$.subscribe(() => inner$.subscribe(...))` — use `switchMap` | 🟠 |
| `index` as trackBy | `trackBy: (i) => i` — same problem as React `key={index}` | 🟠 |
| HTTP call in component | `this.http.get(...)` in a component instead of a service | 🟠 |

## VueJS (Vue 3) Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `v-html` with user content | `v-html="userInput"` without sanitization | 🔴 |
| `v-for` without `:key` | `v-for="item in items"` missing `:key` | 🟠 |
| `:key="index"` | `v-for` using loop index as key on dynamic list | 🟠 |
| Reactive object destructured | `const { count } = reactive(state)` — loses reactivity | 🟠 |
| Props mutated directly | `props.value = newVal` instead of emitting | 🟠 |
| Options API in new code | Using Options API (`data()`, `methods:`) in new Vue 3 components | 🟡 |
| Missing `defineEmits` types | `defineEmits(["click"])` without TypeScript event types | 🟡 |
| Pinia state mutated outside action | Direct mutation of store state outside an action | 🟠 |

## Generation Checklist
- [ ] Angular: `ChangeDetectionStrategy.OnPush` + `async` pipe or `takeUntilDestroyed()`
- [ ] Angular: `trackBy` on all `*ngFor` with a stable unique ID
- [ ] Vue: `<script setup lang="ts">` with typed `defineProps` and `defineEmits`
- [ ] Vue: `storeToRefs()` when destructuring Pinia store state
- [ ] Both: no `v-html` / `bypassSecurityTrust` on user-supplied content
