# Signal Migration Anti-Patterns

## Anti-Pattern 1: Using model() for Mutated Input Objects

**Real failure:** Training-status-modal was migrated to `model()`, then reverted 4 minutes later, then fixed with `linkedSignal` the next day. 3 wasted commits.

```typescript
// WRONG — model() makes item a function, breaks [(ngModel)]="item.field"
item = model<TrainingDTO>();
// Template: [(ngModel)]="item().trainingTime" — DOES NOT WORK with ngModel

// RIGHT — linkedSignal for mutable derived state
itemInput = input<TrainingDTO>();
item = linkedSignal(() => this.itemInput() ? { ...this.itemInput()! } : null);

// OR — alias + local copy for zero template changes
itemInput = input<TrainingDTO>(undefined, { alias: 'item' });
item: TrainingDTO;  // local mutable copy
```

**How to detect:** Look for `[(ngModel)]="inputProp.nestedField"` in templates.

## Anti-Pattern 2: Piecemeal Migration of Complex Components

**Real failure:** Grid component took 5 commits over 3 days because it was migrated property-by-property.

```
Commit 1: Simple inputs → signals
Commit 2: page → model() (broke offset/count computed)
Commit 3: sort, filter, selected → model() (broke two-way bindings)
Commit 4: Fix missing () in templates
Commit 5: rows, columns → input() with effect()
```

**Right approach:** Migrate ALL properties of a complex component in ONE commit:
1. Read the entire component and its template
2. Classify every property (input/output/model/computed/effect)
3. Write the complete migration
4. Run RULE 7 verification on both .ts and .html
5. Build and test
6. Single commit

## Anti-Pattern 3: Forgetting Signal () Calls

**Real failure:** 2 separate commits (ce65088, 0efebf2) were entirely about adding missing `()`.

```typescript
// WRONG — signal function is always truthy!
if (this.isContractor) {  // Always true — isContractor is a function reference
  doContractorStuff();
}

// RIGHT
if (this.isContractor()) {
  doContractorStuff();
}
```

```html
<!-- WRONG — canEdit is always truthy -->
@if (canEdit) { <button>Save</button> }

<!-- RIGHT -->
@if (canEdit()) { <button>Save</button> }
```

**In templates:**
```html
<!-- WRONG -->
[selected]="selected"
[disabled]="disabled"

<!-- RIGHT -->
[selected]="selected()"
[disabled]="disabled()"
```

**In tests:**
```typescript
// WRONG
expect(component.menus).toEqual([...]);

// RIGHT
expect(component.menus()).toEqual([...]);
```

## Anti-Pattern 4: Mutating Signal Values

```typescript
// WRONG — mutation doesn't trigger change detection
this.page().totalCount = 10;
this.activeImage().rotate -= 90;
this.rows().push(newRow);

// RIGHT — create new object/array
this.page.set({ ...this.page(), totalCount: 10 });
this.activeImage.set({ ...this.activeImage(), rotate: this.activeImage().rotate - 90 });
this.rows.set([...this.rows(), newRow]);
```

## Anti-Pattern 5: Effect Without untracked()

```typescript
// WRONG — reads additional signals, creating unwanted dependencies
effect(() => {
  const rows = this.rows();
  if (this.groupRowsBy() && this.dataTable?.bodyComponent) {  // groupRowsBy tracked!
    this.processRows();
  }
});

// RIGHT — use untracked for reads that shouldn't re-trigger the effect
effect(() => {
  const rows = this.rows();  // Only track rows
  if (untracked(() => this.groupRowsBy()) && this.dataTable?.bodyComponent) {
    this.processRows();
  }
});
```

## Anti-Pattern 6: Using setTimeout to Fix Signal Timing

```typescript
// WRONG — masking a timing issue
effect(() => {
  const data = this.data();
  setTimeout(() => {
    this.processData(data);
    this.cdr.detectChanges();  // If you need this, something is wrong
  }, 0);
});

// RIGHT — let signals handle reactivity
effect(() => {
  const data = this.data();
  if (data) {
    untracked(() => this.processData(data));
  }
});
```

## Anti-Pattern 7: Converting Every Property to a Signal

Not every property needs to be a signal. Only convert:
- Properties that come from parent components (`@Input` → `input()`)
- Properties that notify parent components (`@Output` → `output()`)
- Properties that drive template rendering (state → `signal()`)
- Properties derived from other signals (→ `computed()`)

**Don't convert:**
- Internal helper variables
- Method parameters
- Loop counters
- Template-local variables

## Anti-Pattern 8: Using viewChild() Instead of viewChild.required() for Static Queries

**Real failure:** Grid component's 27 `@ViewChild({ static: true })` template refs were converted to `viewChild()` (non-required). Result: 73 NG0100 test failures because `viewChild()` is not the static equivalent.

```typescript
// WRONG — viewChild() is the non-static equivalent, returns undefined initially
switchTpl = viewChild<TemplateRef<any>>('switchTpl');

ngOnInit() {
  this.setColumns();
  // switchTpl() is UNDEFINED here!
}

// RIGHT — viewChild.required() is the { static: true } equivalent
switchTpl = viewChild.required<TemplateRef<any>>('switchTpl');

ngOnInit() {
  this.setColumns();  // switchTpl() is available here
}
```

**Key rule:** `@ViewChild({ static: true })` → `viewChild.required()`, NOT `viewChild()`.

## Anti-Pattern 9: model() with One-Way [data] Binding in Parent

**Real failure:** All truck register sub-forms (v-tax-form, v-general-form, v-license-form, etc.) used `model()` for `data` but parents bound with `[data]="data"` (one-way). Child's `patchData` called `this.data.set({ ...this.data(), [field]: value })` — the signal fired, CD worked, but the new object never propagated back to the parent. On submit, the parent sent stale data. Date pickers and all fields changed through child sub-forms were silently lost.

Same bug affected admin/truck e-form sub-forms (v-1-truck-type-form, v-2-compartment-form, v-3-inspection-checkbox-form).

```html
<!-- WRONG — child creates new object via data.set(), parent never receives it -->
<v-tax-form [data]="data"></v-tax-form>

<!-- RIGHT — two-way binding propagates child's data.set() back to parent -->
<v-tax-form [(data)]="data"></v-tax-form>
```

**Detection:**
```bash
# Find model() properties in child components
grep -rn "data.*=.*model" --include="*.ts" src/ projects/

# Then check if parent templates use one-way [data] for those components
grep -rn "\[data\]=" --include="*.html" src/ projects/
```

**Rule:** When a child uses `model()` with `patchData`-style updates (`data.set({...})`), the parent **MUST** use `[(data)]` two-way binding. Otherwise the parent's property is never updated and submissions send stale data.

## Anti-Pattern 10: linkedSignal with Spread in patchData Disconnecting from Parent

**Real failure:** Driver profile's `v-user-profile` used `linkedSignal(() => this.setData())` and `patchData` did `this.data.set({ ...this.data(), [field]: value })`. After the first `patchData` call, the linkedSignal held a new object disconnected from the parent's source. All subsequent edits in v-user-profile were lost on submit because the parent's `this.data` was never updated.

```typescript
// WRONG — creates new object, disconnects from parent's source
patchData(field: string, value: any) {
  this.data.set({ ...this.data(), [field]: value });
}

// RIGHT — mutate source input first (updates parent), then trigger signal for CD
patchData(field: string, value: any) {
  this.setData()[field] = value;          // mutate parent's object
  this.data.set({ ...this.setData() });   // new ref triggers CD
}
```

**Why it works:** `this.setData()` returns the parent's input object. Mutating it ensures the parent's `this.data` has the updated field. Then creating a new object reference via spread triggers the linkedSignal's CD. On subsequent calls, `this.setData()` still returns the parent's object (which was mutated), so all changes accumulate correctly.

**Rule:** When using `linkedSignal` with `patchData`, always mutate the source input object (`this.setData()[field] = value`) before triggering the signal. Never spread only from `this.data()` which may be a disconnected copy.

## Anti-Pattern 11: Mixing signal() and @Input()

During incremental migration, you might have both patterns in one component. This is OK temporarily, but:

```typescript
// WRONG — confusing which need () and which don't
disabled = input(false);     // signal — use disabled()
@Input() label: string;     // decorator — use label directly
loading = signal(false);     // signal — use loading()
active = true;               // plain — use active directly

// RIGHT — migrate ALL inputs in one pass for a given component
disabled = input(false);
label = input<string>();
loading = signal(false);     // state signals are fine to add incrementally
```
