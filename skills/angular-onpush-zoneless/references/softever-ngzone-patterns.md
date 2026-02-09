# NgZone Patterns in Softever Codebase

Real-world NgZone usage patterns from the TOP-CMS codebase, cataloged during OnPush/zoneless migration. All 7 usages were analyzed against official Angular v21 documentation and found to be **intentionally kept and zoneless-compatible**.

## Official Angular v21 Guidance

From the Angular zoneless migration guide:

> `NgZone.run` and `NgZone.runOutsideAngular` do not need to be removed in order for code to be compatible with Zoneless applications. In fact, removing these calls can lead to performance regressions for libraries that are used in applications that still rely on ZoneJS.

### Compatible APIs (KEEP)
| API | Zoneless Behavior | Action |
|-----|-------------------|--------|
| `NgZone.run()` | Executes synchronously, no-op wrapper in zoneless | Keep — harmless, aids ZoneJS compatibility |
| `NgZone.runOutsideAngular()` | Executes synchronously, no-op wrapper in zoneless | Keep — performance optimization still valid |

### Incompatible APIs (MUST REPLACE)
| API | Zoneless Behavior | Replacement |
|-----|-------------------|-------------|
| `NgZone.onStable` | Never fires (no zone to stabilize) | `afterNextRender()` or `afterRender()` |
| `NgZone.onMicrotaskEmpty` | Never fires | `afterNextRender()` |
| `NgZone.isStable` | Always `true` | Remove or use `PendingTasks` |

## All 7 NgZone Usages — Catalog

### Pattern A: `runOutsideAngular` for High-Frequency DOM Events (4 usages)

These prevent CD from firing on every mouse/drag/scroll event. Even in zoneless (where CD isn't triggered by events automatically), this pattern is harmless and documents intent.

#### 1. `src/app/app.component.ts` — Idle Detection
```typescript
this.zone.runOutsideAngular(() => {
  document.addEventListener('mousemove', () => this.resetIdleTimer());
  document.addEventListener('keydown', () => this.resetIdleTimer());
});
```
**Why keep:** Mouse/keyboard listeners fire hundreds of times per second. Running inside Angular would trigger CD on every event. The `zone.run()` callback only fires once when the idle timeout actually expires.

#### 2. `projects/@lib/src/lib/directives/drag-to-scroll.directive.ts` — Mouse Drag
```typescript
this.zone.runOutsideAngular(() => {
  el.addEventListener('mousedown', ...);
  el.addEventListener('mousemove', ...);
  el.addEventListener('mouseup', ...);
});
```
**Why keep:** Drag events fire at 60fps+. Running inside Angular would cause continuous CD during drag operations.

#### 3. `projects/soft-ngx/src/lib/soft-tooltip/soft-tooltip.directive.ts` — Tippy.js Init
```typescript
this.zone.runOutsideAngular(() => {
  this.tippyInstance = tippy(this.el.nativeElement, { ... });
});
```
**Why keep:** Tippy.js creates internal event listeners (hover, focus, click). Running outside Angular prevents those internal listeners from triggering CD.

#### 4. `src/app/features/training-dashboard/training-dashboard.component.ts` — TUI Calendar
```typescript
this.zone.runOutsideAngular(() => {
  this.calendar = new Calendar('#calendar', { ... });
});
```
**Why keep:** TUI Calendar creates dozens of internal DOM event listeners. Initializing outside Angular prevents all those listeners from triggering CD.

#### 5. `projects/@lib/src/lib/components/color-picker/color-picker.component.ts` — Pickr Init
```typescript
this.zone.runOutsideAngular(() => {
  this.pickr = Pickr.create({ ... });
});
```
**Why keep:** Pickr creates internal mouse/touch event listeners for color selection. Running outside Angular prevents those from triggering CD.

### Pattern B: `zone.run()` for Third-Party Callbacks (5 usages)

These re-enter Angular's zone from callbacks fired by third-party libraries (Highcharts, Pickr, tippy.js, XHR). In zoneless mode, `zone.run()` becomes a no-op but is harmless to keep.

#### 1. `src/app/app.component.ts` — Idle Timeout Callback
```typescript
// After idle timeout expires (inside runOutsideAngular context)
this.zone.run(() => {
  this.authService.logout();
});
```

#### 2. `projects/penalty-inform/src/app/app.component.ts` — XHR Version Check
```typescript
// Inside XMLHttpRequest.onload callback
this.zone.run(() => {
  this.hasNewVersion.set(true);
});
```
**Note:** Since `hasNewVersion` is already a signal, `zone.run()` is technically redundant here — `signal.set()` triggers CD in both zoned and zoneless. But keeping it is harmless.

#### 3. `src/app/shared/chart/shared-stack-bar-chart/shared-stack-bar-chart.component.ts` — Highcharts
```typescript
// In Highcharts legendItemClick callback
this.zone.run(() => { ... });
```

#### 4. `src/app/features/penalty/v-incident-stack-bar-chart/v-incident-stack-bar-chart.component.ts` — Highcharts
Same pattern as above — Highcharts callback re-entering Angular zone.

#### 5. `projects/@lib/src/lib/components/color-picker/color-picker.component.ts` — Pickr Save
```typescript
this.pickr.on('save', (color) => {
  this.zone.run(() => {
    this.colorChange.emit(color.toHEXA().toString());
  });
});
```

## Migration Decision Matrix

When encountering `NgZone` during zoneless migration:

| Pattern | Action | Reason |
|---------|--------|--------|
| `runOutsideAngular` + DOM events | **KEEP** | Performance optimization, documents intent |
| `runOutsideAngular` + 3rd-party init | **KEEP** | Prevents internal listener CD triggers |
| `zone.run()` in 3rd-party callback | **KEEP** | Harmless no-op in zoneless, aids ZoneJS compat |
| `zone.run()` after signal.set() | **KEEP** (optional) | Redundant but harmless — signal already triggers CD |
| `onStable` / `onMicrotaskEmpty` | **REPLACE** | These never fire in zoneless |
| `isStable` | **REPLACE** | Always returns `true` in zoneless |

## Verification Command

```bash
# Find all NgZone usages in the codebase
grep -rn "NgZone\|\.zone\." --include="*.ts" src/ projects/ | grep -v node_modules | grep -v ".spec.ts"

# Check for incompatible patterns specifically
grep -rn "onStable\|onMicrotaskEmpty\|isStable" --include="*.ts" src/ projects/
```

## Current Status

As of the OnPush migration completion:
- **0 incompatible patterns** found (`onStable`, `onMicrotaskEmpty`, `isStable`)
- **7 compatible patterns** found (all `run()` / `runOutsideAngular()`)
- **All 7 intentionally kept** for performance and third-party compatibility
