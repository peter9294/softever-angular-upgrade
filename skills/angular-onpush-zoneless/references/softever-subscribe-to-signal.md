# Softever Subscribe-to-Signal Migration — Real Examples

Before/after examples from a real 50-file migration converting plain property assignments in `.subscribe()` callbacks to `signal().set()` for OnPush compatibility.

## Why This Migration is Needed

Under `ChangeDetectionStrategy.OnPush`, Angular only re-renders when:
1. Signal values change
2. Input references change
3. Template events fire
4. `async` pipe emits
5. `markForCheck()` is called

**Plain property assignments in `.subscribe()` are NONE of these.** The template won't update.

## Scope Reduction Strategy

Before converting all 63 candidate files, check if the base class already handles CD:

```typescript
// BaseFeatureGridComponent has a rows setter with markForCheck:
private _rows: PropType[];
get rows() { return this._rows; }
set rows(value: PropType[]) {
  this._rows = value;
  this.cdr.markForCheck(); // ← Already handles OnPush!
}
```

**Impact:** ~40 components extending BaseFeatureGridComponent DON'T need `rows` converted to signal. Only OTHER properties in their subscribe callbacks need conversion.

## Example 1: Chart Data (Simple Signal)

```typescript
// BEFORE — breaks with OnPush
penaltyResultPieChartDTO: PenaltyResultPieChartDTO;
penaltyResultBarChartDTO: PenaltyResultIncidentCaseChartDTO;

private getPieChartData() {
  this.pieChartDataSub = this.penaltiesService.apiPenaltiesPieChartGet$()
    .subscribe(data => {
      this.penaltyResultPieChartDTO = data; // Won't trigger re-render!
    });
}

// AFTER — signal notifies Angular
penaltyResultPieChartDTO = signal<PenaltyResultPieChartDTO>(null);
penaltyResultBarChartDTO = signal<PenaltyResultIncidentCaseChartDTO>(null);

private getPieChartData() {
  this.pieChartDataSub = this.penaltiesService.apiPenaltiesPieChartGet$()
    .subscribe(data => {
      this.penaltyResultPieChartDTO.set(data); // Signal → CD triggered
    });
}
```

Template update:
```html
<!-- BEFORE -->
@if (penaltyResultPieChartDTO) {
  <shared-pie-chart [series]="penaltyResultPieChartDTO.series"></shared-pie-chart>
}

<!-- AFTER -->
@if (penaltyResultPieChartDTO()) {
  <shared-pie-chart [series]="penaltyResultPieChartDTO().series"></shared-pie-chart>
}
```

## Example 2: Workflow Count + Selected Status

```typescript
// BEFORE
workflowCount: WorkflowCountDTO;
selectedWorkflowStatus: WorkflowStatus;

getWorkflowCount() {
  this.workerflowCountSub = this.workersService.apiWorkflowsStatusGet$()
    .subscribe(data => {
      this.workflowCount = data;
    });
}

workflowCountClick(status: WorkflowStatus) {
  this.selectedWorkflowStatus = status; // Plain assignment
  this.filterChange();
}

clear() {
  this.selectedWorkflowStatus = undefined; // Plain assignment
}

// AFTER
workflowCount = signal<WorkflowCountDTO>(null);
selectedWorkflowStatus = signal<WorkflowStatus>(null);

getWorkflowCount() {
  this.workerflowCountSub = this.workersService.apiWorkflowsStatusGet$()
    .subscribe(data => {
      this.workflowCount.set(data);
    });
}

workflowCountClick(status: WorkflowStatus) {
  this.selectedWorkflowStatus.set(status);
  this.filterChange();
}

clear() {
  this.selectedWorkflowStatus.set(null); // Use null, not undefined
}
```

## Example 3: Complex Object with Breadcrumb

```typescript
// BEFORE
data: MasterWorkerDTO;
breadcrumbData: { workerNo: string, firstNameTH: string, lastNameTH: string };

getData() {
  this.dataSub = this.workersService.apiWorkersIdGet$(this.workerId)
    .subscribe(data => {
      this.data = data;
      this.breadcrumbData = {
        workerNo: data.workerNo,
        firstNameTH: data.firstNameTH,
        lastNameTH: data.lastNameTH,
      };
    });
}

get isThai(): boolean {
  return !this.data.nationality || this.data.nationality?.key === 'Thai';
}

// AFTER
data = signal<MasterWorkerDTO>(null as any);
breadcrumbData = signal<{ workerNo: string, firstNameTH: string, lastNameTH: string } | null>(null);

getData() {
  this.dataSub = this.workersService.apiWorkersIdGet$(this.workerId)
    .subscribe(data => {
      this.data.set(data);
      this.breadcrumbData.set({
        workerNo: data.workerNo,
        firstNameTH: data.firstNameTH,
        lastNameTH: data.lastNameTH,
      });
    });
}

get isThai(): boolean {
  return !this.data()?.nationality || this.data()?.nationality?.key === 'Thai';
}
```

Template update:
```html
<!-- BEFORE -->
@if (breadcrumbData) {
  <h2>&nbsp;> {{breadcrumbData.workerNo}}</h2>
}
<shared-worker-info-form [data]="data"></shared-worker-info-form>

<!-- AFTER -->
@if (breadcrumbData()) {
  <h2>&nbsp;> {{breadcrumbData().workerNo}}</h2>
}
<shared-worker-info-form [data]="data()"></shared-worker-info-form>
```

## Example 4: Multiple Array Signals (QR Code)

```typescript
// BEFORE
data: WorkerPrintingDTO;
rows: Row[] = [];
specialRows: Row[] = [];

getData() {
  this.apiService.getData$().subscribe(data => {
    this.data = data;
    this.setRows();
    this.setSpecialRows();
  });
}

setRows() {
  this.rows = [
    { name: 'Position', value: this.data.position },
    // ...
  ];
}

// AFTER
data = signal<WorkerPrintingDTO>(null as any);
rows = signal<Row[]>([]);
specialRows = signal<Row[]>([]);

getData() {
  this.apiService.getData$().subscribe(data => {
    this.data.set(data);
    this.setRows(data);          // Pass data directly
    this.setSpecialRows(data);   // Don't read signal in same tick
  });
}

setRows(data: WorkerPrintingDTO) {
  const newRows = [
    { name: 'Position', value: data.position },
    // ...
  ];
  this.rows.set(newRows); // Set whole array at once
}
```

## Example 5: App State (Penalty-Inform)

```typescript
// BEFORE
state: State;
hasNewVersion = false;

ngOnInit() {
  this.authService.isLoggedIn$().subscribe(isLoggedIn => {
    this.state = isLoggedIn ? State.Home : State.Login;
  });
  this.versionService.hasNewVersion$.subscribe(() => {
    this.hasNewVersion = true;
  });
}

// AFTER
state = signal<State>(undefined as any);
hasNewVersion = signal(false);

ngOnInit() {
  this.authService.isLoggedIn$().subscribe(isLoggedIn => {
    this.state.set(isLoggedIn ? State.Home : State.Login);
  });
  this.versionService.hasNewVersion$.subscribe(() => {
    this.hasNewVersion.set(true);
  });
}
```

## Example 6: Auth Login (Multiple Related Signals)

```typescript
// BEFORE
returnUrl: string;
isNeedReload: boolean;

// In subscribe chain:
this.returnUrl = result;
this.isNeedReload = this.locale.setLocaleByRole();

// In enterSite():
if (this.isNeedReload) {
  if (this.returnUrl) { window.location.href = basePath + this.returnUrl.slice(1); }
}

// AFTER
returnUrl = signal('');
isNeedReload = signal(false);

// In subscribe chain:
this.returnUrl.set(result);
this.isNeedReload.set(this.locale.setLocaleByRole());

// In enterSite():
if (this.isNeedReload()) {
  if (this.returnUrl()) { window.location.href = basePath + this.returnUrl().slice(1); }
}
```

## Common Bug: Overwriting Signal Reference

```typescript
// BUG — overwrites the signal function itself!
this.selectedWorkflowStatus = workflowStatus;
// After this, selectedWorkflowStatus is a plain value, not a signal
// Template reads selectedWorkflowStatus() will throw TypeError

// CORRECT
this.selectedWorkflowStatus.set(workflowStatus);
```

**This compiles without TypeScript errors** because `WritableSignal<T>` can be assigned `T` (the variable type is inferred as the broader union). Always use `.set()`.

## Properties That DON'T Need Conversion

- Properties used with `[(ngModel)]` on nested fields (e.g., `[(ngModel)]="data.firstName"`)
- Properties only assigned synchronously in click handlers
- Properties only assigned in `ngOnInit` from route snapshots
- Properties managed by `BaseFeatureGridComponent` via setter+markForCheck (`rows`, `page` in some patterns)
