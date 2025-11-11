# Chapter 7: Signals - The Future of Reactivity

> *"Do I really need RxJS for everything? My component state is getting too complex!"*

## The Problem

After mastering Zone.js, Alex was tasked with building a complex shopping cart feature. The requirements:

1. Display cart items with live totals
2. Apply discount codes
3. Show tax calculations
4. Real-time inventory checks
5. Sync with backend

Alex reached for RxJS, the familiar tool:

```typescript
@Component({
  selector: 'app-cart',
  template: `
    <div *ngFor="let item of items$ | async">
      {{ item.name }}: {{ item.price }}
    </div>
    <div>Subtotal: {{ subtotal$ | async }}</div>
    <div>Tax: {{ tax$ | async }}</div>
    <div>Total: {{ total$ | async }}</div>
  `
})
export class CartComponent implements OnInit, OnDestroy {
  private itemsSubject = new BehaviorSubject<CartItem[]>([]);
  items$ = this.itemsSubject.asObservable();

  private discountSubject = new BehaviorSubject<number>(0);
  discount$ = this.discountSubject.asObservable();

  subtotal$ = this.items$.pipe(
    map(items => items.reduce((sum, item) => sum + item.price * item.quantity, 0))
  );

  discountedSubtotal$ = combineLatest([this.subtotal$, this.discount$]).pipe(
    map(([subtotal, discount]) => subtotal * (1 - discount))
  );

  tax$ = this.discountedSubtotal$.pipe(
    map(subtotal => subtotal * 0.08)
  );

  total$ = combineLatest([this.discountedSubtotal$, this.tax$]).pipe(
    map(([subtotal, tax]) => subtotal + tax)
  );

  private destroy$ = new Subject<void>();

  ngOnInit() {
    // Load items
    this.cartService.getItems()
      .pipe(takeUntil(this.destroy$))
      .subscribe(items => this.itemsSubject.next(items));
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  applyDiscount(code: string) {
    this.discountService.validateCode(code)
      .pipe(takeUntil(this.destroy$))
      .subscribe(discount => this.discountSubject.next(discount));
  }
}
```

The code worked, but Alex noticed problems:

1. **Too many subscriptions** - Memory leak risks everywhere
2. **Async pipes everywhere** - 5 subscriptions in the template!
3. **Complex dependency management** - `combineLatest` for simple calculations
4. **Boilerplate heavy** - Subject creation, unsubscribe logic, etc.
5. **Hard to debug** - Which observable fired? Why did it update?

**"There has to be a simpler way!"**

Then Alex discovered Angular Signals (Angular 16+) - a new reactivity primitive designed specifically for synchronous state management.

## The Investigation Begins

Alex needed to understand:
- What are Signals?
- How do they differ from Observables?
- How do they work internally?
- When should you use Signals vs RxJS?

### Discovery 1: Signals are Simple Reactive Values

Alex rewrote the cart with Signals:

```typescript
@Component({
  selector: 'app-cart',
  template: `
    <div *ngFor="let item of items()">
      {{ item.name }}: {{ item.price }}
    </div>
    <div>Subtotal: {{ subtotal() }}</div>
    <div>Tax: {{ tax() }}</div>
    <div>Total: {{ total() }}</div>
  `
})
export class CartComponent {
  // Writable signals
  items = signal<CartItem[]>([]);
  discount = signal<number>(0);

  // Computed signals (automatically derived)
  subtotal = computed(() =>
    this.items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );

  discountedSubtotal = computed(() =>
    this.subtotal() * (1 - this.discount())
  );

  tax = computed(() =>
    this.discountedSubtotal() * 0.08
  );

  total = computed(() =>
    this.discountedSubtotal() + this.tax()
  );

  constructor(private cartService: CartService) {
    // Load items
    this.cartService.getItems().subscribe(items => {
      this.items.set(items);  // Update signal
    });
  }

  applyDiscount(code: string) {
    this.discountService.validateCode(code).subscribe(discount => {
      this.discount.set(discount);  // Update signal
    });
  }
}
```

**Benefits immediately obvious**:
- ✅ No `async` pipes needed
- ✅ No manual subscription management
- ✅ No `combineLatest` complexity
- ✅ No `ngOnDestroy` needed
- ✅ Automatic dependency tracking

💡 **Key Insight #1**: Signals are **synchronous reactive values** - simpler than Observables for state management!

### Discovery 2: The Reactive Graph

Alex explored the Signals source code:

```typescript
// packages/core/primitives/signals/src/signal.ts

/**
 * A reactive value that notifies consumers when it changes.
 */
export interface Signal<T> {
  (): T;  // Read the value
}

export interface WritableSignal<T> extends Signal<T> {
  set(value: T): void;              // Replace value
  update(updateFn: (value: T) => T): void;  // Update based on current value
  asReadonly(): Signal<T>;          // Create readonly view
}

/**
 * Create a writable signal
 */
export function signal<T>(initialValue: T): WritableSignal<T> {
  const node = createSignal(initialValue);

  const signalFn = () => {
    // Reading a signal
    return signalGetFn(node);
  };

  signalFn.set = (newValue: T) => {
    // Writing a signal
    signalSetFn(node, newValue);
  };

  signalFn.update = (updateFn: (value: T) => T) => {
    // Update based on current value
    signalSetFn(node, updateFn(node.value));
  };

  signalFn.asReadonly = () => {
    // Return function without set/update
    return () => signalGetFn(node);
  };

  return signalFn as WritableSignal<T>;
}
```

💡 **Key Insight #2**: A Signal is **just a function** that you call to read the value!

### Discovery 3: Computed Signals

Alex investigated computed signals:

```typescript
// packages/core/primitives/signals/src/computed.ts

/**
 * A signal that derives its value from other signals
 */
export function computed<T>(computation: () => T): Signal<T> {
  const node = createComputed(computation);

  const computedFn = () => {
    return producerReadValue(node);
  };

  return computedFn;
}

function createComputed<T>(computation: () => T): ComputedNode<T> {
  const node: ComputedNode<T> = {
    value: undefined as any,
    computation,
    dirty: true,
    consumers: new Set(),
    producers: new Set(),
    equal: Object.is,
  };

  return node;
}

function producerReadValue<T>(node: ComputedNode<T>): T {
  // If dirty, recompute
  if (node.dirty) {
    // Set this node as current consumer
    const prevConsumer = activeConsumer;
    activeConsumer = node;

    // Clear old dependencies
    for (const producer of node.producers) {
      producer.consumers.delete(node);
    }
    node.producers.clear();

    // Recompute (automatically tracks new dependencies!)
    node.value = node.computation();

    // Restore previous consumer
    activeConsumer = prevConsumer;
    node.dirty = false;
  }

  // Track as dependency of current consumer
  if (activeConsumer !== null) {
    activeConsumer.producers.add(node);
    node.consumers.add(activeConsumer);
  }

  return node.value;
}
```

💡 **Key Insight #3**: Computed signals **automatically track dependencies** when you read other signals inside the computation function!

### Discovery 4: The Dependency Graph

Alex visualized how the reactive graph works:

```typescript
// Create signals
const count = signal(1);
const multiplier = signal(2);

// Computed signals build dependency graph
const doubled = computed(() => count() * 2);          // depends on count
const quadrupled = computed(() => doubled() * 2);     // depends on doubled
const total = computed(() => doubled() + quadrupled() + multiplier());  // depends on all

// Dependency graph created:
//
//     count ─────┬──> doubled ─┬──> total
//                │              │
//                └──────────────┴──> quadrupled ──> total
//
//     multiplier ────────────────────────────────> total
```

When `count` changes:

```typescript
count.set(2);

// Propagation:
// 1. count changes (value: 1 → 2)
// 2. Mark consumers dirty: doubled, quadrupled, total
// 3. Next read recomputes:
//    - doubled: 2 * 2 = 4
//    - quadrupled: 4 * 2 = 8
//    - total: 4 + 8 + 2 = 14
```

💡 **Key Insight #4**: Signals create a **reactive dependency graph** - changes propagate automatically through the graph!

## The Deep Dive: How Signals Work Internally

### The Signal Node

```typescript
// packages/core/primitives/signals/src/graph.ts

interface ReactiveNode {
  /** Current value */
  value: unknown;

  /** Version number (increments on change) */
  version: number;

  /** Is this node dirty? */
  dirty: boolean;

  /** Nodes that depend on this node */
  consumers: Set<ReactiveNode>;

  /** Nodes this node depends on */
  producers: Set<ReactiveNode>;

  /** Equality function */
  equal: (a: unknown, b: unknown) => boolean;
}

/** Currently executing reactive computation */
let activeConsumer: ReactiveNode | null = null;
```

### Reading a Signal

```typescript
function signalGetFn<T>(node: SignalNode<T>): T {
  // Track as dependency of current consumer
  if (activeConsumer !== null) {
    // Add bidirectional link
    activeConsumer.producers.add(node);
    node.consumers.add(activeConsumer);
  }

  return node.value;
}
```

### Writing a Signal

```typescript
function signalSetFn<T>(node: SignalNode<T>, newValue: T): void {
  // Check if value actually changed
  if (node.equal(node.value, newValue)) {
    return;  // No change, don't propagate
  }

  // Update value
  node.value = newValue;
  node.version++;

  // Mark all consumers as dirty
  for (const consumer of node.consumers) {
    consumerMarkDirty(consumer);
  }

  // Schedule change detection (if in Angular zone)
  if (isInAngularZone()) {
    scheduleChangeDetection();
  }
}

function consumerMarkDirty(node: ReactiveNode): void {
  node.dirty = true;

  // Recursively mark consumers dirty
  for (const consumer of node.consumers) {
    consumerMarkDirty(consumer);
  }
}
```

### Effects

Effects are reactive functions that run automatically when dependencies change:

```typescript
// packages/core/primitives/signals/src/effect.ts

/**
 * Create an effect that runs when signals change
 */
export function effect(effectFn: () => void): EffectRef {
  const node = createEffect(effectFn);

  // Run immediately
  runEffect(node);

  return {
    destroy: () => destroyEffect(node)
  };
}

function createEffect(effectFn: () => void): EffectNode {
  const node: EffectNode = {
    computation: effectFn,
    dirty: true,
    consumers: new Set(),
    producers: new Set(),
  };

  return node;
}

function runEffect(node: EffectNode): void {
  // Clear old dependencies
  for (const producer of node.producers) {
    producer.consumers.delete(node);
  }
  node.producers.clear();

  // Set as active consumer
  const prevConsumer = activeConsumer;
  activeConsumer = node;

  try {
    // Run effect (tracks dependencies)
    node.computation();
  } finally {
    activeConsumer = prevConsumer;
    node.dirty = false;
  }
}
```

Example effect:

```typescript
const count = signal(0);
const name = signal('Alice');

effect(() => {
  // This effect automatically tracks count and name
  console.log(`${name()} has count: ${count()}`);
});

// Logs: "Alice has count: 0"

count.set(5);
// Logs: "Alice has count: 5"

name.set('Bob');
// Logs: "Bob has count: 5"
```

## Signals vs RxJS: A Detailed Comparison

Alex created a comprehensive comparison:

### Use Case 1: Simple Counter

**With RxJS**:
```typescript
@Component({
  template: `
    <button (click)="increment()">Count: {{ count$ | async }}</button>
  `
})
export class CounterComponent implements OnDestroy {
  private countSubject = new BehaviorSubject(0);
  count$ = this.countSubject.asObservable();

  increment() {
    this.countSubject.next(this.countSubject.value + 1);
  }

  ngOnDestroy() {
    this.countSubject.complete();
  }
}
```

**With Signals**:
```typescript
@Component({
  template: `
    <button (click)="increment()">Count: {{ count() }}</button>
  `
})
export class CounterComponent {
  count = signal(0);

  increment() {
    this.count.update(c => c + 1);
  }
}
```

**Winner**: Signals (much simpler!)

### Use Case 2: HTTP Requests

**With RxJS**:
```typescript
@Component({
  template: `
    <div *ngIf="user$ | async as user">
      {{ user.name }}
    </div>
  `
})
export class UserComponent {
  user$ = this.http.get<User>('/api/user');

  constructor(private http: HttpClient) {}
}
```

**With Signals**:
```typescript
@Component({
  template: `
    <div *ngIf="user() as user">
      {{ user.name }}
    </div>
  `
})
export class UserComponent {
  // Convert Observable to Signal
  user = toSignal(this.http.get<User>('/api/user'));

  constructor(private http: HttpClient) {}
}
```

**Winner**: Tie (both simple, Signals slightly cleaner)

### Use Case 3: Complex Derived State

**With RxJS**:
```typescript
export class ProductListComponent {
  private products$ = this.http.get<Product[]>('/api/products');
  private searchTerm$ = new BehaviorSubject('');
  private category$ = new BehaviorSubject('all');
  private sortBy$ = new BehaviorSubject<'name' | 'price'>('name');

  filteredProducts$ = combineLatest([
    this.products$,
    this.searchTerm$,
    this.category$,
    this.sortBy$
  ]).pipe(
    map(([products, search, category, sortBy]) => {
      let filtered = products;

      if (search) {
        filtered = filtered.filter(p => p.name.includes(search));
      }

      if (category !== 'all') {
        filtered = filtered.filter(p => p.category === category);
      }

      return filtered.sort((a, b) => {
        if (sortBy === 'name') return a.name.localeCompare(b.name);
        return a.price - b.price;
      });
    })
  );

  setSearchTerm(term: string) {
    this.searchTerm$.next(term);
  }

  setCategory(category: string) {
    this.category$.next(category);
  }

  setSortBy(sortBy: 'name' | 'price') {
    this.sortBy$.next(sortBy);
  }
}
```

**With Signals**:
```typescript
export class ProductListComponent {
  private productsSignal = signal<Product[]>([]);
  searchTerm = signal('');
  category = signal('all');
  sortBy = signal<'name' | 'price'>('name');

  // Automatically recomputes when ANY dependency changes
  filteredProducts = computed(() => {
    let filtered = this.productsSignal();

    if (this.searchTerm()) {
      filtered = filtered.filter(p => p.name.includes(this.searchTerm()));
    }

    if (this.category() !== 'all') {
      filtered = filtered.filter(p => p.category === this.category());
    }

    return filtered.sort((a, b) => {
      if (this.sortBy() === 'name') return a.name.localeCompare(b.name);
      return a.price - b.price;
    });
  });

  constructor(http: HttpClient) {
    http.get<Product[]>('/api/products').subscribe(products => {
      this.productsSignal.set(products);
    });
  }
}
```

**Winner**: Signals (much more readable, no combineLatest!)

### Use Case 4: Event Streams

**With RxJS**:
```typescript
export class SearchComponent {
  searchResults$ = this.searchInput$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => this.searchService.search(term)),
    catchError(err => of([]))
  );

  private searchInput$ = new Subject<string>();

  onSearchInput(term: string) {
    this.searchInput$.next(term);
  }
}
```

**With Signals**:
```typescript
// Signals don't have built-in operators like debounce/switchMap
// Still need RxJS for this!
export class SearchComponent {
  private searchTerm = signal('');

  searchResults = toSignal(
    toObservable(this.searchTerm).pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(term => this.searchService.search(term)),
      catchError(err => of([]))
    ),
    { initialValue: [] }
  );

  onSearchInput(term: string) {
    this.searchTerm.set(term);
  }
}
```

**Winner**: RxJS (Signals can't do async operators)

## When to Use Signals vs RxJS

Alex created decision guidelines:

### Use Signals For:

1. **Local Component State**
```typescript
class FormComponent {
  email = signal('');
  password = signal('');

  isValid = computed(() =>
    this.email().includes('@') && this.password().length >= 8
  );
}
```

2. **Derived/Computed State**
```typescript
class ShoppingCart {
  items = signal<Item[]>([]);

  subtotal = computed(() =>
    this.items().reduce((sum, item) => sum + item.price, 0)
  );

  tax = computed(() => this.subtotal() * 0.08);
  total = computed(() => this.subtotal() + this.tax());
}
```

3. **Simple State Management**
```typescript
class ThemeService {
  private themeSignal = signal<'light' | 'dark'>('light');
  theme = this.themeSignal.asReadonly();

  isDark = computed(() => this.theme() === 'dark');

  toggle() {
    this.themeSignal.update(t => t === 'light' ? 'dark' : 'light');
  }
}
```

### Use RxJS For:

1. **HTTP Requests**
```typescript
class DataService {
  loadUser() {
    return this.http.get<User>('/api/user');
  }
}
```

2. **Event Streams**
```typescript
class KeyboardService {
  keyPress$ = fromEvent<KeyboardEvent>(document, 'keydown');

  escapeKey$ = this.keyPress$.pipe(
    filter(e => e.key === 'Escape')
  );
}
```

3. **Complex Async Operations**
```typescript
class AutoSaveService {
  autoSave$ = this.formChanges$.pipe(
    debounceTime(2000),
    distinctUntilChanged(),
    switchMap(data => this.save(data)),
    retry(3),
    catchError(err => this.handleError(err))
  );
}
```

### Combine Both:

```typescript
class SmartComponent {
  // Signal for local state
  filter = signal('all');
  sortOrder = signal<'asc' | 'desc'>('asc');

  // RxJS for HTTP
  private data$ = this.http.get<Data[]>('/api/data');

  // Convert Observable → Signal
  private dataSignal = toSignal(this.data$, { initialValue: [] });

  // Computed signal combining both
  filteredData = computed(() => {
    const data = this.dataSignal();
    const filter = this.filter();
    const order = this.sortOrder();

    let result = data.filter(d => this.matchesFilter(d, filter));

    return order === 'asc'
      ? result.sort((a, b) => a.value - b.value)
      : result.sort((a, b) => b.value - a.value);
  });
}
```

## Advanced Signal Patterns

### Pattern 1: Signal-based State Management

```typescript
// Global state service
@Injectable({ providedIn: 'root' })
export class CartStateService {
  // Private writable signals
  private itemsSignal = signal<CartItem[]>([]);
  private loadingSignal = signal(false);
  private errorSignal = signal<string | null>(null);

  // Public readonly signals
  items = this.itemsSignal.asReadonly();
  loading = this.loadingSignal.asReadonly();
  error = this.errorSignal.asReadonly();

  // Computed state
  itemCount = computed(() => this.items().length);
  total = computed(() =>
    this.items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );
  isEmpty = computed(() => this.itemCount() === 0);

  constructor(private http: HttpClient) {
    this.loadItems();
  }

  private loadItems() {
    this.loadingSignal.set(true);

    this.http.get<CartItem[]>('/api/cart').subscribe({
      next: items => {
        this.itemsSignal.set(items);
        this.loadingSignal.set(false);
      },
      error: err => {
        this.errorSignal.set(err.message);
        this.loadingSignal.set(false);
      }
    });
  }

  addItem(item: CartItem) {
    this.itemsSignal.update(items => [...items, item]);
  }

  removeItem(id: string) {
    this.itemsSignal.update(items => items.filter(item => item.id !== id));
  }

  clearCart() {
    this.itemsSignal.set([]);
  }
}
```

### Pattern 2: Effect for Side Effects

```typescript
@Component({/* ... */})
export class AutoSaveComponent {
  formData = signal({ name: '', email: '' });

  constructor(private saveService: SaveService) {
    // Auto-save when formData changes
    effect(() => {
      const data = this.formData();

      // Debounce in effect (for demo purposes)
      setTimeout(() => {
        this.saveService.save(data).subscribe();
      }, 1000);
    });
  }
}
```

### Pattern 3: Custom Signal Utilities

```typescript
// Create a signal with localStorage persistence
function persistedSignal<T>(key: string, initialValue: T): WritableSignal<T> {
  // Load from localStorage
  const storedValue = localStorage.getItem(key);
  const value = storedValue ? JSON.parse(storedValue) : initialValue;

  const s = signal(value);

  // Persist on change
  effect(() => {
    localStorage.setItem(key, JSON.stringify(s()));
  });

  return s;
}

// Usage
class PreferencesService {
  theme = persistedSignal<'light' | 'dark'>('theme', 'light');
  fontSize = persistedSignal('fontSize', 14);
}
```

### Pattern 4: Async Signal

```typescript
// Convert async operation to signal
function asyncSignal<T>(
  promise: Promise<T>,
  initialValue: T
): Signal<T> {
  const s = signal(initialValue);

  promise.then(value => s.set(value));

  return s.asReadonly();
}

// Usage
class UserComponent {
  user = asyncSignal(
    fetch('/api/user').then(r => r.json()),
    null
  );
}
```

## Signals and Change Detection

Signals integrate seamlessly with Angular's change detection:

```typescript
// Signals automatically trigger change detection!
@Component({
  template: `<div>{{ count() }}</div>`
})
export class CounterComponent {
  count = signal(0);

  increment() {
    this.count.update(c => c + 1);  // ← Triggers change detection!
  }
}
```

Under the hood:

```typescript
// packages/core/primitives/signals/src/signal.ts

function signalSetFn<T>(node: SignalNode<T>, newValue: T): void {
  // ... update value ...

  // Mark Angular for check
  markForCheck();  // ← Schedules change detection
}

function markForCheck(): void {
  // Tell Angular this component needs checking
  if (currentView) {
    markViewDirty(currentView);
  }
}
```

**Signals work with OnPush!**

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,  // ← OnPush
  template: `<div>{{ count() }}</div>`
})
export class OptimizedComponent {
  count = signal(0);

  increment() {
    this.count.update(c => c + 1);  // ✅ Still triggers CD with OnPush!
  }
}
```

## Migrating from RxJS to Signals

Alex created a migration guide:

### Before (RxJS):
```typescript
@Component({
  template: `
    <input [value]="searchTerm$ | async" (input)="onSearch($event)">
    <div *ngFor="let result of results$ | async">
      {{ result.name }}
    </div>
  `
})
export class SearchComponent implements OnDestroy {
  private searchSubject = new BehaviorSubject('');
  searchTerm$ = this.searchSubject.asObservable();

  results$ = this.searchTerm$.pipe(
    debounceTime(300),
    switchMap(term => this.searchService.search(term))
  );

  onSearch(event: Event) {
    const term = (event.target as HTMLInputElement).value;
    this.searchSubject.next(term);
  }

  ngOnDestroy() {
    this.searchSubject.complete();
  }
}
```

### After (Signals):
```typescript
@Component({
  template: `
    <input [value]="searchTerm()" (input)="onSearch($event)">
    <div *ngFor="let result of results()">
      {{ result.name }}
    </div>
  `
})
export class SearchComponent {
  searchTerm = signal('');

  // Convert Observable to Signal
  results = toSignal(
    toObservable(this.searchTerm).pipe(
      debounceTime(300),
      switchMap(term => this.searchService.search(term))
    ),
    { initialValue: [] }
  );

  onSearch(event: Event) {
    const term = (event.target as HTMLInputElement).value;
    this.searchTerm.set(term);
  }

  // No ngOnDestroy needed!
}
```

## Key Takeaways

After mastering Signals, Alex understood:

### 1. **Signals are Synchronous Reactive Primitives**
Perfect for local state, computed values, and simple reactivity.

### 2. **Automatic Dependency Tracking**
Computed signals automatically track dependencies - no manual wiring!

### 3. **No Subscription Management**
No need for `async` pipe, `takeUntil`, or `ngOnDestroy` cleanup.

### 4. **Fine-Grained Reactivity**
Only affected computations re-run when signals change.

### 5. **RxJS Still Needed for Async**
Use RxJS for HTTP, events, and complex async operations.

### 6. **Combine Both for Best Results**
Use Signals for state, RxJS for async, `toSignal()`/`toObservable()` to bridge.

### 7. **Better Performance**
Signals with OnPush provide optimal change detection performance.

### 8. **Simpler Code**
Less boilerplate, easier to read, fewer bugs.

## Practical Applications

Alex now uses Signals for:

1. **Component State** - All local state
2. **Derived Values** - Computed properties
3. **Global State** - Service-based state management
4. **Form State** - Form values and validation
5. **UI State** - Theme, sidebar open/closed, etc.

And RxJS for:

1. **HTTP Requests** - API calls
2. **WebSockets** - Real-time data
3. **Event Streams** - User input, keyboard events
4. **Complex Async Logic** - Retry, debounce, switchMap

## What's Next?

Understanding Signals revealed Angular's future direction for reactivity. But one more core system remained:

*How does the Angular Router work? How does it navigate between views and lazy-load modules?*

This led to investigating the Router...

---

**Next**: [Chapter 8: Router Internals](08-router.md)

## Further Reading

- Source: `packages/core/primitives/signals/`
- Signals RFC: https://github.com/angular/angular/discussions/49090
- Documentation: https://angular.dev/guide/signals
- Migration Guide: https://angular.dev/guide/signals/rxjs-interop

## Code Example

See `code-examples/07-signals/` for complete examples:
- Basic signals usage
- Computed signals
- Effects
- Signal-based state management
- RxJS interop
- Migration examples

Run it:
```bash
cd code-examples/07-signals/
npm install
npm start
```

## Notes from Alex's Journal

*"Signals are a game-changer! This is what I always wanted from RxJS for local state.*

*The mental model is so simple: signals are reactive values, computed signals derive from other signals, effects run when signals change. That's it.*

*No more BehaviorSubject, no more combineLatest, no more subscription management nightmares. Just clean, simple reactive code.*

*Key insights:*
*- Signals = synchronous reactive values*
*- Computed = automatic dependency tracking*
*- Effects = side effects when dependencies change*
*- RxJS still needed for async operations*

*The cart component went from 80 lines of RxJS boilerplate to 30 lines of clean Signal code. And it's faster and easier to understand.*

*This is the future of Angular. Simpler, faster, better.*

*Final chapter: the Router. Let's understand how navigation works under the hood!"*
