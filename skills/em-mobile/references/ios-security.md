# iOS / Swift — Security & Environment

Covers: Keychain over UserDefaults, HTTPS, SSL pinning, environment profiling,
structured logging with PII masking, client-side validation, third-party safety.

---

## 7. Security — Keychain, HTTPS, SSL Pinning

```swift
// ❌ Bad — sensitive data in UserDefaults (plain text, not encrypted)
UserDefaults.standard.set(authToken, forKey: "authToken")
UserDefaults.standard.set(userPassword, forKey: "password")  // NEVER

// ✅ Good — sensitive data in Keychain
import Security

class KeychainManager {
  static func save(value: String, for key: String) -> Bool {
    guard let data = value.data(using: .utf8) else { return false }

    let query: [CFString: Any] = [
      kSecClass:            kSecClassGenericPassword,
      kSecAttrAccount:      key,
      kSecValueData:        data,
      kSecAttrAccessible:   kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
    ]

    SecItemDelete(query as CFDictionary) // delete existing before add
    let status = SecItemAdd(query as CFDictionary, nil)
    return status == errSecSuccess
  }

  static func load(key: String) -> String? {
    let query: [CFString: Any] = [
      kSecClass:            kSecClassGenericPassword,
      kSecAttrAccount:      key,
      kSecReturnData:       true,
      kSecMatchLimit:       kSecMatchLimitOne,
    ]

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)

    guard status == errSecSuccess,
          let data = result as? Data,
          let value = String(data: data, encoding: .utf8) else { return nil }
    return value
  }
}

// ✅ Usage
KeychainManager.save(value: authToken, for: AppConstants.Keychain.authTokenKey)
let token = KeychainManager.load(key: AppConstants.Keychain.authTokenKey)
```

```swift
// ✅ Good — SSL Pinning with Alamofire (MITM prevention)
import Alamofire

class NetworkManager {
  static let shared = NetworkManager()

  private let session: Session

  private init() {
    // Load your server's certificate from the app bundle
    let certificates = Bundle.main.paths(forResourcesOfType: "cer", inDirectory: nil)
      .compactMap { SecCertificateCreateWithData(nil, try! Data(contentsOf: URL(fileURLWithPath: $0)) as CFData) }

    let evaluators: [String: ServerTrustEvaluating] = [
      "api.yourapp.com": PinnedCertificatesTrustEvaluator(certificates: certificates),
    ]

    let trustManager = ServerTrustManager(evaluators: evaluators)
    session = Session(serverTrustManager: trustManager)
  }

  func request<T: Decodable>(_ url: String, type: T.Type) async throws -> T {
    let data = try await session.request(url).serializingDecodable(T.self).value
    return data
  }
}

// ✅ Good — Info.plist: Never disable ATS, even for ad SDKs
// NSAppTransportSecurity should NOT contain NSAllowsArbitraryLoads = true
// If a third-party SDK requires it, find a different SDK
```

---

## 8. Environment Profiling

```swift
// ✅ Good — Environment enum controls configuration per build target
enum Environment {
  case development
  case staging
  case production

  static var current: Environment {
    #if DEBUG
      return .development
    #elseif STAGING
      return .staging
    #else
      return .production
    #endif
  }

  var apiBaseURL: String {
    switch self {
    case .development:  return "https://dev-api.yourapp.com"
    case .staging:      return "https://staging-api.yourapp.com"
    case .production:   return RemoteConfig.remoteConfig()
                               .configValue(forKey: "api_base_url").stringValue
                          ?? "https://api.yourapp.com"
    }
  }

  var logLevel: LogLevel {
    switch self {
    case .development:  return .debug
    case .staging:      return .info
    case .production:   return .warning
    }
  }

  var isCertificatePinningEnabled: Bool {
    self == .production || self == .staging
  }

  var isAnalyticsEnabled: Bool {
    self == .production
  }
}

// ✅ Usage throughout the app
let baseURL = Environment.current.apiBaseURL
logger.setLevel(Environment.current.logLevel)
```

---

## 9. Logging — Structured, PII-Safe

```swift
// ❌ Bad — print() in production, PII exposed
print("User logged in: \(user.email)")        // PII in logs!
print("Auth token: \(authToken)")             // credential in logs!
print("Error: \(error)")                      // no log level

// ✅ Good — os.Logger with subsystem/category and PII masking
import os.log

struct AppLogger {
  private static let subsystem = Bundle.main.bundleIdentifier ?? "com.app"

  static let auth     = Logger(subsystem: subsystem, category: "auth")
  static let network  = Logger(subsystem: subsystem, category: "network")
  static let payment  = Logger(subsystem: subsystem, category: "payment")
  static let ui       = Logger(subsystem: subsystem, category: "ui")
}

// ✅ Usage — log IDs and events, never PII or credentials
AppLogger.auth.info("User login attempt, userId: \(userId, privacy: .public)")
AppLogger.auth.warning("Login failed, attempt: \(attemptCount, privacy: .public)")
AppLogger.network.error("Request failed: \(error.localizedDescription, privacy: .public)")

// ✅ Mask sensitive data before any log
extension String {
  var maskedEmail: String {
    let parts = self.split(separator: "@")
    guard parts.count == 2 else { return "***@***" }
    let local  = String(parts[0])
    let domain = String(parts[1])
    let masked = String(local.prefix(2)) + "***"
    return "\(masked)@\(domain)"
  }

  var maskedCard: String {
    return "**** **** **** " + String(self.suffix(4))
  }
}

// ✅ Log levels match environment
// DEBUG   → development only
// INFO    → significant lifecycle events (service started, user registered)
// WARNING → recoverable issues (retry attempt, fallback triggered)
// ERROR   → unexpected failures requiring investigation
```

---

## 10. Client-Side Input Validation

```swift
// ✅ Good — centralised, reusable validators
enum ValidationResult {
  case valid
  case invalid(String) // error message
}

struct InputValidator {

  static func email(_ value: String) -> ValidationResult {
    let trimmed = value.trimmingCharacters(in: .whitespacesAndNewlines)
    guard !trimmed.isEmpty else { return .invalid("Email is required") }
    let emailRegex = #"^[A-Z0-9._%+\-]+@[A-Z0-9.\-]+\.[A-Z]{2,}$"#
    let pred = NSPredicate(format: "SELF MATCHES[c] %@", emailRegex)
    return pred.evaluate(with: trimmed)
      ? .valid
      : .invalid("Enter a valid email address")
  }

  static func phone(_ value: String) -> ValidationResult {
    let digits = value.filter(\.isNumber)
    guard digits.count >= 8 && digits.count <= 15 else {
      return .invalid("Enter a valid phone number (8-15 digits)")
    }
    return .valid
  }

  static func password(_ value: String) -> ValidationResult {
    guard value.count >= 8  else { return .invalid("Password must be at least 8 characters") }
    guard value.count <= 128 else { return .invalid("Password too long") }
    guard value.rangeOfCharacter(from: .uppercaseLetters) != nil else {
      return .invalid("Password must contain at least one uppercase letter")
    }
    guard value.rangeOfCharacter(from: .decimalDigits) != nil else {
      return .invalid("Password must contain at least one number")
    }
    return .valid
  }

  static func nonEmpty(_ value: String, fieldName: String) -> ValidationResult {
    value.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty
      ? .invalid("\(fieldName) is required")
      : .valid
  }
}

// ✅ Good — validate on form submit AND real-time on field blur
class LoginViewController: UIViewController {

  @IBAction func loginTapped(_ sender: UIButton) {
    let emailResult    = InputValidator.email(emailField.text ?? "")
    let passwordResult = InputValidator.password(passwordField.text ?? "")

    if case .invalid(let msg) = emailResult {
      showFieldError(emailField, message: msg)
      return
    }
    if case .invalid(let msg) = passwordResult {
      showFieldError(passwordField, message: msg)
      return
    }

    viewModel.login(email: emailField.text!, password: passwordField.text!)
  }
}
```

---

## 11. Code Formatting Rules (Swift)

```swift
// ✅ Good — long function calls: one parameter per line
let result = reticulateSplines(
  spline:             splines,
  adjustmentFactor:   1.3,
  translateConstant:  2,
  comment:            "normalize the display"
)

// ✅ Good — multi-condition guard on multiple lines
guard
  let userId = session.userId,
  let token  = session.token,
  !token.isEmpty
else {
  return .failure(.unauthorized)
}

// ✅ Good — maximum 200 lines per file; functions max ~30 lines
// If a function exceeds this: extract helpers, apply SRP

// ✅ Good — imports: only what is needed, no redundant imports
import UIKit         // provides UIKit + Foundation — no need to also import Foundation
// import Foundation  ← remove: redundant when UIKit is imported

// Where Foundation suffices (non-UI code), import only Foundation:
import Foundation    // models, services, repositories — no UIKit needed
// import UIKit      ← remove: model should not depend on UI framework

// ✅ Good — remove all dead code: unused variables, commented-out blocks,
//           Xcode template placeholders
// ❌ Bad:
// TODO: implement this
// override func didReceiveMemoryWarning() {
//     super.didReceiveMemoryWarning()
//     // Dispose of any resources that can be recreated.
// }

// ✅ Good — no trailing whitespace; blank line between methods
func methodOne() {
  // ...
}

func methodTwo() {
  // ...
}
```

---

## 12. Immutability — `let` Over `var`

```swift
// ❌ Bad — var used when value never changes
var userId = getUserId()      // never reassigned — should be let
var formatter = DateFormatter() // created once — should be let

// ✅ Good — let by default; var only when mutation is required
let userId     = getUserId()
let formatter  = DateFormatter()
var retryCount = 0           // mutated in retry loop — var is correct

// ✅ Good — struct properties: prefer let
struct OrderSummary {
  let id:          String       // never changes after init
  let userId:      String
  var status:      OrderStatus  // can change — var is correct
  var updatedAt:   Date
}

// ✅ Good — immutable value types prevent unintended sharing
// Swift structs are value types — copies are independent
var original = OrderSummary(id: "1", userId: "u1", status: .pending, updatedAt: .now)
var copy     = original
copy.status  = .completed  // only 'copy' changes; 'original' is unaffected
```

---

## 13. Third-Party Library Safety

```swift
// ✅ Good — vetting process before adding any dependency
/*
  Before adding a Swift Package / CocoaPod / Carthage dependency:
  [ ] Is there a native Apple API that does this? (prefer built-in)
  [ ] Last commit: actively maintained (not abandoned)
  [ ] Known CVEs: check GitHub Security Advisories
  [ ] Licence: compatible with your app's distribution (App Store)
  [ ] Source code review: no obfuscated code, no unexpected network calls
  [ ] Does NOT require disabling ATS (App Transport Security)
  [ ] Pin to exact version in Package.swift / Podfile
*/

// ✅ Good — exact version pinning in Package.swift
// .package(url: "https://github.com/Alamofire/Alamofire", exact: "5.9.1")
// Not: .upToNextMajor — unpinned versions introduce untested changes

// ❌ Bad — disabling ATS for any SDK
// In Info.plist:
// <key>NSAllowsArbitraryLoads</key><true/>  ← NEVER
// <key>NSAllowsArbitraryLoadsForMedia</key><true/>  ← avoid

// ✅ Good — if an ad SDK requires ATS disable, replace it with a compliant SDK
```

---

## Gap Detection Table (iOS / Swift)

| Gap | What to Look For | Severity |
|---|---|---|
| `as!` force cast | `as! MyCell`, `as! String` without guard | 🟠 |
| `!` force unwrap | `user.name!`, `urlString!` in production code | 🟠 |
| Sensitive data in UserDefaults | `UserDefaults.set(token, forKey:)`, `UserDefaults.set(password, forKey:)` | 🔴 |
| PII in log statement | `print(user.email)`, `Logger.info("email: \(email)")` without masking | 🔴 |
| Credential in logs | `print("token: \(authToken)")` | 🔴 |
| Hardcoded API key or base URL | String literal `"https://api..."` or `"sk_live_..."` in source | 🔴 |
| Base URL not in remote config | URL hardcoded — requires release to change environment | 🟠 |
| HTTP instead of HTTPS | `http://` in any URL configuration | 🔴 |
| ATS disabled in Info.plist | `NSAllowsArbitraryLoads = true` | 🔴 |
| No SSL pinning | HTTPS used but no certificate pinning — MITM possible | 🟠 |
| No SSL pinning for third-party SDK | Ad/analytics SDK with ATS exception | 🟠 |
| `var` where `let` is correct | Variable never reassigned after declaration | 🟡 |
| `print()` used as logger | `print(...)` in any non-test source file | 🟡 |
| No os.Logger / structured logging | No subsystem/category logging — not filterable | 🟠 |
| Protocol conformance in class body | No separate extension per protocol | 🟡 |
| Delegate method missing source param | `func didSelectItem()` — no delegate source first param | 🟡 |
| Unused imports | `import UIKit` in a model/service file | 🟡 |
| Dead code or template comments | `// TODO: implement`, `super.didReceiveMemoryWarning()` placeholder | 🟡 |
| Missing MARK comments | Large file with no `// MARK: -` navigation markers | 🟡 |
| Constants not in enum namespace | Loose global constants not grouped in Constants.swift | 🟡 |
| No Environment profiling | Single hardcoded config for all environments | 🟠 |
| No input validation on form fields | Text inputs submitted without email/phone/password validation | 🟠 |
| Unnecessary `self` usage | `self.title = "..."` where `self` is not required | 🟡 |
| `(Void)` as parameter type | `func doSomething(Void)` — use `()` | 🟡 |
| No `[weak self]` in timer/async closure | Strong capture of `self` in Timer, NotificationCenter, async | 🟠 |
| Function exceeds 30 lines | Single function doing multiple things | 🟠 |
| File exceeds 200 lines | God class/controller — split by SRP | 🟠 |
| Version not pinned for dependency | `upToNextMajor` in Package.swift or unpinned Pod | 🟠 |
