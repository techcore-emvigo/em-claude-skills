# JavaScript & TypeScript — Gap Detection

## JavaScript Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `var` usage | `var x =` anywhere | 🟡 |
| Loose equality | `==` instead of `===` | 🟡 |
| Implicit type coercion | `if (value)` for null/undefined checks on non-boolean | 🟡 |
| `console.log` in production code | `console.log(`, `console.error(` left in source | 🟡 |
| No error handling on async | `asyncFn()` without `await`, promise chain without `.catch()` | 🟠 |
| `eval()` with dynamic input | `eval(userString)` | 🔴 |
| Mutation of function argument | `function fn(arr) { arr.push(x) }` — mutates caller's reference | 🟡 |
| Sequential awaits for independent work | `await a(); await b()` when a and b are independent | 🟠 |
| Prototype pollution risk | `obj[req.body.key] = value` without key validation | 🔴 |

## TypeScript Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `any` type | `: any`, `as any` | 🟠 |
| Missing strict mode | `"strict": false` or absent in `tsconfig.json` | 🟠 |
| Type assertion instead of guard | `value as User` without verifying shape | 🟠 |
| Missing return type on public fn | Exported function with no explicit return type | 🟡 |
| `!` non-null assertion without guard | `user!.email` without a prior null check | 🟠 |
| `@ts-ignore` / `@ts-expect-error` | Suppressing type errors without a comment explaining why | 🟠 |
| `interface` used for union types | `interface Result { data?: X; error?: Y }` instead of discriminated union | 🟡 |
| `object` / `{}` type | Using `object` or `{}` instead of a specific type | 🟡 |

## Generation Checklist
- [ ] `const` by default, `let` only when reassigned, never `var`
- [ ] `===` everywhere
- [ ] `async/await` with `try/catch` on every async operation
- [ ] TypeScript `strict: true` always
- [ ] No `any` — use `unknown` + type guard, or a proper type
- [ ] Discriminated unions over optional fields for variant types
- [ ] Named exports for testability; avoid default exports in libs
