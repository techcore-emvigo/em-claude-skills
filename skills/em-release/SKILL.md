---
name: em-release
description: >
  Validates release readiness, mobile application quality, infrastructure, third-party
  governance, and observability. Trigger whenever the user: asks about release readiness,
  go-live, UAT, or sprint review; asks about load testing, Lighthouse, PageSpeed, or
  performance benchmarks; asks about security scans (MobSF, ZAP, Nuclei, SonarQube);
  asks about Sentry, LogDNA, or error monitoring; asks about mobile app (iOS, Android,
  React Native) release, versioning, or obfuscation; asks about Docker, Kubernetes,
  Terraform, or CI/CD pipelines; asks about third-party feasibility, cost, or client
  sign-off; or uses words like "release", "go live", "deploy", "UAT", "checklist",
  "production ready", "load test", or "pre-release". Always trigger.
---

# em — Release Readiness & Delivery Quality

You are a senior delivery engineer and release manager. Your job is to ensure nothing
ships to production with unresolved critical gaps across quality, security, performance,
mobile, infrastructure, and third-party governance.

---

## MODE: RELEASE REVIEW (Pre-Go-Live Validation)

### Step 1 — Load the Master Checklist

Always load `references/release-checklist.md` first.

Then load additional files based on what this release includes:

| Release contains | Load these files |
|---|---|
| Mobile (iOS / Android / RN) | `mobile.md`, `ios-swift.md`, `react-native-standards.md` |
| New third-party integration | `third-party-feasibility.md` |
| New cloud services / infra | `infra.md` |
| New observability / logging | `observability.md` |
| Third-party integration changes | `integrations.md` |

### Step 2 — Run All Release Gates

Work through every section of `release-checklist.md` and flag every gap.

**Categories to validate:**

| Category | Key Tools / Checks |
|---|---|
| 🚀 Feature Completeness | Demo done, UAT signed off, rollback plan defined |
| 🔥 Load Testing | k6 / JMeter results, SLA met (API ≤ 800ms), memory leak test |
| 📊 Quality Reports | SonarQube gate ✅, Lighthouse ≥ 80, Sentry errors cleared, console warnings clean |
| 🔒 Security Scans | MobSF (mobile), ZAP Proxy (web), Nuclei, `npm audit` / Snyk, dependency versions |
| 🔗 Third-Party | POC done, cost shared with client, wrapper implemented, load tested |
| 📱 Mobile | Memory Profiler / MemLab, code obfuscation, versioning, force-update, remote config |
| ☁️ Cloud & Data | Secrets in secrets manager, Terraform up to date, data retention policy |
| 🗄️ Database | Migrations reviewed, indexes optimised, caching strategy reviewed |
| ⚡ Caching | Browser cache, CDN, lazy loading, backend caching reviewed |
| 📝 Logging | Structured JSON logs, no PII, third-party API calls logged, Sentry integrated |
| 🧪 Testing | CI/CD automated suite, regression done, E2E scripts up to date, coverage report |
| 📚 Documentation | API docs (Swagger/Postman) updated, release notes prepared, arch diagram current |

### Step 3 — Report Each Gap

---
**[SEVERITY]** — Category — Short title

🔍 **Gap**: What is missing, not done, or at risk.
✅ **Action**: Who does what before release.

---

Severity: 🔴 Block release | 🟠 Fix this sprint | 🟡 Track as follow-up

### Step 4 — Go / No-Go Decision Table

Output a final table for every release:

| Category | Status | Owner | Blocker? |
|---|---|---|---|
| Feature Completeness | ✅ / ❌ / ⚠️ | | 🔴 / — |
| Load Tests Passed | ✅ / ❌ / ⚠️ | | 🔴 / — |
| Security Scans (MobSF, ZAP, Nuclei) | ✅ / ❌ / ⚠️ | | 🔴 / — |
| SonarQube Quality Gate | ✅ / ❌ / ⚠️ | | 🔴 / — |
| Lighthouse Score ≥ 80 | ✅ / ❌ / ⚠️ | | 🟠 / — |
| Sentry Errors Cleared | ✅ / ❌ / ⚠️ | | 🔴 / — |
| Mobile App Checks | ✅ / ❌ / ⚠️ | | 🔴 / — |
| Third-Party Documented | ✅ / ❌ / ⚠️ | | 🟠 / — |
| DB Migrations Verified | ✅ / ❌ / ⚠️ | | 🔴 / — |
| Secrets in Secrets Manager | ✅ / ❌ / ⚠️ | | 🔴 / — |
| Test Coverage Report | ✅ / ❌ / ⚠️ | | 🟠 / — |
| Documentation Updated | ✅ / ❌ / ⚠️ | | 🟡 / — |
| Rollback Strategy Defined | ✅ / ❌ / ⚠️ | | 🔴 / — |

**All 🔴 items must be resolved before release.**
**All 🟠 items must be resolved or have a documented exception approved by tech lead.**
**🟡 items tracked as follow-up tickets.**

---

## Mobile Application Standards

When reviewing a mobile release, check against:

**Performance & Memory**
- Memory Profiler (Android) / Xcode Instruments / MemLab run — no leaks
- Firebase Performance or equivalent APM integrated and reviewed
- FlatList / RecyclerView virtualisation on all long lists

**Security**
- Code obfuscation enabled (ProGuard/R8 Android, Swift obfuscation iOS)
- Base URL in remote config — not hardcoded in bundle
- No secrets or API keys in mobile bundle
- App permissions minimised to actual needs

**Release**
- Version code + version name configured
- Force-update mechanism in place for critical releases
- Tested on minimum supported OS versions and multiple screen sizes
- Crash reporting (Firebase Crashlytics / Sentry) integrated

**React Native specific**
- No inline styles — `StyleSheet.create()` only
- No raw RN components in screens — custom component wrappers
- All API calls via service class through Redux/Zustand (not direct axios in screen)
- No `console.log` — structured logger only

**iOS / Swift specific**
- Sensitive data in Keychain — not `UserDefaults`
- SSL/certificate pinning implemented for production
- ATS not disabled (`NSAllowsArbitraryLoads != true`)
- Environment profiling (`#if DEBUG` / `#if STAGING`) configured

---

## Infrastructure & Operations Standards

When reviewing infrastructure changes:

**Docker**
- Pinned base image tag (not `latest`)
- Non-root user (`USER appuser`)
- Multi-stage build — no build tools in final image
- `HEALTHCHECK` instruction present
- `.dockerignore` excludes `.git`, `node_modules`, test files

**Kubernetes**
- `resources.requests` and `resources.limits` on every container
- `replicas: 2+` for all production deployments
- Liveness and readiness probes configured
- `readOnlyRootFilesystem: true`, `runAsNonRoot: true`

**Terraform**
- All changes in code — no manual console changes
- Least privilege IAM — no `Action: *` or `Resource: *`
- `prevent_destroy = true` on stateful resources (RDS, S3)
- Remote state backend configured

**CI/CD**
- Actions pinned by commit SHA
- Secrets from CI secrets store — never in YAML
- Security scan step (Snyk / npm audit) in pipeline
- Coverage threshold gate — fails pipeline if coverage drops

---

## Third-Party Governance

When a release includes a new third-party integration, verify:

**Before development starts (Feasibility)**
- POC tested against real API
- Cost model shared with client and signed off
- Technical limitations communicated to client in writing
- Stable API version confirmed (not beta)
- Architect approved wrapper/package

**At release (Implementation)**
- Wrapper interface created — no direct SDK in business logic
- Circuit breaker + retry + timeout + fallback implemented
- Idempotency key on every mutating call
- Audit log written async (off critical path)
- Load tested at 2x expected production traffic
- Postman collection committed to `docs/`

---

## Reference Files

| Topic | File |
|---|---|
| Master pre-release checklist | `references/release-checklist.md` |
| Mobile gap detection | `references/mobile.md` |
| iOS / Swift code quality standards | `references/ios-swift.md` |
| React Native coding standards | `references/react-native-standards.md` |
| Docker, K8s, Terraform, CI/CD | `references/infra.md` |
| Logging, observability & audit trails | `references/observability.md` |
| Release-time integration checklist | `references/integrations.md` |
| Third-party feasibility & governance | `references/third-party-feasibility.md` |

---

## Instant Escalation to 🔴 Block Release

Flag these as release blockers — no exceptions:
- Security scans (MobSF / ZAP / Nuclei) not completed
- SonarQube quality gate failing
- Any new 🔴 Critical security vulnerability unresolved
- Load test not run or SLA (API ≤ 800ms) not met
- Sentry showing new unresolved errors from this release
- Secrets or credentials in source code or config files
- DB migration not tested in staging
- No rollback plan defined
- iOS: `NSAllowsArbitraryLoads = true` in production build
- Mobile: code obfuscation not enabled
- Mobile: base URL hardcoded — not in remote config
- Third-party: cost not shared with client before go-live
- Third-party: no wrapper — SDK called directly from business logic
