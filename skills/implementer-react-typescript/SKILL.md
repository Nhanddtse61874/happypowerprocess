---
name: implementer-react-typescript
description: Use when implementing React TypeScript tasks that require robust state handling, clear component APIs, and production-safe frontend behavior.
---

# Implementer React/TypeScript

Apply this skill for React implementation tasks.

## Required Rules

- Keep component contracts small, explicit, and typed.
- Use stable data-fetching and state management patterns.
- Handle loading, error, and empty states consistently.
- Avoid unnecessary complexity and re-render risks.
- Write self-documenting code — naming over comments.

## Minimum Quality Gates

- Add or update tests for changed user-critical behavior.
- Verify lint/type/tests pass for changed scope.
- Record remaining trade-offs and risk notes.
- No `any` types in changed files without explicit justification.

## Output Expectations

- Changed files with rationale
- Test and verification evidence
- Risk and follow-up items

---

## Architecture

### Folder Structure (Feature-based)

```
src/
├── features/
│   └── auth/
│       ├── components/       # Presentational components for this feature
│       ├── hooks/            # Custom hooks (useAuth, useLoginForm)
│       ├── services/         # API calls (authService.ts)
│       ├── store/            # Zustand slice or context
│       ├── types.ts          # Feature-level types
│       └── index.ts          # Public API of the feature
├── components/               # Shared/generic UI components
├── hooks/                    # Global custom hooks
├── lib/                      # Third-party wrappers (queryClient, axios instance)
├── types/                    # Global types
└── utils/                    # Pure utility functions
```

### Component Hierarchy

- **Page/Route components** — orchestrate layout, fetch top-level data, no UI logic
- **Feature components** — contain business logic, call hooks, compose smaller components
- **Presentational components** — pure UI, receive props, no side effects
- **UI primitives** — Button, Input, Modal — fully generic, no domain knowledge

**Rule:** Business logic must NOT live in presentational components. If a component needs `useQuery` or `useStore`, it is a feature component.

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
// ✅ Use discriminated unions for state modeling — impossible states become unrepresentable
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

// ✅ Use unknown + type guard at boundaries (API responses, external input)
function isUser(value: unknown): value is User {
  return typeof value === 'object' && value !== null && 'id' in value;
}

// ✅ Prefer interface for object shapes (extensible), type for unions/aliases
interface UserProfile {
  id: string;
  name: string;
  role: 'admin' | 'viewer';
}

// ✅ Use generics to avoid duplication
function useLocalStorage<T>(key: string, initial: T): [T, (val: T) => void] { ... }

// ❌ Never use any — use unknown + narrowing, or explicit cast with comment
const data = response as unknown as MyType; // acceptable only at verified boundaries

// ✅ Type component props explicitly — never rely on inference alone
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
  variant?: 'primary' | 'secondary' | 'ghost';
}
```

---

## Component Patterns

### Functional Components (always)

```typescript
// ✅ Explicit props interface + default export
interface CardProps {
  title: string;
  description?: string;
  actions?: React.ReactNode;
}

export default function Card({ title, description, actions }: CardProps) {
  return (
    <div className="card">
      <h2>{title}</h2>
      {description && <p>{description}</p>}
      {actions && <div className="card-actions">{actions}</div>}
    </div>
  );
}
```

### Compound Components (for related UI sets)

```typescript
// ✅ Use compound pattern for semantically related components
const Modal = ({ children }: { children: React.ReactNode }) => <div role="dialog">{children}</div>;
Modal.Header = ({ title }: { title: string }) => <h2>{title}</h2>;
Modal.Body = ({ children }: { children: React.ReactNode }) => <div>{children}</div>;
Modal.Footer = ({ children }: { children: React.ReactNode }) => <div>{children}</div>;

// Usage:
<Modal>
  <Modal.Header title="Confirm" />
  <Modal.Body>Are you sure?</Modal.Body>
  <Modal.Footer><Button>Confirm</Button></Modal.Footer>
</Modal>
```

### Render Props / Children as Function (when logic needs to be shared across layout variants)

```typescript
interface DataLoaderProps<T> {
  query: UseQueryResult<T>;
  children: (data: T) => React.ReactNode;
  loadingFallback?: React.ReactNode;
  errorFallback?: (error: Error) => React.ReactNode;
}

function DataLoader<T>({ query, children, loadingFallback, errorFallback }: DataLoaderProps<T>) {
  if (query.isLoading) return <>{loadingFallback ?? <Spinner />}</>;
  if (query.isError) return <>{errorFallback?.(query.error) ?? <ErrorMessage />}</>;
  if (!query.data) return <EmptyState />;
  return <>{children(query.data)}</>;
}
```

---

## Hooks Rules

### Rules of Hooks (enforced by lint)

- Only call hooks at the top level — never inside conditionals, loops, or callbacks
- Only call hooks from React function components or custom hooks

### Custom Hook Patterns

```typescript
// ✅ Extract stateful logic into named custom hooks — one concern per hook
function useUserProfile(userId: string) {
  const query = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000, // 5 min
  });

  return {
    user: query.data,
    isLoading: query.isLoading,
    error: query.error,
    refetch: query.refetch,
  };
}

// ✅ Cleanup side effects in useEffect
function useEventListener<K extends keyof WindowEventMap>(
  event: K,
  handler: (e: WindowEventMap[K]) => void
) {
  useEffect(() => {
    window.addEventListener(event, handler);
    return () => window.removeEventListener(event, handler); // cleanup
  }, [event, handler]);
}

// ✅ useReducer for complex state with multiple sub-values
type Action =
  | { type: 'SET_FILTER'; filter: string }
  | { type: 'RESET' }
  | { type: 'SET_PAGE'; page: number };

function tableReducer(state: TableState, action: Action): TableState {
  switch (action.type) {
    case 'SET_FILTER': return { ...state, filter: action.filter, page: 1 };
    case 'RESET': return initialTableState;
    case 'SET_PAGE': return { ...state, page: action.page };
  }
}
```

---

## State Management

### Decision Tree

```
Is the state only used in one component?
  → useState

Is the state shared between a small subtree (2-4 components)?
  → lift to parent + prop drilling OR useContext

Is it server/async data (fetched from API)?
  → TanStack Query (NOT useState + useEffect)

Is it global UI state (theme, sidebar, modals)?
  → Zustand slice

Is it form state?
  → React Hook Form + Zod validation

Is it genuinely complex global state with many actors?
  → Redux Toolkit (only for enterprise scale)
```

### TanStack Query (server state — primary pattern)

```typescript
// ✅ Standard query
const { data, isLoading, error } = useQuery({
  queryKey: ['products', { category, page }],  // key includes all dependencies
  queryFn: () => productService.list({ category, page }),
  staleTime: 60_000,       // 1 min before refetch
  gcTime: 5 * 60_000,      // 5 min before cache eviction
});

// ✅ Mutation with optimistic update
const mutation = useMutation({
  mutationFn: (id: string) => productService.delete(id),
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ['products'] });
    const previous = queryClient.getQueryData<Product[]>(['products']);
    queryClient.setQueryData<Product[]>(['products'], (old) =>
      old?.filter((p) => p.id !== id) ?? []
    );
    return { previous };
  },
  onError: (_err, _id, context) => {
    queryClient.setQueryData(['products'], context?.previous); // rollback
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['products'] }),
});
```

### Zustand (global UI state)

```typescript
// ✅ Slice pattern — one file per domain
interface SidebarStore {
  isOpen: boolean;
  open: () => void;
  close: () => void;
  toggle: () => void;
}

export const useSidebarStore = create<SidebarStore>((set) => ({
  isOpen: false,
  open: () => set({ isOpen: true }),
  close: () => set({ isOpen: false }),
  toggle: () => set((s) => ({ isOpen: !s.isOpen })),
}));

// ✅ Select only what you need — avoid whole-store subscriptions
const isOpen = useSidebarStore((s) => s.isOpen);  // only re-renders when isOpen changes
```

### React Hook Form + Zod

```typescript
const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
});

type FormValues = z.infer<typeof schema>;

function LoginForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormValues>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: FormValues) => {
    await authService.login(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}
      <button disabled={isSubmitting}>Login</button>
    </form>
  );
}
```

---

## Error Handling

### Error Boundaries (render-time errors)

```typescript
// ✅ Wrap feature sections, not individual components
<ErrorBoundary fallback={<FeatureErrorFallback />}>
  <ProductList />
</ErrorBoundary>

// ✅ Use react-error-boundary for hooks-based usage
import { useErrorBoundary } from 'react-error-boundary';

function ProductList() {
  const { showBoundary } = useErrorBoundary();
  // ...
  if (criticalError) showBoundary(criticalError);
}
```

### Async Error Surfaces

```typescript
// ✅ Every async operation must expose error state explicitly
function UserCard({ userId }: { userId: string }) {
  const { data, isLoading, error } = useUserProfile(userId);

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage message={error.message} onRetry={refetch} />;
  if (!data) return <EmptyState message="User not found" />;

  return <Card title={data.name} />;
}

// ❌ Never swallow errors silently
try {
  await doSomething();
} catch {
  // silent catch — forbidden
}
```

---

## Performance

### When to memoize

```typescript
// ✅ useMemo — only for genuinely expensive derived values
const sortedItems = useMemo(
  () => items.slice().sort((a, b) => a.price - b.price),
  [items]
);
// ❌ Do NOT useMemo for simple transforms — overhead > benefit
const label = useMemo(() => `Hello ${name}`, [name]); // wrong

// ✅ useCallback — only for stable props passed to memoized children
const handleDelete = useCallback((id: string) => {
  mutation.mutate(id);
}, [mutation]);

// ✅ React.memo — only for components that render often with same props
const Row = React.memo(function Row({ item }: { item: Product }) {
  return <li>{item.name}</li>;
});
```

### Code Splitting

```typescript
// ✅ Lazy-load routes and heavy features
const Dashboard = React.lazy(() => import('./features/dashboard/Dashboard'));

<Suspense fallback={<PageSkeleton />}>
  <Dashboard />
</Suspense>
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why | Fix |
|---|---|---|
| `useEffect` for derived state | Creates stale closure bugs, extra render | Compute directly during render |
| `useState` for server data | Manual loading/error/cache management | Use TanStack Query |
| `any` type | Defeats TypeScript | Use `unknown` + type guard |
| Prop drilling 3+ levels | Brittle, hard to refactor | Lift to context or Zustand |
| Business logic in UI components | Hard to test, violates SRP | Extract to custom hook |
| Inline object/array in JSX | New reference on every render, breaks memo | Move outside component or useMemo |
| Missing cleanup in useEffect | Memory leaks, stale handlers | Always return cleanup function |
| Catching errors silently | Hides bugs | Surface to error boundary or state |

---

## Verification Matrix

| Gate | Command | Expected |
|---|---|---|
| Type check | `tsc --noEmit` | 0 errors |
| Lint | `eslint src --ext .ts,.tsx` | 0 errors |
| Unit tests | `vitest run` or `jest` | All pass |
| Coverage | `vitest run --coverage` | Changed lines covered |
| Build | `vite build` or `next build` | 0 errors, bundle delta noted |
| Runtime | Manual verification in browser | Loading/error/empty states reachable |
