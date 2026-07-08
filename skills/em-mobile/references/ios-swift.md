# iOS / Swift — Code Quality & Standards

Covers: naming conventions, protocol extensions, delegate naming, optionals,
void/closure conventions, constants organisation, code formatting, immutability.

---

# iOS / Swift — Coding Best Practices

Based on Swift API Design Guidelines and the Ray Wenderlich Swift Style Guide.
Apply during every iOS code review and code generation.

Reference: https://github.com/raywenderlich/swift-style-guide

---

## 1. Naming Conventions

```swift
// ✅ Good — class, struct, enum, protocol: UpperCamelCase
class UserProfileViewController: UIViewController {}
struct OrderSummary {}
enum PaymentStatus { case pending, completed, failed }
protocol DataFetchable {}

// ✅ Good — variables, functions, parameters: lowerCamelCase
let userEmail = "user@example.com"
func fetchOrderDetails(for userId: String) -> Order {}
var isLoading: Bool = false

// ✅ Good — constants: lowerCamelCase (Swift convention) or UPPER_SNAKE if ObjC-interop
let maxRetryCount = 3
let defaultTimeout: TimeInterval = 30.0

// ✅ Good — generic type parameters: descriptive UpperCamelCase when meaningful
struct Stack<Element> { private var storage: [Element] = [] }
func write<Target: OutputStream>(to target: inout Target) {}

// When no meaningful role: single uppercase letter
func swap<T>(_ a: inout T, _ b: inout T) {}

// ❌ Bad — abbreviations, single letters for non-generics, unclear names
var usrNm = ""
func getData() {}
class VC: UIViewController {}
let n = 3
```

---

## 2. Code Structure — Extensions & Protocol Conformance

Use a separate extension per protocol conformance. This groups related methods,
makes navigation cleaner, and enables MARK comments for Xcode's jump bar.

```swift
// ❌ Bad — all conformances crammed into the class body
class OrderViewController: UIViewController, UITableViewDataSource,
                            UITableViewDelegate, UIScrollViewDelegate {
  // 300 lines mixing VC lifecycle, table data, scroll handling...
}

// ✅ Good — separate extension per protocol, MARK for navigation
class OrderViewController: UIViewController {
  // MARK: - Properties
  private var orders: [Order] = []
  private var viewModel: OrderViewModel

  // MARK: - Lifecycle
  override func viewDidLoad() {
    super.viewDidLoad()
    setupViews()
    bindViewModel()
  }
}

// MARK: - UITableViewDataSource
extension OrderViewController: UITableViewDataSource {
  func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return orders.count
  }

  func tableView(_ tableView: UITableView,
                 cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(
      withIdentifier: OrderCell.reuseIdentifier,
      for: indexPath
    ) as! OrderCell
    cell.configure(with: orders[indexPath.row])
    return cell
  }
}

// MARK: - UITableViewDelegate
extension OrderViewController: UITableViewDelegate {
  func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    let order = orders[indexPath.row]
    router.navigate(to: .orderDetail(order.id))
  }
}

// MARK: - UIScrollViewDelegate
extension OrderViewController: UIScrollViewDelegate {
  func scrollViewDidScroll(_ scrollView: UIScrollView) {
    // load more when near bottom
    if scrollView.isNearBottom { viewModel.loadNextPage() }
  }
}
```

---

## 3. Delegate Method Naming

Custom delegate methods must include the delegate source as an unnamed first parameter.
This mirrors UIKit conventions and makes it clear which object triggered the callback.

```swift
// ❌ Bad — missing delegate source; ambiguous in multi-component UIs
protocol OrderPickerDelegate: AnyObject {
  func didSelectOrder(_ orderId: String)       // which picker?
  func didCancelSelection()                     // from where?
}

// ✅ Good — delegate source is the first (unnamed) parameter
protocol OrderPickerViewDelegate: AnyObject {
  func orderPickerView(_ orderPickerView: OrderPickerView, didSelectOrder order: Order)
  func orderPickerViewDidCancel(_ orderPickerView: OrderPickerView)
}

// ✅ Good — usage: caller knows exactly which view triggered the callback
class OrderListViewController: UIViewController, OrderPickerViewDelegate {
  func orderPickerView(_ orderPickerView: OrderPickerView, didSelectOrder order: Order) {
    viewModel.confirm(order: order)
  }
  func orderPickerViewDidCancel(_ orderPickerView: OrderPickerView) {
    orderPickerView.dismiss(animated: true)
  }
}
```

---

## 4. Optionals — Correct Usage

```swift
// ✅ Good — declare optional where nil is a valid semantic value
var selectedDate: Date?          // nil = user hasn't selected yet — meaningful
var middleName: String?          // nil = user has no middle name — valid

// ❌ Bad — force unwrap crashes in production when value is nil
let name = user.name!            // crash if name is nil
let cell = tableView.dequeueReusableCell(withIdentifier: "Cell") as! MyCell  // crash

// ✅ Good — safe optional binding
if let name = user.name {
  titleLabel.text = name
} else {
  titleLabel.text = "Anonymous"
}

// ✅ Good — guard let for early exit (preferred for required values)
func processOrder(_ orderId: String?) {
  guard let orderId = orderId, !orderId.isEmpty else {
    logger.warning("processOrder called with nil/empty orderId")
    return
  }
  // orderId is guaranteed non-nil and non-empty from here on
  orderService.fetch(id: orderId)
}

// ✅ Good — nil-coalescing for default values
let displayName = user.nickname ?? user.fullName ?? "Anonymous"

// ✅ Good — optional chaining instead of force unwrap
let cityName = user.address?.city?.name    // nil if any step is nil — no crash
```

---

## 5. Void and Closure Type Conventions

```swift
// ✅ Good — use () for empty parameter list; Void for return type
func updateConstraints() -> Void { }   // explicit Void return
func refresh() { }                     // implicit Void — also fine

// ❌ Bad — (Void) as parameter type is wrong
func doSomething(Void) { }             // wrong — use ()

// ✅ Good — typealias uses Void for closure return types
typealias CompletionHandler = (Result<Order, Error>) -> Void
typealias VoidClosure = () -> Void

// ✅ Good — avoid explicit self unless required (capture lists, disambiguation)
class OrderViewModel {
  var orderId: String = ""
  var title: String { return "Order #\(orderId)" }  // no self needed

  // ✅ Required in closures to clarify capture semantics
  func startPolling() {
    timer = Timer.scheduledTimer(withTimeInterval: 5.0, repeats: true) { [weak self] _ in
      self?.fetchUpdates()  // weak self required here — explicit self needed
    }
  }
}

// ❌ Bad — unnecessary self everywhere
class ProfileViewController: UIViewController {
  override func viewDidLoad() {
    self.title = "Profile"           // unnecessary self
    self.view.backgroundColor = .white  // unnecessary self
    self.setupNavigationBar()        // unnecessary self
  }
}

// ✅ Good — self only where required
override func viewDidLoad() {
  title = "Profile"
  view.backgroundColor = .white
  setupNavigationBar()
}
```

---

## 6. Constants — Scope and Organisation

Keep constant scope as small as possible. Use enums in a `Constants.swift` for app-wide values.

```swift
// ❌ Bad — global loose constants, no grouping, wrong scope
let maxRetries = 3
let apiBaseURL = "https://api.example.com"   // hardcoded! should be config
let animationDuration = 0.3
let cellReuseId = "OrderCell"

// ✅ Good — private constants inside the type that needs them
class AnimatedButton: UIButton {
  private enum Layout {
    static let cornerRadius: CGFloat  = 8.0
    static let animationDuration      = 0.25
    static let shadowOpacity: Float   = 0.3
  }
  // Only AnimatedButton can access these
}

// ✅ Good — app-wide constants in grouped enum namespaces (Constants.swift)
enum AppConstants {
  enum API {
    // Never hardcode secrets — read from config or remote config
    static let timeoutSeconds: TimeInterval = 30.0
    static let maxRetries: Int              = 3
    static let pageSize: Int                = 20
  }

  enum Animation {
    static let standard:  TimeInterval = 0.3
    static let quick:     TimeInterval = 0.15
    static let springDamping: CGFloat  = 0.7
  }

  enum Keychain {
    static let authTokenKey   = "com.app.authToken"
    static let refreshTokenKey = "com.app.refreshToken"
    static let biometricKey   = "com.app.biometricEnabled"
  }
}

// ✅ Good — base URL from remote config, not hardcoded
class AppConfig {
  static var apiBaseURL: String {
    // Read from Firebase Remote Config or equivalent
    return RemoteConfig.remoteConfig().configValue(forKey: "api_base_url").stringValue
      ?? "https://api.fallback.example.com"
  }
}
```

---

