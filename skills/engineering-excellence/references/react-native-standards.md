# React Native — Coding Standards & Best Practices

Covers: file/class naming, constants, no inline styles, containers vs components,
micro-components, Flux architecture, Android permissions, unused imports.

---

## Coding Standards & Best Practices (React Native)

### File & Class Naming

```typescript
// ❌ Bad — file name doesn't match class name, wrong case
// file: userProfile.tsx
export default function profile() { ... }

// ✅ Good — PascalCase file name matches exported component name
// file: UserProfileScreen.tsx
export function UserProfileScreen() { ... }

// ✅ Good — PascalCase components, kebab-case utility files
// UserProfileScreen.tsx   → screen component
// useUserProfile.ts       → custom hook (camelCase prefix: use*)
// user-profile.utils.ts   → utility functions
// user-profile.constants.ts → constants
```

### Constant File — Centralised Per Domain

```typescript
// ✅ Good — constants file used everywhere
// constants/storage.constants.ts
export const STORAGE_KEYS = {
  AUTH_TOKEN:          '@app/auth_token',    // prefix to avoid collisions
  REFRESH_TOKEN:       '@app/refresh_token',
  DEVICE_ID:           '@app/device_id',
  ONBOARDING_COMPLETE: '@app/onboarding',
} as const;
// NEVER store PII in AsyncStorage/SharedPreferences

// constants/api.constants.ts
export const API_TIMEOUTS = {
  DEFAULT:  10_000,   // 10 seconds
  UPLOAD:   60_000,   // 60 seconds for file uploads
  PAYMENT:  30_000,   // 30 seconds for payment operations
} as const;

// constants/permissions.constants.ts
export const REQUIRED_PERMISSIONS = {
  CAMERA:   'camera',          // requested only when user opens camera feature
  LOCATION: 'location',        // requested only when user enables location features
  // STORAGE: removed — Android 11+ deprecated WRITE_EXTERNAL_STORAGE
} as const;
```

### No Inline Styles — StyleSheet Only

```typescript
// ❌ Bad — new object created on every render, no reuse, slows React Native bridge
<View style={{ flex: 1, backgroundColor: '#fff', paddingHorizontal: 16 }}>
  <Text style={{ fontSize: 18, color: '#333', fontWeight: '600' }}>Hello</Text>
</View>

// ✅ Good — StyleSheet.create: computed once, optimised by bridge
import { StyleSheet } from 'react-native';
import { colors, spacing, typography } from '../theme';

export function ProfileHeader() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Hello</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex:               1,
    backgroundColor:    colors.background,
    paddingHorizontal:  spacing.md,
  },
  title: {
    fontSize:   typography.sizes.lg,
    color:      colors.textPrimary,
    fontWeight: '600',
  },
});
```

### Containers & Micro-Components

```typescript
// ✅ Good — screen → container → micro-components pattern
// screens/OrderHistoryScreen.tsx  — navigation only
export function OrderHistoryScreen({ navigation }: Props) {
  return <OrderHistoryContainer onOrderPress={id => navigation.navigate('OrderDetail', { id })} />;
}

// containers/OrderHistoryContainer.tsx  — data fetching and state
export function OrderHistoryContainer({ onOrderPress }: Props) {
  const { orders, isLoading, error } = useOrders();
  if (isLoading) return <OrderListSkeleton />;
  if (error)     return <ErrorState message={error.message} />;
  return <OrderList orders={orders} onOrderPress={onOrderPress} />;
}

// components/OrderList.tsx  — pure presentational, accepts only props
export function OrderList({ orders, onOrderPress }: Props) {
  return (
    <FlatList
      data={orders}
      keyExtractor={item => item.id}
      renderItem={({ item }) => (
        <OrderListItem order={item} onPress={() => onOrderPress(item.id)} />
      )}
    />
  );
}

// components/OrderListItem.tsx  — atomic micro-component
export function OrderListItem({ order, onPress }: Props) {
  return (
    <Pressable style={styles.item} onPress={onPress}>
      <AppText variant="subtitle">{order.reference}</AppText>
      <OrderStatusBadge status={order.status} />
      <AppText variant="caption">{formatDate(order.createdAt)}</AppText>
    </Pressable>
  );
}
```

### Flux Architecture — State Management

```typescript
// ❌ Bad — API called directly from screen, no state management
function OrderScreen() {
  useEffect(() => {
    axios.get('/orders').then(res => setOrders(res.data));
  }, []);
}

// ✅ Good — Redux Toolkit / Zustand: API called from defined action
// store/orders/orders.thunks.ts
export const fetchOrders = createAsyncThunk(
  'orders/fetchAll',
  async (_, { rejectWithValue }) => {
    try {
      return await orderService.getAll();  // service class, not direct axios
    } catch (error) {
      logger.error({ error }, 'Failed to fetch orders.');
      return rejectWithValue(error.message);
    }
  }
);

// screen reads from store, dispatches actions — never calls axios directly
function OrderScreen() {
  const dispatch = useAppDispatch();
  const { orders, status } = useAppSelector(s => s.orders);
  useEffect(() => { dispatch(fetchOrders()); }, [dispatch]);
}
```

### Code Quality — Remove Unused Imports & Consoles

```typescript
// ❌ Bad — unused imports, stray console
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { View, Text, StyleSheet, Platform, Dimensions } from 'react-native';
import { SomeLibrary } from 'some-unused-library';

console.log('DEBUG user:', user);  // stray debug log

// ✅ Good — import only what is used; no console.log
import React, { useEffect } from 'react';
import { View } from 'react-native';
// Use logger or Sentry, never console.log
logger.debug({ userId: user.id }, 'User loaded.');
```

### Android Permissions — Modern Policy

```typescript
// ✅ Good — Android 11+ compatible permissions
// AndroidManifest.xml — do NOT include deprecated WRITE_EXTERNAL_STORAGE
// Instead: use scoped storage (MediaStore API) for Android 10+

// React Native — request permissions only when feature is used
import { PermissionsAndroid, Platform } from 'react-native';

async function requestCameraPermission(): Promise<boolean> {
  if (Platform.OS !== 'android') return true;

  const granted = await PermissionsAndroid.request(
    PermissionsAndroid.PERMISSIONS.CAMERA,
    {
      title:   'Camera Permission',
      message: 'This app needs camera access to scan QR codes.',
      buttonPositive: 'Allow',
    }
  );
  return granted === PermissionsAndroid.RESULTS.GRANTED;
}
// Request ONLY at the point of use — not at app startup
```

---

## Generation Checklist (Additions)

- [ ] File name matches exported component/class name in PascalCase
- [ ] All constants in `constants/` domain files — no inline magic values
- [ ] `StyleSheet.create()` used — no inline style objects
- [ ] Container/screen/component/micro-component separation maintained
- [ ] Custom base components wrap RN primitives — no raw `Text`/`View` in screens
- [ ] All API calls dispatched through Redux/Zustand actions via service class
- [ ] Unused imports removed; no `console.log` — use structured logger
- [ ] `WRITE_EXTERNAL_STORAGE` not in AndroidManifest for Android 11+ targets
- [ ] Permissions requested at point of use, not at app startup
- [ ] Sentry integrated and wrapping root component

---

## Gap Detection Table (Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| File name doesn't match class name | `userProfile.tsx` exports `ProfilePage` | 🟡 |
| camelCase component name | `export function orderList()` — should be `OrderList` | 🟡 |
| Inline styles on any component | `style={{ color: 'red' }}` | 🟡 |
| Raw RN components in screen/container | `import { Text, View } from 'react-native'` in a screen | 🟠 |
| API call directly in screen | `axios.get(...)` or direct service call in screen component | 🟠 |
| No constants file | Magic strings/numbers inline in components | 🟡 |
| Unused imports present | `import { X }` where X is never referenced | 🟡 |
| `console.log` in any component | Debug logs left in mobile source | 🟡 |
| `WRITE_EXTERNAL_STORAGE` in manifest | Android 11+ deprecated permission | 🔴 |
| Permissions requested at startup | Permission checks in `App.tsx` / `index.js` instead of at point of use | 🟠 |
| No Sentry in mobile app | No `@sentry/react-native` installed | 🔴 |
| Mixed semicolons | Some statements end with `;`, others don't in same file | 🟡 |

---

## iOS / Swift — Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Sensitive data in UserDefaults | `UserDefaults.set(token/password/pin)` — unencrypted storage | 🔴 |
| HTTP instead of HTTPS | Any `http://` URL in iOS network config or Info.plist | 🔴 |
| ATS disabled in Info.plist | `NSAllowsArbitraryLoads = true` | 🔴 |
| No SSL/certificate pinning | HTTPS only — no pinning — MITM attacks possible | 🟠 |
| SSL pinning disabled for ad SDK | Third-party SDK requires ATS exception | 🟠 |
| Hardcoded API key or base URL | String literal secrets or URLs in Swift source files | 🔴 |
| Base URL not in remote config | Environment URL hardcoded — requires app release to change | 🟠 |
| PII or credentials logged | `print(user.email)`, `Logger.info("token: \(token)")` | 🔴 |
| Keychain not used for credentials | Auth token stored in `UserDefaults` instead of Keychain | 🔴 |
| No environment profiling | Single config for dev/staging/prod — no `#if DEBUG` separation | 🟠 |
| No jailbreak/root detection | App running on compromised device with no detection | 🟠 |
| No input validation on form fields | Email/phone/password fields submitted without validation | 🟠 |
| Third-party SDK not version-pinned | `upToNextMajor` or unpinned Pod — untested version changes | 🟠 |
| SDK added without security review | Dependency added without CVE/ATS/licence check | 🟠 |

---

## iOS / Swift — Code Quality Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `as!` force cast without guard | `cell as! OrderCell` — crashes if type is wrong | 🟠 |
| `!` force unwrap | `user.name!`, `url!` — crashes on nil in production | 🟠 |
| `var` where `let` is correct | Variable declared `var` but never reassigned | 🟡 |
| `print()` used as production logger | `print(...)` in non-test Swift files | 🟡 |
| No `os.Logger` / structured logging | No subsystem/category — logs not filterable in Console.app | 🟠 |
| Protocol conformance in class body | No `// MARK: -` + separate extension per protocol | 🟡 |
| Delegate method missing source param | `func didSelectItem()` — should include delegate source | 🟡 |
| Unused imports | `import UIKit` in model/service — should be `import Foundation` | 🟡 |
| Dead code or template placeholders | `// TODO: implement`, `didReceiveMemoryWarning` stub | 🟡 |
| Constants not in enum namespace | Loose global `let` constants not grouped in `Constants.swift` | 🟡 |
| Unnecessary `self` keyword | `self.title = "..."` where `self` is not required by compiler | 🟡 |
| `(Void)` as parameter type | `func fn(Void)` — should be `func fn()` | 🟡 |
| No `[weak self]` in closure capturing self | Strong cycle in Timer, NotificationCenter, URLSession completion | 🟠 |
| Function exceeds ~30 lines | Single function mixing multiple responsibilities | 🟠 |
| File exceeds 200 lines | God class — split by SRP and protocol extensions | 🟠 |
| Missing MARK navigation comments | Large file with no `// MARK: -` section headers | 🟡 |
