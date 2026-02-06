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

## Anti-Pattern 8: Mixing signal() and @Input()

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
