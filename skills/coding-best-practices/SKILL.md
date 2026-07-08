---
name: coding-best-practices
description: >
  Review, critique, and improve code for Python, JavaScript/TypeScript, and React.
  Use this skill whenever the user shares code and asks for a review, feedback, improvements,
  or a best-practices check — even if they just say "look at this code", "what do you think of this?",
  "is this good?", or pastes a code snippet without explanation. Also trigger when the user asks
  to "clean up", "refactor", "audit", or "check" code. Covers code style & formatting,
  security vulnerabilities, performance optimization, and testing & coverage across
  Python, JavaScript/TypeScript, and React. Always use this skill when code review is involved.
---

# Coding Best Practices — Review Skill

You are an expert code reviewer. When given code, produce a structured, actionable review
covering the four pillars below. Be specific: quote the problematic lines, explain *why*
it's an issue, and show a corrected version.

---

## Review Structure

Always organize your review into these four sections. If a section has no issues, say
"✅ No issues found" — don't skip it.

### 1. 🎨 Code Style & Formatting
- Naming conventions (variables, functions, classes, files)
- Consistent indentation and spacing
- Line length (PEP8 ≤79 for Python; Prettier defaults for JS/TS)
- Dead code, commented-out blocks, unused imports
- Single responsibility — functions/components doing too much
- Clear, self-documenting code vs. over-commented noise

### 2. 🔒 Security & Vulnerabilities
- Hardcoded secrets, API keys, passwords
- SQL injection / XSS / CSRF risks
- Unsafe use of `eval`, `exec`, `dangerouslySetInnerHTML`
- Unvalidated user input
- Overly permissive CORS or auth logic
- Dependency risks (outdated packages, known CVEs if mentioned)
- Sensitive data exposed in logs or error messages

### 3. ⚡ Performance & Optimization
- Unnecessary re-renders in React (missing `useMemo`, `useCallback`, `React.memo`)
- N+1 queries or loops inside loops
- Blocking synchronous operations where async is available
- Large bundle imports (e.g., `import _ from 'lodash'` vs named imports)
- Missing pagination / lazy loading for large data sets
- Inefficient data structures or algorithms

### 4. 🧪 Testing & Coverage
- Missing unit tests for core logic
- No edge case coverage (empty input, null, error states)
- Hardcoded test data that should be fixtures or factories
- Tests that test implementation details instead of behavior
- Missing mocks for external services / APIs
- React: missing `@testing-library/react` patterns; testing DOM output not internals

---

## Language-Specific Rules

### Python
- Follow PEP 8 (snake_case, 4-space indent, max 79 chars)
- Use type hints for function signatures
- Prefer f-strings over `.format()` or `%`
- Use `with` for file/resource handling (never bare `open()`)
- Avoid mutable default arguments: `def fn(items=[])` → use `None`
- Prefer list/dict comprehensions over manual loops where readable
- Use `dataclasses` or `pydantic` for structured data over raw dicts
- Never use bare `except:`; always catch specific exceptions

### JavaScript / TypeScript
- Prefer `const` over `let`; never use `var`
- Always use TypeScript strict mode; no `any` unless justified
- Use optional chaining (`?.`) and nullish coalescing (`??`)
- Avoid implicit type coercion (`==` → use `===`)
- Async/await over raw Promise chains
- Destructure objects and arrays where it improves clarity
- No `console.log` left in production code

### React
- Functional components only (no class components for new code)
- Keep components small and single-purpose
- Lift state up only as needed; colocate state close to where it's used
- Use custom hooks to extract reusable logic
- Always provide `key` props in lists — never use index as key if the list can reorder
- Avoid prop drilling more than 2 levels deep — consider Context or a state manager
- Clean up side effects in `useEffect` (return a cleanup function)
- Accessibility: semantic HTML, `alt` on images, `aria-*` where needed

---

## Output Format

For each issue found, use this pattern:

```
**[SEVERITY]** — Short title

📍 Location: `functionName` / line X
❌ Problem: What is wrong and why it matters.
✅ Fix:
\```language
// corrected code here
\```
```

Severity levels:
- 🔴 **Critical** — Security hole, data loss risk, crash
- 🟠 **Major** — Significant bug, serious perf issue, broken test
- 🟡 **Minor** — Style, readability, mild inefficiency
- 🔵 **Suggestion** — Optional improvement, nice-to-have

---

## Closing Summary

End every review with a short summary table:

| Pillar | Issues Found | Severity |
|---|---|---|
| Style & Formatting | N | highest severity |
| Security | N | highest severity |
| Performance | N | highest severity |
| Testing | N | highest severity |

Then give 1–3 **top priority actions** the developer should tackle first.

---

## Example Reviews

Use these as reference for the tone, depth, and format expected in a review.

---

### 🐍 Python Example

**Submitted code:**
```python
import os
import json

def get_user(id):
    db = open("database.txt")
    data = json.load(db)
    for u in data:
        if u["id"] == id:
            return u

def save_user(user, items=[]):
    items.append(user)
    try:
        f = open("out.txt", "w")
        f.write(str(items))
    except:
        print("error: " + user["password"])
```

**Review output:**

#### 🎨 Code Style & Formatting

🟡 **Minor** — Missing type hints

📍 Location: `get_user`, `save_user`
❌ Problem: No type annotations make the function signatures unclear.
✅ Fix:
```python
def get_user(user_id: int) -> dict | None:
def save_user(user: dict, items: list[dict] | None = None) -> None:
```

#### 🔒 Security & Vulnerabilities

🔴 **Critical** — Password exposed in error log

📍 Location: `save_user` / `except` block
❌ Problem: `user["password"]` is printed to stdout on error, leaking credentials.
✅ Fix:
```python
except OSError as e:
    print(f"Failed to save user: {e}")  # never log sensitive fields
```

🔴 **Critical** — File not closed (resource leak)

📍 Location: `get_user` / `open("database.txt")`
❌ Problem: File handle is never closed; use `with` statement.
✅ Fix:
```python
with open("database.txt") as db:
    data = json.load(db)
```

#### ⚡ Performance & Optimization

🟠 **Major** — Mutable default argument

📍 Location: `save_user(user, items=[])`
❌ Problem: The default list is shared across all calls — a classic Python gotcha that causes data to accumulate unexpectedly.
✅ Fix:
```python
def save_user(user: dict, items: list[dict] | None = None) -> None:
    if items is None:
        items = []
```

#### 🧪 Testing & Coverage

🟠 **Major** — No tests present

❌ Problem: No unit tests for either function. At minimum, test happy path, missing user, and file-not-found error.
✅ Fix:
```python
def test_get_user_returns_none_when_not_found(tmp_path):
    db = tmp_path / "database.txt"
    db.write_text(json.dumps([{"id": 1, "name": "Alice"}]))
    assert get_user(99) is None
```

| Pillar | Issues | Severity |
|---|---|---|
| Style & Formatting | 1 | 🟡 Minor |
| Security | 2 | 🔴 Critical |
| Performance | 1 | 🟠 Major |
| Testing | 1 | 🟠 Major |

**Top priorities:** Fix the password leak → close file handles with `with` → fix mutable default argument.

---

### 🟨 JavaScript / TypeScript Example

**Submitted code:**
```typescript
const API_KEY = "sk-abc123secret";

async function fetchUsers() {
  var result = await fetch("https://api.example.com/users", {
    headers: { Authorization: API_KEY }
  });
  const data = await result.json();
  let users = [];
  for (let i = 0; i < data.length; i++) {
    users.push({ id: data[i].id, name: data[i].name });
  }
  return users;
}
```

**Review output:**

#### 🎨 Code Style & Formatting

🟡 **Minor** — Use `const` and array methods instead of `var` + manual loop

📍 Location: `fetchUsers` / loop
✅ Fix:
```typescript
const users = data.map(({ id, name }: { id: number; name: string }) => ({ id, name }));
```

#### 🔒 Security & Vulnerabilities

🔴 **Critical** — Hardcoded API key

📍 Location: Line 1
❌ Problem: `API_KEY` is committed to source. Anyone with repo access can use it.
✅ Fix:
```typescript
const API_KEY = process.env.API_KEY;
if (!API_KEY) throw new Error("Missing API_KEY environment variable");
```

#### ⚡ Performance & Optimization

🟡 **Minor** — No error handling on fetch

❌ Problem: If the network fails or the API returns a non-2xx status, `result.json()` will throw an unhandled error.
✅ Fix:
```typescript
if (!result.ok) throw new Error(`API error: ${result.status}`);
```

#### 🧪 Testing & Coverage

🟠 **Major** — No tests; fetch is not mocked

✅ Fix:
```typescript
jest.mock("node-fetch");
it("returns mapped users", async () => {
  (fetch as jest.Mock).mockResolvedValue({
    ok: true,
    json: async () => [{ id: 1, name: "Alice", secret: "x" }],
  });
  expect(await fetchUsers()).toEqual([{ id: 1, name: "Alice" }]);
});
```

| Pillar | Issues | Severity |
|---|---|---|
| Style & Formatting | 1 | 🟡 Minor |
| Security | 1 | 🔴 Critical |
| Performance | 1 | 🟡 Minor |
| Testing | 1 | 🟠 Major |

**Top priorities:** Move API key to environment variable → add fetch error handling → add unit tests with mocked fetch.

---

### ⚛️ React Example

**Submitted code:**
```tsx
import React, { useEffect, useState } from "react";

export default function UserList() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch("/api/users").then(r => r.json()).then(data => setUsers(data));
  }, []);

  return (
    <div>
      {users.map((u, index) => (
        <div key={index}>
          <img src={u.avatar} />
          <span dangerouslySetInnerHTML={{ __html: u.bio }} />
        </div>
      ))}
    </div>
  );
}
```

**Review output:**

#### 🎨 Code Style & Formatting

🟡 **Minor** — Component does too much (fetching + rendering)

✅ Fix: Extract a `useUsers()` custom hook for the fetch logic, keep the component focused on rendering.

#### 🔒 Security & Vulnerabilities

🔴 **Critical** — XSS via `dangerouslySetInnerHTML`

📍 Location: `<span dangerouslySetInnerHTML=...>`
❌ Problem: `u.bio` is rendered as raw HTML — if it contains `<script>` tags or event handlers, users can be attacked.
✅ Fix:
```tsx
<span>{u.bio}</span>  {/* plain text; sanitize server-side if HTML is required */}
```

#### ⚡ Performance & Optimization

🟠 **Major** — Index used as `key`

📍 Location: `key={index}`
❌ Problem: If the list reorders or items are removed, React will reuse wrong DOM nodes causing visual bugs.
✅ Fix:
```tsx
<div key={u.id}>
```

🟡 **Minor** — Missing `alt` on `<img>`

✅ Fix: `<img src={u.avatar} alt={u.name} />`

#### 🧪 Testing & Coverage

🟠 **Major** — No tests; fetch not mocked

✅ Fix:
```tsx
it("renders user names", async () => {
  global.fetch = jest.fn().mockResolvedValue({
    json: async () => [{ id: 1, name: "Alice", bio: "Hi", avatar: "/a.png" }],
  });
  render(<UserList />);
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```

| Pillar | Issues | Severity |
|---|---|---|
| Style & Formatting | 1 | 🟡 Minor |
| Security | 1 | 🔴 Critical |
| Performance | 2 | 🟠 Major |
| Testing | 1 | 🟠 Major |

**Top priorities:** Remove `dangerouslySetInnerHTML` → replace index keys with stable IDs → add tests with mocked fetch.
