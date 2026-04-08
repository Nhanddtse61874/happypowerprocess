---
name: implementer-react-native-typescript
description: Use when implementing React Native TypeScript tasks that need stable mobile behavior, maintainable UI boundaries, and release safety.
---

# Implementer React Native/TypeScript

Apply this skill for React Native implementation tasks.

## Required Rules

- Keep component/state boundaries explicit and predictable.
- Handle loading/error/offline states for user-facing flows.
- Account for iOS/Android differences and device limitations.
- Keep implementation aligned to approved acceptance criteria.
- Keep side effects isolated and cleaned up (`useEffect` cleanup, unsubscribe listeners).

## Minimum Quality Gates

- Add or update tests for critical user paths.
- Verify runtime behavior for changed flows on at least one platform.
- Note platform-specific assumptions and risks.
- No `any` types in changed files without explicit justification.

## Output Expectations

- Changed files with rationale
- Test and verification evidence
- Platform risks and follow-up items

---

## Architecture

### Folder Structure (Feature-based)

```
src/
├── navigation/
│   ├── RootNavigator.tsx       # Top-level navigator — Stack + Tab composition
│   ├── AppNavigator.tsx        # Authenticated routes
│   ├── AuthNavigator.tsx       # Unauthenticated routes
│   └── types.ts                # RootStackParamList + navigation prop types
├── features/
│   └── orders/
│       ├── screens/            # Full-screen route components
│       │   ├── OrderListScreen.tsx
│       │   └── OrderDetailScreen.tsx
│       ├── components/         # Feature-specific UI components
│       ├── hooks/              # useOrders, useOrderDetail
│       ├── services/           # orderService.ts (API + business logic)
│       └── types.ts
├── components/                 # Shared generic UI (Button, Modal, Skeleton)
├── hooks/                      # Global hooks (useNetworkStatus, usePermissions)
├── store/                      # Zustand or Context global state
├── lib/                        # axios instance, queryClient, mmkv storage
└── utils/                      # Pure utilities
```

### Component Hierarchy

- **Screen components** — route entry points, fetch data, compose feature components, no raw UI
- **Feature components** — business logic, service calls, state management
- **Presentational components** — pure UI, typed props, `React.memo` eligible
- **UI primitives** — Button, Input, Text — platform-aware, no business logic

---

## TypeScript Rules

### Strict Config (required)

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### Type Patterns

```typescript
// ✅ Type navigation params — never pass untyped route params
export type RootStackParamList = {
  Home: undefined;
  OrderDetail: { orderId: string };
  Profile: { userId: string; readonly?: boolean };
};

// In screen component:
type Props = NativeStackScreenProps<RootStackParamList, 'OrderDetail'>;

function OrderDetailScreen({ route, navigation }: Props) {
  const { orderId } = route.params; // fully typed
}

// ✅ Discriminated union for async state
type RemoteData<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }
  | { status: 'offline' };

// ✅ Device model interfaces — typed payloads from native modules
interface SensorReading {
  deviceId: string;
  temperature: number;
  humidity: number;
  batteryLevel: number;
  timestamp: string; // ISO 8601
}

// ❌ Avoid any — use unknown + type guard at native module boundaries
function parseSensorData(raw: unknown): SensorReading {
  if (!isSensorReading(raw)) throw new Error('Invalid sensor payload');
  return raw;
}
```

---

## Navigation (React Navigation)

### Setup Pattern

```typescript
// ✅ Typed navigator — all routes typed in one place
const Stack = createNativeStackNavigator<RootStackParamList>();

export function RootNavigator() {
  const { isAuthenticated } = useAuthStore();

  return (
    <NavigationContainer>
      <Stack.Navigator screenOptions={{ headerShown: false }}>
        {isAuthenticated ? (
          <Stack.Screen name="App" component={AppNavigator} />
        ) : (
          <Stack.Screen name="Auth" component={AuthNavigator} />
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
}

// ✅ Navigate with type safety — params checked at compile time
navigation.navigate('OrderDetail', { orderId: order.id });

// ✅ Use useNavigation hook — avoid passing navigation as prop
function SomeNestedComponent() {
  const navigation = useNavigation<NativeStackNavigationProp<RootStackParamList>>();
  navigation.navigate('Home');
}
```

### Deep Linking

```typescript
// ✅ Configure linking at NavigationContainer level
const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      App: {
        screens: {
          OrderDetail: 'orders/:orderId',
        }
      }
    }
  }
};
<NavigationContainer linking={linking}>...</NavigationContainer>
```

---

## State Management

### Decision Tree

```
Local to one screen?
  → useState / useReducer

Shared between siblings in same navigator?
  → lift to parent Screen, pass as params or callback

Server/async data (API calls)?
  → TanStack Query — NOT useState + useEffect

Global UI state (auth, theme, cart, offline status)?
  → Zustand

Persisted across app restarts?
  → MMKV (fast) or AsyncStorage (simple)

Complex domain state with many actors?
  → Redux Toolkit (enterprise only)
```

### TanStack Query (server state)

```typescript
// ✅ Standard query with offline handling
const { data, isLoading, error, refetch } = useQuery({
  queryKey: ['orders', userId],
  queryFn: () => orderService.getOrders(userId),
  staleTime: 2 * 60_000,      // 2 min
  retry: 3,
  networkMode: 'offlineFirst', // use cache when offline
});

// ✅ Mutation with optimistic update
const mutation = useMutation({
  mutationFn: orderService.completeOrder,
  onMutate: async (orderId) => {
    await queryClient.cancelQueries({ queryKey: ['orders'] });
    const snapshot = queryClient.getQueryData<Order[]>(['orders', userId]);
    queryClient.setQueryData<Order[]>(['orders', userId], (old) =>
      old?.map(o => o.id === orderId ? { ...o, status: 'completed' } : o) ?? []
    );
    return { snapshot };
  },
  onError: (_err, _id, context) => {
    queryClient.setQueryData(['orders', userId], context?.snapshot);
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['orders'] }),
});
```

### Zustand (global state)

```typescript
interface AuthStore {
  user: User | null;
  token: string | null;
  login: (user: User, token: string) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      login: (user, token) => set({ user, token }),
      logout: () => set({ user: null, token: null }),
    }),
    { name: 'auth-store', storage: createJSONStorage(() => MMKV) }
  )
);
```

---

## Performance

### FlatList Best Practices

```typescript
// ✅ All required FlatList optimizations
<FlatList
  data={orders}
  keyExtractor={(item) => item.id}        // stable, unique key — not index
  renderItem={renderOrderItem}             // stable reference (useCallback or module-level)
  getItemLayout={(_, index) => ({          // enables scroll-to-index, skips layout calculation
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
  maxToRenderPerBatch={10}                 // limit off-screen renders
  windowSize={5}                           // virtual window size
  removeClippedSubviews={true}             // unmount off-screen items (Android)
  initialNumToRender={10}                  // first render batch
/>

// ✅ Stable renderItem — defined outside component or wrapped in useCallback
const renderOrderItem = useCallback(
  ({ item }: ListRenderItemInfo<Order>) => (
    <OrderRow order={item} onPress={handlePress} />
  ),
  [handlePress]
);
```

### Memoization Rules

```typescript
// ✅ React.memo for list items — renders often with same props
const OrderRow = React.memo(function OrderRow({ order, onPress }: OrderRowProps) {
  return <TouchableOpacity onPress={() => onPress(order.id)}>...</TouchableOpacity>;
});

// ✅ useMemo for expensive derived values only
const groupedOrders = useMemo(
  () => groupBy(orders, (o) => o.status),
  [orders]
);

// ✅ useCallback for stable function props to memoized children
const handlePress = useCallback((orderId: string) => {
  navigation.navigate('OrderDetail', { orderId });
}, [navigation]);

// ❌ Do NOT memo everything — overhead > benefit for simple components
const Label = React.memo(({ text }: { text: string }) => <Text>{text}</Text>); // unnecessary
```

### Image Optimization

```typescript
// ✅ Always specify dimensions — prevents layout shift
<Image source={{ uri: user.avatarUrl }} style={{ width: 48, height: 48 }} />

// ✅ Use react-native-fast-image for remote images — better caching
import FastImage from 'react-native-fast-image';
<FastImage
  source={{ uri: product.imageUrl, priority: FastImage.priority.normal }}
  style={{ width: 200, height: 200 }}
  resizeMode={FastImage.resizeMode.cover}
/>
```

---

## Error Handling

### Error Boundaries (render-time)

```typescript
// ✅ Wrap screen-level components
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary
  fallbackRender={({ error, resetErrorBoundary }) => (
    <ErrorScreen message={error.message} onRetry={resetErrorBoundary} />
  )}
>
  <OrderListScreen />
</ErrorBoundary>
```

### Async Error Surfaces

```typescript
// ✅ Every data-dependent screen must handle all states
function OrderListScreen() {
  const { data, isLoading, error, refetch } = useOrders();
  const isOffline = useNetworkStatus() === 'offline';

  if (isLoading) return <SkeletonList />;
  if (isOffline && !data) return <OfflineBanner onRetry={refetch} />;
  if (error) return <ErrorMessage message={error.message} onRetry={refetch} />;
  if (!data?.length) return <EmptyState message="No orders yet" />;

  return <FlatList data={data} renderItem={renderOrderItem} />;
}

// ❌ Never swallow async errors
try {
  await doSomething();
} catch {
  // silent — FORBIDDEN
}
```

---

## Platform Differences

```typescript
import { Platform, StyleSheet } from 'react-native';

// ✅ Platform-specific values inline
const shadowStyle = Platform.select({
  ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 2 }, shadowOpacity: 0.1 },
  android: { elevation: 4 },
});

// ✅ Platform-specific files — RN resolves automatically
// Button.ios.tsx    — iOS implementation
// Button.android.tsx — Android implementation

// ✅ Safe area handling — always
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function Screen({ children }: { children: React.ReactNode }) {
  const insets = useSafeAreaInsets();
  return (
    <View style={{ flex: 1, paddingTop: insets.top, paddingBottom: insets.bottom }}>
      {children}
    </View>
  );
}
```

---

## Side Effects & Cleanup

```typescript
// ✅ Always cleanup subscriptions, timers, event listeners
useEffect(() => {
  const subscription = DeviceEventEmitter.addListener('sensorData', handleData);
  return () => subscription.remove(); // cleanup on unmount
}, []);

useEffect(() => {
  const timer = setTimeout(doSomething, 1000);
  return () => clearTimeout(timer); // cleanup
}, [dependency]);

// ✅ AppState for background/foreground transitions
useEffect(() => {
  const subscription = AppState.addEventListener('change', (state) => {
    if (state === 'active') refetchCriticalData();
  });
  return () => subscription.remove();
}, []);
```

---

## Release Safety

### Permissions

```typescript
// ✅ Request permissions at the point of use — not at app startup
async function requestCameraPermission(): Promise<boolean> {
  const result = await request(
    Platform.OS === 'ios'
      ? PERMISSIONS.IOS.CAMERA
      : PERMISSIONS.ANDROID.CAMERA
  );
  return result === RESULTS.GRANTED;
}

// ✅ Handle all permission states
switch (permissionStatus) {
  case RESULTS.GRANTED: enableCamera(); break;
  case RESULTS.DENIED: showPermissionRationale(); break;
  case RESULTS.BLOCKED: showOpenSettingsPrompt(); break;
  case RESULTS.UNAVAILABLE: showUnavailableMessage(); break;
}
```

### OTA Updates (CodePush)

```typescript
// ✅ Apply updates at safe points — never during active flows
useEffect(() => {
  CodePush.sync({
    installMode: CodePush.InstallMode.ON_NEXT_RESTART, // don't interrupt user
    checkFrequency: CodePush.CheckFrequency.ON_APP_RESUME,
  });
}, []);

// ❌ Immediate restart during active transaction — data loss risk
CodePush.sync({ installMode: CodePush.InstallMode.IMMEDIATE }); // use with caution
```

---

## Testing Patterns

```typescript
// ✅ Unit test hooks in isolation with renderHook
import { renderHook, waitFor } from '@testing-library/react-native';

test('useOrders fetches and returns orders', async () => {
  const wrapper = ({ children }) => <QueryClientProvider client={testQueryClient}>{children}</QueryClientProvider>;
  const { result } = renderHook(() => useOrders('user-1'), { wrapper });
  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data).toHaveLength(3);
});

// ✅ Component test with user interaction
import { render, fireEvent, screen } from '@testing-library/react-native';

test('OrderRow calls onPress with correct id', () => {
  const onPress = jest.fn();
  render(<OrderRow order={mockOrder} onPress={onPress} />);
  fireEvent.press(screen.getByText(mockOrder.title));
  expect(onPress).toHaveBeenCalledWith(mockOrder.id);
});

// ✅ Mock native modules — always in jest setup
jest.mock('react-native-mmkv', () => ({
  MMKV: jest.fn().mockImplementation(() => ({
    set: jest.fn(),
    getString: jest.fn(),
  }))
}));
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why | Fix |
|---|---|---|
| `useState` for server data | Manual loading/error, no cache | TanStack Query |
| Index as `keyExtractor` | Wrong items update on list changes | Use stable unique ID |
| Heavy closures in `renderItem` | Re-create on every render, breaks memo | `useCallback` or module-level function |
| Business logic in Screen components | Hard to test, bloated file | Extract to custom hook |
| `useEffect` for derived state | Extra render, stale closure | Compute inline |
| Missing `useEffect` cleanup | Memory leak, stale listener | Always return cleanup |
| `any` for native module data | Runtime crashes | Interface + type guard |
| Permissions at app startup | Bad UX, OS may deny | Request at point of use |
| Ignoring platform differences | iOS/Android visual bugs | `Platform.select()` or `.ios.tsx` / `.android.tsx` |
| `CodePush.IMMEDIATE` during flows | Data loss risk | `ON_NEXT_RESTART` default |

---

## Conditional Add-Ons

- If task includes MQTT/BLE/device connectivity → also apply `skills/implementer-iot-edge/SKILL.md`
- If app uses global navigation guards or crash analytics → include verification for those integrations

---

## Verification Matrix

| Gate | Command / Method | Expected |
|---|---|---|
| Type check | `tsc --noEmit` | 0 errors |
| Lint | `eslint src --ext .ts,.tsx` | 0 errors |
| Unit tests | `jest --testPathPattern=changed` | All pass |
| Coverage | `jest --coverage` | Changed paths covered |
| Runtime iOS | Run on iOS simulator | Changed flows work, no red screen |
| Runtime Android | Run on Android emulator | Changed flows work, no crash |
| Memory | Flipper → RAM monitor | No leak after navigate away and back |
| Offline | Toggle airplane mode | Offline state renders, retry works |
| Platform | Test on both platforms | No platform-specific visual regressions |
