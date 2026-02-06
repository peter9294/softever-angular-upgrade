# Input Mutation Catalog

## How to Identify Input-Mutating Components

An "input-mutating component" is one that modifies properties of an object received via `@Input()`. This is the #1 cause of signal migration failures.

### Detection Commands

```bash
# Find [(ngModel)] bound to @Input properties in templates
grep -rn "ngModel.*\.\(data\|item\|filter\|config\|form\)" --include="*.html" src/ projects/

# Find direct property assignment on @Input objects in TypeScript
grep -rn "this\.\(data\|item\|filter\|config\)\.[a-zA-Z].*=" --include="*.ts" src/ projects/ | \
  grep -v "this\.\(data\|item\|filter\|config\)\.\(subscribe\|pipe\|map\|filter\)"

# Find array mutations on @Input objects
grep -rn "this\.\(data\|items\|rows\)\.\(push\|splice\|sort\|reverse\|shift\|pop\)" --include="*.ts" src/ projects/
```

### Classification Matrix

| Pattern Found | Risk Level | Migration Strategy |
|--------------|------------|-------------------|
| `[(ngModel)]="inputProp.field"` | HIGH | Alias + local copy (Pattern 2) |
| `this.inputProp.field = value` in methods | HIGH | Alias + local copy or linkedSignal |
| `this.inputProp.push(item)` | HIGH | Alias + local copy with spread |
| `[ngModel]="inputProp.field"` (one-way only) | LOW | Simple `input()` with `()` in template |
| `[(prop)]` two-way on parent side | MEDIUM | `model()` with immutable updates |

## Real Examples from This Codebase

### Example 1: shared-training-status-modal (DEFERRED)

```typescript
// This component mutates item.trainingTime via [(ngModel)]
@Input() item: TrainingDTO;
// Template: <input [(ngModel)]="item.trainingTime">

// Migration attempt 1: model() — FAILED
// item() is a function, can't do [(ngModel)]="item().trainingTime"
// Reverted after 4 minutes.

// Migration attempt 2: linkedSignal — WORKED
itemInput = input<TrainingDTO>();
item = linkedSignal(() => this.itemInput() ? { ...this.itemInput()! } : null);
// Template: <input [(ngModel)]="item()!.trainingTime">
// BUT this requires changing EVERY template reference to item()!.xxx
```

### Example 2: fe-incident-report-form

```typescript
// data has many [(ngModel)] bindings to nested properties
@Input() data: IncidentReportDTO;
// Template: [(ngModel)]="data.description", [(ngModel)]="data.incidentDate", etc.

// Solution: alias + local copy (Pattern 2) — ZERO template changes
dataInput = input<IncidentReportDTO>(undefined, { alias: 'data' });
data: IncidentReportDTO;

constructor() {
  effect(() => {
    const d = this.dataInput();
    if (d) Object.assign(this.data, d);
  });
}
```

### Example 3: shared-document-approval-list

```typescript
// filter is passed from parent but also has [(ngModel)] on properties
@Input() filter: any = {};
// Template: [(ngModel)]="filter.status", etc.

// Solution: alias + local copy
filterInput = input({} as any, { alias: 'filter' });
filter = {} as any;

constructor() {
  effect(() => {
    const f = this.filterInput();
    if (f) Object.assign(this.filter, f);
  });
}
```

### Example 4: fe-contract-worker-basic-info (DEFERRED)

```typescript
// data is mutated directly in methods, not just via ngModel
@Input() data: WorkerDTO;

onSave() {
  this.data.status = 'approved';  // Direct mutation
  this.data.approvedDate = new Date();
}

// This needs writable signal pattern (not implemented yet)
// Deferred because it changes the component's behavioral contract
```

### Example 5: Grid filter (model signal)

```typescript
// filter is two-way bound [(filter)] and has [(ngModel)] on nested props
filter = model<{ [key: string]: any }>({});

// Template split pattern:
// <input [ngModel]="filter()[column.prop]"
//        (ngModelChange)="onFilterPropChange(column.prop, $event)">

onFilterPropChange(prop: string, value: any) {
  this.filter.update(f => ({ ...f, [prop]: value }));
  this.filterChange.emit(this.filter());
}
```

## Decision Guide

```
Does component mutate @Input via [(ngModel)]="input.prop"?
├── YES: How many ngModel bindings?
│   ├── 1-3 bindings → linkedSignal (Pattern 4)
│   │   Pro: Clean signal semantics
│   │   Con: Must change template to input()!.prop
│   └── 4+ bindings → Alias + local copy (Pattern 2)
│       Pro: ZERO template changes
│       Con: Extra property + effect() boilerplate
├── NO: Does component mutate @Input in methods?
│   ├── YES → Defer or use writable signal pattern
│   └── NO → Simple input() (Pattern 1)
```
