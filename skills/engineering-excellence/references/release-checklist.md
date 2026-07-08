# Release Readiness — Gap Detection & Pre-Release Checklist

This file captures all pre-release validation requirements.
Load this file for any release review, go-live assessment, or sprint completion check.

---

## 🚀 Release Feature Completeness

| Check | Gap if Missing | Severity |
|---|---|---|
| All release features completed and demoed | Incomplete features shipped to production | 🔴 |
| Release checklist created and signed off | Steps missed due to no structured checklist | 🟠 |
| UAT (User Acceptance Testing) completed before release | Real-world defects reach production | 🔴 |
| Stakeholders briefed on features and timelines | Misaligned expectations, escalations post-release | 🟠 |
| Release notes prepared for clients/end-users | Users unaware of changes, support overload | 🟡 |
| Post-release tasks documented and assigned | Follow-ups fall through the cracks | 🟡 |
| Rollback or hotfix strategy defined | No recovery path if release breaks production | 🔴 |
| Common/shared story list reviewed and covered | Shared tasks missed across teams | 🟠 |

---

## 🔥 Load Testing & Performance Benchmarks

| Check | Gap if Missing | Severity |
|---|---|---|
| Load tests executed for all web pages and APIs in this release | Performance regressions undetected before go-live | 🔴 |
| Real-world traffic patterns simulated during load tests | Tests don't reflect actual user behaviour | 🟠 |
| Performance benchmarks defined for pages and APIs (response time SLA) | No baseline to compare against | 🟠 |
| Page and API response times within defined SLA limits | Slow endpoints shipped to production | 🔴 |
| Memory leak detection run during load testing | Memory leaks cause production crashes under sustained load | 🔴 |
| Stress testing run to identify breaking point | System capacity unknown | 🟠 |
| Load test results documented for future reference | No baseline for regression comparison | 🟡 |
| Soak tests run for resilience under sustained load | Time-bomb issues (memory growth, connection leaks) undetected | 🟠 |

**Tools to reference:** k6, JMeter, Gatling, Locust, Artillery

---

## 📊 Quality Reports & Audits

| Check | Gap if Missing | Severity |
|---|---|---|
| SonarQube/SonarCloud quality gate passed — report reviewed | Code quality regressions merged to production | 🔴 |
| Google Lighthouse report run on all pages in this release | Page performance, accessibility, SEO issues undetected | 🟠 |
| Lighthouse scores within acceptable thresholds (Performance ≥ 80, Accessibility ≥ 90) | Poor user experience shipped | 🟠 |
| Network performance checked (waterfall, bundle sizes, TTFB) | Slow load times not caught pre-release | 🟠 |
| Sentry error/log report reviewed and all new errors addressed | Known errors shipped intentionally | 🔴 |
| Console errors and warnings in browser/app reviewed and cleared | JavaScript errors silently failing in production | 🟠 |
| Static analysis (lint) passing with no new violations | Code quality standards not enforced | 🟡 |
| Reports integrated into CI/CD pipeline and blocking merge on failure | Quality checks bypassed | 🟠 |
| Key findings from reports shared with the entire development team | Issues siloed, not learned from | 🟡 |

---

## 🔒 Security Checks

| Check | Gap if Missing | Severity |
|---|---|---|
| MobSF scan completed for mobile application | Mobile security vulnerabilities undetected | 🔴 |
| OWASP ZAP Proxy scan completed on web application | Web application attack vectors undetected | 🔴 |
| Nuclei scan completed (https://github.com/projectdiscovery/nuclei) | Template-based vulnerability patterns missed | 🔴 |
| Dependency vulnerability scan run (npm audit, pip audit, Snyk, Dependabot) | Known CVEs in shipped dependencies | 🔴 |
| All library and plugin versions reviewed — no outdated/vulnerable packages | Supply chain attack surface | 🔴 |
| Content Security Policy (CSP) implemented for XSS prevention | XSS attack vector open | 🔴 |
| Security patches applied for all dependencies and plugins | Known exploits unpatched | 🔴 |
| API endpoints protected against injection attacks (SQL, XML, NoSQL) | Injection vulnerabilities in production | 🔴 |
| Authentication tokens securely stored and managed | Token theft/misuse | 🔴 |
| MFA enforced for admin access | Admin account compromise risk | 🔴 |
| Open ports and attack surface scanned | Unnecessary exposure | 🟠 |
| Code review focused on security vulnerabilities completed | Security gaps missed in code review | 🟠 |

---

## 🔗 Third-Party Integrations

| Check | Gap if Missing | Severity |
|---|---|---|
| All new third-party integrations documented (method, auth, endpoints, data flow) | Knowledge siloed; future maintainers can't understand integration | 🟠 |
| Integration method explained and reviewed (OAuth2, API key, webhook, SDK) | Insecure integration method used | 🟠 |
| OAuth2 or equivalent secure auth used for all third-party services | Credentials exposed in plain API key | 🟠 |
| Third-party integrations tested under load | Integrations break at scale | 🟠 |
| Errors and timeouts handled gracefully (retry, circuit breaker, fallback) | One failing integration brings down the feature | 🔴 |
| Third-party API calls logged with request/response metadata | Impossible to debug integration issues in production | 🟠 |
| Data privacy policy of third-party services reviewed for compliance | GDPR/CCPA violation risk | 🔴 |
| Third-party integrations monitored for real-time issues | Failures invisible until users complain | 🟠 |
| Scalability of third-party rate limits reviewed for production load | Rate limit exceeded under normal traffic | 🟠 |

---

## 📱 Mobile Application Checks

| Check | Gap if Missing | Severity |
|---|---|---|
| Memory Profiler / MemLab analysis completed | Memory leaks cause app crashes on device | 🔴 |
| App tested on multiple OS versions and device sizes | Crashes on common devices undetected | 🔴 |
| Firebase Performance (or equivalent APM) integrated and reviewed | Performance regressions invisible | 🟠 |
| App permissions minimised to necessary access only | Privacy violation; app store rejection risk | 🔴 |
| Push notifications handled securely and efficiently | Token misuse; notification spam | 🟠 |
| App versioning implemented (version code + version name) | Can't track which version is in production | 🟠 |
| Force update feature implemented for critical releases | Users stuck on broken old versions | 🟠 |
| Code obfuscation enabled (ProGuard/R8 for Android, Swift obfuscation for iOS) | Reverse engineering of business logic | 🔴 |
| Base URL and environment config stored in remote config (not hardcoded) | Requires app release to change environment | 🟠 |
| Offline mode supported where applicable | App unusable without connectivity | 🟡 |
| Mobile analytics reviewed to understand user behaviour trends | Feature decisions made without data | 🟡 |

---

## ☁️ Cloud Services & Data Management

| Check | Gap if Missing | Severity |
|---|---|---|
| All new cloud services in this release identified and documented | Untracked cloud resources, cost surprises | 🟠 |
| Cloud service usage and costs monitored for optimisation | Unexpected cost overruns | 🟠 |
| Secrets/credentials/keys/configs stored in secrets manager (AWS Secrets Manager, Vault, GCP Secret Manager) | Credentials exposed in code or config files | 🔴 |
| No sensitive data stored in local storage, cookies, or env files committed to git | Credential exposure | 🔴 |
| Data retention policies followed for sensitive data | Compliance violation | 🔴 |
| Cloud-stored data backed up and backup verified | Data loss with no recovery | 🔴 |
| Access to cloud services monitored for unauthorized use | Breach goes undetected | 🔴 |
| Least privilege principle applied to all cloud configurations | Over-permissioned service = blast radius | 🔴 |
| Terraform scripts updated and reflect current infrastructure state | Infra drift; can't reproduce environment | 🟠 |
| Infrastructure as code maintained for consistent deployments | Manual snowflake environments | 🟠 |

---

## 🗄️ Database & Migration

| Check | Gap if Missing | Severity |
|---|---|---|
| All new database queries optimised and indexed | Slow queries under production load | 🔴 |
| Database query changes explained (indexes added/removed, schema changes) | Unreviewed schema changes break production | 🟠 |
| Migration scripts reviewed for potential data loss | Irreversible data loss in production | 🔴 |
| Migration scripts up to date and tested in staging | Migration fails in production | 🔴 |
| Database backups automated and recovery verified | Data loss with no recovery | 🔴 |
| Database load monitored to prevent performance bottlenecks | Undetected DB saturation | 🟠 |
| Database sharding/partitioning considered if scale requires | DB becomes a single bottleneck | 🟡 |
| Database caching strategy reviewed (query cache, Redis, in-memory) | Repeated expensive queries hit DB unnecessarily | 🟠 |

---

## ⚡ Caching & Frontend Performance

| Check | Gap if Missing | Severity |
|---|---|---|
| Caching mechanisms identified and implemented for slow/repeated data reads | Unnecessary DB/API load; slow pages | 🟠 |
| Browser caching enabled for static assets (Cache-Control headers) | Assets re-downloaded on every visit | 🟡 |
| CDN considered or implemented for static assets | High latency for geographically distributed users | 🟡 |
| Lazy loading implemented for images and heavy assets | Slow initial page load | 🟡 |
| Backend in-memory caching (Redis) implemented where applicable | Repeated expensive computations | 🟠 |
| Excessive API calls avoided via local data storage or debouncing | Unnecessary network load | 🟡 |
| Caching strategy performance impact tested | Cache invalidation bugs; stale data | 🟠 |
| Need for lazy loading in background reviewed to improve page load | Pages slow while fetching non-critical data | 🟡 |

---

## 📝 Compatibility & Logging

| Check | Gap if Missing | Severity |
|---|---|---|
| Cross-platform testing completed on commonly used browsers (Chrome, Safari, Firefox, Edge) | Features broken on specific browsers | 🟠 |
| Testing completed on different OS versions and screen sizes | Crashes or layout breaks on common devices | 🟠 |
| Logs structured (JSON) for easy parsing and analysis | Logs not searchable or machine-parseable | 🟠 |
| Both successful and failed API calls logged | Failures invisible; debugging impossible | 🟠 |
| Third-party API request/response communications logged | Integration issues invisible in production | 🟠 |
| Logs encrypted at rest | Sensitive data in plaintext log files | 🟠 |
| Logs centralised for easier monitoring and management | Distributed logs, no single view | 🟠 |
| Logs regularly reviewed for suspicious activity | Attacks go undetected | 🟠 |
| No PII or credentials in any log statement | Data breach via log access | 🔴 |
| Sentry (or equivalent) integrated; all new errors in this release addressed | Unmonitored production errors | 🔴 |

---

## 🧪 Testing & Coverage

| Check | Gap if Missing | Severity |
|---|---|---|
| Automated test suite runs in CI/CD pipeline | Regressions merged without detection | 🔴 |
| Unit tests cover edge cases and business logic | Defects in untested paths | 🟠 |
| Frontend unit test coverage report reviewed | Coverage regression undetected | 🟠 |
| Regression tests completed before release | Previous features broken by new changes | 🔴 |
| E2E and regression test scripts up to date for this release | Automation doesn't cover new features | 🟠 |
| All test scenarios documented and coverage report reviewed | Unknown gaps in test coverage | 🟠 |
| Stress and soak tests performed for resilience | Time-bomb performance issues undetected | 🟠 |
| Test data sets maintained to cover wide range of scenarios | Edge cases not tested | 🟡 |
| QA sign-off completed before release | Untested features shipped | 🔴 |

---

## 📚 Documentation

| Check | Gap if Missing | Severity |
|---|---|---|
| Technical documentation updated for this release | Outdated docs mislead future developers | 🟠 |
| API documentation (Swagger/Postman) updated and interactive | Consumers don't know about new endpoints | 🟠 |
| Code comments and inline documentation comprehensive for complex logic | Future developers can't maintain the code | 🟡 |
| Version history maintained in technical documents | Can't trace when changes were made | 🟡 |
| Release notes detailed and shared with stakeholders | Stakeholders uninformed about changes | 🟡 |
| Documentation update plan part of each sprint | Docs drift out of sync with code | 🟡 |

---

## 🏗️ Best Practices & Standards

| Check | Gap if Missing | Severity |
|---|---|---|
| Code reviewed against established design patterns (SOLID, OOP) | Technical debt accumulates | 🟠 |
| Coding conventions enforced for readability and maintainability | Inconsistent codebase | 🟡 |
| Static analysis tools (ESLint, Pylint, SonarQube) passing | Code quality standards not enforced | 🟠 |
| Methods modular and independently testable | Hard to unit test; tightly coupled | 🟠 |
| Technical debt assessed and documented | Debt grows silently | 🟡 |
| App permissions minimised (mobile and web) | Privacy violation; data exposure | 🔴 |
| Local storage access follows best practices (no sensitive data, TTL enforced) | Token/credential exposure client-side | 🔴 |
| Best practice document points reviewed and followed | Standards not applied consistently | 🟡 |

---

## ✅ Release Go/No-Go Gate

All 🔴 Critical items must be resolved before release.
All 🟠 Major items must be resolved or have a documented exception approved by tech lead.
🟡 Minor items should be tracked as follow-up tickets.

| Category | Status | Owner | Notes |
|---|---|---|---|
| Feature Completeness | ⬜ | | |
| Load Tests Passed | ⬜ | | |
| Security Scans (MobSF, ZAP, Nuclei) | ⬜ | | |
| SonarQube Quality Gate | ⬜ | | |
| Lighthouse Report | ⬜ | | |
| Sentry Errors Cleared | ⬜ | | |
| Mobile App Checks | ⬜ | | |
| Third-Party Integrations Documented | ⬜ | | |
| Database Migrations Verified | ⬜ | | |
| Secrets in Secrets Manager | ⬜ | | |
| Test Coverage Report | ⬜ | | |
| Documentation Updated | ⬜ | | |
| Rollback Strategy Defined | ⬜ | | |
