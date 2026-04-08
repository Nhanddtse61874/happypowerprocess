---
name: implementer-angular-typescript
description: Use when implementing Angular TypeScript tasks that require clear module boundaries, strict typing, and production-ready UI behavior.
---

# Implementer Angular/TypeScript

Apply this skill for Angular implementation tasks.

## Required Rules

- Preserve clear feature/module boundaries.
- Keep typed contracts and predictable state transitions.
- Ensure loading/error/empty states are explicit.
- Maintain accessibility and rendering performance awareness.
- Clean up all subscriptions — no memory leaks.

## Minimum Quality Gates

- Add or update tests for changed components and logic.
- Verify build/type checks and changed flows.
- Record risks for UX/performance/regression.
- No unsubscribed Observables in changed code.

## Output Expectations

- Changed files with rationale
- Test and verification evidence
- Risk notes and follow-up items

---

## Architecture

### Module Structure

```
src/
├── core/                     # Singleton services, guards, interceptors (imported once in AppModule)
│   ├── interceptors/
│   ├── guards/
│   └── services/             # AuthService, LoggerService, etc.
├── shared/                   # Reusable components, pipes, directives (no business logic)
│   ├── components/           # ButtonComponent, ModalComponent, etc.
│   ├── pipes/
│   └── directives/
└── features/
    └── orders/               # Self-contained feature module
        ├── components/       # Smart + dumb components
        ├── services/         # OrderService (HTTP + business logic)
        ├── store/            # Signal-based store or NgRx slice
        ├── models/           # TypeScript interfaces for this feature
        ├── orders.routes.ts  # Lazy-loaded routing
        └── orders.component.ts
```

### Component Types

- **Smart (Container) components** — inject services, manage state, compose dumb components
- **Dumb (Presentational) components** — receive `@Input()`, emit `@Output()`, no service dependencies
- **Rule:** Dumb components must use `ChangeDetectionStrategy.OnPush` — always.

---

## TypeScript Rules

### Strict Config (required)

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "strictTemplates": true   // in angularCompilerOptions
  }
}
```

### Type Patterns

```typescript
// ✅ Discriminated unions for async state
type LoadState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string };

// ✅ Use interfaces for service contracts
interface OrderService {
  getOrders(params: OrderParams): Observable<Order[]>;
  createOrder(payload: CreateOrderDto): Observable<Order>;
}

// ✅ Readonly input models — prevent mutation
interface CreateOrderDto {
  readonly productId: string;
  readonly quantity: number;
}

// ❌ Avoid any — use unknown + type narrowing at boundaries
const response = await http.get<unknown>('/api/orders');
if (isOrderArray(response)) { /* safe to use */ }
```

---

## Signals (Angular 17+ — Primary Pattern for Local State)

Signals provide **fine-grained reactivity** without Zone.js overhead. Use signals for component-local and moderately shared state.

```typescript
import { signal, computed, effect } from '@angular/core';

@Component({ ... })
export class ProductListComponent {
  // ✅ Signal for mutable state
  private readonly searchTerm = signal('');
  private readonly products = signal<Product[]>([]);

  // ✅ computed() for derived state — recalculates only when dependencies change
  readonly filteredProducts = computed(() =>
    this.products().filter(p => p.name.includes(this.searchTerm()))
  );

  // ✅ effect() for side effects that depend on signals
  constructor() {
    effect(() => {
      // Runs whenever searchTerm changes — auto-cleanup on destroy
      console.log('Search changed:', this.searchTerm());
    });
  }

  updateSearch(term: string) {
    this.searchTerm.set(term);
  }
}
```

### Signal-based Inputs / Outputs (Angular 17.1+)

```typescript
@Component({ ... })
export class UserCardComponent {
  // ✅ input() instead of @Input() — typed signal
  readonly userId = input.required<string>();
  readonly variant = input<'compact' | 'full'>('full');

  // ✅ output() instead of @Output() EventEmitter
  readonly selected = output<string>();

  select() {
    this.selected.emit(this.userId());
  }
}
```

---

## RxJS Rules

Signals handle state. **RxJS handles async workflows** — HTTP, WebSockets, polling, complex event composition.

### Subscription Cleanup (CRITICAL)

```typescript
// ✅ Option 1: takeUntilDestroyed (Angular 16+ — preferred)
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ ... })
export class OrderListComponent {
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.orderService.getOrders().pipe(
      takeUntilDestroyed(this.destroyRef)  // auto-unsubscribes on destroy
    ).subscribe(orders => this.orders.set(orders));
  }
}

// ✅ Option 2: async pipe in template — auto-unsubscribes (preferred for template binding)
@Component({
  template: `
    @if (orders$ | async; as orders) {
      <order-list [orders]="orders" />
    }
  `
})
export class OrderListComponent {
  readonly orders$ = this.orderService.getOrders();
}

// ❌ Manual subscribe without cleanup — memory leak
ngOnInit() {
  this.orderService.getOrders().subscribe(...); // FORBIDDEN without takeUntilDestroyed
}
```

### Operator Selection

```typescript
// ✅ switchMap — cancel previous, use for search/navigation (latest wins)
readonly results$ = this.searchTerm$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.searchService.search(term))  // cancels in-flight request
);

// ✅ mergeMap — run all concurrently, use for independent parallel tasks
readonly uploads$ = this.files$.pipe(
  mergeMap(file => this.uploadService.upload(file))  // all uploads run in parallel
);

// ✅ concatMap — queue sequentially, use for ordered operations
readonly saved$ = this.saveActions$.pipe(
  concatMap(action => this.saveService.save(action))  // one at a time, in order
);

// ✅ exhaustMap — ignore new until current completes, use for form submit
readonly submit$ = this.submitClick$.pipe(
  exhaustMap(() => this.formService.submit(this.form.value))  // ignores double-clicks
);

// ❌ Avoid nested subscribe — use higher-order operators instead
this.outer$.subscribe(val => {
  this.inner$(val).subscribe(...);  // FORBIDDEN — use switchMap/mergeMap
});
```

### Error Handling in Streams

```typescript
// ✅ catchError to recover — stream continues
readonly data$ = this.dataService.getData().pipe(
  catchError(err => {
    this.errorMessage.set(err.message);
    return of([]); // return safe fallback, stream stays alive
  })
);

// ✅ retry with delay for transient errors
readonly data$ = this.dataService.getData().pipe(
  retry({ count: 3, delay: 1000 }),
  catchError(err => of(null))
);
```

---

## Services & HTTP

### HTTP Service Pattern

```typescript
@Injectable({ providedIn: 'root' })
export class OrderService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = '/api/orders';

  getOrders(params: OrderParams): Observable<Order[]> {
    return this.http.get<Order[]>(this.baseUrl, { params: params as HttpParams });
  }

  createOrder(payload: CreateOrderDto): Observable<Order> {
    return this.http.post<Order>(this.baseUrl, payload);
  }

  deleteOrder(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

### HTTP Interceptors

```typescript
// ✅ Auth interceptor — attach token
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  if (!token) return next(req);
  return next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));
};

// ✅ Error interceptor — global error handling
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) inject(AuthService).logout();
      if (err.status >= 500) inject(ToastService).error('Server error, please retry');
      return throwError(() => err);
    })
  );
};

// Register in app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
  ]
};
```

---

## Forms

### Reactive Forms (complex forms — preferred)

```typescript
@Component({ ... })
export class OrderFormComponent {
  private readonly fb = inject(FormBuilder);

  readonly form = this.fb.group({
    productId: ['', [Validators.required]],
    quantity: [1, [Validators.required, Validators.min(1), Validators.max(100)]],
    notes: [''],
  });

  get quantityControl() { return this.form.get('quantity')!; }

  submit() {
    if (this.form.invalid) return;
    const value = this.form.getRawValue(); // typed
    this.orderService.createOrder(value).pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe();
  }
}
```

### Signal Forms (Angular 19+ — for simpler cases)

```typescript
// ✅ Signal-based form state for lightweight forms
readonly email = signal('');
readonly emailError = computed(() =>
  this.email().includes('@') ? null : 'Invalid email'
);
```

---

## Change Detection

```typescript
// ✅ Always use OnPush for presentational components
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<span>{{ label }}</span>`
})
export class LabelComponent {
  readonly label = input.required<string>();
}

// ✅ Trigger manual detection only when needed
@Component({ changeDetection: ChangeDetectionStrategy.OnPush })
export class ChartComponent {
  private readonly cdr = inject(ChangeDetectorRef);

  updateChart(data: ChartData) {
    this.chartData = data;
    this.cdr.markForCheck(); // schedule re-render
  }
}
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why | Fix |
|---|---|---|
| `subscribe()` without cleanup | Memory leak | `takeUntilDestroyed` or `async` pipe |
| Nested `subscribe()` | Unmanageable, leaks | Higher-order operators (switchMap etc.) |
| Business logic in templates | Untestable, slow | Move to component class or service |
| God service with 20+ methods | Hard to maintain | Split by bounded context |
| `@Input()` mutation inside component | Breaks data flow | Emit `@Output()` and let parent mutate |
| `any` type in HTTP responses | Runtime surprises | Type responses explicitly, narrow at boundary |
| Missing `trackBy` in `*ngFor` | Full re-render on list change | Always provide `trackBy: (i, item) => item.id` |
| `ChangeDetectionStrategy.Default` on dumb components | Unnecessary checks | Always `OnPush` for presentational |

---

## Verification Matrix

| Gate | Command | Expected |
|---|---|---|
| Type check | `ng build` or `tsc --noEmit` | 0 errors |
| Lint | `ng lint` | 0 errors |
| Unit tests | `ng test --watch=false` | All pass |
| Coverage | `ng test --code-coverage` | Changed paths covered |
| Bundle | `ng build --stats-json` + `webpack-bundle-analyzer` | Delta noted |
| Runtime | Manual in browser | Loading/error/empty states reachable, no console errors |
| Memory | Chrome DevTools Heap snapshot | No subscription leaks after navigate away |
