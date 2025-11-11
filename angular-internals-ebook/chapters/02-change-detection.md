# Chapter 2: The Change Detection Enigma

> *"I clicked the button, but the UI didn't update!"*

## The Problem

Fresh from the dependency injection victory, Alex felt confident. The next task seemed straightforward: build a real-time dashboard showing live order updates for the e-commerce platform.

The requirements were clear:
- Display a live count of pending orders
- Update every second without user interaction
- Show order status changes in real-time
- Handle hundreds of orders efficiently

Alex created a component that fetched data every second:

```typescript
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <h2>Live Orders: {{ orders.length }}</h2>
      <div class="stats">
        <span>Pending: {{ getPendingCount() }}</span>
        <span>Processing: {{ getProcessingCount() }}</span>
      </div>
      <div *ngFor="let order of orders" class="order-card">
        {{ order.id }} - {{ order.status }} - ${{ order.total }}
      </div>
    </div>
  `,
  standalone: true,
  imports: [CommonModule]
})
export class DashboardComponent implements OnInit {
  orders: Order[] = [];

  constructor(private orderService: OrderService) {}

  ngOnInit() {
    // Poll for updates using native setInterval
    setInterval(() => {
      this.orderService.getOrders().subscribe(orders => {
        this.orders = orders;
        console.log('Orders updated:', orders.length); // Logs correctly every second!
      });
    }, 1000);
  }

  getPendingCount(): number {
    console.log('Calculating pending...'); // This never logs!
    return this.orders.filter(o => o.status === 'pending').length;
  }

  getProcessingCount(): number {
    return this.orders.filter(o => o.status === 'processing').length;
  }
}
```

Alex ran the app and watched the console. Every second: `Orders updated: 15`, `Orders updated: 17`, `Orders updated: 19`...

But the screen? It showed "Live Orders: 0" and never changed.

**"The data is updating, but the view isn't!"** Alex was completely baffled.

Then Alex tried something out of desperation: clicking anywhere on the page. Suddenly, the UI updated with all the pending changes. The count jumped from 0 to 19, all the orders appeared, and everything was current.

*"Wait... what? Clicks trigger updates, but data changes don't? How does that even make sense?"*

Alex tried another experiment - hovering over a button. No update. Clicking that button? Instant update. Typing in an input field? Update! Moving the mouse? Nothing.

**Something was watching for specific events and triggering UI updates.**

## The Investigation Begins

This mystery led Alex on a deep dive into Angular's change detection system. Unlike the dependency injection problem, which produced a clear error message, this issue was silent. The application ran without errors - it just... didn't update.

### Discovery 1: Change Detection Runs on Events

Alex opened the Angular repository and navigated to `packages/core/src/render3/instructions/`. The file `change_detection.ts` caught Alex's attention.

```typescript
// packages/core/src/render3/instructions/change_detection.ts

/**
 * The maximum number of times the change detection traversal will rerun before throwing an error.
 */
export const MAXIMUM_REFRESH_RERUNS = 100;

export function detectChangesInternal(lView: LView, mode = ChangeDetectionMode.Global) {
  const environment = lView[ENVIRONMENT];
  const rendererFactory = environment.rendererFactory;

  // Check no changes mode is a dev only mode used to verify that bindings have not changed
  // since they were assigned. We do not want to invoke renderer factory functions in that mode
  // to avoid any possible side-effects.
  const checkNoChangesMode = !!ngDevMode && isInCheckNoChangesMode();

  if (!checkNoChangesMode) {
    rendererFactory.begin?.();
  }

  try {
    detectChangesInViewWhileDirty(lView, mode);
  } finally {
    if (!checkNoChangesMode) {
      rendererFactory.end?.();
    }
  }
}

function detectChangesInViewWhileDirty(lView: LView, mode: ChangeDetectionMode) {
  const lastIsRefreshingViewsValue = isRefreshingViews();
  try {
    setIsRefreshingViews(true);
    detectChangesInView(lView, mode);

    let retries = 0;
    // If after running change detection, this view still needs to be refreshed or there are
    // descendants views that need to be refreshed due to re-dirtying during the change detection
    // run, detect changes on the view again. We run change detection in `Targeted` mode to only
    // refresh views with the `RefreshView` flag.
    while (requiresRefreshOrTraversal(lView)) {
      if (retries === MAXIMUM_REFRESH_RERUNS) {
        throw new RuntimeError(
          RuntimeErrorCode.INFINITE_CHANGE_DETECTION,
          ngDevMode &&
            'Infinite change detection while trying to refresh views. ' +
              'There may be components which each cause the other to require a refresh, ' +
              'causing an infinite loop.',
        );
      }
      retries++;
      detectChangesInView(lView, ChangeDetectionMode.Targeted);
    }
  } finally {
    setIsRefreshingViews(lastIsRefreshingViewsValue);
  }
}
```

💡 **Key Insight #1**: Change detection doesn't run automatically - something must explicitly call `detectChangesInternal()`!

Alex discovered that Angular uses a concept called **LView** (Logical View) to represent each component's state. Every component gets its own LView, which contains all the data needed for change detection.

But who calls `detectChangesInternal()`?

### Discovery 2: The markViewDirty Function

Alex found another crucial piece in `mark_view_dirty.ts`:

```typescript
// packages/core/src/render3/instructions/mark_view_dirty.ts

/**
 * Marks current view and all ancestors dirty.
 *
 * Returns the root view because it is found as a byproduct of marking the view tree
 * dirty, and can be used by methods that consume markViewDirty() to easily schedule
 * change detection. Otherwise, such methods would need to traverse up the view tree
 * an additional time to get the root view and schedule a tick on it.
 *
 * @param lView The starting LView to mark dirty
 * @returns the root LView
 */
export function markViewDirty(lView: LView, source: NotificationSource): LView | null {
  const dirtyBitsToUse = isRefreshingViews()
    ? LViewFlags.Dirty
    : LViewFlags.RefreshView | LViewFlags.Dirty;

  // Notify the change detection scheduler
  lView[ENVIRONMENT].changeDetectionScheduler?.notify(source);

  // Walk up the view tree, marking each ancestor dirty
  while (lView) {
    lView[FLAGS] |= dirtyBitsToUse;  // Set dirty bits using bitwise OR
    const parent = getLViewParent(lView);

    // Stop at root view
    if (isRootView(lView) && !parent) {
      return lView;
    }

    lView = parent!;
  }
  return null;
}
```

This was fascinating! Angular marks views as "dirty" using **bit flags**. But again, who calls `markViewDirty()`?

### Discovery 3: Zone.js - The Monkey Patcher

The answer lay in a library called **Zone.js**. Alex discovered that Angular patches all asynchronous browser APIs:

```typescript
// Conceptual representation of Zone.js patching

// Original setTimeout
const originalSetTimeout = window.setTimeout;

// Patched setTimeout
window.setTimeout = function(callback, delay) {
  return originalSetTimeout(() => {
    try {
      // Run the callback
      callback();
    } finally {
      // Trigger Angular's change detection!
      triggerChangeDetection();
    }
  }, delay);
};
```

Zone.js patches:
- ✅ `setTimeout` / `setInterval` / `setImmediate`
- ✅ `Promise.then()` / `async`/`await`
- ✅ `XMLHttpRequest` / `fetch()`
- ✅ `addEventListener` (for all DOM events)
- ✅ `requestAnimationFrame`
- ✅ `MutationObserver`
- ✅ `IntersectionObserver`

When any of these complete, Zone.js automatically triggers change detection.

### Discovery 4: The Problem Revealed

But here's what Alex discovered: **The `setInterval` in the component wasn't being patched!**

After extensive debugging with breakpoints, Alex realized the interval was created in a context where Zone.js patches weren't active. This can happen when:
- Code runs during SSR (Server-Side Rendering)
- Code runs in a Web Worker
- Code runs in `NgZone.runOutsideAngular()`
- Code runs before Angular fully initializes

The solution became clear: **manually trigger change detection**.

## The Solution

Alex tried several approaches:

### Approach 1: Run Inside NgZone

```typescript
import { Component, OnInit, NgZone } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <h2>Live Orders: {{ orders.length }}</h2>
      <div *ngFor="let order of orders">
        {{ order.id }} - {{ order.status }}
      </div>
    </div>
  `,
  standalone: true,
  imports: [CommonModule]
})
export class DashboardComponent implements OnInit {
  orders: Order[] = [];

  constructor(
    private orderService: OrderService,
    private ngZone: NgZone  // Inject NgZone
  ) {}

  ngOnInit() {
    // Ensure the interval runs inside Angular's zone
    this.ngZone.run(() => {
      setInterval(() => {
        this.orderService.getOrders().subscribe(orders => {
          this.orders = orders;
          // Change detection triggered automatically!
        });
      }, 1000);
    });
  }
}
```

✅ **This worked!** The UI now updated every second.

### Approach 2: Use RxJS (Recommended)

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { timer } from 'rxjs';
import { switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-dashboard',
  changeDetection: ChangeDetectionStrategy.OnPush,  // More efficient!
  template: `
    <div class="dashboard">
      <h2>Live Orders: {{ (orders$ | async)?.length }}</h2>
      <div *ngFor="let order of (orders$ | async)">
        {{ order.id }} - {{ order.status }}
      </div>
    </div>
  `,
  standalone: true,
  imports: [CommonModule]
})
export class DashboardComponent {
  // RxJS timer runs in Angular zone automatically
  orders$ = timer(0, 1000).pipe(
    switchMap(() => this.orderService.getOrders())
  );

  constructor(private orderService: OrderService) {}
}
```

✅ **Even better!** Using `async` pipe handles subscriptions AND change detection automatically.

But why did clicking update the UI earlier? **Because click events are automatically patched by Zone.js!**

```typescript
// When you click a button:
// 1. Zone.js intercepts the click event
// 2. Zone.js runs your click handler
// 3. Zone.js triggers change detection
// 4. Angular checks all components for changes
// 5. UI updates!
```

## Deep Dive: The Change Detection Algorithm

Now that the immediate problem was solved, Alex wanted to understand the complete mechanism. How does change detection actually work?

### The LView Data Structure

Alex discovered that every component is backed by an **LView** (Logical View) array:

```typescript
// packages/core/src/render3/interfaces/view.ts

/**
 * LView is an array that holds all the data needed to render and check a view
 */
export interface LView<T = unknown> extends Array<any> {
  /** The node into which this LView is inserted */
  [HOST]: RElement | null;

  /** The static data for this view (TView) */
  readonly [TVIEW]: TView;

  /** Flags for this view. See LViewFlags for more info */
  [FLAGS]: LViewFlags;

  /** Parent view (for walking up the tree) */
  [PARENT]: LView | LContainer | null;

  /** Context (usually the component instance) */
  [CONTEXT]: T;

  /** Injector for this view */
  readonly [INJECTOR]: Injector;

  /** Environment (shared across views) */
  [ENVIRONMENT]: LViewEnvironment;

  // ... and 20+ more properties!
}
```

An LView is literally an array with special indices. For example:
- `lView[0]` = HOST element
- `lView[1]` = TVIEW (template view)
- `lView[2]` = FLAGS (our dirty bits!)
- `lView[3]` = PARENT view
- etc.

This array-based structure is **extremely fast** - array access is O(1), and V8 optimizes array access aggressively.

### The LViewFlags System

Alex found the complete set of flags in `view.ts`:

```typescript
// packages/core/src/render3/interfaces/view.ts

export const enum LViewFlags {
  /** The state of the init phase on the first 2 bits */
  InitPhaseStateIncrementer = 0b00000000001,
  InitPhaseStateMask = 0b00000000011,

  /**
   * Whether or not the view is in creationMode.
   * This must be stored in the view rather than using `data` as a marker.
   */
  CreationMode = 1 << 2,  // 0b00000100

  /**
   * Whether or not this LView instance is on its first processing pass.
   */
  FirstLViewPass = 1 << 3,  // 0b00001000

  /** Whether this view has default change detection strategy (checks always) or onPush */
  CheckAlways = 1 << 4,  // 0b00010000

  /** Whether there are any i18n blocks inside this LView */
  HasI18n = 1 << 5,  // 0b00100000

  /** Whether or not this view is currently dirty (needing check) */
  Dirty = 1 << 6,  // 0b01000000

  /** Whether or not this view is currently attached to change detection tree */
  Attached = 1 << 7,  // 0b10000000

  /** Whether or not this view is destroyed */
  Destroyed = 1 << 8,

  /** Whether or not this view is the root view */
  IsRoot = 1 << 9,

  /**
   * Whether this moved LView needs to be refreshed. Similar to the Dirty flag,
   * but used for transplanted and signal views where the parent/ancestor views
   * are not marked dirty as well.
   */
  RefreshView = 1 << 10,

  /** Indicates that the view **or any of its ancestors** have an embedded view injector */
  HasEmbeddedViewInjector = 1 << 11,

  /** Indicates that the view was created with `signals: true` (Angular 17+) */
  SignalView = 1 << 12,

  /**
   * Indicates that this LView has a view underneath it that needs to be refreshed
   * during change detection.
   */
  HasChildViewsToRefresh = 1 << 13,
}
```

These bit flags allow Angular to check multiple conditions with a single bitwise operation:

```typescript
// Check if view should be skipped
function shouldSkipView(lView: LView): boolean {
  const flags = lView[FLAGS];

  // Single bitwise check for multiple conditions!
  return !!(
    flags & LViewFlags.Destroyed ||   // View is destroyed
    flags & LViewFlags.Detached ||    // View is detached
    (!(flags & LViewFlags.CheckAlways) && !(flags & LViewFlags.Dirty))  // OnPush and clean
  );
}
```

### The Change Detection Tree Walk

Alex found the actual tree traversal code:

```typescript
// Simplified from packages/core/src/render3/instructions/change_detection.ts

function detectChangesInView(lView: LView, mode: ChangeDetectionMode): void {
  const tView = lView[TVIEW];
  const flags = lView[FLAGS];

  // Skip if view shouldn't be checked
  if (flags & LViewFlags.Destroyed) return;
  if (!(flags & LViewFlags.Attached)) return;

  // For OnPush components, only check if marked dirty or in Global mode
  if (!(flags & LViewFlags.CheckAlways)) {
    if (mode === ChangeDetectionMode.Targeted && !(flags & LViewFlags.RefreshView)) {
      return;
    }
  }

  // Enter this view's context
  enterView(lView);

  try {
    // 1. Run OnInit, DoCheck hooks
    if (flags & LViewFlags.FirstLViewPass) {
      executeInitAndCheckHooks(lView, tView);
    } else {
      executeCheckHooks(lView, tView);
    }

    // 2. Refresh the view (check bindings, update DOM)
    refreshView(tView, lView, tView.template, lView[CONTEXT]);

    // 3. Check content queries
    refreshContentQueries(tView, lView);

    // 4. Clear the dirty flag
    lView[FLAGS] &= ~(LViewFlags.Dirty | LViewFlags.RefreshView);

    // 5. Recursively check child components
    const components = tView.components;
    if (components !== null) {
      for (let i = 0; i < components.length; i++) {
        const componentIndex = components[i];
        const componentView = getComponentLViewByIndex(componentIndex, lView);

        // Recursive call!
        detectChangesInView(componentView, mode);
      }
    }

    // 6. Run AfterViewChecked
    if (tView.viewHooks !== null) {
      executeViewHooks(lView, tView.viewHooks);
    }

  } finally {
    leaveView();
  }
}
```

💡 **Key Insight #2**: Change detection is a **depth-first tree traversal** from root to leaves!

The tree looks like this:

```
┌─────────────────┐
│   AppComponent  │  ← Root (always checked)
│  CD: Default    │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
┌───┴────┐ ┌─┴──────┐
│Dashboard│ │ Sidebar│
│ Default │ │ OnPush │  ← Only checked if dirty
└───┬────┘ └────────┘
    │
┌───┴──────┐
│OrderCard │
│  OnPush  │  ← Only checked if parent checked AND dirty
└──────────┘
```

### Change Detection Strategies

Alex opened `constants.ts` to understand the two strategies:

```typescript
// packages/core/src/change_detection/constants.ts

export enum ChangeDetectionStrategy {
  /**
   * Use the `CheckOnce` strategy, meaning that automatic change detection is deactivated
   * until reactivated by setting the strategy to `Default` (`CheckAlways`).
   * Change detection can still be explicitly invoked.
   */
  OnPush = 0,

  /**
   * Use the default `CheckAlways` strategy, in which change detection is automatic until
   * explicitly deactivated.
   */
  Default = 1,
}
```

#### Default Strategy: Check Always

```typescript
@Component({
  selector: 'app-user-list',
  changeDetection: ChangeDetectionStrategy.Default,  // Default
  template: `
    <div *ngFor="let user of users">
      {{ user.name }} - {{ user.status }}
      <button (click)="refresh()">Refresh</button>
    </div>
  `
})
export class UserListComponent {
  users: User[] = [];

  constructor(private userService: UserService) {
    // ANY async operation triggers CD for this component
    setInterval(() => {
      this.users = [...this.users];  // Even if nothing changed!
    }, 1000);
  }

  refresh() {
    // This also triggers CD for entire app
    this.userService.refresh();
  }
}
```

With `Default` strategy:
- ✅ Component is **always checked** during CD
- ✅ Every async event checks **every Default component**
- ❌ Can be slow with many components
- ❌ Checks happen even when nothing changed

#### OnPush Strategy: Check Selectively

```typescript
@Component({
  selector: 'app-user-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>Status: {{ user.status }}</p>
      <button (click)="toggleStatus()">Toggle</button>
    </div>
  `
})
export class UserCardComponent {
  @Input() user!: User;

  toggleStatus() {
    // ❌ Won't update view - same object reference!
    this.user.status = this.user.status === 'active' ? 'inactive' : 'active';

    // ✅ This works - new object reference
    this.user = { ...this.user, status: this.user.status === 'active' ? 'inactive' : 'active' };
  }
}
```

OnPush only checks when:

1. **@Input() reference changes** (shallow comparison, not deep equality!)
   ```typescript
   // Parent component
   this.user = { ...this.user };  // ✅ New reference, child checks
   this.user.name = 'New Name';    // ❌ Same reference, child doesn't check
   ```

2. **Event handler in template runs**
   ```typescript
   // (click)="toggleStatus()" triggers CD for this component
   ```

3. **Async pipe emits new value**
   ```typescript
   // {{ data$ | async }} triggers CD when observable emits
   ```

4. **Manually marked via ChangeDetectorRef.markForCheck()**
   ```typescript
   this.cdr.markForCheck();
   ```

### ChangeDetectorRef: Manual Control

Alex discovered the `ChangeDetectorRef` API:

```typescript
// packages/core/src/change_detection/change_detector_ref.ts

export abstract class ChangeDetectorRef {
  /**
   * When a view uses the OnPush change detection strategy, explicitly marks the view
   * as changed so that it can be checked again.
   */
  abstract markForCheck(): void;

  /**
   * Detaches this view from the change-detection tree.
   * A detached view is not checked until it is reattached.
   */
  abstract detach(): void;

  /**
   * Checks this view and its children. Use in combination with detach()
   * to implement local change detection checks.
   */
  abstract detectChanges(): void;

  /**
   * Re-attaches the previously detached view to the change detection tree.
   */
  abstract reattach(): void;
}
```

Practical usage:

```typescript
@Component({
  selector: 'app-streaming-data',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div *ngFor="let item of items">{{ item }}</div>
    <button (click)="pause()">Pause</button>
    <button (click)="resume()">Resume</button>
  `
})
export class StreamingDataComponent implements OnInit, OnDestroy {
  items: string[] = [];
  private subscription?: Subscription;

  constructor(
    private dataService: DataService,
    private cdr: ChangeDetectorRef
  ) {}

  ngOnInit() {
    this.subscription = this.dataService.stream$.subscribe(item => {
      this.items.push(item);

      // Data changed, but we're OnPush - manually mark dirty
      this.cdr.markForCheck();
    });
  }

  pause() {
    // Detach from CD tree - we won't be checked anymore
    this.cdr.detach();
  }

  resume() {
    // Re-attach to CD tree
    this.cdr.reattach();

    // Force immediate check
    this.cdr.detectChanges();
  }

  ngOnDestroy() {
    this.subscription?.unsubscribe();
  }
}
```

## Zone.js Deep Dive

Alex wanted to understand exactly how Zone.js works.

### What is a Zone?

A Zone is an execution context that persists across async operations:

```typescript
// Conceptual Zone implementation
class Zone {
  constructor(private parent: Zone | null, private name: string) {}

  // Run a function in this zone
  run<T>(callback: () => T): T {
    const previousZone = currentZone;
    currentZone = this;

    try {
      return callback();
    } finally {
      currentZone = previousZone;
    }
  }

  // Fork creates a child zone with custom hooks
  fork(spec: ZoneSpec): Zone {
    return new Zone(this, spec.name);
  }
}
```

### How Angular Uses Zone.js

Angular creates a special zone with hooks:

```typescript
// Simplified from @angular/core

class NgZone {
  private _inner: Zone;
  private _outer: Zone;

  constructor() {
    this._outer = Zone.current;  // The zone outside Angular

    this._inner = Zone.current.fork({
      name: 'angular',
      onInvokeTask: (delegate, current, target, task, applyThis, applyArgs) => {
        try {
          // Run the task
          return delegate.invokeTask(target, task, applyThis, applyArgs);
        } finally {
          // After task completes, check if we should trigger CD
          if (this.shouldScheduleChangeDetection()) {
            this.onMicrotaskEmpty.emit();  // Triggers CD!
          }
        }
      }
    });
  }

  run<T>(fn: () => T): T {
    return this._inner.run(fn);
  }

  runOutsideAngular<T>(fn: () => T): T {
    return this._outer.run(fn);
  }
}
```

### Performance: Running Outside Angular

For performance-critical code that doesn't affect the UI, run outside Angular:

```typescript
@Component({
  selector: 'app-animation',
  template: `<canvas #canvas></canvas>`
})
export class AnimationComponent implements OnInit {
  @ViewChild('canvas', { static: true }) canvas!: ElementRef<HTMLCanvasElement>;
  private animationId?: number;

  constructor(private ngZone: NgZone) {}

  ngOnInit() {
    const ctx = this.canvas.nativeElement.getContext('2d')!;

    // Run animation loop outside Angular - CD would be triggered 60 times/second!
    this.ngZone.runOutsideAngular(() => {
      const animate = () => {
        // Draw frame
        ctx.clearRect(0, 0, 800, 600);
        ctx.fillRect(Math.random() * 800, Math.random() * 600, 50, 50);

        // Schedule next frame (outside Angular)
        this.animationId = requestAnimationFrame(animate);
      };

      animate();
    });
  }

  ngOnDestroy() {
    // Clean up
    if (this.animationId) {
      cancelAnimationFrame(this.animationId);
    }
  }
}
```

Without `runOutsideAngular`, requestAnimationFrame would trigger change detection **60 times per second**, checking every component!

## Performance Optimization Patterns

Armed with deep knowledge, Alex optimized the dashboard application.

### Pattern 1: OnPush Everything

```typescript
// Before: Slow (Default CD)
@Component({
  selector: 'app-order-list',
  changeDetection: ChangeDetectionStrategy.Default,  // Checked every CD cycle
  template: `
    <app-order-card *ngFor="let order of orders" [order]="order"></app-order-card>
  `
})
export class OrderListComponent {
  @Input() orders: Order[] = [];
}

// After: Fast (OnPush CD)
@Component({
  selector: 'app-order-list',
  changeDetection: ChangeDetectionStrategy.OnPush,  // Only checked when inputs change
  template: `
    <app-order-card
      *ngFor="let order of orders; trackBy: trackById"
      [order]="order">
    </app-order-card>
  `
})
export class OrderListComponent {
  @Input() orders: Order[] = [];

  // trackBy prevents unnecessary DOM updates
  trackById(index: number, order: Order): number {
    return order.id;
  }
}
```

### Pattern 2: Immutable Data Structures

```typescript
// ❌ Bad: Mutating objects (OnPush won't detect)
updateOrder(orderId: number, status: string) {
  const order = this.orders.find(o => o.id === orderId);
  if (order) {
    order.status = status;  // Same reference!
  }
}

// ✅ Good: Creating new objects (OnPush detects)
updateOrder(orderId: number, status: string) {
  this.orders = this.orders.map(order =>
    order.id === orderId
      ? { ...order, status }  // New object
      : order
  );
}
```

### Pattern 3: Async Pipe Everywhere

```typescript
// ❌ Bad: Manual subscription (must call markForCheck)
@Component({
  selector: 'app-data',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<div>{{ data }}</div>`
})
export class DataComponent implements OnInit, OnDestroy {
  data: any;
  private sub?: Subscription;

  constructor(private service: DataService, private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    this.sub = this.service.getData().subscribe(data => {
      this.data = data;
      this.cdr.markForCheck();  // Easy to forget!
    });
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();  // Easy to forget!
  }
}

// ✅ Good: Async pipe (handles everything)
@Component({
  selector: 'app-data',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<div>{{ data$ | async }}</div>`  // Automatically calls markForCheck!
})
export class DataComponent {
  data$ = this.service.getData();  // Automatically unsubscribes!

  constructor(private service: DataService) {}
}
```

### Pattern 4: Signals (Angular 17+)

Angular 17 introduced Signals, which integrate seamlessly with change detection:

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-counter',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>
      <p>Count: {{ count() }}</p>
      <p>Double: {{ double() }}</p>
      <button (click)="increment()">+1</button>
    </div>
  `
})
export class CounterComponent {
  // Signal automatically triggers CD when changed
  count = signal(0);

  // Computed signal automatically recalculates
  double = computed(() => this.count() * 2);

  increment() {
    this.count.update(n => n + 1);  // CD triggered automatically!
  }
}
```

Signals are **the future** of Angular change detection. They provide:
- ✅ Fine-grained reactivity (only affected components check)
- ✅ Automatic dependency tracking
- ✅ Better performance than Zone.js
- ✅ Works perfectly with OnPush

## Real-World Example: Optimized Dashboard

Alex rebuilt the dashboard with all the learnings:

```typescript
// order.service.ts
import { Injectable, signal } from '@angular/core';
import { interval } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class OrderService {
  // Using signals for reactive state
  private ordersSignal = signal<Order[]>([]);

  // Public read-only signal
  orders = this.ordersSignal.asReadonly();

  // Computed signals for derived state
  pendingCount = computed(() =>
    this.orders().filter(o => o.status === 'pending').length
  );

  processingCount = computed(() =>
    this.orders().filter(o => o.status === 'processing').length
  );

  constructor(private http: HttpClient) {
    // Poll for updates
    interval(1000).pipe(
      switchMap(() => this.http.get<Order[]>('/api/orders'))
    ).subscribe(orders => {
      this.ordersSignal.set(orders);  // CD triggered automatically!
    });
  }
}

// dashboard.component.ts
@Component({
  selector: 'app-dashboard',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="dashboard">
      <div class="stats">
        <h2>Live Orders: {{ orderService.orders().length }}</h2>
        <span>Pending: {{ orderService.pendingCount() }}</span>
        <span>Processing: {{ orderService.processingCount() }}</span>
      </div>

      <app-order-list [orders]="orderService.orders()"></app-order-list>
    </div>
  `,
  standalone: true,
  imports: [CommonModule, OrderListComponent]
})
export class DashboardComponent {
  // Inject service - signals handle CD automatically
  constructor(public orderService: OrderService) {}
}

// order-list.component.ts
@Component({
  selector: 'app-order-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="order-list">
      <app-order-card
        *ngFor="let order of orders; trackBy: trackById"
        [order]="order">
      </app-order-card>
    </div>
  `,
  standalone: true,
  imports: [CommonModule, OrderCardComponent]
})
export class OrderListComponent {
  @Input() orders: Order[] = [];

  trackById(index: number, order: Order): number {
    return order.id;
  }
}

// order-card.component.ts
@Component({
  selector: 'app-order-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="card" [class.pending]="order.status === 'pending'">
      <span class="id">{{ order.id }}</span>
      <span class="status">{{ order.status }}</span>
      <span class="total">\${{ order.total }}</span>
    </div>
  `,
  standalone: true,
  imports: [CommonModule]
})
export class OrderCardComponent {
  @Input() order!: Order;
}
```

**Performance Results:**
- Before: ~45ms per CD cycle (all components checked)
- After: ~3ms per CD cycle (only dirty components checked)
- **15x faster!** ⚡

## Debugging Change Detection Issues

Alex compiled a debugging toolkit:

### 1. Angular DevTools Profiler

Chrome DevTools → Angular tab → Profiler

Shows:
- Which components are being checked
- How long each component takes
- Change detection cycles

### 2. Enable Change Detection Logging

```typescript
// main.ts
import { enableDebugTools } from '@angular/platform-browser';
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

platformBrowserDynamic().bootstrapModule(AppModule)
  .then(moduleRef => {
    const applicationRef = moduleRef.injector.get(ApplicationRef);
    const componentRef = applicationRef.components[0];

    // Enable debug tools
    enableDebugTools(componentRef);

    // In console:
    // ng.profiler.timeChangeDetection() - measure CD performance
  });
```

### 3. Manual CD Inspection

```typescript
import { Component, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-debug',
  template: `<div>{{ data }}</div>`
})
export class DebugComponent implements OnInit {
  data = 'initial';

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    // Check CD state
    console.log('Detached?', (this.cdr as any)._view?.state & 2);

    // Manually trigger CD
    this.data = 'changed';
    this.cdr.detectChanges();  // Sync CD

    // Or mark for next CD cycle
    this.cdr.markForCheck();  // Async CD
  }
}
```

### 4. ExpressionChangedAfterItHasBeenCheckedError

A common error:

```typescript
// ❌ Causes error
@Component({
  selector: 'app-parent',
  template: `<app-child [value]="value"></app-child>`
})
export class ParentComponent implements AfterViewInit {
  value = 1;

  ngAfterViewInit() {
    // Changes value AFTER CD checked it
    this.value = 2;  // ERROR in dev mode!
  }
}

// ✅ Fix: Schedule change for next tick
@Component({
  selector: 'app-parent',
  template: `<app-child [value]="value"></app-child>`
})
export class ParentComponent implements AfterViewInit {
  value = 1;

  constructor(private cdr: ChangeDetectorRef) {}

  ngAfterViewInit() {
    // Schedule for next CD cycle
    setTimeout(() => {
      this.value = 2;  // No error
    });

    // Or use Promise
    Promise.resolve().then(() => {
      this.value = 3;
    });
  }
}
```

### 5. Zone.js Debugging

```typescript
// Track zone tasks
Zone.current.onScheduleTask = (delegate, current, target, task) => {
  console.log('Task scheduled:', task.type, task.source);
  return delegate.scheduleTask(target, task);
};

// Check if code is running in Angular zone
function isInAngularZone(): boolean {
  return Zone.current.name === 'angular';
}

// Run with debug
import 'zone.js/plugins/zone-error';  // Better error messages
```

## Key Takeaways

After this deep dive, Alex had mastered change detection:

### 1. **Change Detection is Tree Traversal**
Angular walks the component tree from root to leaves, checking bindings.

### 2. **Zone.js Automates CD**
Zone.js patches async APIs and triggers change detection automatically.

### 3. **LView and Flags are Fast**
Array-based LView and bit flags make CD extremely efficient.

### 4. **OnPush is Your Friend**
OnPush strategy provides massive performance gains with minimal effort:
- Only check when inputs change
- Requires immutable data patterns
- Works perfectly with async pipe and signals

### 5. **Immutability Matters**
OnPush only detects reference changes, not deep equality:
```typescript
// ❌ Won't detect
obj.property = newValue;

// ✅ Will detect
obj = { ...obj, property: newValue };
```

### 6. **Async Pipe is Magic**
- Automatically subscribes/unsubscribes
- Automatically calls markForCheck()
- Works perfectly with OnPush

### 7. **Signals are the Future**
Angular 17+ signals provide fine-grained reactivity without Zone.js overhead.

### 8. **ChangeDetectorRef for Manual Control**
- `markForCheck()` - mark view and ancestors dirty
- `detectChanges()` - run CD immediately (sync)
- `detach()` / `reattach()` - opt in/out of CD tree

### 9. **Run Outside Angular When Possible**
```typescript
this.ngZone.runOutsideAngular(() => {
  // High-frequency operations that don't affect UI
});
```

### 10. **Profile and Optimize**
Use Angular DevTools profiler to find slow components.

## Practical Applications

Alex now applies this knowledge to:

1. **Build Performant Apps** - OnPush everywhere, immutable data, async pipe
2. **Debug Faster** - Understanding CD makes issues obvious
3. **Optimize Strategically** - Profile first, optimize hot paths
4. **Design Better** - Consider CD when designing component architecture

## What's Next?

Understanding change detection unlocked incredible performance improvements. But new questions emerged:

- *When exactly do lifecycle hooks run?*
- *What's the difference between OnInit, AfterViewInit, and AfterContentInit?*
- *In what order do hooks execute?*
- *When should I load data or manipulate the DOM?*

These questions led to the next investigation...

---

**Next**: [Chapter 3: The Lifecycle Chronicles](03-component-lifecycle.md)

## Further Reading

- Source: `packages/core/src/change_detection/`
- Source: `packages/core/src/render3/instructions/change_detection.ts`
- Source: `packages/core/src/render3/interfaces/view.ts`
- Zone.js: `packages/zone.js/`
- Signals: `packages/core/primitives/signals/`
- Documentation: https://angular.dev/guide/change-detection
- OnPush Guide: https://angular.dev/best-practices/skipping-subtrees
- Signals Guide: https://angular.dev/guide/signals

## Notes from Alex's Journal

*"Mind = blown. Change detection isn't magic - it's just tree traversal with Zone.js! And those LViewFlags using bit operations? Brilliant for performance.*

*The key realization: immutability + OnPush + async pipe = fast apps. Super simple pattern.*

*Signals are even better - fine-grained reactivity without Zone.js overhead. That's the future right there.*

*Most important lesson: Zone.js triggers CD automatically, but you can opt out with runOutsideAngular(). Game changer for animations and high-frequency operations.*

*Next up: figure out lifecycle hooks. When exactly does ngOnInit run vs ngAfterViewInit? And what order do hooks fire in parent vs child components?"*
