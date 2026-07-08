# em-mobile — Rules Reference

## 1. Security
| Rule | Description | Severity |
|---|---|---|
| MB-01 | Sensitive data (token, PIN, password) in Keychain/Keystore — never `UserDefaults`/`SharedPreferences` | 🔴 |
| MB-02 | No secrets, API keys, or credentials hardcoded in bundle | 🔴 |
| MB-03 | Base URL and environment config in remote config — not compiled into app | 🟠 |
| MB-04 | HTTPS enforced — `http://` URLs forbidden in any network config | 🔴 |
| MB-05 | ATS never disabled — `NSAllowsArbitraryLoads = false` always | 🔴 |
| MB-06 | SSL/certificate pinning active in production builds | 🟠 |
| MB-07 | Code obfuscation enabled — ProGuard/R8 (Android), Swift obfuscation (iOS) | 🔴 |
| MB-08 | App permissions minimised to actual feature requirements | 🔴 |
| MB-09 | Payment SDK (Stripe, GooglePay) called from backend only — never from screen | 🔴 |
| MB-10 | Card numbers, CVVs, bank data never logged | 🔴 |
| MB-11 | `WRITE_EXTERNAL_STORAGE` not in manifest for Android API 29+ | 🔴 |

## 2. Performance & Memory
| Rule | Description | Severity |
|---|---|---|
| MB-12 | Memory Profiler (Android) / Xcode Instruments / MemLab run before release | 🔴 |
| MB-13 | Long lists use `FlatList` / `RecyclerView` with `keyExtractor` — no `.map()` rendering | 🟠 |
| MB-14 | No heavy computation on main thread — offload to worker | 🟠 |
| MB-15 | API calls debounced/throttled on rapid user interactions | 🟠 |
| MB-16 | Firebase Performance or equivalent APM integrated | 🟠 |

## 3. React Native
| Rule | Description | Severity |
|---|---|---|
| RN-01 | No inline styles — `StyleSheet.create()` only | 🟡 |
| RN-02 | No raw RN primitives (`Text`, `View`, `Button`) in screen/container files | 🟠 |
| RN-03 | Screen → Container → Component → Micro-component pattern enforced | 🟠 |
| RN-04 | API calls via Redux/Zustand actions from service class — not direct axios in screen | 🟠 |
| RN-05 | `key` props use stable unique IDs — never array index | 🟠 |
| RN-06 | No `console.log` — structured logger or Sentry only | 🟡 |
| RN-07 | No unused imports; no stray debug statements | 🟡 |
| RN-08 | Constants in domain constants file — no magic strings in components | 🟡 |

## 4. iOS / Swift
| Rule | Description | Severity |
|---|---|---|
| IOS-01 | No force unwrap `!` in production — `guard let` or optional chaining | 🟠 |
| IOS-02 | Protocol conformance in separate `extension` with `// MARK: -` | 🟡 |
| IOS-03 | `let` by default; `var` only when mutation required | 🟡 |
| IOS-04 | `[weak self]` in all Timer, NotificationCenter, and async closures | 🟠 |
| IOS-05 | `os.Logger` with subsystem/category — no `print()` | 🟡 |
| IOS-06 | No PII in any log statement — mask email, phone, token | 🔴 |
| IOS-07 | Environment profiling with `#if DEBUG` / `#if STAGING` | 🟠 |
| IOS-08 | Delegate methods include delegate source as unnamed first parameter | 🟡 |

## 5. Release Standards
| Rule | Description | Severity |
|---|---|---|
| MB-17 | App version code (integer) and version name (string) configured | 🟠 |
| MB-18 | Force-update mechanism implemented for critical releases | 🟠 |
| MB-19 | Crash reporting (Firebase Crashlytics / Sentry) integrated | 🔴 |
| MB-20 | Tested on minimum supported OS versions and multiple screen sizes | 🔴 |
| MB-21 | Push notification tokens not logged or exposed | 🟠 |

## 6. Logging & Quality
| Rule | Description | Severity |
|---|---|---|
| MB-22 | No PII (email, phone, name, token) in any log statement | 🔴 |
| MB-23 | No `console.log` / `print()` in production — structured logger only | 🟡 |
| MB-24 | Sentry integrated and wrapping root component | 🔴 |
| MB-25 | Functions ≤ 30 lines; files ≤ 200 lines | 🟠 |
| MB-26 | Named constants/enums — no magic strings or numbers in components | 🟡 |

## 7. Instant Escalation — 🔴
| # | Violation |
|---|---|
| ESC-01 | Sensitive data in `UserDefaults` / `SharedPreferences` |
| ESC-02 | API key, secret, or base URL hardcoded in bundle |
| ESC-03 | `NSAllowsArbitraryLoads = true` in any build config |
| ESC-04 | Code obfuscation not enabled in production |
| ESC-05 | HTTP used anywhere in mobile network config |
| ESC-06 | Payment SDK called from screen — secret key in bundle |
| ESC-07 | PII in any log statement |
| ESC-08 | Crash reporting not integrated |
