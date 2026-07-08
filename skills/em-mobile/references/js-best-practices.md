# JavaScript & React Native — Core Best Practices

Covers: var/const/let, strict equality, arrow functions, destructuring, async/await,
Promise.all, repository pattern, React Native containers and components.

---

# JavaScript & React Native — Coding Best Practices

## 1. JavaScript Fundamentals

### Variable Declarations

```javascript
// ❌ Bad
var counter = 0;          // var leaks to function scope, hoisted
var userList = [];

// ✅ Good — const by default, let only when reassigning
const MAX_RETRIES = 3;    // never changes
let retryCount = 0;        // changes in a loop
const users = [];          // reference doesn't change, contents may
```

### Strict Equality

```javascript
// ❌ Bad — loose equality causes unexpected coercions
if (userId == '123')  { }  // '123' == 123 is true!
if (value == null)    { }  // matches both null and undefined

// ✅ Good — strict equality
if (userId === '123') { }
if (value === null || value === undefined) { }
if (value == null) { } // This one exception: intentional null/undefined check
```

### Arrow Functions & Implicit Return

```javascript
// ❌ Bad — verbose callbacks
const doubled = numbers.map(function(n) { return n * 2; });
const actives = users.filter(function(u) { return u.isActive; });

// ✅ Good — arrow functions with implicit return
const doubled = numbers.map(n => n * 2);
const actives = users.filter(u => u.isActive);

// ✅ Good — explicit return for multi-line arrow functions
const formatUser = (user) => {
  const fullName = `${user.firstName} ${user.lastName}`;
  return { ...user, fullName };
};
```

### Destructuring

```javascript
// ❌ Bad — repetitive property access
const name = user.name;
const email = user.email;
const role = user.role;

const first = arr[0];
const second = arr[1];

// ✅ Good — destructuring
const { name, email, role } = user;
const { name, email, role = 'member' } = user; // with default

const [first, second, ...rest] = arr;

// ✅ Good — destructuring in function parameters
function sendEmail({ to, subject, body, cc = [] }) {
  // ...
}
```

### Async/Await

```javascript
// ❌ Bad — callback hell
getUserById(id, function(err, user) {
  if (err) return handleError(err);
  getOrdersByUser(user.id, function(err, orders) {
    if (err) return handleError(err);
    formatOrders(orders, function(err, result) {
      // ...
    });
  });
});

// ❌ Bad — promise chain harder to read and debug
getUserById(id)
  .then(user => getOrdersByUser(user.id))
  .then(orders => formatOrders(orders))
  .catch(handleError);

// ✅ Good — async/await with try/catch
async function getUserOrders(id) {
  try {
    const user   = await getUserById(id);
    const orders = await getOrdersByUser(user.id);
    return formatOrders(orders);
  } catch (error) {
    logger.error('Failed to get user orders', { userId: id, error: error.message });
    throw error;
  }
}
```

### Promise.all for Independent Operations

```javascript
// ❌ Bad — sequential when operations are independent (slow)
const user   = await fetchUser(userId);         // wait 100ms
const orders = await fetchOrders(userId);       // then wait 100ms
const stats  = await fetchStats(userId);        // then wait 100ms
// Total: ~300ms

// ✅ Good — parallel execution (fast)
const [user, orders, stats] = await Promise.all([
  fetchUser(userId),    // all 3 start simultaneously
  fetchOrders(userId),
  fetchStats(userId),
]);
// Total: ~100ms (only as slow as the slowest)
```

### Module Pattern

```javascript
// ✅ Good — named exports for testability and tree-shaking
// utils/date.utils.js
export function formatDate(date, format = 'ISO') { /* ... */ }
export function isExpired(date) { /* ... */ }
export function addDays(date, days) { /* ... */ }

// For a single primary export, default export is acceptable
// services/user.service.js
export default class UserService { /* ... */ }

// ✅ Use axios interceptors for common request parameters
// api/axios.config.js
const apiClient = axios.create({ baseURL: process.env.API_BASE_URL });

apiClient.interceptors.request.use((config) => {
  const token = authStore.getToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  config.headers['X-App-Version'] = APP_VERSION;
  return config;
});

apiClient.interceptors.response.use(
  response => response.data,
  error => {
    if (error.response?.status === 401) authStore.logout();
    return Promise.reject(normalizeApiError(error));
  },
);

export default apiClient;
```

---

## 2. Repository Pattern for Backend

```javascript
// ✅ Good — repository wraps DB access; service never touches DB directly

// repositories/user.repository.js
class UserRepository {
  async findById(id)           { return prisma.user.findUnique({ where: { id } }); }
  async findByEmail(email)     { return prisma.user.findUnique({ where: { email } }); }
  async findAllActive()        { return prisma.user.findMany({ where: { isActive: true } }); }
  async save(user)             { return prisma.user.upsert({ ... }); }
  async softDelete(id)         { return prisma.user.update({ where: { id }, data: { deletedAt: new Date() } }); }
}

// services/user.service.js
class UserService {
  constructor(userRepository, emailService) {
    this.userRepo     = userRepository;
    this.emailService = emailService;
  }

  async register(dto) {
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) throw new ConflictError('Email already registered');

    const user = await this.userRepo.save(new User(dto));
    await this.emailService.sendWelcome(user);
    return user;
  }
}
```

---

## 3. React Native Best Practices

### Component Structure

```jsx
// ❌ Bad — all logic inline in screen
function ProfileScreen({ navigation }) {
  const [user, setUser] = useState(null);
  return (
    <View style={{ flex: 1, backgroundColor: '#fff', padding: 16 }}>
      <Text style={{ fontSize: 24, fontWeight: 'bold' }}>Hello</Text>
      <TouchableOpacity style={{ backgroundColor: '#007bff', padding: 12 }}
        onPress={() => navigation.navigate('Edit')}>
        <Text style={{ color: '#fff' }}>Edit</Text>
      </TouchableOpacity>
    </View>
  );
}

// ✅ Good — screen uses containers and custom components, no inline styles
// screens/ProfileScreen.tsx
export function ProfileScreen({ navigation }: ProfileScreenProps) {
  return <ProfileContainer onEditPress={() => navigation.navigate('Edit')} />;
}

// containers/ProfileContainer.tsx — handles data and logic
export function ProfileContainer({ onEditPress }: Props) {
  const { user, isLoading } = useUser();
  if (isLoading) return <ProfileSkeleton />;
  return <ProfileView user={user} onEditPress={onEditPress} />;
}

// components/ProfileView.tsx — pure presentational component
export function ProfileView({ user, onEditPress }: Props) {
  return (
    <ScreenWrapper>
      <Heading>{user.name}</Heading>
      <PrimaryButton onPress={onEditPress} label="Edit Profile" />
    </ScreenWrapper>
  );
}
```

### Custom Base Components (No Raw RN Components in Screens)

```jsx
// ❌ Bad — raw React Native components in screens and containers
import { Text, View, TouchableOpacity } from 'react-native';

// ✅ Good — custom components wrapping RN primitives
// components/ui/AppText.tsx
export function AppText({ variant = 'body', style, children }: Props) {
  return (
    <Text style={[styles[variant], style]}>{children}</Text>
  );
}

// components/ui/PrimaryButton.tsx
export function PrimaryButton({ onPress, label, disabled }: Props) {
  return (
    <TouchableOpacity
      style={[styles.button, disabled && styles.disabled]}
      onPress={onPress}
      disabled={disabled}
      accessibilityRole="button">
      <AppText variant="buttonLabel">{label}</AppText>
    </TouchableOpacity>
  );
}
```

### No Inline Styles

```jsx
// ❌ Bad — inline styles: no reuse, performance issue (new object every render)
<View style={{ flex: 1, paddingHorizontal: 16, backgroundColor: '#f5f5f5' }}>
  <Text style={{ fontSize: 18, fontWeight: '600', color: '#333' }}>Title</Text>
</View>

// ✅ Good — StyleSheet.create: styles computed once, not per render
import { StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    flex:               1,
    paddingHorizontal:  16,
    backgroundColor:    colors.background,
  },
  title: {
    fontSize:   18,
    fontWeight: '600',
    color:      colors.textPrimary,
  },
});
```

### Constants File

```typescript
// ✅ Good — all constants in one place per domain
// constants/api.constants.ts
export const API = {
  BASE_URL:    process.env.EXPO_PUBLIC_API_BASE_URL,  // from env / remote config
  TIMEOUT_MS:  10_000,
  MAX_RETRIES: 3,
} as const;

// constants/storage.constants.ts
export const STORAGE_KEYS = {
  AUTH_TOKEN:    'auth_token',
  DEVICE_ID:     'device_id',
  ONBOARDING:    'onboarding_complete',
} as const;
// Never store PII in AsyncStorage
```

### Flux Architecture (State Management)

```typescript
// ✅ Good — Redux Toolkit / Zustand — API calls from service class, not component
// store/orders.slice.ts (Redux Toolkit)
export const fetchOrders = createAsyncThunk(
  'orders/fetchAll',
  async (userId: string, { rejectWithValue }) => {
    try {
      return await orderService.getByUser(userId); // call from service, not direct axios
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

// ❌ Bad — axios call directly in component
function OrderList() {
  useEffect(() => {
    axios.get('/api/orders').then(setOrders); // no error handling, not reusable
  }, []);
}

// ✅ Good — dispatch action, read from store
function OrderList() {
  const dispatch = useDispatch();
  const { orders, status } = useSelector(state => state.orders);
  useEffect(() => { dispatch(fetchOrders(userId)); }, [userId]);
}
```

---

## 4. Node Version Management

```json
// package.json — specify node version clearly
{
  "name": "my-service",
  "engines": {
    "node": ">=20.0.0",
    "npm":  ">=10.0.0"
  }
}
```

```
# .nvmrc — pin exact version for the project
20.11.0
```

---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| `var` declaration | `var x =` anywhere | 🟡 |
| Loose equality | `==` not `===` | 🟡 |
| `console.log` in production | `console.log(`, `console.info(` | 🟡 |
| Sequential awaits for independent ops | `await a(); await b()` when independent | 🟠 |
| Inline styles in React Native | `style={{ ... }}` on any component | 🟡 |
| Raw RN components in screens | `import { Text, View } from 'react-native'` in a screen file | 🟠 |
| `this` assigned to variable | `const self = this` | 🟡 |
| No axios interceptor for auth | Auth token manually added per API call | 🟡 |
| No repository pattern on backend | Service directly imports ORM/Prisma | 🟠 |
| API called directly in component | `axios.get(...)` inside `useEffect` | 🟠 |
| Node version unspecified | No `engines` in `package.json` | 🟡 |
| `package-lock.json` deleted | No lock file committed | 🟠 |
| Flux architecture violated | Direct API call in React Native screen | 🟠 |

---

