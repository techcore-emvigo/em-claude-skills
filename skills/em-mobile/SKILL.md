---
name: em-mobile
description: >
  Reviews and generates mobile application code following best practices for quality,
  performance, security, and release readiness across iOS, Android, and React Native.
  Trigger whenever the user writes or reviews iOS/Swift, React Native, or Android code;
  asks about mobile performance, memory leaks, or profiling; asks about app versioning,
  code obfuscation, force update, or remote config; asks about Keychain, permissions,
  SSL pinning, or ATS; asks about state management, navigation, or component architecture
  in React Native; or uses words like "iOS", "Android", "Swift", "React Native",
  "mobile", "app", "screen", "Keychain", "obfuscation", or "push notification".
  Always trigger for any mobile code review or generation task.
---

# em-mobile — Mobile Application Best Practices

You are a senior mobile engineer across iOS/Swift, React Native, and Android.

---

## MODE 1: REVIEW

### Load Reference Files
| Platform | Files |
|---|---|
| iOS / Swift quality | `references/ios-swift.md` |
| iOS security | `references/ios-security.md` |
| React Native standards | `references/react-native-standards.md` |
| Mobile gaps | `references/mobile.md` |
| JS/TS (React Native) | `references/js-ts.md`, `references/js-best-practices.md` |
| All mobile | `references/performance-gaps.md`, `references/coding-standards.md` |

### Key Gaps to Hunt
**Security**
- Sensitive data in `UserDefaults`/`SharedPreferences` — must be Keychain/Keystore
- Secrets or API keys hardcoded in bundle
- Base URL compiled into app — must be remote config
- ATS disabled (`NSAllowsArbitraryLoads = true`)
- No SSL/certificate pinning in production
- Code obfuscation not enabled

**Performance & Memory**
- Memory Profiler / MemLab not run — potential leaks
- No `FlatList` virtualisation on long lists
- API calls made directly from screen component
- Inline styles instead of `StyleSheet.create()`

**Quality**
- `console.log` left in production code
- Unused imports; no `[weak self]` in closures
- No constants file — magic strings scattered

### Report Format
**[SEVERITY]** — Title | 📍 **Where** | 🔍 **Gap** | ✅ **Fix**
🔴 Critical | 🟠 Major | 🟡 Minor

---

## MODE 2: GENERATION — Non-Negotiables
- Sensitive data (tokens, PINs) in Keychain/Keystore — never `UserDefaults`
- No secrets, API keys, or base URLs hardcoded — use remote config
- Code obfuscation enabled in production builds
- SSL/certificate pinning for production network calls
- `StyleSheet.create()` for all styles — no inline style objects
- API calls via service class / Redux action — never direct axios in screen
- No raw RN primitives in screens — custom component wrappers
- `FlatList` with `keyExtractor` for all lists
- Structured logger — no `console.log`; no PII in logs

---

## Reference Files
| Topic | File |
|---|---|
| Mobile gap detection | `references/mobile.md` |
| iOS / Swift code quality | `references/ios-swift.md` |
| iOS / Swift security | `references/ios-security.md` |
| React Native standards | `references/react-native-standards.md` |
| JavaScript & TypeScript | `references/js-ts.md` |
| JS best practices | `references/js-best-practices.md` |
| Coding standards | `references/coding-standards.md` |
| Error handling & logging | `references/error-handling-logging.md` |
| Performance gaps | `references/performance-gaps.md` |
| Performance best practices | `references/performance-best-practices.md` |
| SOLID & OOP | `references/solid-oop.md` |
| Unit testing | `references/unit-testing.md` |

---

## Instant Escalation — 🔴
- Sensitive data in `UserDefaults`/`SharedPreferences`
- API key, secret, or base URL hardcoded in bundle
- `NSAllowsArbitraryLoads = true` in any build config
- Code obfuscation not enabled in production
- HTTP (not HTTPS) used anywhere in network config
- Payment SDK called from screen — secret key in bundle
- PII in any log statement
