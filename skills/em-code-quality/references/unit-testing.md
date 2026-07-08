# Unit Testing Best Practices

## Core Principle
Tests are not optional. They are the safety net that lets you refactor confidently,
ship faster, and catch bugs before users do. Every business logic path must be tested.

---

## 1. What to Test

Test behaviour, not implementation. Every test should verify a business rule or outcome,
not assert that a specific internal function was called.

```typescript
// ❌ Bad — tests implementation detail (which function was called)
it('calls userRepository.save', async () => {
  await userService.register(dto);
  expect(mockRepo.save).toHaveBeenCalled(); // proves nothing about correctness
});

// ✅ Good — tests the business outcome
it('returns the created user with hashed password', async () => {
  const user = await userService.register({ email: 'a@b.com', password: 'secret' });

  expect(user.email).toBe('a@b.com');
  expect(user.passwordHash).not.toBe('secret');         // password was hashed
  expect(user.passwordHash).toMatch(/^\$2[aby]/);       // is a bcrypt hash
});
```

---

## 2. Cover All Scenarios

Every unit test must cover: **happy path**, **edge cases**, **error conditions**, **boundary values**.

```typescript
// Service under test: OrderService.create(dto, userId)

describe('OrderService.create', () => {

  // ✅ Happy path
  it('creates order and returns it when user and items are valid', async () => {
    const order = await orderService.create(validOrderDto, userId);
    expect(order.id).toBeDefined();
    expect(order.status).toBe(OrderStatus.PENDING);
    expect(order.items).toHaveLength(validOrderDto.items.length);
  });

  // ✅ Validation / edge cases
  it('throws ValidationError when items array is empty', async () => {
    const dto = { ...validOrderDto, items: [] };
    await expect(orderService.create(dto, userId))
      .rejects.toThrow(ValidationError);
  });

  it('throws ValidationError when item quantity is zero', async () => {
    const dto = { ...validOrderDto, items: [{ sku: 'ABC', quantity: 0 }] };
    await expect(orderService.create(dto, userId))
      .rejects.toThrow(ValidationError);
  });

  // ✅ Error conditions
  it('throws UserNotFoundError when user does not exist', async () => {
    mockUserRepo.findById.mockResolvedValue(null);
    await expect(orderService.create(validOrderDto, 'nonexistent-id'))
      .rejects.toThrow(UserNotFoundError);
  });

  it('throws InsufficientStockError when item is out of stock', async () => {
    mockInventoryService.check.mockResolvedValue({ available: 0 });
    await expect(orderService.create(validOrderDto, userId))
      .rejects.toThrow(InsufficientStockError);
  });

  // ✅ Boundary values
  it('creates order with exactly 50 items (maximum allowed)', async () => {
    const dto = { items: Array.from({ length: 50 }, makeItem) };
    await expect(orderService.create(dto, userId)).resolves.toBeDefined();
  });

  it('throws when order has 51 items (over limit)', async () => {
    const dto = { items: Array.from({ length: 51 }, makeItem) };
    await expect(orderService.create(dto, userId)).rejects.toThrow();
  });

  // ✅ Side effects
  it('publishes OrderCreatedEvent after successful creation', async () => {
    await orderService.create(validOrderDto, userId);
    expect(mockEventBus.publish).toHaveBeenCalledWith(
      expect.objectContaining({ type: 'OrderCreated' })
    );
  });

  it('does NOT publish event when order creation fails', async () => {
    mockOrderRepo.save.mockRejectedValue(new Error('DB error'));
    await expect(orderService.create(validOrderDto, userId)).rejects.toThrow();
    expect(mockEventBus.publish).not.toHaveBeenCalled();
  });
});
```

---

## 3. Test Structure — AAA Pattern

Every test follows Arrange → Act → Assert:

```typescript
it('applies discount correctly for premium users', () => {
  // Arrange — set up the data and mocks
  const premiumUser = createUser({ plan: Plan.PREMIUM });
  const cart        = createCart({ items: [{ price: 100 }] });
  const service     = new PricingService();

  // Act — call the function under test
  const discountedCart = service.applyDiscount(cart, premiumUser);

  // Assert — verify the outcome
  expect(discountedCart.total).toBe(80);         // 20% discount applied
  expect(discountedCart.discountApplied).toBe(20);
  expect(discountedCart.discountRate).toBe(0.2);
});
```

---

## 4. Mocking Dependencies

```typescript
// ✅ Good — mock only external dependencies, test the unit in isolation
describe('UserService', () => {
  let userService: UserService;
  let mockUserRepo: jest.Mocked<IUserRepository>;
  let mockEmailService: jest.Mocked<IEmailService>;

  beforeEach(() => {
    mockUserRepo     = { findByEmail: jest.fn(), save: jest.fn(), findById: jest.fn() };
    mockEmailService = { sendWelcome: jest.fn(), sendReset: jest.fn() };
    userService      = new UserService(mockUserRepo, mockEmailService);
  });

  afterEach(() => jest.clearAllMocks());

  it('sends welcome email after registration', async () => {
    mockUserRepo.findByEmail.mockResolvedValue(null);
    mockUserRepo.save.mockResolvedValue(mockUser);

    await userService.register(validDto);

    expect(mockEmailService.sendWelcome).toHaveBeenCalledWith(
      expect.objectContaining({ email: validDto.email })
    );
  });
});
```

---

## 5. Component Testing (React / React Native)

```typescript
// ✅ Good — test user-visible behaviour, not DOM structure
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';

describe('LoginForm', () => {

  it('shows validation error when email is invalid', async () => {
    render(<LoginForm onLogin={jest.fn()} />);

    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'notanemail');
    fireEvent.press(screen.getByText('Sign In'));

    await waitFor(() => {
      expect(screen.getByText('Please enter a valid email')).toBeTruthy();
    });
  });

  it('calls onLogin with credentials when form is valid', async () => {
    const mockLogin = jest.fn().mockResolvedValue({ user: mockUser });
    render(<LoginForm onLogin={mockLogin} />);

    fireEvent.changeText(screen.getByPlaceholderText('Email'),    'user@test.com');
    fireEvent.changeText(screen.getByPlaceholderText('Password'), 'ValidPass1!');
    fireEvent.press(screen.getByText('Sign In'));

    await waitFor(() => {
      expect(mockLogin).toHaveBeenCalledWith({
        email:    'user@test.com',
        password: 'ValidPass1!',
      });
    });
  });

  it('disables submit button while login is in progress', async () => {
    const mockLogin = jest.fn(() => new Promise(() => {})); // never resolves
    render(<LoginForm onLogin={mockLogin} />);
    // Fill in and submit form...
    fireEvent.press(screen.getByText('Sign In'));
    expect(screen.getByText('Sign In')).toBeDisabled();
  });
});
```

---

## 6. Coverage Standards

| Layer | Minimum Coverage |
|---|---|
| Domain / business logic | 90%+ line coverage |
| Service layer | 80%+ |
| API controllers | 70%+ |
| Utility functions | 90%+ |
| React components (critical) | 70%+ |

```json
// jest.config.js — enforce coverage thresholds in CI
{
  "coverageThreshold": {
    "global": {
      "branches":   70,
      "functions":  80,
      "lines":      80,
      "statements": 80
    },
    "./src/domain/**/*.ts": {
      "lines": 90,
      "functions": 90
    }
  }
}
```

---

## 7. Test Data Factories

Never hardcode test data — use factories so tests are readable and DRY:

```typescript
// test/factories/user.factory.ts
import { faker } from '@faker-js/faker';

export function createUser(overrides: Partial<User> = {}): User {
  return {
    id:              faker.string.uuid(),
    email:           faker.internet.email(),
    name:            faker.person.fullName(),
    plan:            Plan.FREE,
    isActive:        true,
    createdAt:       new Date(),
    ...overrides,     // allow any property to be overridden
  };
}

// ✅ Usage — clear intent, no magic values
const adminUser   = createUser({ role: UserRole.ADMIN });
const premiumUser = createUser({ plan: Plan.PREMIUM, isActive: true });
const deletedUser = createUser({ isActive: false, deletedAt: new Date() });
```

---

## 8. SonarLint / SonarQube Integration

```bash
# Run SonarQube analysis as part of CI (GitHub Actions example)
- name: SonarQube Scan
  uses: SonarSource/sonarcloud-github-action@master
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN:  ${{ secrets.SONAR_TOKEN }}
  with:
    args: >
      -Dsonar.projectKey=my-project
      -Dsonar.coverage.exclusions=**/*.test.ts,**/test/**
      -Dsonar.qualitygate.wait=true   # fail pipeline if quality gate fails
```

---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| No unit tests for business logic | Service or domain class with no `.test.ts` file | 🔴 |
| Only happy path tested | Test file with only one `it()` per function | 🟠 |
| No error scenario tests | `try/catch` in service with no test for the catch path | 🟠 |
| No boundary value tests | Min/max limits not tested (0, -1, max+1) | 🟠 |
| Testing implementation not behaviour | `expect(mockFn).toHaveBeenCalled()` as the only assertion | 🟡 |
| No test data factory | Hardcoded `{ email: 'test@test.com', ... }` in every test | 🟡 |
| Coverage below threshold | Coverage < 80% on business logic | 🟠 |
| No SonarQube integration | No `sonar-project.properties` or SonarLint not installed | 🟠 |
| Tests not in CI pipeline | No test step in `.github/workflows` or equivalent | 🔴 |
| Acceptance criteria not covered | Story has 5 AC but only happy path tested | 🟠 |

---

## 9. SonarLint — IDE Integration (Mandatory)

Every developer must have SonarLint installed in their IDE. It catches issues before commit.

```
# IDE plugin installation:
# VS Code:      Extensions → search "SonarLint" → install
# IntelliJ/WebStorm: Plugins → search "SonarLint" → install

# Connect to your SonarQube server for consistent rules:
# VS Code settings.json:
{
  "sonarlint.connectedMode.project": {
    "connectionId": "my-sonarqube",
    "projectKey":   "my-project-key"
  }
}
```

SonarLint enforces in real-time:
- Code smells (functions too long, duplicated code, complexity)
- Bug patterns (null dereference, resource leaks)
- Security hotspots (SQL injection, hardcoded credentials, XSS)
- Code coverage visibility

**If SonarLint shows a red/orange issue — fix it before committing. Do not suppress without comment.**

---

## 10. Architecture & Design Alignment

```typescript
// ❌ Bad — implementation drifts from agreed architecture without documentation
// Developer adds a new pattern (direct DB call from controller) without design review

// ✅ Good — implementation aligns with architecture; deviations documented
/**
 * NOTE: This endpoint uses a direct repository call (bypassing the service layer)
 * because it is a health check and adding service overhead would mask the DB status.
 * Architecture Decision Record: docs/adr/002-health-check-repository.md
 * Approved by: Tech Lead on 2024-01-15
 */
@Get('health/db')
async checkDatabase() {
  return this.userRepo.ping(); // intentional layer skip — documented
}
```

**Rules:**
- If implementation deviates from the agreed architecture, raise it in PR and document
- Changes to architecture must be discussed, approved, and documented in an ADR
- Output of each feature must align with the documented design — not improvised

---

## Gap Detection Table (Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| SonarLint not installed | Developer has unresolved SonarLint issues in committed code | 🟠 |
| SonarQube issues not resolved | Open Sonar issues in the project without tracking/resolution | 🟠 |
| Implementation doesn't match design | Code structure differs from agreed architecture with no ADR | 🟠 |
| Acceptance criteria not tested | Story with 5 AC but only happy path in unit tests | 🟠 |
| No component test for UI behaviour | React component with no `@testing-library` test | 🟠 |
| Test only verifies mock was called | `expect(mockFn).toHaveBeenCalled()` — no outcome verified | 🟡 |
