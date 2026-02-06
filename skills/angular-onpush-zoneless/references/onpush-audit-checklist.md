# OnPush Audit Checklist

## Per-Component Audit

For each component being migrated to OnPush, verify every item:

### 1. Subscribe Callbacks (RULE 4)

```bash
grep -n "\.subscribe(" component-name.component.ts
```

For each `.subscribe()`:
- [ ] Does the callback set `this.someProperty = value`?
- [ ] If YES: Is `someProperty` a signal? (Safe if yes)
- [ ] If NO: Convert to signal OR add `markForCheck()`
- [ ] Is there cleanup? (`takeUntilDestroyed()`, `unsubscribe()`, or `takeUntil()`)

### 2. setTimeout / setInterval

```bash
grep -n "setTimeout\|setInterval" component-name.component.ts
```

For each timer:
- [ ] Does the callback set component state?
- [ ] If YES: Convert state to signal
- [ ] Is there cleanup? (`clearTimeout`/`clearInterval` in `ngOnDestroy`)

### 3. Third-Party Library Callbacks

```bash
grep -n "\.on(\|addEventListener\|fromEvent" component-name.component.ts
```

Common offenders:
- Highcharts click events → Use `NgZone.run()` wrapper
- Calendar widget events → Use `NgZone.run()` wrapper
- Dropzone callbacks → Convert to signal state

For each:
- [ ] Does the callback set component state?
- [ ] If YES: Wrap in `NgZone.run()` or use signals

### 4. Object Mutations

```bash
grep -n "this\.\w\+\.\w\+\s*=" component-name.component.ts | grep -v "this\.\w\+\.set\|this\.\w\+\.update"
```

For each mutation:
- [ ] Is the property bound in the template?
- [ ] If YES: Convert to immutable update pattern

### 5. Template Bindings

Check the template for:
- [ ] All signal properties use `()` syntax
- [ ] No direct property mutation from template events
- [ ] `async` pipe used for Observables (auto-triggers `markForCheck()`)

### 6. Animation Callbacks

```bash
grep -n "@.*\.done\|animationDone\|animation.*callback" component-name.component.ts
```

- [ ] Animation state is a signal (not a plain property)
- [ ] Completion callbacks use signals or `markForCheck()`

### 7. CDR Usage

```bash
grep -n "ChangeDetectorRef\|cdr\." component-name.component.ts
```

- [ ] `detectChanges()` calls replaced with signal updates (preferred)
- [ ] `markForCheck()` only used as escape hatch for unavoidable plain properties
- [ ] Remove CDR injection if no longer needed

## Quick Classification

| Component Type | OnPush Difficulty | Approach |
|---------------|-------------------|----------|
| Pure display (inputs only) | Trivial | Just add OnPush |
| Form component | Low | Signals + OnPush |
| List with API data | Medium | Signal state + subscribe audit |
| Dashboard with charts | Medium | NgZone.run() for chart callbacks |
| Real-time data (WebSocket) | Hard | Async pipe or signal state |
| Complex form wizard | Hard | Full signal migration first |

## Build Verification After Each Component

```bash
yarn build:dev    # Must compile
yarn lint         # Must pass
# Manual: verify the component renders and updates correctly
```

## Rollback Plan

If a component breaks after OnPush:
1. **Don't remove OnPush** — find the root cause
2. Check the subscribe audit — did you miss a callback?
3. Check for object mutations
4. As last resort: add `markForCheck()` after the problematic state change
