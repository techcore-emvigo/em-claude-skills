# Mobile Application — Gap Detection

Covers: performance/memory gaps, security gaps (MobSF, permissions, obfuscation,
remote config), release/versioning gaps, compatibility, analytics, iOS/Swift gaps.

---

# Mobile Application — Gap Detection

## Performance & Memory Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No memory profiling done | Memory Profiler (Android) / Xcode Instruments / MemLab not run | 🔴 |
| Memory leaks in component lifecycle | Objects held in memory after screen/component destroyed | 🔴 |
| No APM tool integrated | Firebase Performance / New Relic / Datadog not integrated | 🟠 |
| Large assets not compressed or lazy-loaded | Images, videos, fonts loaded eagerly at startup | 🟠 |
| Unoptimised list rendering | Long lists without virtualisation (FlatList, RecyclerView) | 🟠 |
| Excessive re-renders in state updates | State change triggers full tree re-render | 🟠 |
| No offline mode for core features | App unusable without connectivity | 🟡 |
| API calls not debounced/throttled | Rapid user interactions trigger excessive API calls | 🟠 |

**Tools:** Android Profiler, Xcode Instruments, MemLab, Firebase Performance, Flipper

---

## Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| MobSF scan not completed | Mobile app security vulnerabilities undetected | 🔴 |
| Sensitive data in local storage | Token, password, PII stored in AsyncStorage / SharedPreferences / NSUserDefaults | 🔴 |
| Base URL or API keys hardcoded in app bundle | Reverse engineering reveals credentials | 🔴 |
| Base URL not in remote config | Requires app store release to change environment/endpoint | 🟠 |
| Code obfuscation not enabled | ProGuard/R8 (Android) or Swift obfuscation not configured | 🔴 |
| Certificate pinning not implemented | Man-in-the-middle attacks possible | 🟠 |
| App permissions not minimised | Requesting camera, contacts, location without necessity | 🔴 |
| Deep links not validated | Malicious apps can trigger deep links | 🟠 |
| No jailbreak/root detection | Compromised device can access app data | 🟠 |
| No biometric/keychain for sensitive data | Sensitive data accessible without auth | 🟠 |

---

## Release & Versioning Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No app versioning (version code + version name) | Can't identify which version is in production | 🟠 |
| No force update mechanism | Users stuck on broken/insecure old versions | 🟠 |
| No staged rollout strategy | Bad release hits 100% of users immediately | 🟠 |
| Push notification token not refreshed on re-install | Stale tokens cause silent notification failures | 🟠 |
| Push notifications not handled securely | Token misuse; notification content exposed | 🟠 |

---

## Compatibility & Testing Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Not tested on minimum supported OS versions | Crashes on common older devices undetected | 🔴 |
| Not tested on various screen sizes and densities | UI breaks on specific devices | 🟠 |
| Not tested on low-end devices | Performance issues invisible on developer devices | 🟠 |
| No crash reporting integrated | Production crashes invisible | 🔴 |
| UI not tested for accessibility (screen readers, font scaling) | App unusable for accessibility needs | 🟠 |

---

## Analytics & Behaviour Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No mobile analytics implemented | Feature decisions made without real data | 🟡 |
| User behaviour not reviewed before release | Usage patterns unknown | 🟡 |
| No funnel tracking on key flows | Drop-offs in critical flows invisible | 🟡 |

---

## Generation Checklist

- [ ] Code obfuscation enabled (ProGuard/R8 for Android, obfuscation for iOS)
- [ ] No secrets, tokens, or base URLs hardcoded — use remote config
- [ ] Sensitive data stored in secure keychain/keystore, not local storage
- [ ] App versioning configured: version code (integer) + version name (string)
- [ ] Force update check on app launch against remote config
- [ ] Permissions declared in manifest/Info.plist minimised to actual needs
- [ ] Memory Profiler run and no leaks found before release
- [ ] Firebase Performance or equivalent APM integrated
- [ ] Crash reporting (Firebase Crashlytics, Sentry) integrated
- [ ] MobSF scan completed and high/critical findings resolved
- [ ] App tested on minimum supported OS version and multiple screen sizes

---

