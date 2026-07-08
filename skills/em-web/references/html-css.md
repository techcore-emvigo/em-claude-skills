# HTML5 & CSS — Gap Detection

## HTML Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `<div onClick>` for action | Clickable `<div>` or `<span>` instead of `<button>` — not keyboard accessible | 🟠 |
| `<a>` without `href` for action | `<a onclick="...">` with no `href` — use `<button>` | 🟠 |
| Missing `alt` on `<img>` | `<img src="...">` with no `alt` attribute | 🟠 |
| `<table>` for layout | Non-data table used for visual layout | 🟡 |
| Missing `<label>` for input | `<input>` without associated `<label for="id">` or `aria-label` | 🟠 |
| Placeholder as only label | Relying on `placeholder` text instead of a visible label | 🟠 |
| Missing `lang` on `<html>` | `<html>` without `lang="en"` (or appropriate language) | 🟡 |
| Missing viewport meta | No `<meta name="viewport" content="width=device-width, initial-scale=1">` | 🟠 |
| `outline: none` with no replacement | Focus style removed with no custom replacement — keyboard inaccessible | 🟠 |
| `innerHTML = userInput` | User content set via `innerHTML` without sanitization | 🔴 |
| Skipped heading level | `<h1>` followed by `<h3>` — skips `<h2>` | 🟡 |
| Missing charset meta | No `<meta charset="UTF-8">` — encoding issues | 🟡 |

## CSS Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `!important` overuse | More than 1-2 `!important` in application styles | 🟡 |
| Magic number in layout | `margin-top: 37px` with no variable/comment | 🟡 |
| Inline styles in HTML | `style="color: red"` attributes on elements | 🟡 |
| Deep selector nesting | CSS selectors 4+ levels deep: `.a .b .c .d .e { }` | 🟡 |
| Fixed pixel widths on layout | `width: 1200px` on a container — breaks on small screens | 🟠 |
| No CSS custom properties for tokens | Colors, spacing, fonts hardcoded as literals throughout CSS | 🟡 |
| Layout-triggering animation | `transition: top`, `transition: left` instead of `transform` | 🟡 |

## Generation Checklist
- [ ] Semantic elements used (`<main>`, `<nav>`, `<article>`, `<section>`, `<button>`)
- [ ] Every `<img>` has a meaningful `alt` attribute
- [ ] Every form input has an associated `<label>`
- [ ] Focus styles visible and not removed without replacement
- [ ] CSS custom properties for colors, spacing, and typography tokens
- [ ] Mobile-first with `min-width` media queries
- [ ] Touch targets minimum 44×44px
