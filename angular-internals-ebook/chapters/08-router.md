# Chapter 8: Router Deep Dive

> *"Why does navigating to /admin/users/123 trigger 3 HTTP requests?"*

## The Problem

After mastering Signals, Alex was tasked with optimizing the admin panel. Users complained that navigating between pages was slow and triggered duplicate API requests.

Alex observed the behavior:

```typescript
// Route configuration
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    children: [
      {
        path: 'users/:id',
        component: UserDetailComponent
      }
    ]
  }
];

// UserDetailComponent
@Component({
  selector: 'app-user-detail',
  template: `<div>{{ user?.name }}</div>`
})
export class UserDetailComponent implements OnInit {
  user?: User;

  constructor(
    private route: ActivatedRoute,
    private userService: UserService
  ) {}

  ngOnInit() {
    // Get user ID from route
    const userId = this.route.snapshot.params['id'];

    // Load user
    this.userService.getUser(userId).subscribe(user => {
      this.user = user;
    });
  }
}
```

When navigating from `/admin/users/1` to `/admin/users/2`:

```
Network tab:
GET /api/users/1  200 OK  (why is this called again?)
GET /api/users/2  200 OK  (expected)
GET /api/users/2  200 OK  (duplicate!)
```

**Three HTTP requests for a single navigation!** 😱

**"Why is the router calling the component twice? And why is it re-fetching the old user?"**

This crisis led Alex to investigate Angular's Router internals.

## The Investigation Begins

Alex needed to understand:
- How does the Router match URLs?
- When are components created vs reused?
- Why multiple API calls?
- How do guards and resolvers work?
- How does lazy loading work?

### Discovery 1: The Navigation Lifecycle

Alex found the Router's navigation flow in the source:

```typescript
// packages/router/src/router.ts

export class Router {
  /**
   * Navigate to a new URL
   */
  navigate(commands: any[], extras?: NavigationExtras): Promise<boolean> {
    const url = this.createUrlTree(commands, extras);
    return this.navigateByUrl(url, extras);
  }

  /**
   * Navigate to a UrlTree
   */
  navigateByUrl(url: UrlTree | string, extras?: NavigationBehaviorOptions): Promise<boolean> {
    // Create navigation object
    const id = ++this.navigationId;
    const navigation = {
      id,
      url: this.parseUrl(url),
      extras: extras || {}
    };

    // Execute navigation
    return this.executeNavigation(navigation);
  }

  private executeNavigation(navigation: Navigation): Promise<boolean> {
    return this.runNavigationSteps(navigation);
  }
}
```

💡 **Key Insight #1**: Every navigation goes through a **multi-step pipeline**!

### Discovery 2: The Navigation Steps

```typescript
// packages/router/src/router.ts (simplified)

private runNavigationSteps(navigation: Navigation): Promise<boolean> {
  return new Promise((resolve) => {
    // STEP 1: Parse URL into UrlTree
    const urlTree = this.parseUrl(navigation.url);

    // STEP 2: Apply redirects
    const redirectedUrl = this.applyRedirects(urlTree);

    // STEP 3: Recognize routes (match URL to route config)
    const recognized = this.recognizer.recognize(redirectedUrl);

    // STEP 4: Run guards (canActivate, canDeactivate, canLoad)
    const guardsResult = this.runGuards(recognized);

    if (!guardsResult) {
      resolve(false);  // Navigation blocked
      return;
    }

    // STEP 5: Run resolvers (load data)
    const resolvedData = this.runResolvers(recognized);

    // STEP 6: Create/update components
    this.activateRoutes(recognized, resolvedData);

    // STEP 7: Update browser URL
    this.updateUrl(urlTree);

    resolve(true);  // Navigation complete
  });
}
```

The complete navigation flow:

```
1. URL Change
   ↓
2. Parse URL → UrlTree
   ↓
3. Apply Redirects
   ↓
4. Recognize Routes (match config)
   ↓
5. Run canDeactivate guards
   ↓
6. Run canActivate guards
   ↓
7. Run canActivateChild guards
   ↓
8. Run canLoad guards
   ↓
9. Run Resolvers
   ↓
10. Activate/Create Components
   ↓
11. Update Browser URL
   ↓
12. Navigation Complete
```

💡 **Key Insight #2**: Navigation is a **12-step pipeline** - any step can block navigation!

### Discovery 3: Route Recognition

Alex found how URLs are matched to routes:

```typescript
// packages/router/src/recognize.ts

export class Recognizer {
  /**
   * Match URL segments to route configuration
   */
  recognize(url: UrlTree, rootComponentType: Type<any>): RecognizedState {
    const rootSegmentGroup = url.root;

    // Start matching from root
    return this.matchSegmentGroup(
      rootSegmentGroup,
      this.config,  // Routes configuration
      rootComponentType
    );
  }

  private matchSegmentGroup(
    segmentGroup: UrlSegmentGroup,
    config: Routes,
    componentType: Type<any>
  ): Match[] {
    const matches: Match[] = [];

    // Try to match each route
    for (const route of config) {
      const match = this.matchSegmentAgainstRoute(
        segmentGroup,
        route,
        segmentGroup.segments
      );

      if (match) {
        matches.push(match);
      }
    }

    // Return best match (first match wins)
    return matches.length > 0 ? matches[0] : null;
  }

  private matchSegmentAgainstRoute(
    rawSegmentGroup: UrlSegmentGroup,
    route: Route,
    segments: UrlSegment[]
  ): Match | null {
    // Static path match
    if (route.path === '') {
      return { route, segments: [] };
    }

    // Dynamic parameter match
    if (route.path?.includes(':')) {
      return this.matchWithParams(route, segments);
    }

    // Exact match
    if (segments[0]?.path === route.path) {
      return {
        route,
        segments: segments.slice(1)  // Remaining segments
      };
    }

    return null;  // No match
  }
}
```

Example matching:

```typescript
// URL: /admin/users/123
// Segments: ['admin', 'users', '123']

// Route config:
{
  path: 'admin',  // Matches 'admin', remaining: ['users', '123']
  children: [
    {
      path: 'users/:id',  // Matches 'users/:id', params: { id: '123' }
      component: UserDetailComponent
    }
  ]
}
```

💡 **Key Insight #3**: Routes are matched **recursively** from top to bottom, **first match wins**!

### Discovery 4: Component Reuse

Alex discovered why components were being recreated:

```typescript
// packages/router/src/operators/activate_routes.ts

function activateRoutes(
  futureState: RouterStateSnapshot,
  currState: RouterStateSnapshot
): Observable<void> {
  // Compare future and current states
  const futureRoot = futureState.root;
  const currRoot = currState ? currState.root : null;

  return activateChildRoutes(futureRoot, currRoot, /* ... */);
}

function activateChildRoutes(
  futureNode: TreeNode<ActivatedRoute>,
  currNode: TreeNode<ActivatedRoute> | null,
  /* ... */
): Observable<void> {
  const futureChildren = futureNode.children;
  const currChildren = currNode ? currNode.children : [];

  // Deactivate old routes
  for (const child of currChildren) {
    if (!futureChildren.includes(child)) {
      deactivateRoute(child);  // ← Component destroyed!
    }
  }

  // Activate new routes
  for (const child of futureChildren) {
    if (!currChildren.includes(child)) {
      activateRoute(child);  // ← Component created!
    }
  }
}

function shouldReuseRoute(
  future: ActivatedRouteSnapshot,
  curr: ActivatedRouteSnapshot
): boolean {
  // Default strategy: reuse if same route config
  return future.routeConfig === curr.routeConfig;
}
```

**The problem Alex found**:

```typescript
// Route config
{
  path: 'users/:id',
  component: UserDetailComponent  // Same config = component REUSED!
}

// Navigation: /users/1 → /users/2
// future.routeConfig === curr.routeConfig → true
// Component is REUSED, not recreated!
// ngOnInit NOT called again!
```

But Alex's component only loaded data in `ngOnInit`:

```typescript
ngOnInit() {
  const userId = this.route.snapshot.params['id'];  // ← Only runs ONCE!
  this.loadUser(userId);
}
```

💡 **Key Insight #4**: Components are **reused** by default when navigating within the same route config! You must handle parameter changes!

## The Solution: Reactive Parameter Subscription

```typescript
@Component({
  selector: 'app-user-detail',
  template: `<div>{{ user()?.name }}</div>`
})
export class UserDetailComponent implements OnInit {
  user = signal<User | undefined>(undefined);

  constructor(
    private route: ActivatedRoute,
    private userService: UserService
  ) {}

  ngOnInit() {
    // ✅ Subscribe to parameter changes
    this.route.params.subscribe(params => {
      const userId = params['id'];
      this.loadUser(userId);  // Called every time params change!
    });

    // Or with Signals (Angular 16+)
    const userId = toSignal(this.route.params.pipe(
      map(params => params['id'])
    ));

    effect(() => {
      const id = userId();
      if (id) {
        this.loadUser(id);
      }
    });
  }

  private loadUser(id: string) {
    this.userService.getUser(id).subscribe(user => {
      this.user.set(user);
    });
  }
}
```

**Result**: Only **1 API call** per navigation! 🎉

## Guards Deep Dive

### CanActivate Guard

```typescript
// packages/router/src/operators/check_guards.ts

function checkGuards(
  moduleInjector: Injector,
  forwardEvent?: (evt: Event) => void
): Observable<boolean | UrlTree> {
  return (source: Observable<NavigationTransition>) => {
    return source.pipe(
      switchMap(t => {
        // Run canDeactivate guards first
        const canDeactivateChecks = runCanDeactivateChecks(t);

        // Then canActivate guards
        const canActivateChecks = runCanActivateChecks(t);

        return forkJoin([canDeactivateChecks, canActivateChecks]).pipe(
          map(([canDeactivate, canActivate]) => {
            return canDeactivate && canActivate;
          })
        );
      })
    );
  };
}

function runCanActivateChecks(
  transition: NavigationTransition
): Observable<boolean | UrlTree> {
  const checks: Observable<boolean | UrlTree>[] = [];

  // Collect all canActivate guards
  for (const route of transition.targetSnapshot.root.children) {
    if (route.routeConfig?.canActivate) {
      for (const guard of route.routeConfig.canActivate) {
        checks.push(executeGuard(guard, route));
      }
    }
  }

  // Run all guards in parallel
  return forkJoin(checks).pipe(
    map(results => results.every(r => r === true))
  );
}

function executeGuard(
  guard: CanActivateFn,
  route: ActivatedRouteSnapshot
): Observable<boolean | UrlTree> {
  const guardResult = guard(route, state);

  // Convert to Observable
  if (isObservable(guardResult)) {
    return guardResult;
  } else if (isPromise(guardResult)) {
    return from(guardResult);
  } else {
    return of(guardResult);
  }
}
```

Example guard:

```typescript
// Functional guard (Angular 15+)
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;  // Allow navigation
  }

  // Redirect to login
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};

// Usage
{
  path: 'admin',
  canActivate: [authGuard],
  loadChildren: () => import('./admin/admin.routes')
}
```

### CanDeactivate Guard

Prevent navigation away from unsaved forms:

```typescript
export const unsavedChangesGuard: CanDeactivateFn<FormComponent> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm('You have unsaved changes. Do you really want to leave?');
  }
  return true;
};

// Component
@Component({/* ... */})
export class FormComponent {
  form = new FormGroup({/* ... */});

  hasUnsavedChanges(): boolean {
    return this.form.dirty;
  }
}

// Route
{
  path: 'edit',
  component: FormComponent,
  canDeactivate: [unsavedChangesGuard]
}
```

### Resolver

Load data before activating route:

```typescript
// packages/router/src/operators/resolve_data.ts

function resolveData(
  paramsInheritanceStrategy: 'emptyOnly' | 'always'
): Observable<NavigationTransition> {
  return (source: Observable<NavigationTransition>) => {
    return source.pipe(
      switchMap(t => {
        const dataResolvers = getDataResolvers(t.targetSnapshot);

        if (dataResolvers.length === 0) {
          return of(t);
        }

        // Run all resolvers in parallel
        return forkJoin(dataResolvers).pipe(
          map(resolvedData => {
            // Attach resolved data to route
            return { ...t, resolvedData };
          })
        );
      })
    );
  };
}
```

Example resolver:

```typescript
// Functional resolver
export const userResolver: ResolveFn<User> = (route, state) => {
  const userService = inject(UserService);
  const userId = route.params['id'];

  return userService.getUser(userId);  // Returns Observable<User>
};

// Route
{
  path: 'users/:id',
  component: UserDetailComponent,
  resolve: {
    user: userResolver  // Data available as route.data.user
  }
}

// Component
export class UserDetailComponent implements OnInit {
  user!: User;

  constructor(private route: ActivatedRoute) {}

  ngOnInit() {
    // Data already resolved!
    this.user = this.route.snapshot.data['user'];

    // Or subscribe to changes
    this.route.data.subscribe(data => {
      this.user = data['user'];
    });
  }
}
```

## Lazy Loading Internals

### How Lazy Loading Works

```typescript
// packages/router/src/operators/apply_redirects.ts

function loadChildren(
  route: Route,
  injector: EnvironmentInjector
): Observable<LoadedRouterConfig> {
  if (route.loadChildren) {
    // Call the loadChildren function
    const loadedChildren$ = wrapIntoObservable(
      route.loadChildren()
    );

    return loadedChildren$.pipe(
      map((module: any) => {
        // Extract routes from loaded module
        const routes = module.default || module;

        return {
          routes: routes as Routes,
          injector: injector
        };
      })
    );
  }

  return of({ routes: route.children || [], injector });
}
```

Example lazy route:

```typescript
// Old way (NgModule)
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
}

// New way (Standalone, Angular 14+)
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES)
}

// admin.routes.ts
export const ADMIN_ROUTES: Routes = [
  {
    path: '',
    component: AdminComponent,
    children: [
      { path: 'users', component: UsersComponent },
      { path: 'settings', component: SettingsComponent }
    ]
  }
];
```

### Preloading Strategy

```typescript
// packages/router/src/router_preloader.ts

export class RouterPreloader implements OnDestroy {
  private subscription?: Subscription;

  constructor(
    private router: Router,
    private injector: EnvironmentInjector,
    private preloadingStrategy: PreloadingStrategy
  ) {}

  preload(): Observable<any> {
    return this.processRoutes(this.router.config);
  }

  private processRoutes(routes: Routes): Observable<void> {
    const observables: Observable<void>[] = [];

    for (const route of routes) {
      if (route.loadChildren) {
        // Ask strategy if should preload
        const shouldPreload$ = this.preloadingStrategy.preload(
          route,
          () => this.loadRoute(route)
        );

        observables.push(shouldPreload$);
      }

      if (route.children) {
        observables.push(this.processRoutes(route.children));
      }
    }

    return forkJoin(observables).pipe(map(() => void 0));
  }
}
```

Preloading strategies:

```typescript
// 1. PreloadAllModules - Load everything after initial render
import { PreloadAllModules } from '@angular/router';

@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: PreloadAllModules
    })
  ]
})
export class AppModule {}

// 2. NoPreloading (default) - Only load on demand
import { NoPreloading } from '@angular/router';

// 3. Custom Strategy - Selective preloading
export class CustomPreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Preload if route has data.preload = true
    if (route.data && route.data['preload']) {
      console.log('Preloading:', route.path);
      return load();
    }

    return of(null);
  }
}

// Route with preload flag
{
  path: 'dashboard',
  data: { preload: true },
  loadChildren: () => import('./dashboard/dashboard.routes')
}
```

## Route Reuse Strategy

Control when components are reused:

```typescript
// packages/router/src/route_reuse_strategy.ts

export abstract class RouteReuseStrategy {
  abstract shouldReuseRoute(future: ActivatedRouteSnapshot, curr: ActivatedRouteSnapshot): boolean;
  abstract retrieve(route: ActivatedRouteSnapshot): DetachedRouteHandle | null;
  abstract shouldAttach(route: ActivatedRouteSnapshot): boolean;
  abstract shouldDetach(route: ActivatedRouteSnapshot): boolean;
  abstract store(route: ActivatedRouteSnapshot, handle: DetachedRouteHandle | null): void;
}

// Default implementation
export class DefaultRouteReuseStrategy implements RouteReuseStrategy {
  shouldReuseRoute(future: ActivatedRouteSnapshot, curr: ActivatedRouteSnapshot): boolean {
    // Reuse if same route config
    return future.routeConfig === curr.routeConfig;
  }

  retrieve(route: ActivatedRouteSnapshot): DetachedRouteHandle | null {
    return null;  // Never retrieve
  }

  shouldAttach(route: ActivatedRouteSnapshot): boolean {
    return false;  // Never attach
  }

  shouldDetach(route: ActivatedRouteSnapshot): boolean {
    return false;  // Never detach
  }

  store(route: ActivatedRouteSnapshot, detachedTree: DetachedRouteHandle): void {
    // Never store
  }
}
```

Custom reuse strategy example:

```typescript
export class CustomReuseStrategy implements RouteReuseStrategy {
  private storedRoutes = new Map<string, DetachedRouteHandle>();

  shouldReuseRoute(future: ActivatedRouteSnapshot, curr: ActivatedRouteSnapshot): boolean {
    return future.routeConfig === curr.routeConfig;
  }

  shouldDetach(route: ActivatedRouteSnapshot): boolean {
    // Store route if it has reuseRoute flag
    return route.data['reuseRoute'] === true;
  }

  store(route: ActivatedRouteSnapshot, handle: DetachedRouteHandle | null): void {
    if (handle) {
      const path = this.getPath(route);
      this.storedRoutes.set(path, handle);
    }
  }

  shouldAttach(route: ActivatedRouteSnapshot): boolean {
    const path = this.getPath(route);
    return this.storedRoutes.has(path);
  }

  retrieve(route: ActivatedRouteSnapshot): DetachedRouteHandle | null {
    const path = this.getPath(route);
    return this.storedRoutes.get(path) || null;
  }

  private getPath(route: ActivatedRouteSnapshot): string {
    return route.pathFromRoot
      .map(r => r.url.map(segment => segment.toString()).join('/'))
      .join('/');
  }
}

// Provide the strategy
{
  provide: RouteReuseStrategy,
  useClass: CustomReuseStrategy
}

// Route with reuse flag
{
  path: 'dashboard',
  component: DashboardComponent,
  data: { reuseRoute: true }  // This component will be cached!
}
```

## Advanced Router Patterns

### Pattern 1: Auxiliary Routes (Named Outlets)

```typescript
// Route configuration
{
  path: 'dashboard',
  component: DashboardComponent,
  children: [
    {
      path: 'stats',
      component: StatsComponent,
      outlet: 'primary'  // Default outlet
    },
    {
      path: 'messages',
      component: MessagesComponent,
      outlet: 'side'  // Named outlet
    }
  ]
}

// Template
<div class="dashboard">
  <router-outlet></router-outlet>  <!-- primary -->
  <router-outlet name="side"></router-outlet>  <!-- side -->
</div>

// Navigate
router.navigate(['/dashboard', {
  outlets: {
    primary: ['stats'],
    side: ['messages']
  }
}]);

// URL: /dashboard/(stats//side:messages)
```

### Pattern 2: Matrix Parameters

```typescript
// URL: /products;category=electronics;sort=price
router.navigate(['/products', { category: 'electronics', sort: 'price' }]);

// Access in component
this.route.params.subscribe(params => {
  const category = params['category'];  // 'electronics'
  const sort = params['sort'];  // 'price'
});
```

### Pattern 3: Route Animation

```typescript
// Define animations
const routeAnimations = trigger('routeAnimations', [
  transition('* <=> *', [
    style({ position: 'relative' }),
    query(':enter, :leave', [
      style({
        position: 'absolute',
        top: 0,
        left: 0,
        width: '100%'
      })
    ], { optional: true }),
    query(':enter', [style({ opacity: 0 })], { optional: true }),
    query(':leave', animateChild(), { optional: true }),
    group([
      query(':leave', [animate('300ms ease-out', style({ opacity: 0 }))], { optional: true }),
      query(':enter', [animate('300ms ease-in', style({ opacity: 1 }))], { optional: true })
    ]),
    query(':enter', animateChild(), { optional: true }),
  ])
]);

// Component
@Component({
  template: `
    <div [@routeAnimations]="prepareRoute(outlet)">
      <router-outlet #outlet="outlet"></router-outlet>
    </div>
  `,
  animations: [routeAnimations]
})
export class AppComponent {
  prepareRoute(outlet: RouterOutlet) {
    return outlet?.activatedRouteData?.['animation'];
  }
}

// Route with animation data
{
  path: 'home',
  component: HomeComponent,
  data: { animation: 'HomePage' }
}
```

### Pattern 4: Scroll Position Restoration

```typescript
// Enable scroll restoration
RouterModule.forRoot(routes, {
  scrollPositionRestoration: 'enabled',  // Restore scroll position
  anchorScrolling: 'enabled'  // Enable fragment scrolling
})

// Manual scroll control
export class AppComponent implements OnInit {
  constructor(
    private router: Router,
    private viewportScroller: ViewportScroller
  ) {}

  ngOnInit() {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe(() => {
      // Scroll to top on navigation
      this.viewportScroller.scrollToPosition([0, 0]);
    });
  }
}
```

## Debugging Router Issues

### Technique 1: Enable Router Tracing

```typescript
RouterModule.forRoot(routes, {
  enableTracing: true  // Logs all router events to console
})
```

### Technique 2: Listen to Router Events

```typescript
export class AppComponent implements OnInit {
  constructor(private router: Router) {}

  ngOnInit() {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        console.log('Navigation started:', event.url);
      }

      if (event instanceof NavigationEnd) {
        console.log('Navigation ended:', event.url);
      }

      if (event instanceof NavigationError) {
        console.error('Navigation error:', event.error);
      }

      if (event instanceof NavigationCancel) {
        console.log('Navigation cancelled:', event.reason);
      }

      if (event instanceof RouteConfigLoadStart) {
        console.log('Loading lazy module...');
      }

      if (event instanceof RouteConfigLoadEnd) {
        console.log('Lazy module loaded!');
      }
    });
  }
}
```

### Technique 3: Inspect Router State

```typescript
export class DebugComponent {
  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}

  logRouterState() {
    console.log('Current URL:', this.router.url);
    console.log('Router state:', this.router.routerState);
    console.log('Route params:', this.route.snapshot.params);
    console.log('Route data:', this.route.snapshot.data);
    console.log('Query params:', this.route.snapshot.queryParams);
    console.log('Fragment:', this.route.snapshot.fragment);
  }
}
```

## Key Takeaways

After mastering the Router, Alex understood:

### 1. **Navigation is a Multi-Step Pipeline**
Parse → Match → Guard → Resolve → Activate → Update URL

### 2. **Components are Reused by Default**
Must subscribe to `route.params` to handle parameter changes!

### 3. **Guards Control Navigation**
Use guards for authentication, authorization, and unsaved changes.

### 4. **Resolvers Preload Data**
Load data before activating routes for better UX.

### 5. **Lazy Loading Reduces Bundle Size**
Load modules on-demand, preload strategically.

### 6. **Route Reuse Strategy Controls Caching**
Customize when components are created vs reused.

### 7. **Router Events Enable Tracking**
Listen to events for loading indicators, analytics, etc.

### 8. **Named Outlets Enable Complex Layouts**
Display multiple independent routes simultaneously.

## Practical Applications

Alex now uses the Router wisely:

1. **Parameter Subscriptions**: Always subscribe to `route.params` for dynamic content
2. **Auth Guards**: Protect admin routes with `canActivate`
3. **Unsaved Changes**: Warn users with `canDeactivate`
4. **Data Resolvers**: Preload user data before rendering
5. **Lazy Loading**: Split admin panel into separate bundle
6. **Preloading**: Preload frequently-used modules
7. **Router Events**: Show loading indicator during navigation

## The Journey Complete

Alex had journeyed through Angular's internals:

1. **Dependency Injection** - How services are created and injected
2. **Change Detection** - How Angular knows when to update the UI
3. **Component Lifecycle** - When components are created and destroyed
4. **Rendering Engine** - How templates become DOM
5. **Compiler** - How templates are compiled to instructions
6. **Zone.js** - How async operations trigger change detection
7. **Signals** - The future of reactive programming
8. **Router** - How navigation works

**Alex now understood Angular at the deepest level.**

---

**Next**: [Chapter 9: Building TaskMaster - Putting It All Together](09-building-taskmaster.md)

## Further Reading

- Source: `packages/router/src/`
- Router Guide: https://angular.dev/guide/routing
- Router API: https://angular.dev/api/router
- Navigation Lifecycle: https://angular.dev/guide/router#router-events

## Code Example

See `code-examples/08-router/` for complete examples:
- Guard implementations
- Resolver examples
- Lazy loading demo
- Custom reuse strategy
- Named outlets
- Router animations

Run it:
```bash
cd code-examples/08-router/
npm install
npm start
```

## Notes from Alex's Journal

*"Finally understand why my components were being reused! The Router reuses components by default when the route config is the same.*

*Key lesson: ALWAYS subscribe to route.params, not route.snapshot.params. The snapshot only gives you the initial value.*

*Navigation pipeline is elegant:*
*1. Parse URL*
*2. Match routes*
*3. Run guards (can block)*
*4. Run resolvers (preload data)*
*5. Activate components*
*6. Update URL*

*Guards are powerful:*
*- canActivate: protect routes*
*- canDeactivate: prevent losing data*
*- canLoad: lazy load authorization*

*Resolvers are brilliant for UX - data is ready before the component renders.*

*The journey through Angular internals is complete. I now understand:*
*- How dependency injection creates services*
*- How change detection propagates updates*
*- How components go through their lifecycle*
*- How rendering converts templates to DOM*
*- How the compiler optimizes everything*
*- How Zone.js makes it all automatic*
*- How Signals will replace much of this*
*- How the Router orchestrates it all*

*Next: build something real with all this knowledge!"*
