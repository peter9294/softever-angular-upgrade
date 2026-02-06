# Signal Migration Patterns — Complete Catalog

## Decision Tree

```
Is the @Input used with [(prop)] two-way binding?
  YES → Does the component mutate the input object?
    YES → Use Pattern 2 (alias + local copy) or Pattern 4 (linkedSignal)
    NO  → Use Pattern 3 (model)
  NO → Is the input used in ngOnChanges?
    YES → Is it a cascading dropdown (reload on parent change)?
      YES → Use Pattern 5 (watchedInputs)
      NO  → Use effect() to replace ngOnChanges logic
    NO → Use Pattern 1 (simple input)
```

## Pattern 1: Simple Input Signal

**When:** Read-only input, no mutation, no ngOnChanges dependency.

```typescript
// Decorator style
@Input() disabled = false;
@Input() label: string;
@Input() items: Item[] = [];

// Signal style
disabled = input(false);
label = input<string>();
items = input<Item[]>([]);

// With alias (when property name differs from binding name)
filterInput = input({} as any, { alias: 'filter' });
```

**Template changes:**
```html
<!-- Before -->
<div [class.disabled]="disabled">{{ label }}</div>

<!-- After -->
<div [class.disabled]="disabled()">{{ label() }}</div>
```

## Pattern 2: Alias + Local Copy

**When:** Parent passes via `[prop]`, component uses `[(ngModel)]` on nested properties.

```typescript
// The input (read-only from parent)
dataInput = input<IncidentReportDTO>(undefined, { alias: 'data' });

// Local mutable copy for template ngModel
data: IncidentReportDTO;

constructor() {
  effect(() => {
    const d = this.dataInput();
    if (d) {
      Object.assign(this.data, d);
      // or: this.data = { ...d };
    }
  });
}
```

**Template:** No changes needed — continues using local `data` property.

**Use over linkedSignal when:** The component has many `[(ngModel)]` bindings to nested properties and you don't want to change every template reference.

## Pattern 3: Model Signal

**When:** Parent uses `[(prop)]` two-way binding AND the component doesn't mutate nested properties via ngModel.

```typescript
// Before
@Input() page: Page = { page: 1, perPage: 10, totalCount: 0 };
@Output() pageChange = new EventEmitter<Page>();

// After — model() replaces both @Input and @Output
page = model<Page>({ page: 1, perPage: 10, totalCount: 0 });
```

**Updating model values:**
```typescript
// Set entire value
this.page.set({ page: 2, perPage: 10, totalCount: 100 });

// Update partial (immutable)
this.page.update(p => ({ ...p, totalCount: newCount }));
```

**Template:**
```html
<!-- Read -->
<span>Page {{ page().page }} of {{ page().totalCount }}</span>

<!-- Two-way child binding -->
<child [(page)]="page"></child>
```

## Pattern 4: Linked Signal

**When:** Input data feeds into a form where local state can diverge from parent.

```typescript
itemInput = input<TrainingDTO>();
item = linkedSignal(() => this.itemInput() ? { ...this.itemInput()! } : null);
```

**Template:**
```html
@if (item(); as item) {
  <input [(ngModel)]="item.trainingTime"
         (ngModelChange)="this.item.set({ ...item, trainingTime: $event })">
}
```

**Note:** `linkedSignal` resets when the input changes, but allows local mutations.

## Pattern 5: watchedInputs() (Cascading Dropdowns)

**When:** A dropdown component reloads its options when a parent-controlled input changes.

**Base class:**
```typescript
protected watchedInputs(): InputSignal<any>[] {
  return [];
}

constructor() {
  effect(() => {
    const signals = this.watchedInputs();
    signals.forEach(s => s()); // Track all signals
    if (!this.initialized) return;
    untracked(() => {
      if (this.autoReset()) { this.reset(); }
      this.getOptionList();
    });
  });
}
```

**Child class:**
```typescript
province = input<ProvinceDTO>();

protected override watchedInputs() {
  return [this.province];
}
```

## Output Signal Pattern

```typescript
// Before
@Output() save = new EventEmitter<SaveDTO>();
@Output() cancel = new EventEmitter<void>();

// After
save = output<SaveDTO>();
cancel = output<void>();

// Emit
this.save.emit(data);
this.cancel.emit();
```

## Computed Signal Pattern

Replace `get` accessors with `computed()`:

```typescript
// Before
get count(): number { return this.page?.totalCount ?? 0; }
get offset(): number { return (this.page?.page ?? 1) - 1; }

// After
count = computed(() => this.page()?.totalCount ?? 0);
offset = computed(() => (this.page()?.page ?? 1) - 1);
```

## Effect Pattern for Side Effects

Replace `ngOnChanges` with `effect()`:

```typescript
// Before
ngOnChanges(changes: SimpleChanges) {
  if (changes['columns'] && this.columns?.length) {
    this.processColumns();
  }
}

// After
columns = input<GridTableColumn[]>([]);

constructor() {
  effect(() => {
    const cols = this.columns();
    if (cols.length) {
      untracked(() => this.processColumns());
    }
  });
}
```

## Effect with Cleanup

```typescript
constructor() {
  effect((onCleanup) => {
    const sub = this.dataSub();
    if (sub) {
      const timer = setTimeout(() => this.loading.set(true), 200);
      sub.add(() => {
        clearTimeout(timer);
        this.loading.set(false);
      });
      onCleanup(() => clearTimeout(timer));
    }
  });
}
```

## NotificationsService Pattern (BehaviorSubject → Signal)

```typescript
// Before
private menuStatSubject = new BehaviorSubject<UserMenuDTO | null>(null);
getMenuStat$() { return this.menuStatSubject.pipe(filter(v => !!v)); }

// After
menuStat = signal<UserMenuDTO | null>(null);

// Consumer — Before
this.notificationsService.getMenuStat$().subscribe(stat => {
  this.menuStat = stat;
});

// Consumer — After
effect(() => {
  const stat = this.notificationsService.menuStat();
  if (stat) {
    this.processMenuStat(stat);
  }
});
```
