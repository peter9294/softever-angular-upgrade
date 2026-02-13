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

## Anti-Pattern 8: Converting @ViewChild({ static: true }) to viewChild()

**Real failure:** Grid component's 27 `@ViewChild({ static: true })` template refs were converted to `viewChild()` signals. Result: 73 NG0100 test failures. The entire batch had to be reverted.

```typescript
// WRONG — viewChild() resolves during CD, NOT before ngOnInit
switchTpl = viewChild<TemplateRef<any>>('switchTpl');
nameTpl = viewChild<TemplateRef<any>>('nameTpl');

ngOnInit() {
  this.setColumns();
  // switchTpl() and nameTpl() are UNDEFINED here!
  // They resolve during the FIRST change detection cycle,
  // but ngOnInit runs BEFORE the first CD for static queries.
}

// RIGHT — keep @ViewChild({ static: true }) for template refs needed in ngOnInit
@ViewChild('switchTpl', { static: true }) switchTpl: TemplateRef<any>;
@ViewChild('nameTpl', { static: true }) nameTpl: TemplateRef<any>;

ngOnInit() {
  this.setColumns();  // Template refs are available here
}
```

**Why NG0100?** When `viewChild()` returns `undefined` in `ngOnInit`, `setColumns()` sets column templates to `undefined`. During the first CD cycle, `viewChild()` resolves to the actual `TemplateRef`. On the dev-mode re-check, Angular sees the binding changed (`undefined` → `TemplateRef`) and throws NG0100. The error aborts CD, preventing child components from rendering.

**Angular's official position:** Signal-based `viewChild()` has NO `{ static: true }` equivalent. The decorator API is fully supported and is the correct tool for static queries. GitHub issue #54376 tracks the feature request.

**Detection:**
```bash
# Find all static ViewChild — these must NOT be converted
grep -rn "ViewChild.*static.*true" --include="*.ts" src/ projects/
```

## Anti-Pattern 9: Using viewChild.required for Testable Properties

**Real failure:** `base-document-check.ts` had modal refs converted to `viewChild.required` with getter accessors. 13 test failures across 2 spec files because tests couldn't mock the modals.

```typescript
// WRONG — viewChild.required creates an unmockable getter
private readonly _approveModal = viewChild.required<SoftModalComponent>('approveModal');
get approveModal() { return this._approveModal(); }

// Tests CANNOT do this:
component.approveModal = jasmine.createSpyObj('modal', ['open', 'close']);
// TypeError: Cannot set property which has only a getter

// RIGHT — plain @ViewChild for properties that tests need to mock
@ViewChild('approveModal') approveModal: SoftModalComponent;
@ViewChild('resultModal') resultModal: SoftModalComponent;

// Tests CAN do this:
component.approveModal = jasmine.createSpyObj('modal', ['open', 'close']);
```

**Rule of thumb:** If a component property is mocked in tests (especially modals, child components), keep it as `@ViewChild` unless you're willing to restructure all the test mocks.

## Anti-Pattern 10: Mixing signal() and @Input()

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
