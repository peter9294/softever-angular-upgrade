---
name: Angular Signals Migration
description: Use this skill when the user asks to "migrate to signals", "convert @Input to signal", "convert @Output to output()", "signal migration", "input() migration", or when working with Angular signal patterns. Covers all 5 migration patterns (simple input, alias+local-copy, model, linkedSignal, watchedInputs), input classification by template binding, and critical anti-patterns. Encodes RULE 1 (input mutation pre-scan), RULE 2 (atomic complex migration), and RULE 7 (missing signal () verification).
version: 1.0.0
---

# Angular Signals Migration

## Purpose

Guide the migration of `@Input()`, `@Output()`, and `ngOnChanges` to Angular's signal-based APIs (`input()`, `output()`, `model()`, `computed()`, `effect()`). This skill prevents the common mistakes that caused ~23% rework in a real 93-commit upgrade.

## CRITICAL: Pre-Migration Scan (RULE 1)

**BEFORE running any signal migration tool**, classify every component's inputs:

### Step 1: Check Template Binding

```bash
# For each @Input property, find how it's used in templates
grep -r "propertyName" --include="*.html" src/ projects/
```

| Template Binding | Signal Type | Notes |
|------------------|-------------|-------|
| `[prop]="x"` (read-only) | `input()` | Simple — tool handles well |
| `[(prop)]="x"` (two-way) | `model()` | Need immutable updates |
| `(propChange)="fn($event)"` | `output()` | Simple — tool handles well |
| `[(ngModel)]="prop.field"` on input's nested property | **Manual** | Tool CANNOT handle — see patterns below |

### Step 2: Check for Input Mutation

```bash
# Find components that mutate their @Input objects
grep -rn "this\.\(data\|item\|filter\|config\)\." --include="*.ts" src/app/ | \
  grep -E "\.(push|splice|sort|reverse|[a-zA-Z]+\s*=)" | head -30
```

Components that mutate inputs need special handling:
- **linkedSignal** — For form fields that derive from parent input
- **Alias + local copy** — For `[(ngModel)]` on nested properties
- **NEVER use `model()`** for objects with `[(ngModel)]` on properties — the signal function breaks ngModel binding

### Step 3: Identify Complex Components (RULE 2)

Components with 10+ inputs, ngOnChanges, ViewChild interactions, or timer side effects must be migrated **atomically in one pass**. The grid component took 5 incremental fix commits because it was done piecemeal.

## Tool-First Strategy

### Run the Bulk Migration Tool

```bash
ng generate @angular/core:signals
```

This handles ~70% of simple conversions. Then manually fix edge cases.

### What the Tool Handles Well
- Simple `@Input()` → `input()`
- Simple `@Output() EventEmitter` → `output()`
- Adding `()` to template reads

### What the Tool CANNOT Handle
- Input-mutating components (RULE 1)
- Complex `ngOnChanges` with conditional logic
- `[(ngModel)]` on signal properties
- Two-way binding with immutable update patterns
- `watchedInputs()` pattern for cascading dropdowns
- Components with timer/subscription side effects in setters

## The 5 Migration Patterns

### Pattern 1: Simple input() (Most Common)

```typescript
// Before
@Input() disabled = false;

// After
disabled = input(false);

// Template: disabled → disabled()
```

### Pattern 2: Alias + Local Copy (Input Mutation with ngModel)

When the parent passes data via `[prop]` and the component uses `[(ngModel)]="prop.field"`:

```typescript
// Before
@Input() data: MyDTO;

// After — signal input with alias, local mutable copy
dataInput = input<MyDTO>(undefined, { alias: 'data' });
data: MyDTO;

constructor() {
  effect(() => {
    const d = this.dataInput();
    if (d) {
      this.data = { ...d };  // or Object.assign(this.data, d)
    }
  });
}
// Template: NO changes needed — still uses local `data` property
```

### Pattern 3: model() (Two-Way Binding)

When the parent uses `[(prop)]`:

```typescript
// Before
@Input() page: Page = { page: 1, perPage: 10, totalCount: 0 };
@Output() pageChange = new EventEmitter<Page>();

// After
page = model<Page>({ page: 1, perPage: 10, totalCount: 0 });

// CRITICAL: Immutable updates only!
// WRONG: this.page().totalCount = 10;
// RIGHT: this.page.set({ ...this.page(), totalCount: 10 });
```

### Pattern 4: linkedSignal (Derived Mutable State)

When input data is used in forms but the local state can diverge from the parent:

```typescript
// Before
@Input() item: TrainingDTO;
// Template: [(ngModel)]="item.trainingTime"

// After
itemInput = input<TrainingDTO>();
item = linkedSignal(() => this.itemInput() ? { ...this.itemInput()! } : undefined);

// Template: [(ngModel)]="item()!.trainingTime"
// NOTE: Need to unwrap with () and non-null assertion
```

### Pattern 5: watchedInputs() (Cascading Dropdowns)

For dropdown components that reload options when a parent input changes:

```typescript
// Before (BaseDropdown child)
@Input() province: ProvinceDTO;
inputChanges = ['province'];
// ngOnChanges checks inputChanges array

// After
province = input<ProvinceDTO>();
protected override watchedInputs(): InputSignal<any>[] {
  return [this.province];
}
// Base class effect() watches these signals automatically
```

## ngModel Split Pattern

When `[(ngModel)]` is bound to a signal property:

```html
<!-- Before -->
<input [(ngModel)]="value">

<!-- After — split into one-way read + change handler -->
<input [ngModel]="value()" (ngModelChange)="value.set($event)">
```

For nested properties on a model():
```html
<!-- Before -->
<input [(ngModel)]="filter[column.prop]">

<!-- After — helper method for immutable update -->
<input [ngModel]="filter()[column.prop]"
       (ngModelChange)="onFilterPropChange(column.prop, $event)">
```

```typescript
onFilterPropChange(prop: string, value: any) {
  this.filter.update(f => ({ ...f, [prop]: value }));
}
```

## Object Identity Rule

**Signals detect changes by object identity, NOT mutations.**

```typescript
// WRONG — mutation, signal won't fire
this.page().totalCount = 10;
this.activeImage().rotate = newRotate;

// RIGHT — new object reference
this.page.set({ ...this.page(), totalCount: 10 });
this.activeImage.set({ ...this.activeImage(), rotate: newRotate });
```

## Post-Migration Verification (RULE 7)

After ANY signal migration, run these checks:

### Check 1: Missing () in Templates

```bash
# Find signal properties used without () in templates
# Look for properties that are now signals but templates still reference without ()
grep -rn "\[.*\]=\"\(disabled\|loading\|viewMode\|isApproval\|canEdit\|isContractor\|selected\|filter\)\"" \
  --include="*.html" src/ projects/ | grep -v "()"
```

### Check 2: Missing () in TypeScript

```bash
# Signal properties used as truthy checks without ()
grep -rn "if (this\.\(isContractor\|canEdit\|disabled\|loading\|viewMode\))" \
  --include="*.ts" src/ projects/
```

### Check 3: Missing () in Tests

```bash
# Test files accessing signal properties without ()
grep -rn "component\.\(disabled\|loading\|page\|sort\|filter\)" \
  --include="*.spec.ts" src/ projects/ | grep -v "()"
```

**Why this matters:** A signal function is always truthy. `if (this.isContractor)` compiles but ALWAYS evaluates to true. This was missed in 2 separate commits.

## Template: Signal Unwrap Pattern

```html
<!-- For nullable signal inputs, unwrap once with @if...as -->
@if (data(); as data) {
  <input [ngModel]="data.name">
  <span>{{ data.status }}</span>
}
```

## Computed Signals for Derived State

Replace getter properties with `computed()`:

```typescript
// Before
get isThai(): boolean {
  return !this.data?.nationality || this.data.nationality?.key === NationalityKey.Thai;
}

// After
isThai = computed(() => {
  const d = this.data();
  return !d?.nationality || d.nationality?.key === NationalityKey.Thai;
});
```

## References

- `references/signal-patterns.md` — Complete pattern catalog with examples
- `references/signal-anti-patterns.md` — What NOT to do (with real failure stories)
- `references/input-mutation-catalog.md` — How to identify and handle input-mutating components
