# Zone Boundary Patterns

## What is a "Zone Boundary"?

A zone boundary is where code runs outside Angular's change detection awareness. With `Default` CD, zone.js patches all async APIs, so everything triggers CD. With `OnPush` or zoneless, code running outside the zone won't trigger CD.

## Common Zone Boundary Sources

### 1. Subscription.add() Callbacks

```typescript
// PROBLEM: Subscription.add() runs the teardown outside Angular zone
const sub = this.apiService.getData().subscribe(data => {
  this.data = data;
});
sub.add(() => {
  this.loading = false;  // This runs outside zone!
});

// FIX: Use signal
loading = signal(true);
sub.add(() => {
  this.loading.set(false);  // Signal notifies CD
});
```

### 2. Third-Party Library Events

```typescript
// PROBLEM: Highcharts click events run outside Angular
chart.ref$.subscribe(chart => {
  chart.series[0].update({
    point: {
      events: {
        click: (e) => {
          this.selectedPoint = e.point;  // Outside zone!
          this.router.navigate(['/detail']);  // May not trigger
        }
      }
    }
  });
});

// FIX: Wrap in NgZone.run()
constructor(private zone: NgZone) {}

click: (e) => {
  this.zone.run(() => {
    this.selectedPoint = e.point;
    this.router.navigate(['/detail']);
  });
}
```

### 3. Calendar Widget Events

```typescript
// PROBLEM: tui-calendar events run outside Angular
this.calendar.on('clickSchedule', (event) => {
  this.selectedEvent = event;  // Outside zone!
});

// FIX: NgZone.run()
this.calendar.on('clickSchedule', (event) => {
  this.zone.run(() => {
    this.selectedEvent.set(event);
  });
});
```

### 4. requestAnimationFrame / requestIdleCallback

```typescript
// PROBLEM: Runs outside Angular zone
requestAnimationFrame(() => {
  this.position = calculatePosition();  // Outside zone!
});

// FIX: Signal or NgZone.run()
requestAnimationFrame(() => {
  this.position.set(calculatePosition());
});
```

### 5. Native Event Listeners

```typescript
// PROBLEM: addEventListener runs outside Angular zone
document.addEventListener('click', (e) => {
  this.showDropdown = false;  // Outside zone!
});

// FIX: Use Angular's HostListener or signal
@HostListener('document:click', ['$event'])
onDocumentClick(e: MouseEvent) {
  this.showDropdown.set(false);  // Runs inside Angular
}
```

### 6. Promise.then() Chains

```typescript
// PROBLEM: Deep promise chains may run outside zone
someLibrary.initialize()
  .then(config => config.loadPlugins())
  .then(plugins => {
    this.plugins = plugins;  // May be outside zone
  });

// FIX: Use async/await (runs in zone) or signal
const plugins = await someLibrary.initialize().then(c => c.loadPlugins());
this.plugins.set(plugins);
```

## NgZone.run() vs Signals

| Approach | When to Use | Pros | Cons |
|----------|-------------|------|------|
| `NgZone.run()` | Third-party callbacks | Minimal code change | Requires NgZone injection |
| `signal.set()` | Own state management | Clean, future-proof | Requires signal migration |
| `markForCheck()` | Quick fix | Minimal change | Not signal-based, escape hatch |
| `async` pipe | Observable in template | Auto cleanup, auto CD | Requires Observable pattern |

## Zoneless Compatibility

### Safe Patterns (work with zoneless)

```typescript
// NgZone.run() — still forces CD in zoneless
this.zone.run(() => this.handleEvent());

// NgZone.runOutsideAngular() — still useful for performance
this.zone.runOutsideAngular(() => this.heavyCalculation());

// Signals — primary CD mechanism in zoneless
this.data.set(newValue);

// Async pipe — triggers markForCheck internally
{{ observable$ | async }}
```

### Incompatible Patterns (break with zoneless)

```typescript
// NgZone.onStable — never fires without zone.js
this.zone.onStable.subscribe(() => { /* dead code */ });

// NgZone.onMicrotaskEmpty — never fires without zone.js
this.zone.onMicrotaskEmpty.subscribe(() => { /* dead code */ });

// NgZone.isStable — always true without zone.js
if (this.zone.isStable) { /* always true */ }
```

### Detection Command

```bash
# Find incompatible patterns
grep -rn "onStable\|onMicrotaskEmpty\|\.isStable" --include="*.ts" src/ projects/
```

If any are found, they must be replaced with signals or polling before enabling zoneless.
