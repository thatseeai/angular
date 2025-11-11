# Chapter 3: The Lifecycle Chronicles

> *"My ViewChild is undefined! But I just declared it!"*

## The Problem

After mastering dependency injection and change detection, Alex felt unstoppable. The next feature seemed straightforward: integrate a third-party charting library to visualize sales data.

The plan was simple:
1. Add a canvas element to the template
2. Get a reference with `@ViewChild`
3. Initialize the chart library
4. Done!

Alex wrote the component:

```typescript
import { Component, ViewChild, ElementRef } from '@angular/core';
import { Chart } from 'chart.js';

@Component({
  selector: 'app-sales-chart',
  template: `
    <div class="chart-container">
      <h2>Sales Overview</h2>
      <canvas #chartCanvas></canvas>
    </div>
  `,
  standalone: true
})
export class SalesChartComponent {
  @ViewChild('chartCanvas') chartCanvas!: ElementRef<HTMLCanvasElement>;

  private chart?: Chart;

  constructor(private salesService: SalesService) {
    console.log('Constructor - chartCanvas:', this.chartCanvas); // undefined

    // Initialize chart
    this.initChart(); // ❌ ERROR: Cannot read property 'nativeElement' of undefined
  }

  initChart() {
    const ctx = this.chartCanvas.nativeElement.getContext('2d');

    this.chart = new Chart(ctx, {
      type: 'line',
      data: {
        labels: ['Jan', 'Feb', 'Mar', 'Apr', 'May'],
        datasets: [{
          label: 'Sales',
          data: [12, 19, 3, 5, 2]
        }]
      }
    });
  }
}
```

Alex ran the app:

```
ERROR TypeError: Cannot read property 'nativeElement' of undefined
    at SalesChartComponent.initChart
    at new SalesChartComponent
```

**"What? I just declared it with @ViewChild!"**

Alex tried moving the code to `ngOnInit`:

```typescript
ngOnInit() {
  console.log('ngOnInit - chartCanvas:', this.chartCanvas); // Still undefined!
  this.initChart(); // Still crashes!
}
```

Still undefined!

Then, in frustration, Alex tried `ngAfterViewInit`:

```typescript
ngAfterViewInit() {
  console.log('ngAfterViewInit - chartCanvas:', this.chartCanvas); // ElementRef!
  this.initChart(); // ✅ Works!
}
```

**It worked!** But why? What's the difference between `ngOnInit` and `ngAfterViewInit`? And why are there so many lifecycle hooks anyway?

This mystery led Alex on a journey through Angular's component lifecycle system.

## The Investigation Begins

Alex needed to understand the order and purpose of every lifecycle hook. Time to dive into the Angular source code.

### Discovery 1: The Complete Lifecycle

Alex found the lifecycle hooks interface definitions:

```typescript
// packages/core/src/core.ts

/**
 * A lifecycle hook that is called when any data-bound property of a directive changes.
 */
export interface OnChanges {
  ngOnChanges(changes: SimpleChanges): void;
}

/**
 * A lifecycle hook that is called after Angular has initialized all data-bound properties.
 */
export interface OnInit {
  ngOnInit(): void;
}

/**
 * A lifecycle hook that is called when Angular checks the component for changes.
 */
export interface DoCheck {
  ngDoCheck(): void;
}

/**
 * A lifecycle hook that is called after Angular has projected external content into the component.
 */
export interface AfterContentInit {
  ngAfterContentInit(): void;
}

/**
 * A lifecycle hook that is called after Angular checks the projected content.
 */
export interface AfterContentChecked {
  ngAfterContentChecked(): void;
}

/**
 * A lifecycle hook that is called after Angular has initialized the component's view.
 */
export interface AfterViewInit {
  ngAfterViewInit(): void;
}

/**
 * A lifecycle hook that is called after Angular checks the component's view.
 */
export interface AfterViewChecked {
  ngAfterViewChecked(): void;
}

/**
 * A lifecycle hook that is called when a component is destroyed.
 */
export interface OnDestroy {
  ngOnDestroy(): void;
}
```

💡 **Key Insight #1**: There are **8 lifecycle hooks**, and they execute in a specific order!

### Discovery 2: The Execution Order

Alex created a test component to log every lifecycle hook:

```typescript
@Component({
  selector: 'app-lifecycle-test',
  template: `
    <div>
      <p>Name: {{ name }}</p>
      <ng-content></ng-content>
      <app-child [data]="childData"></app-child>
    </div>
  `,
  standalone: true,
  imports: [ChildComponent]
})
export class LifecycleTestComponent implements
  OnChanges, OnInit, DoCheck,
  AfterContentInit, AfterContentChecked,
  AfterViewInit, AfterViewChecked, OnDestroy {

  @Input() name = '';
  childData = 'test';

  constructor() {
    console.log('Parent: Constructor');
  }

  ngOnChanges(changes: SimpleChanges) {
    console.log('Parent: ngOnChanges', changes);
  }

  ngOnInit() {
    console.log('Parent: ngOnInit');
  }

  ngDoCheck() {
    console.log('Parent: ngDoCheck');
  }

  ngAfterContentInit() {
    console.log('Parent: ngAfterContentInit');
  }

  ngAfterContentChecked() {
    console.log('Parent: ngAfterContentChecked');
  }

  ngAfterViewInit() {
    console.log('Parent: ngAfterViewInit');
  }

  ngAfterViewChecked() {
    console.log('Parent: ngAfterViewChecked');
  }

  ngOnDestroy() {
    console.log('Parent: ngOnDestroy');
  }
}
```

The console output revealed the order:

```
// INITIAL RENDER
Parent: Constructor
Parent: ngOnChanges { name: { firstChange: true, ... } }
Parent: ngOnInit
Parent: ngDoCheck
  Child: Constructor
  Child: ngOnChanges
  Child: ngOnInit
  Child: ngDoCheck
  Child: ngAfterContentInit
  Child: ngAfterContentChecked
  Child: ngAfterViewInit
  Child: ngAfterViewChecked
Parent: ngAfterContentInit
Parent: ngAfterContentChecked
Parent: ngAfterViewInit
Parent: ngAfterViewChecked

// ON EVERY CHANGE DETECTION
Parent: ngDoCheck
  Child: ngDoCheck
  Child: ngAfterContentChecked
  Child: ngAfterViewChecked
Parent: ngAfterContentChecked
Parent: ngAfterViewChecked

// ON DESTROY
Parent: ngOnDestroy
  Child: ngOnDestroy
```

💡 **Key Insight #2**: Lifecycle hooks follow a **parent-before-child pattern** for initialization, but **child-before-parent** for destruction!

### Discovery 3: When ViewChild Becomes Available

Alex traced through the Angular source to understand when `@ViewChild` queries are resolved:

```typescript
// packages/core/src/render3/hooks.ts (simplified)

/**
 * Calls lifecycle hooks for a view
 */
export function callHooks(currentView: LView, hooks: HookData) {
  for (let i = 0; i < hooks.length; i += 2) {
    const hook = hooks[i] as number;
    const hookFn = hooks[i + 1] as () => void;

    switch (hook) {
      case LifecycleHooks.OnInit:
        hookFn.call(currentView[CONTEXT]);
        break;
      case LifecycleHooks.AfterViewInit:
        // View queries are resolved BEFORE this hook!
        hookFn.call(currentView[CONTEXT]);
        break;
      // ... other hooks
    }
  }
}

/**
 * Refreshes view including child components and queries
 */
export function refreshView(tView: TView, lView: LView, template: ComponentTemplate | null, context: any) {
  // 1. Update component bindings
  executeTemplate(tView, lView, template, RenderFlags.Update, context);

  // 2. Refresh child components
  refreshChildComponents(lView);

  // 3. Resolve ViewChild queries ← This happens BEFORE AfterViewInit!
  refreshViewQueries(tView, lView);

  // 4. Call AfterViewInit hook
  if (tView.viewHooks !== null) {
    executeViewHooks(lView, tView.viewHooks, InitPhaseState.AfterViewInit);
  }
}
```

💡 **Key Insight #3**: `@ViewChild` queries are resolved **between view creation and AfterViewInit**. That's why they're undefined in `ngOnInit` but available in `ngAfterViewInit`!

### Discovery 4: Content vs View

Alex was confused about the difference between "Content" and "View". The source code clarified:

```typescript
// packages/core/src/render3/instructions/projection.ts

/**
 * Adds LView or LContainer to the end of the current view tree.
 *
 * This structure will be used to project content into the current view.
 */
export function ɵɵprojection(
  nodeIndex: number,
  selectorIndex: number = 0,
  attrs?: TAttributes
) {
  const lView = getLView();
  const tView = getTView();
  const tProjectionNode = getOrCreateTNode(tView, HEADER_OFFSET + nodeIndex, TNodeType.Projection, null, attrs);

  // We can't use viewData[HOST_NODE] because projection nodes can be nested in embedded views.
  if (tProjectionNode.projection === null) {
    tProjectionNode.projection = selectorIndex;
  }

  // `<ng-content>` has no content itself, we're simply projecting children
  setCurrentTNodeAsNotParent();

  // Get parent's content children
  const componentView = lView[DECLARATION_COMPONENT_VIEW];
  const componentHost = componentView[T_HOST] as TElementNode;
  const componentTemplate = componentHost.projection as (TNode | null)[];

  // Project the appropriate content
  project(lView, tView, selectorIndex);
}
```

Alex realized:

- **View** = Component's own template (defined in `@Component.template`)
- **Content** = Children projected via `<ng-content>` from parent

Example:

```typescript
// Parent component
@Component({
  selector: 'app-parent',
  template: `
    <app-card>
      <p>This is CONTENT (from parent)</p>  <!-- ← This is content -->
    </app-card>
  `
})
export class ParentComponent {}

// Child component
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <h2>Card Title</h2>  <!-- ← This is view -->
      <ng-content></ng-content>  <!-- Content projected here -->
    </div>
  `
})
export class CardComponent {
  @ContentChild('content') content?: ElementRef;  // Access projected content
  @ViewChild('title') title?: ElementRef;         // Access own view
}
```

Timeline:

1. `ngAfterContentInit` - Content (`<p>This is CONTENT</p>`) is projected
2. `ngAfterViewInit` - View (`<h2>Card Title</h2>`) is initialized

## The Complete Lifecycle Deep Dive

Armed with understanding, Alex documented each lifecycle hook in detail.

### 0. Constructor

**When**: During component instantiation, before any lifecycle hooks

**Purpose**: Dependency injection

**Available**:
- ✅ Injected services
- ❌ `@Input()` properties (not set yet)
- ❌ `@ViewChild()` / `@ContentChild()` (not resolved yet)
- ❌ DOM elements

**Source Code**:

```typescript
// packages/core/src/render3/component_ref.ts

export class ComponentFactory<T> extends AbstractComponentFactory<T> {
  create(
    injector: Injector,
    projectableNodes?: any[][],
    rootSelectorOrNode?: any,
    environmentInjector?: EnvironmentInjector | NgModuleRef<any>
  ): ComponentRef<T> {
    // ... setup code

    // Call constructor
    const instance = new this.componentType(...dependencies);  // ← Constructor called here

    // Set up view
    const lView = createLView(/* ... */);

    // Run initialization hooks
    initializeDirectives(tView, lView, /* ... */);

    return new ComponentRef(/* ... */);
  }
}
```

**Best Practices**:

```typescript
// ✅ Good - Simple initialization
constructor(
  private http: HttpClient,
  private router: Router
) {
  // Initialize simple properties
  this.items = [];
  this.count = 0;
}

// ❌ Bad - Complex logic
constructor(private service: DataService) {
  // DON'T do HTTP requests
  this.service.getData().subscribe(/* ... */);  // ❌

  // DON'T access inputs
  console.log(this.userId);  // ❌ undefined!

  // DON'T access DOM
  document.querySelector('.my-element');  // ❌ Not rendered yet!
}
```

### 1. ngOnChanges

**When**:
- First call: Before `ngOnInit` (if component has inputs)
- Subsequent calls: Whenever `@Input()` properties change

**Purpose**: React to input property changes

**Receives**: `SimpleChanges` object mapping property names to changes

**Source Code**:

```typescript
// packages/core/src/render3/instructions/shared.ts

/**
 * Updates input bindings and calls ngOnChanges if needed
 */
export function refreshDynamicEmbeddedViews(lView: LView) {
  for (let current = lView[CHILD_HEAD]; current !== null; current = current[NEXT]) {
    const tView = current[TVIEW];

    // Check if any inputs changed
    if (tView.type === TViewType.Embedded) {
      const changesObj = collectInputChanges(current);

      if (changesObj !== null) {
        const instance = current[CONTEXT];

        // Call ngOnChanges with changes
        if (instance.ngOnChanges) {
          instance.ngOnChanges(changesObj);
        }
      }
    }
  }
}
```

**Usage**:

```typescript
@Component({
  selector: 'app-user-card',
  template: `
    <div class="card" [class.premium]="isPremium">
      <h3>{{ user?.name }}</h3>
      <p>Member since: {{ memberSince }}</p>
    </div>
  `
})
export class UserCardComponent implements OnChanges {
  @Input() user?: User;
  @Input() showDetails = false;

  isPremium = false;
  memberSince = '';

  ngOnChanges(changes: SimpleChanges) {
    console.log('Changes:', changes);

    // Check specific property
    if (changes['user']) {
      const change = changes['user'];

      console.log('First change?', change.firstChange);
      console.log('Previous:', change.previousValue);
      console.log('Current:', change.currentValue);

      if (change.currentValue) {
        this.isPremium = change.currentValue.tier === 'premium';
        this.memberSince = new Date(change.currentValue.joinedDate).toLocaleDateString();
      }
    }

    if (changes['showDetails'] && !changes['showDetails'].firstChange) {
      console.log('Details visibility changed');
    }
  }
}
```

**SimpleChanges Structure**:

```typescript
interface SimpleChanges {
  [propName: string]: SimpleChange;
}

interface SimpleChange {
  previousValue: any;
  currentValue: any;
  firstChange: boolean;
  isFirstChange(): boolean;
}

// Example:
{
  user: {
    previousValue: { id: 1, name: 'Alice' },
    currentValue: { id: 1, name: 'Alice Updated' },
    firstChange: false
  },
  showDetails: {
    previousValue: false,
    currentValue: true,
    firstChange: false
  }
}
```

### 2. ngOnInit

**When**: Once, after first `ngOnChanges` (or immediately if no inputs)

**Purpose**: Component initialization logic

**Available**:
- ✅ `@Input()` properties (set)
- ✅ Injected services
- ❌ `@ViewChild()` / `@ViewChildren()` (not resolved yet)
- ❌ `@ContentChild()` / `@ContentChildren()` (not resolved yet)

**Source Code**:

```typescript
// packages/core/src/render3/hooks.ts

export function executeInitAndCheckHooks(lView: LView, tView: TView) {
  const initHooks = tView.initHooks;

  if (initHooks !== null) {
    for (let i = 0; i < initHooks.length; i += 2) {
      const hook = initHooks[i + 1] as () => void;

      // Call ngOnInit
      hook.call(lView[CONTEXT]);
    }
  }

  // Mark that init phase is complete
  lView[FLAGS] &= ~LViewFlags.CreationMode;
}
```

**Usage**:

```typescript
@Component({
  selector: 'app-dashboard',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="!loading && data">
      <h2>{{ data.title }}</h2>
      <p>{{ data.description }}</p>
    </div>
  `
})
export class DashboardComponent implements OnInit {
  @Input() userId!: number;  // Available here!

  data?: DashboardData;
  loading = false;

  constructor(private dataService: DataService) {
    // userId is undefined here
  }

  ngOnInit() {
    // ✅ Perfect place for data loading
    this.loading = true;

    this.dataService.getDashboard(this.userId).subscribe({
      next: (data) => {
        this.data = data;
        this.loading = false;
      },
      error: (err) => {
        console.error('Failed to load dashboard', err);
        this.loading = false;
      }
    });
  }
}
```

**Best Practices**:

```typescript
// ✅ Good - Data loading
ngOnInit() {
  this.loadData();
  this.initializeForm();
  this.setupSubscriptions();
}

// ✅ Good - Use inputs
ngOnInit() {
  console.log('User ID:', this.userId);  // Available!
}

// ❌ Bad - Accessing view children
ngOnInit() {
  console.log(this.myViewChild);  // undefined!
}
```

### 3. ngDoCheck

**When**: Every change detection cycle, immediately after `ngOnChanges` or `ngOnInit`

**Purpose**: Custom change detection logic

**Warning**: Called **very frequently** - keep it fast!

**Source Code**:

```typescript
// packages/core/src/render3/instructions/change_detection.ts

function detectChangesInView(lView: LView, mode: ChangeDetectionMode) {
  // ... setup

  // Call DoCheck hook every CD cycle
  executeCheckHooks(lView, tView);

  // ... refresh view
}

function executeCheckHooks(lView: LView, tView: TView) {
  const checkHooks = tView.checkHooks;

  if (checkHooks !== null) {
    for (let i = 0; i < checkHooks.length; i++) {
      const hook = checkHooks[i] as () => void;
      hook.call(lView[CONTEXT]);  // Call ngDoCheck
    }
  }
}
```

**Usage**:

```typescript
@Component({
  selector: 'app-complex-list',
  template: `
    <div *ngFor="let item of items">
      {{ item.name }} - {{ item.value }}
    </div>
  `
})
export class ComplexListComponent implements DoCheck {
  @Input() items: Item[] = [];

  private differ: IterableDiffer<Item>;

  constructor(private differs: IterableDiffers) {
    this.differ = this.differs.find([]).create<Item>((index, item) => item.id);
  }

  ngDoCheck() {
    // Detect changes in array contents (not just reference)
    const changes = this.differ.diff(this.items);

    if (changes) {
      console.log('Array contents changed!');

      changes.forEachAddedItem(record => {
        console.log('Added:', record.item);
      });

      changes.forEachRemovedItem(record => {
        console.log('Removed:', record.item);
      });

      changes.forEachMovedItem(record => {
        console.log('Moved:', record.item);
      });
    }
  }
}
```

**Performance Warning**:

```typescript
// ❌ VERY BAD - Heavy computation on every CD!
ngDoCheck() {
  // Called on EVERY change detection (could be 100+ times/second)
  this.expensiveCalculation();  // ❌ Will kill performance!

  this.http.get('/api/data').subscribe(/* ... */);  // ❌ HTTP request every CD!?
}

// ✅ Good - Fast checks only
ngDoCheck() {
  // Quick reference check
  if (this.previousValue !== this.currentValue) {
    this.handleChange();
  }
}
```

### 4. ngAfterContentInit

**When**: Once, after first `ngDoCheck`, when projected content is initialized

**Purpose**: Access and initialize `@ContentChild` / `@ContentChildren`

**Available**:
- ✅ `@Input()` properties
- ✅ `@ContentChild()` / `@ContentChildren()` (just resolved!)
- ❌ `@ViewChild()` / `@ViewChildren()` (not resolved yet)

**Source Code**:

```typescript
// packages/core/src/render3/instructions/change_detection.ts

function detectChangesInView(lView: LView, mode: ChangeDetectionMode) {
  // ... execute hooks

  // Refresh content queries BEFORE AfterContentInit
  refreshContentQueries(tView, lView);

  // Call AfterContentInit
  if (tView.contentHooks !== null) {
    executeContentHooks(lView, tView.contentHooks, InitPhaseState.AfterContentInit);
  }
}

function refreshContentQueries(tView: TView, lView: LView) {
  const contentQueries = tView.contentQueries;

  if (contentQueries !== null) {
    for (let i = 0; i < contentQueries.length; i += 2) {
      const queryIndex = contentQueries[i];
      const queryList = lView[queryIndex] as QueryList<any>;

      // Update the query results
      queryList.setDirty();
    }
  }
}
```

**Usage**:

```typescript
// Parent component
@Component({
  selector: 'app-parent',
  template: `
    <app-tabs>
      <app-tab #tab1 title="Tab 1">Content 1</app-tab>
      <app-tab #tab2 title="Tab 2">Content 2</app-tab>
      <app-tab #tab3 title="Tab 3">Content 3</app-tab>
    </app-tabs>
  `
})
export class ParentComponent {}

// Tabs component
@Component({
  selector: 'app-tabs',
  template: `
    <div class="tabs">
      <div class="tab-headers">
        <button *ngFor="let tab of tabs; let i = index"
                (click)="selectTab(i)"
                [class.active]="i === activeIndex">
          {{ tab.title }}
        </button>
      </div>
      <div class="tab-content">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class TabsComponent implements AfterContentInit {
  @ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;

  activeIndex = 0;

  ngAfterContentInit() {
    // ✅ tabs is now available!
    console.log('Found', this.tabs.length, 'tabs');

    // Show first tab by default
    if (this.tabs.length > 0) {
      this.selectTab(0);
    }

    // Listen for dynamic tab changes
    this.tabs.changes.subscribe(() => {
      console.log('Tabs changed!', this.tabs.length);
    });
  }

  selectTab(index: number) {
    this.tabs.forEach((tab, i) => {
      tab.active = i === index;
    });
    this.activeIndex = index;
  }
}

// Tab component
@Component({
  selector: 'app-tab',
  template: `
    <div class="tab-pane" [hidden]="!active">
      <ng-content></ng-content>
    </div>
  `
})
export class TabComponent {
  @Input() title = '';
  active = false;
}
```

### 5. ngAfterContentChecked

**When**: After `ngAfterContentInit` and every subsequent `ngDoCheck`

**Purpose**: React to changes in projected content

**Warning**: Called on every change detection - keep it fast!

**Usage**:

```typescript
@Component({
  selector: 'app-accordion',
  template: `
    <div class="accordion">
      <ng-content></ng-content>
    </div>
  `
})
export class AccordionComponent implements AfterContentChecked {
  @ContentChildren(AccordionItemComponent) items!: QueryList<AccordionItemComponent>;

  ngAfterContentChecked() {
    // Ensure only one item is expanded at a time
    const expandedItems = this.items.filter(item => item.expanded);

    if (expandedItems.length > 1) {
      // Close all except the first one
      expandedItems.slice(1).forEach(item => {
        item.expanded = false;
      });
    }
  }
}
```

### 6. ngAfterViewInit

**When**: Once, after first `ngAfterContentChecked`, when component's view is initialized

**Purpose**: Access and initialize `@ViewChild` / `@ViewChildren`, perform DOM operations

**Available**:
- ✅ All `@Input()` properties
- ✅ All `@ContentChild()` / `@ContentChildren()`
- ✅ All `@ViewChild()` / `@ViewChildren()` (just resolved!)
- ✅ DOM elements

**Source Code**:

```typescript
// packages/core/src/render3/instructions/change_detection.ts

function detectChangesInView(lView: LView, mode: ChangeDetectionMode) {
  // ... refresh view and content

  // Refresh view queries BEFORE AfterViewInit
  refreshViewQueries(tView, lView);

  // Call AfterViewInit
  if (tView.viewHooks !== null) {
    executeViewHooks(lView, tView.viewHooks, InitPhaseState.AfterViewInit);
  }
}

function refreshViewQueries(tView: TView, lView: LView) {
  const viewQuery = tView.viewQuery;

  if (viewQuery !== null) {
    // Execute view query function
    viewQuery(RenderFlags.Update, lView[CONTEXT]);
  }
}
```

**Usage** (solving Alex's original problem):

```typescript
@Component({
  selector: 'app-sales-chart',
  template: `
    <div class="chart-container">
      <h2>Sales Overview</h2>
      <canvas #chartCanvas></canvas>
    </div>
  `
})
export class SalesChartComponent implements AfterViewInit, OnDestroy {
  @ViewChild('chartCanvas') chartCanvas!: ElementRef<HTMLCanvasElement>;

  private chart?: Chart;

  constructor(private salesService: SalesService) {
    // chartCanvas is undefined here
  }

  ngAfterViewInit() {
    // ✅ chartCanvas is NOW available!
    this.initChart();
  }

  initChart() {
    const ctx = this.chartCanvas.nativeElement.getContext('2d')!;

    this.chart = new Chart(ctx, {
      type: 'line',
      data: this.salesService.getChartData()
    });
  }

  ngOnDestroy() {
    // Clean up chart
    this.chart?.destroy();
  }
}
```

**More Examples**:

```typescript
@Component({
  selector: 'app-autofocus',
  template: `<input #inputElement type="text">`
})
export class AutofocusComponent implements AfterViewInit {
  @ViewChild('inputElement') input!: ElementRef<HTMLInputElement>;

  ngAfterViewInit() {
    // Focus the input automatically
    this.input.nativeElement.focus();
  }
}

@Component({
  selector: 'app-animation',
  template: `
    <div #animTarget class="animation-target">
      Animate me!
    </div>
  `
})
export class AnimationComponent implements AfterViewInit {
  @ViewChild('animTarget') target!: ElementRef;

  ngAfterViewInit() {
    // Start animation
    anime({
      targets: this.target.nativeElement,
      translateX: 250,
      rotate: '1turn',
      duration: 800
    });
  }
}
```

### 7. ngAfterViewChecked

**When**: After `ngAfterViewInit` and every subsequent `ngAfterContentChecked`

**Purpose**: React to changes in component's view

**Warning**: Called on every change detection - keep it VERY fast!

**Common Use Case**: Synchronize with third-party libraries

```typescript
@Component({
  selector: 'app-scroll-tracker',
  template: `
    <div #scrollContainer class="scroll-container">
      <div *ngFor="let item of items">{{ item }}</div>
    </div>
    <p>Scroll position: {{ scrollPosition }}</p>
  `
})
export class ScrollTrackerComponent implements AfterViewChecked {
  @ViewChild('scrollContainer') container!: ElementRef;

  items: string[] = [];
  scrollPosition = 0;

  ngAfterViewChecked() {
    // Update scroll position after view is checked
    // (e.g., after items array changes)
    this.scrollPosition = this.container.nativeElement.scrollTop;
  }
}
```

**ExpressionChangedAfterItHasBeenCheckedError**:

This hook is where the infamous error often occurs:

```typescript
// ❌ BAD - Triggers error in dev mode!
@Component({
  selector: 'app-parent',
  template: `
    <p>Count: {{ count }}</p>
    <app-child></app-child>
  `
})
export class ParentComponent implements AfterViewChecked {
  count = 0;

  ngAfterViewChecked() {
    // This changes data AFTER Angular checked bindings!
    this.count++;  // ❌ ExpressionChangedAfterItHasBeenCheckedError
  }
}

// ✅ GOOD - Use next tick
ngAfterViewChecked() {
  setTimeout(() => {
    this.count++;  // Scheduled for next change detection cycle
  });
}

// ✅ BETTER - Use AfterViewInit for one-time changes
ngAfterViewInit() {
  this.count++;  // Only runs once
}
```

### 8. ngOnDestroy

**When**: Just before component is destroyed and removed from DOM

**Purpose**: Cleanup - unsubscribe, detach event handlers, clear timers

**Source Code**:

```typescript
// packages/core/src/render3/node_manipulation.ts

export function destroyViewTree(rootView: LView): void {
  let lViewToDestroy = rootView[CHILD_HEAD];

  while (lViewToDestroy !== null) {
    // Call ngOnDestroy hooks
    executeOnDestroys(lViewToDestroy[TVIEW], lViewToDestroy);

    // Move to next view
    lViewToDestroy = lViewToDestroy[NEXT];
  }
}

function executeOnDestroys(tView: TView, lView: LView): void {
  const destroyHooks = tView.destroyHooks;

  if (destroyHooks !== null) {
    for (let i = 0; i < destroyHooks.length; i += 2) {
      const hook = destroyHooks[i + 1] as () => void;

      try {
        hook.call(lView[CONTEXT]);
      } catch (err) {
        handleError(lView, err);
      }
    }
  }
}
```

**Critical Usage**:

```typescript
@Component({
  selector: 'app-live-data',
  template: `<div>{{ data }}</div>`
})
export class LiveDataComponent implements OnInit, OnDestroy {
  data: any;

  private subscription?: Subscription;
  private intervalId?: number;
  private resizeListener?: () => void;

  constructor(private dataService: DataService) {}

  ngOnInit() {
    // 1. RxJS subscriptions
    this.subscription = this.dataService.stream$.subscribe(data => {
      this.data = data;
    });

    // 2. Timers
    this.intervalId = window.setInterval(() => {
      console.log('Polling...');
    }, 1000);

    // 3. Event listeners
    this.resizeListener = () => console.log('Resized');
    window.addEventListener('resize', this.resizeListener);
  }

  ngOnDestroy() {
    // ✅ CRITICAL: Clean up everything!

    // 1. Unsubscribe from observables
    this.subscription?.unsubscribe();

    // 2. Clear timers
    if (this.intervalId !== undefined) {
      window.clearInterval(this.intervalId);
    }

    // 3. Remove event listeners
    if (this.resizeListener) {
      window.removeEventListener('resize', this.resizeListener);
    }

    console.log('Component destroyed - all cleaned up!');
  }
}
```

**Memory Leak Prevention**:

```typescript
// ❌ Memory leak - subscription never cleaned up
export class LeakyComponent implements OnInit {
  ngOnInit() {
    interval(1000).subscribe(val => {
      console.log(val);  // Runs forever, even after component destroyed!
    });
  }
}

// ✅ Proper cleanup
export class CleanComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    interval(1000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(val => {
        console.log(val);
      });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Parent-Child Lifecycle Order

Alex created a comprehensive test to understand the exact order:

```typescript
@Component({
  selector: 'app-root',
  template: `<app-parent [value]="rootValue"></app-parent>`,
})
export class AppComponent {
  rootValue = 1;
}

@Component({
  selector: 'app-parent',
  template: `
    <div>
      <p>Parent: {{ value }}</p>
      <ng-content></ng-content>
      <app-child [childValue]="value * 2"></app-child>
    </div>
  `,
})
export class ParentComponent implements OnChanges, OnInit, DoCheck,
  AfterContentInit, AfterContentChecked, AfterViewInit, AfterViewChecked, OnDestroy {

  @Input() value = 0;
  @ContentChild('projected') projected?: ElementRef;
  @ViewChild(ChildComponent) child?: ChildComponent;

  ngOnChanges() { console.log('1. Parent: ngOnChanges'); }
  ngOnInit() { console.log('2. Parent: ngOnInit'); }
  ngDoCheck() { console.log('3. Parent: ngDoCheck'); }
  ngAfterContentInit() { console.log('7. Parent: ngAfterContentInit'); }
  ngAfterContentChecked() { console.log('8. Parent: ngAfterContentChecked'); }
  ngAfterViewInit() { console.log('11. Parent: ngAfterViewInit'); }
  ngAfterViewChecked() { console.log('12. Parent: ngAfterViewChecked'); }
  ngOnDestroy() { console.log('13. Parent: ngOnDestroy'); }
}

@Component({
  selector: 'app-child',
  template: `<p>Child: {{ childValue }}</p>`,
})
export class ChildComponent implements OnChanges, OnInit, DoCheck,
  AfterContentInit, AfterContentChecked, AfterViewInit, AfterViewChecked, OnDestroy {

  @Input() childValue = 0;

  ngOnChanges() { console.log('  4. Child: ngOnChanges'); }
  ngOnInit() { console.log('  5. Child: ngOnInit'); }
  ngDoCheck() { console.log('  6. Child: ngDoCheck'); }
  ngAfterContentInit() { console.log('  9. Child: ngAfterContentInit'); }
  ngAfterContentChecked() { console.log('  10. Child: ngAfterContentChecked'); }
  ngAfterViewInit() { console.log('  11. Child: ngAfterViewInit'); }
  ngAfterViewChecked() { console.log('  12. Child: ngAfterViewChecked'); }
  ngOnDestroy() { console.log('  14. Child: ngOnDestroy'); }
}
```

**Console Output (Initial Render)**:

```
1. Parent: ngOnChanges
2. Parent: ngOnInit
3. Parent: ngDoCheck
  4. Child: ngOnChanges
  5. Child: ngOnInit
  6. Child: ngDoCheck
  7. Child: ngAfterContentInit
  8. Child: ngAfterContentChecked
  9. Child: ngAfterViewInit
  10. Child: ngAfterViewChecked
11. Parent: ngAfterContentInit
12. Parent: ngAfterContentChecked
13. Parent: ngAfterViewInit
14. Parent: ngAfterViewChecked

// On Change Detection:
3. Parent: ngDoCheck
  6. Child: ngDoCheck
  8. Child: ngAfterContentChecked
  10. Child: ngAfterViewChecked
12. Parent: ngAfterContentChecked
14. Parent: ngAfterViewChecked

// On Destroy:
15. Parent: ngOnDestroy
  16. Child: ngOnDestroy
```

💡 **Key Pattern**: Parent initializes first, children complete their lifecycle, then parent completes. Think of it as **depth-first tree traversal**!

## Advanced Patterns

### Pattern 1: DestroyRef (Angular 16+)

Modern alternative to `ngOnDestroy`:

```typescript
import { Component, OnInit, inject, DestroyRef } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-modern',
  template: `<div>{{ data }}</div>`
})
export class ModernComponent implements OnInit {
  private destroyRef = inject(DestroyRef);
  data: any;

  constructor(private dataService: DataService) {}

  ngOnInit() {
    // Automatically unsubscribes when component is destroyed!
    this.dataService.stream$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(data => {
        this.data = data;
      });

    // Or register cleanup callback
    this.destroyRef.onDestroy(() => {
      console.log('Component is being destroyed!');
    });
  }
}
```

### Pattern 2: OnPush with Lifecycle Hooks

```typescript
@Component({
  selector: 'app-optimized',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<div>{{ displayValue }}</div>`
})
export class OptimizedComponent implements OnInit, OnDestroy {
  @Input() value = 0;
  displayValue = '';

  private destroy$ = new Subject<void>();

  constructor(
    private cdr: ChangeDetectorRef,
    private dataService: DataService
  ) {}

  ngOnInit() {
    this.dataService.updates$
      .pipe(takeUntil(this.destroy$))
      .subscribe(update => {
        this.displayValue = update;

        // OnPush requires manual marking
        this.cdr.markForCheck();
      });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Pattern 3: Dynamic Component Loading

```typescript
@Component({
  selector: 'app-dynamic-host',
  template: `
    <div #container></div>
    <button (click)="loadComponent()">Load Component</button>
  `
})
export class DynamicHostComponent implements AfterViewInit, OnDestroy {
  @ViewChild('container', { read: ViewContainerRef }) container!: ViewContainerRef;

  private componentRef?: ComponentRef<any>;

  ngAfterViewInit() {
    // Container is available now
    console.log('Container ready');
  }

  loadComponent() {
    // Clear previous component
    this.container.clear();

    // Create new component
    this.componentRef = this.container.createComponent(DynamicComponent);

    // Set inputs
    this.componentRef.instance.data = 'Hello from parent';

    // Listen to outputs
    this.componentRef.instance.outputEvent.subscribe(value => {
      console.log('Output:', value);
    });
  }

  ngOnDestroy() {
    // Clean up dynamic component
    this.componentRef?.destroy();
  }
}
```

## Debugging Lifecycle Issues

### Technique 1: Lifecycle Logger

```typescript
export function logLifecycle(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;

  descriptor.value = function(...args: any[]) {
    console.log(`[${target.constructor.name}] ${propertyKey}`, args);
    return original.apply(this, args);
  };

  return descriptor;
}

@Component({/* ... */})
export class MyComponent {
  @logLifecycle
  ngOnInit() {
    // Logs: [MyComponent] ngOnInit []
  }

  @logLifecycle
  ngOnChanges(changes: SimpleChanges) {
    // Logs: [MyComponent] ngOnChanges [SimpleChanges]
  }
}
```

### Technique 2: Angular DevTools

Chrome DevTools → Angular Tab → Component Explorer

Shows:
- Current lifecycle phase
- Input/output values
- Change detection status

### Technique 3: Manual Inspection

```typescript
@Component({/* ... */})
export class DebugComponent {
  constructor() {
    console.log('Constructor');
    console.trace(); // Full stack trace
  }

  ngOnInit() {
    console.log('Inputs:', {
      prop1: this.prop1,
      prop2: this.prop2
    });

    console.log('Services:', {
      service: this.service
    });
  }

  ngAfterViewInit() {
    console.log('ViewChildren:', {
      child1: this.child1,
      child2: this.child2
    });
  }
}
```

## Common Pitfalls and Solutions

### Pitfall 1: Accessing ViewChild in ngOnInit

```typescript
// ❌ Wrong
ngOnInit() {
  console.log(this.myViewChild);  // undefined!
}

// ✅ Correct
ngAfterViewInit() {
  console.log(this.myViewChild);  // Available!
}
```

### Pitfall 2: Changing Data in AfterViewChecked

```typescript
// ❌ Wrong - Infinite loop!
ngAfterViewChecked() {
  this.count++;  // Triggers new CD, which calls this hook again!
}

// ✅ Correct
ngAfterViewInit() {
  this.count++;  // Only once
}
```

### Pitfall 3: Heavy Computation in DoCheck

```typescript
// ❌ Wrong - Performance killer!
ngDoCheck() {
  this.items.forEach(item => {
    this.expensiveCalculation(item);  // Called 100+ times/second!
  });
}

// ✅ Correct - Use memoization
private cache = new Map();

ngDoCheck() {
  this.items.forEach(item => {
    if (!this.cache.has(item.id)) {
      this.cache.set(item.id, this.expensiveCalculation(item));
    }
  });
}
```

### Pitfall 4: Forgetting to Unsubscribe

```typescript
// ❌ Wrong - Memory leak!
ngOnInit() {
  this.service.data$.subscribe(data => {
    this.data = data;  // Subscription lives forever!
  });
}

// ✅ Correct
private destroy$ = new Subject<void>();

ngOnInit() {
  this.service.data$
    .pipe(takeUntil(this.destroy$))
    .subscribe(data => {
      this.data = data;
    });
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

## Key Takeaways

After mastering lifecycle hooks, Alex understood:

### 1. **Execution Order Matters**
- Constructor → OnChanges → OnInit → DoCheck → AfterContent* → AfterView* → OnDestroy
- Children complete their lifecycle before parent's AfterContent*/AfterView* hooks

### 2. **Query Resolution Timeline**
- `@ContentChild` available in `ngAfterContentInit`
- `@ViewChild` available in `ngAfterViewInit`
- Both undefined before their respective hooks

### 3. **Frequency Awareness**
- **Once**: OnInit, AfterContentInit, AfterViewInit
- **Every CD**: DoCheck, AfterContentChecked, AfterViewChecked
- **On input change**: OnChanges
- **On destroy**: OnDestroy

### 4. **Performance Considerations**
- Keep DoCheck, AfterContentChecked, AfterViewChecked extremely fast
- Avoid heavy computation in frequently-called hooks
- Use memoization when necessary

### 5. **Cleanup is Critical**
- Always unsubscribe in ngOnDestroy
- Clear timers and intervals
- Remove event listeners
- Destroy dynamic components

### 6. **Modern Patterns**
- Use `DestroyRef` for cleaner destruction handling (Angular 16+)
- Use `takeUntilDestroyed()` for automatic unsubscription
- Leverage OnPush with lifecycle hooks for performance

## What's Next?

Understanding lifecycle hooks revealed when components initialize and update. But a new mystery emerged:

*How does Angular actually render components to the DOM? What happens between template code and pixels on screen?*

This led to investigating Angular's rendering engine...

---

**Next**: [Chapter 4: The Rendering Engine Revealed](04-rendering-engine.md)

## Further Reading

- Source: `packages/core/src/render3/interfaces/lifecycle_hooks.ts`
- Source: `packages/core/src/render3/hooks.ts`
- Source: `packages/core/src/render3/instructions/change_detection.ts`
- Documentation: https://angular.dev/guide/lifecycle-hooks
- DestroyRef: https://angular.dev/api/core/DestroyRef

## Code Example

See `code-examples/03-lifecycle/` for complete examples:
- All lifecycle hooks demonstrated
- Parent-child lifecycle order
- ViewChild/ContentChild usage
- Proper cleanup patterns
- Performance optimization

Run it:
```bash
cd code-examples/03-lifecycle/
npm install
npm start
```

## Notes from Alex's Journal

*"Finally figured out the ViewChild mystery! It's all about timing - queries are resolved at specific points in the lifecycle.*

*The key insight: lifecycle hooks execute in a precise order, and understanding this order prevents so many bugs.*

*Most important lessons:*
*- ngOnInit for data loading*
*- ngAfterViewInit for DOM access*
*- ngOnDestroy for cleanup (NEVER forget this!)*
*- DoCheck/AfterViewChecked are called frequently - keep them fast*

*The parent-child order is elegant: parent starts initialization, children complete fully, then parent finishes. Like a depth-first tree traversal.*

*Next mystery: how does Angular convert my templates into actual DOM elements? Time to explore the rendering engine!"*
