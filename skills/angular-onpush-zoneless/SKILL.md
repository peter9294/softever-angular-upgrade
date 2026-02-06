---
name: Angular OnPush & Zoneless Migration
description: Use this skill when the user asks to "migrate to OnPush", "enable OnPush change detection", "zoneless Angular", "remove zone.js", "provideZonelessChangeDetection", or when working with change detection strategy. Covers the three-phase OnPush approach (trivial → outputs → complex), subscribe() callback auditing, object mutation fixes, and zoneless enablement. Encodes RULE 4 (subscribe callback audit).
version: 1.0.0
---

# Angular OnPush & Zoneless Migration

## Purpose

Guide the migration from `ChangeDetectionStrategy.Default` to `OnPush`, and then to full zoneless change detection. This is the final phase of an Angular modernization — it requires signals migration to be substantially complete first.

## Prerequisites

Before starting OnPush migration:
1. Most `@Input()` should be migrated to `input()` signals
2. Most `@Output()` should be migrated to `output()` signals
3. Key state properties should be `signal()` or `computed()`

## CRITICAL: Subscribe Callback Audit (RULE 4)

**This is the #1 source of OnPush bugs.** Every `.subscribe()` callback that sets a component property will silently break with OnPush.

### Pre-Migration Audit

```bash
# Find all subscribe callbacks that set properties
grep -rn "\.subscribe(" --include="*.ts" src/app/ projects/ -A 5 | \
  grep "this\.[a-zA-Z].*="
```

### The Problem

With `Default` change detection, zone.js triggers CD after every async operation. With `OnPush`, CD only runs when:
1. An input reference changes
2. A template event fires
3. A signal notifies
4. `markForCheck()` is called explicitly
5. The `async` pipe triggers

**Subscribe callbacks are NONE of these.** Setting `this.data = response` inside `.subscribe()` won't update the template.

### The Fix

For each subscribe callback that sets state:

```typescript
// BEFORE (breaks with OnPush)
this.apiService.getData().subscribe(data => {
  this.data = data;  // Template won't update!
});

// FIX 1: Convert to signal (preferred)
data = signal<DataDTO | null>(null);
// ...
this.apiService.getData().subscribe(data => {
  this.data.set(data);  // Signal notifies CD
});

// FIX 2: Use markForCheck (escape hatch)
constructor(private cdr: ChangeDetectorRef) {}
// ...
this.apiService.getData().subscribe(data => {
  this.data = data;
  this.cdr.markForCheck();
});

// FIX 3: Use async pipe (cleanest for templates)
data$ = this.apiService.getData();
// Template: @if (data$ | async; as data) { ... }
```

## MCP Tool Integration

Use the Angular CLI MCP tool for analysis:

```
mcp__angular-cli__onpush_zoneless_migration({ fileOrDirPath: '/path/to/component' })
```

This tool:
- Analyzes a component/directory
- Identifies the next action to take
- Provides step-by-step instructions
- Must be called repeatedly until no more actions needed

**Limitation:** The tool identifies issues but doesn't always fix them correctly for complex cases (subscribe callbacks, timer-based code, third-party library callbacks). Use the tool for analysis, apply fixes manually.

## Three-Phase OnPush Approach

### Phase 1: Trivial Components

Components with no reactive state — just add OnPush.

**Criteria:**
- No `subscribe()` calls
- No `setTimeout()` / `setInterval()`
- No manual DOM manipulation
- Template only uses inputs, outputs, and static content

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  // ...
})
```

### Phase 2: Components with Outputs

Components that emit events — convert outputs to `output()` signals, then add OnPush.

```typescript
// Before
@Output() loginSuccess = new EventEmitter<void>();

// After
loginSuccess = output<void>();
```

### Phase 3: Complex Components

Components with subscriptions, timers, or third-party library callbacks.

For each one:
1. Run the subscribe audit (RULE 4)
2. Convert state properties to signals
3. Add `takeUntilDestroyed()` for cleanup
4. Add OnPush
5. Build and test

## Object Mutation Fixes

OnPush requires immutable state updates. Common fixes:

### Image Rotation
```typescript
// BEFORE (mutation — OnPush won't detect)
this.activeImage.rotate = this.activeImage.rotate - 90;

// AFTER (new object — OnPush detects)
this.activeImage.set({
  ...this.activeImage(),
  rotate: this.activeImage().rotate - 90
});
```

### Array Updates
```typescript
// BEFORE
this.rows.push(newRow);

// AFTER
this.rows.set([...this.rows(), newRow]);
```

### Nested Object Updates
```typescript
// BEFORE
this.config.settings.theme = 'dark';

// AFTER
this.config.set({
  ...this.config(),
  settings: { ...this.config().settings, theme: 'dark' }
});
```

## Animation Callbacks with OnPush

Animation completion callbacks run outside Angular's CD. With OnPush, use signals:

```typescript
animationState = signal<'open' | 'closed' | 'void'>('open');

onConfirm() {
  this.animationState.set('closed');  // Signal triggers CD
}

onAnimationDone(event: AnimationEvent) {
  if (event.toState === 'closed') {
    this.close();
  }
}
```

## NgZone Compatibility Assessment

**These NgZone patterns ARE compatible with zoneless:**
- `NgZone.run()` — Forces synchronous CD, works in zoneless
- `NgZone.runOutsideAngular()` — Still useful for performance

**These NgZone patterns are NOT compatible:**
- `NgZone.onStable` — No zone = never fires
- `NgZone.onMicrotaskEmpty` — No zone = never fires
- `NgZone.isStable` — Always true in zoneless

```bash
# Check for incompatible patterns
grep -rn "onStable\|onMicrotaskEmpty\|isStable" --include="*.ts" src/ projects/
```

## Enabling Zoneless Change Detection

After ALL components are OnPush-compatible:

### Step 1: Add provider

```typescript
// app.module.ts (or app.config.ts for standalone)
import { provideZonelessChangeDetection } from '@angular/core';

@NgModule({
  providers: [
    provideZonelessChangeDetection(),
  ],
})
```

### Step 2: Remove zone.js from polyfills

```json
// angular.json
"polyfills": [
  "@angular/localize/init"
  // Remove: "zone.js"
  // Remove: "zone-flags.ts"
]
```

### Step 3: Verify bundle savings

Expected: ~92KB reduction in polyfills bundle.

```bash
yarn build:dev
# Compare polyfills bundle size before and after
```

## OnPush Audit Checklist

See `references/onpush-audit-checklist.md` for the complete per-component audit.

## References

- `references/onpush-audit-checklist.md` — Per-component audit checklist
- `references/zone-boundary-patterns.md` — Common zone boundary issues and fixes
- `references/softever-subscribe-to-signal.md` — Real before/after examples from a 50-file subscribe→signal migration, including scope reduction strategy and common bugs
