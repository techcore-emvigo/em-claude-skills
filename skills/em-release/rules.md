# em-release — Rules Reference

Non-negotiable rules enforced before every release, sprint completion, and go-live.
Every 🔴 item is a release blocker. Every 🟠 item must be resolved or have a
documented exception approved by the Tech Lead before deployment.

Prefix: **RL** = Release Process | **LT** = Load Testing | **QR** = Quality Reports |
**SC** = Security Scans | **TP** = Third-Party | **MB** = Mobile |
**CI** = Cloud & Infrastructure | **DB** = Database & Migrations |
**CH** = Caching & Performance | **LG** = Logging & Compatibility |
**TT** = Testing | **DC** = Documentation

---

## 1. Release Process & Feature Completeness

| Rule | Description | Severity |
|---|---|---|
| RL-01 | All release features completed, reviewed, and demoed before go-live | 🔴 |
| RL-02 | UAT (User Acceptance Testing) signed off before release | 🔴 |
| RL-03 | Release checklist created and every item verified before deployment | 🔴 |
| RL-04 | Rollback or hotfix strategy defined and documented before deployment | 🔴 |
| RL-05 | Stakeholders briefed on release features and timelines | 🟠 |
| RL-06 | Release notes prepared and shared with client/end-users | 🟡 |
| RL-07 | Post-release tasks documented and assigned to owners | 🟡 |
| RL-08 | Common/shared story list reviewed and confirmed covered | 🟠 |
| RL-09 | All tech debt gaps identified in this release added to backlog | 🟠 |
| RL-10 | Client aware of all tech debt stories related to this release | 🟠 |

---

## 2. Load Testing & Performance Benchmarks

| Rule | Description | Severity |
|---|---|---|
| LT-01 | Load tests executed for all new web pages and APIs in this release | 🔴 |
| LT-02 | Real-world traffic patterns simulated — not synthetic flat load | 🟠 |
| LT-03 | Performance benchmarks defined for all pages and APIs (response time SLA) | 🟠 |
| LT-04 | All API response times within SLA — ≤ 800ms for Lambda and GraphQL endpoints | 🔴 |
| LT-05 | Memory leak detection run during load testing — no growth under sustained load | 🔴 |
| LT-06 | Stress test run to identify system breaking point | 🟠 |
| LT-07 | Soak test run for resilience under sustained load | 🟠 |
| LT-08 | Load test results documented for future regression baseline | 🟡 |
| LT-09 | Google Lighthouse run on all pages in this release | 🟠 |
| LT-10 | Lighthouse Performance score ≥ 80, Accessibility score ≥ 90 | 🟠 |
| LT-11 | Network waterfall reviewed — bundle sizes, TTFB, render-blocking resources | 🟠 |
| LT-12 | Mobile Memory Profiler / MemLab run for mobile releases — no leaks found | 🔴 |

---

## 3. Quality Reports & Audits

| Rule | Description | Severity |
|---|---|---|
| QR-01 | SonarQube quality gate passed — report reviewed before release | 🔴 |
| QR-02 | SonarQube integrated in all repositories — no repo exempted | 🟠 |
| QR-03 | All SonarQube issues resolved — no open issues in release scope | 🟠 |
| QR-04 | Sentry error report reviewed — all new errors from this release addressed | 🔴 |
| QR-05 | Browser console errors and warnings cleared before release | 🟠 |
| QR-06 | Static analysis (ESLint, Pylint) passing — no new violations | 🟡 |
| QR-07 | Quality gate blocks CI merge on failure — not bypassed | 🟠 |
| QR-08 | Key findings from reports shared with the full development team | 🟡 |

---

## 4. Security Scans

| Rule | Description | Severity |
|---|---|---|
| SC-01 | MobSF scan completed for every mobile application release | 🔴 |
| SC-02 | OWASP ZAP Proxy scan completed on web application | 🔴 |
| SC-03 | Nuclei scan completed — template-based vulnerability check | 🔴 |
| SC-04 | Dependency vulnerability scan run — `npm audit`, `pip audit`, or Snyk | 🔴 |
| SC-05 | All high/critical CVEs in dependencies resolved before release | 🔴 |
| SC-06 | Library and plugin versions reviewed — no outdated or known-vulnerable packages | 🔴 |
| SC-07 | Security patches applied for all framework and server components | 🔴 |
| SC-08 | CSP (Content Security Policy) header implemented for XSS prevention | 🔴 |
| SC-09 | MFA enforced for all admin access | 🔴 |
| SC-10 | Security scan results reviewed and all critical findings resolved | 🔴 |

---

## 5. Third-Party Integrations

| Rule | Description | Severity |
|---|---|---|
| TP-01 | POC built and tested against real API before integration committed | 🟠 |
| TP-02 | All new integrations documented — method, auth, endpoints, data flow | 🟠 |
| TP-03 | Cost model shared with client before development started — sign-off obtained | 🔴 |
| TP-04 | All technical limitations and challenges communicated to client in writing | 🟠 |
| TP-05 | Wrapper interface created — no direct SDK calls in business logic or UI | 🔴 |
| TP-06 | OAuth2 or equivalent secure auth used — no plain API key without rotation plan | 🟠 |
| TP-07 | Circuit breaker + timeout + retry + fallback implemented | 🔴 |
| TP-08 | Idempotency key on every mutating call (charge, send, publish) | 🔴 |
| TP-09 | Third-party integrations load tested at 2× expected production traffic | 🟠 |
| TP-10 | Errors and timeouts handled gracefully — no third-party failure crashes the feature | 🔴 |
| TP-11 | Audit log written async (off critical path) for every request/response | 🔴 |
| TP-12 | Data privacy policy of third-party reviewed for GDPR/CCPA compliance | 🔴 |
| TP-13 | Approved/latest stable API version in use — no beta or deprecated endpoints | 🟠 |
| TP-14 | Postman collection maintained and committed to `docs/` | 🟡 |
| TP-15 | Architect approval obtained for any new external package or wrapper | 🟠 |

---

## 6. Mobile Application

| Rule | Description | Severity |
|---|---|---|
| MB-01 | Memory Profiler (Android) / Xcode Instruments / MemLab run — no leaks | 🔴 |
| MB-02 | App tested on minimum supported OS versions (iOS and Android) | 🔴 |
| MB-03 | App tested on multiple screen sizes and densities | 🟠 |
| MB-04 | Firebase Performance or equivalent APM reviewed | 🟠 |
| MB-05 | App permissions minimised to actual feature requirements | 🔴 |
| MB-06 | App versioning configured — version code (integer) and version name (string) | 🟠 |
| MB-07 | Force-update mechanism in place for critical releases | 🟠 |
| MB-08 | Code obfuscation enabled — ProGuard/R8 (Android), Swift obfuscation (iOS) | 🔴 |
| MB-09 | Base URL and environment config in remote config — not hardcoded in bundle | 🟠 |
| MB-10 | No secrets or API keys compiled into mobile bundle | 🔴 |
| MB-11 | Push notifications handled securely — token not logged or exposed | 🟠 |
| MB-12 | Crash reporting (Firebase Crashlytics / Sentry) integrated and active | 🔴 |
| MB-13 | `WRITE_EXTERNAL_STORAGE` not in Android manifest for API 29+ targets | 🔴 |
| MB-14 | iOS: `NSAllowsArbitraryLoads = false` — ATS not disabled in production build | 🔴 |
| MB-15 | iOS: sensitive data in Keychain — not `UserDefaults` | 🔴 |
| MB-16 | iOS: SSL/certificate pinning active in production build | 🟠 |
| MB-17 | React Native: no inline styles — `StyleSheet.create()` only | 🟡 |
| MB-18 | React Native: no raw RN primitives in screens — custom component wrappers | 🟠 |
| MB-19 | React Native: API calls via service class / Redux action — not direct axios in screen | 🟠 |

---

## 7. Cloud Services & Infrastructure

| Rule | Description | Severity |
|---|---|---|
| CI-01 | All new cloud services in this release documented and tracked | 🟠 |
| CI-02 | Secrets and credentials stored in secrets manager — never in code or config files | 🔴 |
| CI-03 | No sensitive data stored in local storage, cookies, or committed `.env` files | 🔴 |
| CI-04 | Terraform scripts updated to reflect current infrastructure state | 🟠 |
| CI-05 | No manual console changes — all infra changes in code (IaC) | 🟠 |
| CI-06 | Least privilege applied to all new IAM roles — no `Action: *` or `Resource: *` | 🔴 |
| CI-07 | Cloud-stored data backed up and backup recovery verified | 🔴 |
| CI-08 | Data retention policies applied to all new data stores | 🟠 |
| CI-09 | Cloud service costs monitored — budget alerts configured | 🟠 |
| CI-10 | Docker: pinned base image tag, non-root user, HEALTHCHECK instruction | 🟠 |
| CI-11 | Kubernetes: resource limits, replicas ≥ 2, liveness + readiness probes configured | 🟠 |
| CI-12 | CI/CD: actions pinned by SHA, secrets from store, security scan step present | 🟠 |
| CI-13 | Coverage threshold gate in CI — pipeline fails if coverage drops below threshold | 🟠 |

---

## 8. Database & Migrations

| Rule | Description | Severity |
|---|---|---|
| DB-01 | All new queries optimised and reviewed for missing indexes | 🔴 |
| DB-02 | Database changes explained — indexes added/removed, schema changes documented | 🟠 |
| DB-03 | Migration scripts reviewed for data loss risk | 🔴 |
| DB-04 | Migration scripts tested in staging — not just local | 🔴 |
| DB-05 | Every migration has a rollback (`down`) counterpart | 🟠 |
| DB-06 | Database backups automated and recovery verified in staging | 🔴 |
| DB-07 | Database caching strategy reviewed — Redis, query cache | 🟠 |
| DB-08 | No manual `ALTER TABLE` in production — all changes via migration file | 🔴 |

---

## 9. Caching & Frontend Performance

| Rule | Description | Severity |
|---|---|---|
| CH-01 | Caching mechanisms reviewed for all slow or frequently repeated reads | 🟠 |
| CH-02 | Browser caching enabled for static assets — `Cache-Control` headers configured | 🟡 |
| CH-03 | CDN configured for static assets where applicable | 🟡 |
| CH-04 | Lazy loading implemented for images and heavy below-fold content | 🟡 |
| CH-05 | Backend Redis caching implemented for expensive repeated computations | 🟠 |
| CH-06 | No full-page spinners — skeleton screens per component | 🟡 |
| CH-07 | No excessive API calls on page load — combined or lazy-loaded | 🟡 |

---

## 10. Logging & Compatibility

| Rule | Description | Severity |
|---|---|---|
| LG-01 | Cross-browser testing completed — Chrome, Safari, Firefox, Edge | 🟠 |
| LG-02 | Cross-device testing completed — multiple OS versions and screen sizes | 🟠 |
| LG-03 | All logs structured JSON — not plain string output | 🟠 |
| LG-04 | Both successful and failed API calls logged | 🟠 |
| LG-05 | Third-party API request/response metadata logged | 🟠 |
| LG-06 | No PII in any log statement — email, phone, name, card data masked | 🔴 |
| LG-07 | Logs centralised — ELK, Datadog, CloudWatch, or Loki | 🟠 |
| LG-08 | Sentry (or equivalent) integrated in all frontend and mobile apps | 🔴 |
| LG-09 | LogDNA / centralised log system active and receiving data | 🟠 |
| LG-10 | Audit trail maintained for all sensitive mutations in this release | 🟠 |

---

## 11. Testing

| Rule | Description | Severity |
|---|---|---|
| TT-01 | Automated test suite runs and passes in CI/CD pipeline | 🔴 |
| TT-02 | Unit test coverage report reviewed — no regression from previous release | 🟠 |
| TT-03 | Regression tests completed before release | 🔴 |
| TT-04 | E2E and regression test scripts updated to cover new features | 🟠 |
| TT-05 | All test scenarios documented — coverage report available | 🟠 |
| TT-06 | QA sign-off obtained before release | 🔴 |
| TT-07 | Stress and soak tests performed for resilience validation | 🟠 |
| TT-08 | All third-party integration scenarios mocked and tested | 🟠 |

---

## 12. Documentation

| Rule | Description | Severity |
|---|---|---|
| DC-01 | Technical documentation updated for all changes in this release | 🟠 |
| DC-02 | API documentation (Swagger/Postman) updated — new endpoints documented | 🟠 |
| DC-03 | Architecture diagram updated if any structural changes were made | 🟡 |
| DC-04 | Release notes detailed and shared with stakeholders | 🟡 |
| DC-05 | ER diagram updated for any database schema changes | 🟡 |
| DC-06 | Third-party integration guides updated for any new or changed integrations | 🟠 |

---

## 13. Go / No-Go Gate

All 🔴 rules must pass before release is approved.
All 🟠 rules must pass or have a documented Tech Lead exception.

| Category | Owner | Status |
|---|---|---|
| Feature Completeness + UAT | PM / BA | ⬜ |
| Load Tests Passed (SLA ≤ 800ms) | Dev / QA | ⬜ |
| Security Scans (MobSF, ZAP, Nuclei) | Dev / Security | ⬜ |
| SonarQube Quality Gate | Dev | ⬜ |
| Lighthouse Score ≥ 80 | Dev / QA | ⬜ |
| Sentry Errors Cleared | Dev | ⬜ |
| Mobile App Checks | Dev / QA | ⬜ |
| Third-Party Documented + Tested | Dev / BA | ⬜ |
| DB Migrations Verified in Staging | Dev | ⬜ |
| Secrets in Secrets Manager | Dev / DevOps | ⬜ |
| Test Coverage Report | Dev / QA | ⬜ |
| Documentation Updated | Dev / BA | ⬜ |
| Rollback Strategy Defined | Tech Lead | ⬜ |

---

## 14. Instant Escalation — 🔴 Release Blockers

| # | Blocker |
|---|---|
| ESC-01 | Security scans (MobSF, ZAP, Nuclei) not completed |
| ESC-02 | SonarQube quality gate failing |
| ESC-03 | Any unresolved 🔴 Critical security vulnerability |
| ESC-04 | Load test not run or API SLA (≤ 800ms) not met |
| ESC-05 | Sentry showing new unresolved errors from this release |
| ESC-06 | Secrets or credentials in source code or config files |
| ESC-07 | DB migration not tested in staging environment |
| ESC-08 | No rollback strategy defined |
| ESC-09 | iOS: `NSAllowsArbitraryLoads = true` in production build |
| ESC-10 | Mobile: code obfuscation not enabled |
| ESC-11 | Mobile: base URL or secrets hardcoded in bundle |
| ESC-12 | Third-party cost not shared with client before go-live |
| ESC-13 | Third-party wrapper missing — SDK called directly from business logic |
| ESC-14 | QA sign-off not obtained |
| ESC-15 | PII present in any log statement going to production |
