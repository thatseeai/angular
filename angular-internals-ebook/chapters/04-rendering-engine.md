# Chapter 4: The Rendering Engine Revealed

> *"Why is my list so slow? React uses Virtual DOM. What does Angular use?"*

## The Problem

After mastering lifecycle hooks, Alex was asked to build a complex data grid showing thousands of products with real-time price updates. Coming from a React background, Alex knew about Virtual DOM and how it optimizes rendering.

But Angular doesn't use Virtual DOM. So how does it update the DOM efficiently?

Alex built a simple product list:

```typescript
@Component({
  selector: 'app-product-list',
  template: `
    <div class="product-grid">
      <div *ngFor="let product of products" class="product-card">
        <img [src]="product.image" [alt]="product.name">
        <h3>{{ product.name }}</h3>
        <p>{{ product.description }}</p>
        <span class="price">\${{ product.price }}</span>
        <button (click)="addToCart(product)">Add to Cart</button>
      </div>
    </div>
  `,
  standalone: true,
  imports: [CommonModule]
})
export class ProductListComponent implements OnInit {
  products: Product[] = [];

  constructor(private productService: ProductService) {}

  ngOnInit() {
    // Load 5000 products
    this.productService.getProducts().subscribe(products => {
      this.products = products;
    });

    // Update prices every second
    setInterval(() => {
      this.products = this.products.map(p => ({
        ...p,
        price: p.price + (Math.random() - 0.5)
      }));
    }, 1000);
  }

  addToCart(product: Product) {
    console.log('Added to cart:', product.name);
  }
}
```

Initial render: **3.2 seconds** 😱

Price update (every second): **1.8 seconds** 😱

The app was completely unusable. Users complained about lag, and scrolling felt like molasses.

**"How does Angular render so slowly? I thought Ivy was supposed to be fast!"**

This crisis forced Alex to understand Angular's rendering engine at the deepest level.

## The Investigation Begins

Alex needed answers:
- How does Angular render templates to DOM?
- Why is it slow with 5000 items?
- How can it be optimized?
- What's the difference between Ivy and Virtual DOM?

Time to dive into the Angular source code.

### Discovery 1: Angular Uses Instructions, Not Virtual DOM

Alex discovered something surprising in `packages/core/src/render3/`:

```typescript
// packages/core/src/render3/instructions/element.ts

/**
 * Create a DOM element.
 *
 * @param index Index in the LView where the element is stored
 * @param name Name of the DOM element
 * @param attrsIndex Index of attributes in the consts array
 * @param localRefsIndex Index of local refs in the consts array
 */
export function ɵɵelementStart(
  index: number,
  name: string,
  attrsIndex?: number | null,
  localRefsIndex?: number
): void {
  const lView = getLView();
  const tView = getTView();
  const adjustedIndex = HEADER_OFFSET + index;

  ngDevMode && assertEqual(
    getBindingIndex(), tView.bindingStartIndex,
    'elements should be created before any bindings'
  );

  const native = lView[adjustedIndex] = createElementNode(
    lView[RENDERER],
    name,
    getNamespace()
  );

  const tNode = getOrCreateTNode(
    tView,
    adjustedIndex,
    TNodeType.Element,
    name,
    null
  );

  if (attrsIndex !== null && attrsIndex !== undefined) {
    setUpAttributes(native, tView.data[attrsIndex]);
  }

  appendChild(tView, lView, native, tNode);
  setCurrentTNode(tNode, true);
}
```

💡 **Key Insight #1**: Angular doesn't create a Virtual DOM tree and diff it. Instead, templates compile to **direct DOM manipulation instructions**!

No Virtual DOM means:
- ✅ No memory overhead for virtual tree
- ✅ No diffing algorithm overhead
- ✅ Direct DOM updates
- ❌ But requires compiler to generate optimal instructions

### Discovery 2: The LView Data Structure

Alex found the core rendering data structure:

```typescript
// packages/core/src/render3/interfaces/view.ts

/**
 * LView is an array representing a logical view in Angular.
 * It's the heart of change detection and rendering.
 */
export interface LView<T = unknown> extends Array<any> {
  // HEADER (indices 0-26)
  /** The host DOM node */
  [HOST]: RElement | null;

  /** The static data for this view (TView) */
  readonly [TVIEW]: TView;

  /** Flags for this view */
  [FLAGS]: LViewFlags;

  /** Parent LView or LContainer */
  [PARENT]: LView | LContainer | null;

  /** Link to next sibling LView */
  [NEXT]: LView | LContainer | null;

  /** Views to refresh due to transplanted views */
  [TRANSPLANTED_VIEWS_TO_REFRESH]: LView[] | null;

  /** The TNode representing the host */
  [T_HOST]: TNode | null;

  /** Cleanup functions for this view */
  [CLEANUP]: any[] | null;

  /** The component instance (context) */
  [CONTEXT]: T;

  /** Injector for this view */
  readonly [INJECTOR]: Injector;

  /** Environment configuration */
  [ENVIRONMENT]: LViewEnvironment;

  /** Renderer instance */
  [RENDERER]: Renderer;

  /** Child LViews */
  [CHILD_HEAD]: LView | LContainer | null;
  [CHILD_TAIL]: LView | LContainer | null;

  /** Declaration location for debugging */
  [DECLARATION_VIEW]: LView | null;
  [DECLARATION_COMPONENT_VIEW]: LView;

  /** Preorder index for tree traversal */
  [PREORDER_HOOK_FLAGS]: PreOrderHookFlags;

  /** Queries for this view */
  [QUERIES]: LQueries | null;

  /** Embedded view IDs */
  [ID]: number;

  // ... 14 more header slots

  // CONTENT starts at index 27 (HEADER_OFFSET)
  // All DOM nodes, component instances, directives stored here by index
}

/** Size of the LView header */
export const HEADER_OFFSET = 27;
```

💡 **Key Insight #2**: LView is a **flat array** where:
- Indices 0-26: Header (metadata)
- Indices 27+: Actual DOM nodes and data

Why an array? **Performance!** Array access by index is O(1) and heavily optimized by JavaScript engines.

### Discovery 3: The TView Structure

Alex discovered LView has a companion - TView (Template View):

```typescript
// packages/core/src/render3/interfaces/view.ts

/**
 * TView contains static template data shared across all instances of a component.
 * This is the "blueprint" - LView is the "instance".
 */
export interface TView {
  /** Type of this view (Root, Component, Embedded) */
  readonly type: TViewType;

  /** Template ID for debugging */
  readonly id: number;

  /** Blueprint for LView - initial values */
  blueprint: LView;

  /** Template function that generates the view */
  template: ComponentTemplate<any> | null;

  /** Static data (attributes, styles, etc.) */
  data: TData;

  /** First child TNode */
  firstChild: TNode | null;

  /** Directives and components on this template */
  directiveRegistry: DirectiveDefList | null;

  /** Lifecycle hooks to execute */
  viewHooks: HookData | null;
  contentHooks: HookData | null;
  contentCheckHooks: HookData | null;
  viewCheckHooks: HookData | null;
  destroyHooks: DestroyHookData | null;

  /** Content queries */
  contentQueries: number[] | null;

  /** Binding start index */
  bindingStartIndex: number;

  /** Expand instructions (for structural directives) */
  expandoStartIndex: number;

  /** Static view queries */
  viewQuery: ViewQueriesFunction<any> | null;
}
```

💡 **Key Insight #3**:
- **TView** = Shared template blueprint (created once)
- **LView** = Instance data (created per component instance)

This is brilliant for memory efficiency! If you have 1000 product cards, there's:
- 1 TView (shared blueprint)
- 1000 LViews (instance data)

### Discovery 4: The Compilation Process

Alex examined what happens when a template is compiled:

```typescript
// Template
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <h2>{{ title }}</h2>
      <p>{{ description }}</p>
      <button (click)="onClick()">Click</button>
    </div>
  `
})
export class CardComponent {
  title = 'My Card';
  description = 'Card description';

  onClick() {
    console.log('Clicked!');
  }
}
```

Compiles to (simplified):

```typescript
// packages/core/src/render3/jit/directive.ts (conceptual output)

function CardComponent_Template(rf: RenderFlags, ctx: CardComponent) {
  // CREATE PHASE - Runs once
  if (rf & RenderFlags.Create) {
    ɵɵelementStart(0, 'div', 0);      // <div class="card">
      ɵɵelementStart(1, 'h2');         // <h2>
        ɵɵtext(2);                      //   [text node]
      ɵɵelementEnd();                   // </h2>
      ɵɵelementStart(3, 'p');          // <p>
        ɵɵtext(4);                      //   [text node]
      ɵɵelementEnd();                   // </p>
      ɵɵelementStart(5, 'button', 1);  // <button (click)>
        ɵɵlistener('click', function() {
          return ctx.onClick();
        });
        ɵɵtext(6, 'Click');
      ɵɵelementEnd();                   // </button>
    ɵɵelementEnd();                     // </div>
  }

  // UPDATE PHASE - Runs on change detection
  if (rf & RenderFlags.Update) {
    ɵɵadvance(2);                       // Move to h2 text node (index 2)
    ɵɵtextInterpolate(ctx.title);       // Update with ctx.title

    ɵɵadvance(2);                       // Move to p text node (index 4)
    ɵɵtextInterpolate(ctx.description); // Update with ctx.description
  }
}

// Attribute arrays stored in TView.data
const _c0 = ['class', 'card'];
const _c1 = [];
```

💡 **Key Insight #4**: Templates compile to **two-phase rendering**:
1. **Create Phase** (once): Build DOM structure
2. **Update Phase** (on changes): Update only changed bindings

This is **massively more efficient** than Virtual DOM diffing!

## The Deep Dive: How Rendering Actually Works

Armed with understanding, Alex traced through a complete render cycle.

### Step 1: Component Creation

```typescript
// packages/core/src/render3/component_ref.ts

export class ComponentFactory<T> extends AbstractComponentFactory<T> {
  create(
    injector: Injector,
    projectableNodes?: any[][],
    rootSelectorOrNode?: any,
    environmentInjector?: EnvironmentInjector | NgModuleRef<any>
  ): ComponentRef<T> {
    // 1. Create TView (template blueprint) if first time
    const componentDef = this.componentDef;
    const tView = getOrCreateComponentTView(componentDef);

    // 2. Create root LView (instance)
    const rootLView = createLView(
      null,                    // parent
      tView,                   // template view
      null,                    // context (component instance)
      LViewFlags.CheckAlways,  // flags
      null,                    // host
      injector,
      environmentInjector,
      renderer,
      null,                    // hydration info
      null                     // embedded view injector
    );

    // 3. Create component instance
    const component = instantiateRootComponent(tView, rootLView, componentDef);

    // 4. Execute template (CREATE phase)
    renderView(tView, rootLView, null);

    return new ComponentRef(/* ... */);
  }
}
```

### Step 2: First Render (Create Phase)

```typescript
// packages/core/src/render3/instructions/shared.ts

export function renderView<T>(
  tView: TView,
  lView: LView<T>,
  context: T
): void {
  const prevView = enterView(lView);

  try {
    // First render - CREATE phase
    if (lView[FLAGS] & LViewFlags.CreationMode) {
      // Execute template with RenderFlags.Create
      executeTemplate(tView, lView, tView.template, RenderFlags.Create, context);

      // Process view queries
      processViewQuery(tView, lView);

      // Mark creation complete
      lView[FLAGS] &= ~LViewFlags.CreationMode;
    }

    // Always run UPDATE phase
    executeTemplate(tView, lView, tView.template, RenderFlags.Update, context);

    // Refresh embedded views and child components
    refreshEmbeddedViews(lView);

  } finally {
    leaveView(prevView);
  }
}
```

### Step 3: Template Execution

```typescript
// packages/core/src/render3/instructions/template.ts

function executeTemplate<T>(
  tView: TView,
  lView: LView<T>,
  templateFn: ComponentTemplate<T> | null,
  rf: RenderFlags,
  context: T
): void {
  // Call the compiled template function
  if (templateFn !== null) {
    templateFn(rf, context);
  }
}
```

### Step 4: Instruction Execution

Let's trace a single instruction:

```typescript
// packages/core/src/render3/instructions/element.ts

export function ɵɵelementStart(
  index: number,
  name: string,
  attrsIndex?: number | null,
  localRefsIndex?: number
): void {
  const lView = getLView();
  const tView = getTView();
  const adjustedIndex = HEADER_OFFSET + index;

  // CREATE phase: Create the actual DOM element
  const renderer = lView[RENDERER];
  const native = renderer.createElement(name);

  // Store in LView at the specified index
  lView[adjustedIndex] = native;

  // Create or get the TNode (template node)
  const tNode = getOrCreateTNode(
    tView,
    adjustedIndex,
    TNodeType.Element,
    name,
    null
  );

  // Set attributes if provided
  if (attrsIndex != null) {
    const attrs = tView.data[attrsIndex] as TAttributes;
    setUpAttributes(renderer, native, attrs);
  }

  // Append to parent in the DOM
  appendChild(tView, lView, native, tNode);

  // Set this as current TNode for child elements
  setCurrentTNode(tNode, true);
}
```

### Step 5: Change Detection Update

```typescript
// packages/core/src/render3/instructions/text.ts

/**
 * Update a text node with new value
 */
export function ɵɵtextInterpolate(value: any): void {
  const lView = getLView();
  const index = getSelectedIndex();
  const tNode = getTNode(lView[TVIEW], index);

  // Get the text node from LView
  const textNode = lView[index] as Text;

  // Update if value changed
  const stringValue = String(value);
  if (textNode.textContent !== stringValue) {
    textNode.textContent = stringValue;
  }
}
```

💡 **Key Insight #5**: Updates are **surgical**! Angular:
1. Knows exactly which bindings to check (from compiled template)
2. Only updates DOM if value actually changed
3. Uses direct DOM API calls (no Virtual DOM diff)

## Understanding TNode (Template Node)

Alex discovered another critical piece:

```typescript
// packages/core/src/render3/interfaces/node.ts

/**
 * TNode represents a static template node.
 * One TNode per template element, shared across all component instances.
 */
export interface TNode {
  /** Type of this node */
  type: TNodeType;

  /** Index in LView array */
  index: number;

  /** Index of TNode in TView.data */
  insertBeforeIndex: InsertBeforeIndex | null;

  /** Tag name for elements */
  value: string | null;

  /** Static attributes */
  attrs: TAttributes | null;

  /** Local names (template variables like #myDiv) */
  localNames: (string | number)[] | null;

  /** Initial input values */
  initialInputs: InitialInputData | null;

  /** Input properties */
  inputs: PropertyAliases | null;

  /** Output properties */
  outputs: PropertyAliases | null;

  /** Directive start index */
  directiveStart: number;

  /** Directive end index */
  directiveEnd: number;

  /** Property bindings */
  propertyBindings: number[] | null;

  /** Flags */
  flags: TNodeFlags;

  /** Tree structure */
  child: TNode | null;
  next: TNode | null;
  prev: TNode | null;
  parent: TNode | null;
  projection: number | null;

  /** Styling context */
  styles: string | null;
  residualStyles: KeyValueArray<any> | null;
  classes: string | null;
  residualClasses: KeyValueArray<any> | null;
}
```

The relationship:

```
TView (Blueprint)
  └── TNode Tree (Template structure)
        ├── TNode: <div>
        │     ├── TNode: <h2>
        │     └── TNode: <p>
        └── TNode: <span>

LView (Instance)
  └── DOM Nodes (Actual elements)
        ├── [27]: HTMLDivElement
        │     ├── [28]: HTMLHeadingElement
        │     └── [29]: HTMLParagraphElement
        └── [30]: HTMLSpanElement
```

## Performance Analysis: Why Was It Slow?

Armed with knowledge, Alex understood the performance problem:

```typescript
// The problematic code
setInterval(() => {
  // ❌ Creates NEW array - changes reference
  this.products = this.products.map(p => ({
    ...p,
    price: p.price + (Math.random() - 0.5)
  }));
}, 1000);
```

What happens:
1. `products` array reference changes
2. `*ngFor` detects the change
3. Angular **destroys all 5000 product card components**
4. Angular **recreates all 5000 product card components**
5. All 5000 DOM subtrees destroyed and recreated

**5000 × (destroy + create)** every second = Disaster! 💥

### The Fix: TrackBy

```typescript
@Component({
  selector: 'app-product-list',
  template: `
    <div class="product-grid">
      <div *ngFor="let product of products; trackBy: trackById" class="product-card">
        <img [src]="product.image" [alt]="product.name">
        <h3>{{ product.name }}</h3>
        <p>{{ product.description }}</p>
        <span class="price">\${{ product.price }}</span>
        <button (click)="addToCart(product)">Add to Cart</button>
      </div>
    </div>
  `
})
export class ProductListComponent {
  products: Product[] = [];

  // TrackBy function - tells Angular how to identify items
  trackById(index: number, product: Product): number {
    return product.id;  // Use stable ID, not object reference
  }

  ngOnInit() {
    // Update with trackBy
    setInterval(() => {
      this.products = this.products.map(p => ({
        ...p,
        price: p.price + (Math.random() - 0.5)
      }));
    }, 1000);
  }
}
```

With `trackBy`, Angular knows:
1. Product with id=123 still exists (same ID)
2. Don't destroy and recreate the component
3. Just run change detection and update the price binding

**Result**: Update time dropped from **1.8 seconds** to **28ms**! 🚀

### Understanding NgFor Internals

Alex examined how `*ngFor` works:

```typescript
// packages/common/src/directives/ng_for_of.ts (simplified)

@Directive({
  selector: '[ngFor][ngForOf]'
})
export class NgForOf<T> implements DoCheck {
  @Input() ngForOf!: NgIterable<T>;
  @Input() ngForTrackBy?: TrackByFunction<T>;

  private _differ: IterableDiffer<T> | null = null;

  constructor(
    private _viewContainer: ViewContainerRef,
    private _template: TemplateRef<NgForOfContext<T>>,
    private _differs: IterableDiffers
  ) {}

  ngDoCheck(): void {
    if (this._differ) {
      // Check for changes in the collection
      const changes = this._differ.diff(this.ngForOf);

      if (changes) {
        // Apply changes to the view
        this._applyChanges(changes);
      }
    }
  }

  private _applyChanges(changes: IterableChanges<T>): void {
    const viewContainer = this._viewContainer;

    // Process removed items
    changes.forEachRemovedItem((record) => {
      const index = record.previousIndex!;
      viewContainer.remove(index);  // Destroy view
    });

    // Process moved items
    changes.forEachMovedItem((record) => {
      const index = record.previousIndex!;
      const view = viewContainer.get(index);
      viewContainer.move(view!, record.currentIndex!);
    });

    // Process added items
    changes.forEachAddedItem((record) => {
      const view = viewContainer.createEmbeddedView(
        this._template,
        new NgForOfContext(record.item, this.ngForOf, record.currentIndex!, count),
        record.currentIndex!
      );

      // Update context
      view.context.$implicit = record.item;
      view.context.index = record.currentIndex!;
    });

    // Update identity changes
    changes.forEachIdentityChange((record) => {
      const view = viewContainer.get(record.currentIndex!);
      view!.context.$implicit = record.item;
    });
  }
}
```

The **IterableDiffer** uses trackBy to determine:
- Which items were added
- Which items were removed
- Which items moved
- Which items changed identity

Without trackBy, it assumes **every item is new** when the array reference changes!

## Optimization Patterns

### Pattern 1: OnPush with Immutable Data

```typescript
@Component({
  selector: 'app-product-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="card">
      <h3>{{ product.name }}</h3>
      <span>\${{ product.price | number:'1.2-2' }}</span>
    </div>
  `
})
export class ProductCardComponent {
  @Input() product!: Product;
}

// Parent updates immutably
updatePrice(productId: number, newPrice: number) {
  this.products = this.products.map(p =>
    p.id === productId
      ? { ...p, price: newPrice }  // New object = OnPush detects change
      : p
  );
}
```

### Pattern 2: Virtual Scrolling

For huge lists, use `@angular/cdk/scrolling`:

```typescript
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  selector: 'app-product-list',
  template: `
    <cdk-virtual-scroll-viewport itemSize="100" class="viewport">
      <div *cdkVirtualFor="let product of products; trackBy: trackById" class="product">
        {{ product.name }} - \${{ product.price }}
      </div>
    </cdk-virtual-scroll-viewport>
  `,
  imports: [ScrollingModule]
})
export class ProductListComponent {
  products: Product[] = [/* 100,000 items */];

  trackById(index: number, product: Product) {
    return product.id;
  }
}
```

Only renders ~20 items at a time, regardless of total count!

### Pattern 3: Detach Change Detection

For static content:

```typescript
@Component({
  selector: 'app-static-chart',
  template: `<canvas #chart></canvas>`
})
export class StaticChartComponent implements AfterViewInit, OnDestroy {
  @ViewChild('chart') canvas!: ElementRef;

  constructor(private cdr: ChangeDetectorRef) {
    // Detach from change detection - this component never changes
    this.cdr.detach();
  }

  ngAfterViewInit() {
    // Render chart once
    this.renderChart();
  }

  // If you need to update later
  forceUpdate() {
    this.renderChart();
    this.cdr.detectChanges();  // Manual one-time check
  }
}
```

### Pattern 4: Memoization

```typescript
@Component({
  selector: 'app-dashboard',
  template: `
    <div>
      Total: {{ getTotal() }}
      Average: {{ getAverage() }}
    </div>
  `
})
export class DashboardComponent {
  @Input() items: Item[] = [];

  private totalCache?: number;
  private averageCache?: number;
  private lastItems?: Item[];

  getTotal(): number {
    // Only recalculate if items changed
    if (this.items !== this.lastItems) {
      this.totalCache = this.items.reduce((sum, item) => sum + item.value, 0);
      this.lastItems = this.items;
    }
    return this.totalCache!;
  }

  getAverage(): number {
    if (this.items !== this.lastItems || !this.averageCache) {
      this.averageCache = this.getTotal() / this.items.length;
    }
    return this.averageCache;
  }
}
```

Or use `@Memo()` decorator:

```typescript
import { Memo } from '@memo-decorator/decorator';

@Component({/*...*/})
export class DashboardComponent {
  @Input() items: Item[] = [];

  @Memo()
  getTotal(): number {
    return this.items.reduce((sum, item) => sum + item.value, 0);
  }

  @Memo()
  getAverage(): number {
    return this.getTotal() / this.items.length;
  }
}
```

## Comparing with Virtual DOM

Alex created a comparison:

### Virtual DOM (React)

```typescript
// React approach
function ProductList({ products }) {
  return (
    <div>
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

// On every render:
// 1. Create Virtual DOM tree (JavaScript objects)
// 2. Diff against previous Virtual DOM tree
// 3. Calculate minimal set of DOM operations
// 4. Apply operations to real DOM
```

**Pros**:
- Simple mental model
- Works without compiler
- Cross-platform (React Native, etc.)

**Cons**:
- Virtual DOM tree memory overhead
- Diffing algorithm overhead (O(n))
- Can't optimize compile-time

### Ivy (Angular)

```typescript
// Angular approach (compiled output)
function Template(rf: RenderFlags, ctx: Component) {
  if (rf & RenderFlags.Create) {
    // Create DOM once
    ɵɵelementStart(0, 'div');
    ɵɵtemplate(1, ProductCard_Template, ...);
    ɵɵelementEnd();
  }
  if (rf & RenderFlags.Update) {
    // Update only changed bindings
    ɵɵadvance(1);
    ɵɵproperty('products', ctx.products);
  }
}

// On change detection:
// 1. Execute update instructions
// 2. Check binding values
// 3. Update DOM only if value changed
```

**Pros**:
- No Virtual DOM memory
- No diffing algorithm
- Compile-time optimizations
- Surgically precise updates

**Cons**:
- Requires AOT compilation
- Less flexible than runtime approach

## Debugging Rendering Performance

### Technique 1: Chrome DevTools Performance Profiler

```typescript
// Add performance marks
@Component({/*...*/})
export class MyComponent implements AfterViewInit {
  ngAfterViewInit() {
    performance.mark('component-rendered');
    performance.measure('render-time', 'navigationStart', 'component-rendered');

    const measure = performance.getEntriesByName('render-time')[0];
    console.log('Render time:', measure.duration, 'ms');
  }
}
```

Record → Profile → Analyze:
- Long tasks (>50ms)
- Scripting time
- Rendering time
- Painting time

### Technique 2: Angular DevTools Profiler

Chrome DevTools → Angular tab → Profiler

Shows:
- Change detection cycles
- Component render times
- Lifecycle hook execution

### Technique 3: Why Did You Render?

```typescript
@Component({/*...*/})
export class DebugComponent implements DoCheck {
  @Input() data: any;

  private previousData: any;

  ngDoCheck() {
    if (this.data !== this.previousData) {
      console.log('Re-rendered because data changed:');
      console.log('Previous:', this.previousData);
      console.log('Current:', this.data);

      this.previousData = this.data;
    }
  }
}
```

### Technique 4: LView Inspection

```typescript
// In dev console
const component = ng.getComponent(document.querySelector('app-my-component'));
const lView = component.__ngContext__;

console.log('LView:', lView);
console.log('Flags:', lView[FLAGS]);
console.log('Context:', lView[CONTEXT]);
console.log('TView:', lView[TVIEW]);
```

## Advanced: Custom Renderers

Angular supports custom renderers for different platforms:

```typescript
// packages/core/src/render3/interfaces/renderer.ts

export interface Renderer {
  destroy(): void;
  createElement(name: string, namespace?: string | null): any;
  createComment(value: string): any;
  createText(value: string): any;
  appendChild(parent: any, newChild: any): void;
  insertBefore(parent: any, newChild: any, refChild: any, isMove?: boolean): void;
  removeChild(parent: any, oldChild: any, isHostElement?: boolean): void;
  selectRootElement(selectorOrNode: any, preserveContent?: boolean): any;
  parentNode(node: any): any;
  nextSibling(node: any): any;
  setAttribute(el: any, name: string, value: string, namespace?: string | null): void;
  removeAttribute(el: any, name: string, namespace?: string | null): void;
  addClass(el: any, name: string): void;
  removeClass(el: any, name: string): void;
  setStyle(el: any, style: string, value: any, flags?: RendererStyleFlags2): void;
  removeStyle(el: any, style: string, flags?: RendererStyleFlags2): void;
  setProperty(el: any, name: string, value: any): void;
  setValue(node: any, value: string): void;
  listen(target: any, eventName: string, callback: (event: any) => boolean | void): () => void;
}
```

Examples:
- **DomRenderer**: Browser DOM (default)
- **ServerRenderer**: Angular Universal (SSR)
- **WebWorkerRenderer**: Render in Web Worker
- **NativeScriptRenderer**: Mobile native UI

## Key Takeaways

After this deep dive, Alex mastered rendering:

### 1. **Instructions, Not Virtual DOM**
Angular compiles templates to direct DOM instructions. No Virtual DOM overhead.

### 2. **Two-Phase Rendering**
- Create phase: Build DOM structure once
- Update phase: Update only changed bindings

### 3. **LView = Instance, TView = Blueprint**
- TView shared across all instances (memory efficient)
- LView is a flat array (access by index is O(1))

### 4. **TrackBy is Critical**
For `*ngFor`, always use `trackBy` to avoid destroying/recreating components.

### 5. **OnPush for Performance**
Combined with immutable data patterns, OnPush drastically reduces change detection work.

### 6. **Virtual Scrolling for Large Lists**
CDK Virtual Scrolling renders only visible items.

### 7. **Detach When Static**
For content that never changes, detach from change detection.

### 8. **Memoize Expensive Computations**
Cache results when inputs haven't changed.

## Practical Applications

Alex now optimizes rendering:

1. **Large Lists**: TrackBy + OnPush + Virtual Scrolling
2. **Complex UIs**: Detach static sections, memoize calculations
3. **Real-time Data**: Immutable updates with OnPush
4. **Profiling**: Chrome DevTools + Angular DevTools for bottlenecks

## What's Next?

Understanding the rendering engine revealed how templates become DOM. But a new question emerged:

*How does the compiler transform templates into these optimized instructions? What happens during the build process?*

This led to investigating the Angular compiler...

---

**Next**: [Chapter 5: The Compiler's Magic](05-compiler.md)

## Further Reading

- Source: `packages/core/src/render3/`
- Source: `packages/core/src/render3/instructions/`
- Source: `packages/core/src/render3/interfaces/view.ts`
- Source: `packages/common/src/directives/ng_for_of.ts`
- Ivy Architecture: https://github.com/angular/angular/blob/main/packages/core/src/render3/README.md
- Documentation: https://angular.dev/guide/change-detection

## Code Example

See `code-examples/04-rendering/` for complete examples:
- LView/TView structure visualization
- Performance comparison with/without trackBy
- Virtual scrolling implementation
- Custom renderer example
- Performance profiling utilities

Run it:
```bash
cd code-examples/04-rendering/
npm install
npm start
```

## Notes from Alex's Journal

*"Mind blown again! Angular doesn't use Virtual DOM at all. It compiles templates to direct DOM instructions.*

*The two-phase rendering (create + update) is genius. Create once, update only what changed.*

*LView as a flat array is brilliant for performance. And TView being shared across instances saves so much memory.*

*The trackBy revelation: my 5000-item list went from 1.8 seconds to 28ms just by adding trackBy. That's a 64x improvement!*

*Key lesson: Angular's rendering is already optimized by the compiler. My job is to give it the right hints (trackBy, OnPush) and avoid common pitfalls (creating new arrays unnecessarily).*

*Next mystery: how does the compiler transform templates into these instructions? Time to explore the compilation process!"*
