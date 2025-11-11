# Chapter 6: Zone.js and the Async World

> *"Why is my animation so janky? It's triggering change detection 60 times per second!"*

## The Problem

After optimizing the bundle size, Alex built a smooth data visualization dashboard with animated charts. The feature looked great in development, but in production, users complained about stuttering animations and poor performance.

Alex opened Chrome DevTools and profiled the app:

```
Performance Profile:
Frame 1: 45ms (should be <16ms for 60fps)
├── Change Detection: 38ms  ← ⚠️ Way too long!
├── Rendering: 5ms
└── Other: 2ms

Frame 2: 42ms
├── Change Detection: 35ms  ← ⚠️ Still too long!
├── Rendering: 5ms
└── Other: 2ms
```

Change detection was running **60 times per second** (once per animation frame)! The app could barely maintain 20fps.

Looking at the code:

```typescript
@Component({
  selector: 'app-chart',
  template: `
    <canvas #canvas></canvas>
    <div>FPS: {{ fps }}</div>
  `
})
export class ChartComponent implements OnInit, OnDestroy {
  @ViewChild('canvas') canvas!: ElementRef<HTMLCanvasElement>;
  fps = 0;
  private animationId?: number;

  ngOnInit() {
    const ctx = this.canvas.nativeElement.getContext('2d')!;
    let frames = 0;
    let lastTime = performance.now();

    const animate = () => {
      // Draw chart
      ctx.clearRect(0, 0, 800, 600);
      this.drawChart(ctx);

      // Calculate FPS
      frames++;
      const now = performance.now();
      if (now - lastTime >= 1000) {
        this.fps = frames;
        frames = 0;
        lastTime = now;
      }

      // Schedule next frame
      this.animationId = requestAnimationFrame(animate);
    };

    animate();
  }

  drawChart(ctx: CanvasRenderingContext2D) {
    // Heavy drawing operations...
  }

  ngOnDestroy() {
    if (this.animationId) {
      cancelAnimationFrame(this.animationId);
    }
  }
}
```

**"Why is change detection running so often? I'm only updating a canvas!"**

Alex discovered that `requestAnimationFrame` was triggering change detection on every frame. But why? And how can it be prevented?

This led to investigating Zone.js - the magic that makes Angular's automatic change detection work.

## The Investigation Begins

Alex needed to understand:
- What is Zone.js?
- How does it track async operations?
- Why does it trigger change detection?
- How can you opt-out when needed?

### Discovery 1: Zone.js is Everywhere

Alex opened the browser console and typed:

```javascript
console.log(window.setTimeout);
```

Expected output:
```javascript
function setTimeout() { [native code] }
```

Actual output:
```javascript
function () {
  // Patched by Zone.js!
  const zone = Zone.current;
  return originalSetTimeout(() => {
    zone.run(callback);
  }, delay);
}
```

**The native `setTimeout` had been replaced!**

Alex checked other APIs:

```javascript
console.log(window.fetch);  // Patched!
console.log(Promise);  // Patched!
console.log(XMLHttpRequest.prototype.send);  // Patched!
console.log(EventTarget.prototype.addEventListener);  // Patched!
```

💡 **Key Insight #1**: Zone.js **monkey-patches** all async browser APIs at startup!

### Discovery 2: The Zone.js Source Code

Alex found Zone.js in the Angular repository:

```typescript
// packages/zone.js/lib/zone.ts

/**
 * A Zone is an execution context that persists across async tasks.
 */
export class Zone {
  static current: Zone = _currentZoneFrame.zone;
  static currentTask: Task | null = null;

  constructor(
    public parent: Zone | null,
    public name: string
  ) {}

  /**
   * Run a function in this zone
   */
  run<T>(callback: Function, applyThis?: any, applyArgs?: any[]): T {
    _currentZoneFrame = { parent: _currentZoneFrame, zone: this };
    try {
      return callback.apply(applyThis, applyArgs);
    } finally {
      _currentZoneFrame = _currentZoneFrame.parent!;
    }
  }

  /**
   * Create a child zone
   */
  fork(zoneSpec: ZoneSpec): Zone {
    return new Zone(this, zoneSpec.name);
  }

  /**
   * Run a callback when the zone becomes stable (no pending tasks)
   */
  runTask(task: Task, applyThis?: any, applyArgs?: any[]): any {
    // Notify zone that task is starting
    const previousTask = Zone.currentTask;
    Zone.currentTask = task;

    try {
      // Execute the task
      return this.run(task.callback, applyThis, applyArgs);
    } finally {
      // Task completed
      Zone.currentTask = previousTask;

      // Check if zone is now stable
      if (this._checkStable()) {
        this.onHasTask(/* ... */);
      }
    }
  }
}
```

💡 **Key Insight #2**: A Zone is an **execution context** that persists across async operations. It tracks when async tasks start and complete.

### Discovery 3: How setTimeout Gets Patched

Alex traced through the patching code:

```typescript
// packages/zone.js/lib/browser/browser.ts

/**
 * Patch setTimeout to run callbacks in zones
 */
Zone.__load_patch('timers', (global: any) => {
  // Save original functions
  const originalSetTimeout = global.setTimeout;
  const originalSetInterval = global.setInterval;
  const originalClearTimeout = global.clearTimeout;

  // Patch setTimeout
  global.setTimeout = function(callback: Function, delay: number, ...args: any[]) {
    // Create a Zone task
    const zone = Zone.current;
    const task = zone.scheduleTask(new MacroTask(
      'setTimeout',
      () => callback(...args),
      { delay },
      () => originalClearTimeout(timerId)
    ));

    // Call original setTimeout with wrapped callback
    const timerId = originalSetTimeout(() => {
      // When timer fires, run callback in the zone
      zone.runTask(task);
    }, delay);

    return task;
  };

  // Similar patching for setInterval...
});
```

The patching process:

```
Original Flow:
setTimeout(callback, 1000)
  → Timer starts
  → 1 second passes
  → callback() executes

Zone.js Patched Flow:
setTimeout(callback, 1000)
  → Zone.current.scheduleTask(task)  ← Zone knows task started
  → Timer starts
  → 1 second passes
  → Zone.current.runTask(task)
  → callback() executes
  → Zone checks if stable  ← Zone knows task completed
  → onMicrotaskEmpty.emit()  ← Triggers change detection!
```

💡 **Key Insight #3**: Zone.js wraps every async operation to know when tasks start and complete!

### Discovery 4: NgZone - Angular's Integration

Alex found how Angular uses Zone.js:

```typescript
// packages/core/src/zone/ng_zone.ts

/**
 * NgZone wraps Zone.js and integrates it with Angular
 */
export class NgZone {
  private readonly _outer: Zone;
  private readonly _inner: Zone;

  readonly onMicrotaskEmpty = new EventEmitter<void>();
  readonly onStable = new EventEmitter<void>();
  readonly onUnstable = new EventEmitter<void>();

  constructor({ shouldCoalesceEventChangeDetection = false }) {
    // Outer zone (outside Angular)
    this._outer = Zone.current;

    // Create Angular zone
    this._inner = this._outer.fork({
      name: 'angular',
      properties: { isAngularZone: true },

      // Called when async task starts
      onScheduleTask: (delegate, current, target, task) => {
        // Mark zone as unstable
        this.onUnstable.emit();
        return delegate.scheduleTask(target, task);
      },

      // Called when no more microtasks
      onInvokeTask: (delegate, current, target, task, applyThis, applyArgs) => {
        try {
          return delegate.invokeTask(target, task, applyThis, applyArgs);
        } finally {
          // Check if zone is stable (no pending tasks)
          if (!this.hasPendingMicrotasks && !this.hasPendingMacrotasks) {
            // Trigger change detection!
            this.onMicrotaskEmpty.emit();
            this.onStable.emit();
          }
        }
      }
    });
  }

  /**
   * Run code inside Angular zone (triggers CD)
   */
  run<T>(fn: () => T): T {
    return this._inner.run(fn);
  }

  /**
   * Run code outside Angular zone (no CD)
   */
  runOutsideAngular<T>(fn: () => T): T {
    return this._outer.run(fn);
  }
}
```

Angular's setup:

```typescript
// packages/core/src/application/application_ref.ts

export class ApplicationRef {
  constructor(private _zone: NgZone) {
    // Subscribe to zone's onMicrotaskEmpty event
    this._zone.onMicrotaskEmpty.subscribe(() => {
      // Trigger change detection on all views!
      this.tick();
    });
  }

  tick(): void {
    // Run change detection on all root views
    for (const view of this._views) {
      detectChangesInView(view);
    }
  }
}
```

💡 **Key Insight #4**: Angular listens to NgZone's `onMicrotaskEmpty` event and triggers change detection automatically!

## The Complete Flow

Alex now understood the entire process:

```
1. User clicks button
   ↓
2. Zone.js intercepts event
   ↓
3. Zone.current.scheduleTask(task)
   ↓
4. NgZone.onUnstable.emit()
   ↓
5. Event handler executes
   this.count++
   ↓
6. Task completes
   ↓
7. Zone checks: any pending tasks?
   No? → Zone is stable
   ↓
8. NgZone.onMicrotaskEmpty.emit()
   ↓
9. ApplicationRef.tick()
   ↓
10. Change detection runs
   ↓
11. UI updates!
```

## The Performance Problem Solved

Now Alex understood why the animation was slow:

```typescript
const animate = () => {
  this.drawChart(ctx);
  this.animationId = requestAnimationFrame(animate);  // ← Patched by Zone.js!
};
```

Each `requestAnimationFrame`:
1. Zone.js schedules task
2. Callback executes
3. Zone becomes stable
4. **Change detection runs** (checks entire component tree!)
5. Next frame...

**60 frames/second × full change detection = disaster!**

### Solution 1: Run Outside Angular

```typescript
@Component({
  selector: 'app-chart',
  template: `
    <canvas #canvas></canvas>
    <div>FPS: {{ fps }}</div>
  `
})
export class ChartComponent implements OnInit {
  @ViewChild('canvas') canvas!: ElementRef<HTMLCanvasElement>;
  fps = 0;

  constructor(
    private ngZone: NgZone,
    private cdr: ChangeDetectorRef
  ) {}

  ngOnInit() {
    const ctx = this.canvas.nativeElement.getContext('2d')!;
    let frames = 0;
    let lastTime = performance.now();

    // ✅ Run animation loop OUTSIDE Angular zone
    this.ngZone.runOutsideAngular(() => {
      const animate = () => {
        // Draw chart (no CD triggered)
        ctx.clearRect(0, 0, 800, 600);
        this.drawChart(ctx);

        // Calculate FPS
        frames++;
        const now = performance.now();
        if (now - lastTime >= 1000) {
          // Update FPS and manually trigger CD (only once per second)
          this.ngZone.run(() => {
            this.fps = frames;
            // CD runs here, but only once per second!
          });
          frames = 0;
          lastTime = now;
        }

        // Schedule next frame (no CD!)
        this.animationId = requestAnimationFrame(animate);
      };

      animate();
    });
  }
}
```

**Results**:
- Before: 20 FPS (change detection 60 times/second)
- After: 60 FPS (change detection 1 time/second)
- **3x performance improvement!** 🚀

## Advanced Zone.js Patterns

### Pattern 1: Zone-Aware Libraries

Create libraries that respect zones:

```typescript
export class DataService {
  constructor(private ngZone: NgZone) {}

  // Heavy computation - run outside zone
  processData(data: any[]): Observable<any> {
    return new Observable(observer => {
      this.ngZone.runOutsideAngular(() => {
        // Heavy work here
        const result = this.heavyComputation(data);

        // Re-enter zone to emit
        this.ngZone.run(() => {
          observer.next(result);
          observer.complete();
        });
      });
    });
  }

  private heavyComputation(data: any[]): any {
    // CPU-intensive work
    return data.map(/* ... */);
  }
}
```

### Pattern 2: WebSocket Optimization

```typescript
@Injectable({ providedIn: 'root' })
export class WebSocketService {
  private socket?: WebSocket;
  private messages$ = new Subject<any>();

  constructor(private ngZone: NgZone) {}

  connect(url: string): Observable<any> {
    // Create WebSocket outside zone (high-frequency messages)
    this.ngZone.runOutsideAngular(() => {
      this.socket = new WebSocket(url);

      this.socket.onmessage = (event) => {
        // Process message outside zone
        const data = JSON.parse(event.data);

        // Only trigger CD when emitting to subscribers
        this.ngZone.run(() => {
          this.messages$.next(data);
        });
      };
    });

    return this.messages$.asObservable();
  }
}
```

### Pattern 3: Polling Without Change Detection

```typescript
@Component({
  selector: 'app-live-ticker',
  template: `
    <div>Stock Price: {{ price }}</div>
    <div>Last Update: {{ lastUpdate }}</div>
  `
})
export class LiveTickerComponent implements OnInit, OnDestroy {
  price = 0;
  lastUpdate = '';
  private intervalId?: number;

  constructor(private ngZone: NgZone) {}

  ngOnInit() {
    // Poll every 100ms outside zone
    this.ngZone.runOutsideAngular(() => {
      this.intervalId = window.setInterval(() => {
        // Fetch data (no CD triggered)
        this.fetchPrice().then(newPrice => {
          // Only trigger CD if price changed
          if (newPrice !== this.price) {
            this.ngZone.run(() => {
              this.price = newPrice;
              this.lastUpdate = new Date().toLocaleTimeString();
            });
          }
        });
      }, 100);
    });
  }

  ngOnDestroy() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
  }

  private async fetchPrice(): Promise<number> {
    // API call
    return Math.random() * 100;
  }
}
```

## Debugging Zone.js Issues

### Technique 1: Check if Code is in Angular Zone

```typescript
import { Component } from '@angular/core';

@Component({/* ... */})
export class DebugComponent {
  checkZone() {
    const zone = (Zone as any).current;
    console.log('Current zone:', zone.name);
    console.log('Is Angular zone?', zone.get('isAngularZone'));

    // Check zone hierarchy
    let current = zone;
    while (current) {
      console.log('Zone:', current.name);
      current = current.parent;
    }
  }
}
```

### Technique 2: Count Change Detection Cycles

```typescript
import { ApplicationRef } from '@angular/core';

@Component({/* ... */})
export class PerformanceMonitorComponent implements OnInit {
  cdCount = 0;

  constructor(private appRef: ApplicationRef) {}

  ngOnInit() {
    const originalTick = this.appRef.tick.bind(this.appRef);

    this.appRef.tick = () => {
      this.cdCount++;
      console.log('Change detection #', this.cdCount);
      console.trace(); // Show stack trace
      return originalTick();
    };
  }
}
```

### Technique 3: Identify Frequent Async Operations

```typescript
// Add to main.ts for debugging
(window as any).Zone['__zone_symbol__BLACK_LISTED_EVENTS'] = [];

// Track all zone tasks
const originalScheduleTask = Zone.prototype.scheduleTask;
Zone.prototype.scheduleTask = function(task: Task) {
  console.log('Task scheduled:', task.type, task.source);
  return originalScheduleTask.call(this, task);
};
```

### Technique 4: Angular DevTools Profiler

Use Angular DevTools to visualize change detection:

```
Chrome DevTools → Angular Tab → Profiler

Shows:
- How many CD cycles
- Which components were checked
- Time spent in each component
- What triggered CD
```

## Zone.js Alternatives

### Option 1: Zone-less Angular (Experimental)

Angular 14+ supports running without Zone.js:

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';

bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection()  // No Zone.js!
  ]
});
```

Without Zone.js, you must trigger CD manually:

```typescript
@Component({/* ... */})
export class ManualCDComponent {
  count = 0;

  constructor(private cdr: ChangeDetectorRef) {}

  increment() {
    this.count++;
    this.cdr.markForCheck();  // Manual CD trigger
  }
}
```

Or use signals (automatic CD):

```typescript
@Component({/* ... */})
export class SignalComponent {
  count = signal(0);  // Signals trigger CD automatically!

  increment() {
    this.count.update(n => n + 1);  // CD triggered
  }
}
```

### Option 2: Hybrid Approach

Use Zone.js for most of the app, but opt-out specific components:

```typescript
// Heavy animation component - no Zone.js
@Component({
  selector: 'app-animation',
  template: `<canvas #canvas></canvas>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AnimationComponent {
  constructor(private ngZone: NgZone) {
    // Detach from CD
    this.cdr.detach();

    // All code runs outside zone
    this.ngZone.runOutsideAngular(() => {
      // Animation loop
    });
  }
}
```

## Common Zone.js Pitfalls

### Pitfall 1: Third-Party Libraries

Some libraries create their own async operations:

```typescript
// ❌ Problem: Library runs in zone, triggers CD
import { Chart } from 'chart.js';

ngOnInit() {
  new Chart(this.canvas.nativeElement, {
    data: this.data,
    options: {
      animation: {
        onProgress: () => {
          // This runs in zone → CD triggered!
        }
      }
    }
  });
}

// ✅ Solution: Initialize outside zone
ngOnInit() {
  this.ngZone.runOutsideAngular(() => {
    new Chart(this.canvas.nativeElement, {
      data: this.data,
      options: {
        animation: {
          onProgress: () => {
            // Runs outside zone → no CD
          }
        }
      }
    });
  });
}
```

### Pitfall 2: Event Listeners on Window/Document

```typescript
// ❌ Triggers CD on every scroll!
ngOnInit() {
  window.addEventListener('scroll', () => {
    this.scrollPosition = window.scrollY;
  });
}

// ✅ Run outside zone
ngOnInit() {
  this.ngZone.runOutsideAngular(() => {
    window.addEventListener('scroll', () => {
      this.scrollPosition = window.scrollY;

      // Manually trigger CD (throttled)
      if (Date.now() - this.lastUpdate > 100) {
        this.ngZone.run(() => {
          this.cdr.detectChanges();
        });
        this.lastUpdate = Date.now();
      }
    });
  });
}
```

### Pitfall 3: Observables from Libraries

```typescript
// ❌ RxJS interval triggers CD every second
interval(1000).subscribe(() => {
  this.time = new Date();
});

// ✅ Run outside zone, manually trigger CD
this.ngZone.runOutsideAngular(() => {
  interval(1000).subscribe(() => {
    this.ngZone.run(() => {
      this.time = new Date();
    });
  });
});

// ✅ Better: Use async pipe (handles CD automatically)
time$ = interval(1000).pipe(
  map(() => new Date())
);
```

## Measuring Zone.js Impact

### Benchmark: With vs Without Zone Optimization

```typescript
// Test case: High-frequency updates
@Component({
  selector: 'app-benchmark',
  template: `
    <div>Updates: {{ updateCount }}</div>
    <div>FPS: {{ fps }}</div>
  `
})
export class BenchmarkComponent {
  updateCount = 0;
  fps = 0;

  // ❌ Without optimization (in zone)
  benchmarkInZone() {
    let frames = 0;
    const start = performance.now();

    const loop = () => {
      this.updateCount++;
      frames++;

      if (performance.now() - start < 5000) {
        requestAnimationFrame(loop);  // CD triggered!
      } else {
        this.fps = frames / 5;
        console.log('In zone FPS:', this.fps);
      }
    };

    loop();
  }

  // ✅ With optimization (outside zone)
  benchmarkOutsideZone() {
    let frames = 0;
    const start = performance.now();

    this.ngZone.runOutsideAngular(() => {
      const loop = () => {
        this.updateCount++;
        frames++;

        if (performance.now() - start < 5000) {
          requestAnimationFrame(loop);  // No CD!
        } else {
          this.ngZone.run(() => {
            this.fps = frames / 5;
            console.log('Outside zone FPS:', this.fps);
          });
        }
      };

      loop();
    });
  }
}

// Results:
// In zone: ~20 FPS (CD overhead kills performance)
// Outside zone: ~60 FPS (smooth!)
```

## Zone.js Configuration

### Disabling Specific Patches

```typescript
// polyfills.ts
(window as any).__Zone_disable_requestAnimationFrame = true;  // Disable RAF patching
(window as any).__Zone_disable_on_property = true;  // Disable onProperty patching
(window as any).__Zone_disable_customElements = true;  // Disable custom elements
(window as any).__Zone_disable_geolocation = true;  // Disable geolocation

// Then import zone.js
import 'zone.js';
```

### Black-listing Events

```typescript
// Don't trigger CD for these events
(window as any).__zone_symbol__BLACK_LISTED_EVENTS = [
  'scroll',
  'mousemove',
  'touchmove'
];
```

## Key Takeaways

After mastering Zone.js, Alex understood:

### 1. **Zone.js is Monkey-Patching Magic**
Zone.js replaces all async browser APIs to track execution context.

### 2. **NgZone Integrates with Angular**
Angular listens to NgZone's `onMicrotaskEmpty` event to trigger change detection automatically.

### 3. **Run Outside Zone for Performance**
Use `NgZone.runOutsideAngular()` for:
- High-frequency operations (animations, polling)
- Third-party libraries
- Operations that don't need CD

### 4. **Manual CD When Needed**
When outside zone, manually trigger CD with:
- `NgZone.run()`
- `ChangeDetectorRef.markForCheck()`
- `ChangeDetectorRef.detectChanges()`

### 5. **Signals are the Future**
Angular 16+ signals provide automatic CD without Zone.js overhead.

### 6. **Profile Before Optimizing**
Use Chrome DevTools and Angular DevTools to identify CD bottlenecks.

### 7. **Be Careful with Third-Party Libraries**
Libraries that create async operations can trigger excessive CD if not handled properly.

## Practical Applications

Alex now uses Zone.js wisely:

1. **Animations**: Always run outside zone
2. **WebSockets**: Handle outside zone, trigger CD only when data changes
3. **Polling**: Run outside zone, batch updates
4. **Event Listeners**: Black-list high-frequency events
5. **Third-Party Libraries**: Initialize outside zone

## What's Next?

Understanding Zone.js revealed how Angular achieves automatic change detection. But Angular 16 introduced a new reactivity primitive that doesn't need Zone.js:

*What are Signals? How do they provide fine-grained reactivity? Can they replace Zone.js entirely?*

This led to investigating Angular's Signals...

---

**Next**: [Chapter 7: Signals - The Future of Reactivity](07-signals.md)

## Further Reading

- Source: `packages/zone.js/lib/zone.ts`
- Source: `packages/core/src/zone/ng_zone.ts`
- Zone.js Spec: https://github.com/angular/angular/tree/main/packages/zone.js
- NgZone API: https://angular.dev/api/core/NgZone
- Zoneless Angular: https://angular.dev/guide/experimental/zoneless

## Code Example

See `code-examples/06-zone/` for complete examples:
- Animation optimization demo
- WebSocket integration
- Polling strategies
- Zone.js profiling tools
- Benchmark comparisons

Run it:
```bash
cd code-examples/06-zone/
npm install
npm start
```

## Notes from Alex's Journal

*"Finally understand the Zone.js magic! It's all monkey-patching - Zone.js literally replaces every async API in the browser.*

*The key insight: Zone.js tracks when async operations complete, and Angular listens to that to trigger change detection automatically.*

*My animation went from 20 FPS to 60 FPS just by running it outside the Angular zone. One line of code: `runOutsideAngular()`. Massive impact.*

*Important lessons:*
*- Not everything needs change detection*
*- High-frequency operations should run outside zone*
*- Manually trigger CD only when data actually changes*
*- Signals might make Zone.js obsolete in the future*

*The pattern is clear: runOutsideAngular() for heavy work, then ngZone.run() to update UI. Simple but powerful.*

*Next up: Signals. If they provide fine-grained reactivity without Zone.js, that could be a game-changer for performance!"*
