# Softever Base Classes — Signal-Migrated Reference

Production-migrated base classes from a real Softever Angular 21 monorepo. Other Softever projects share this architecture with slight modifications.

## Architecture

```
projects/@base/src/lib/
├── base-dropdown/base-dropdown.ts   ← CVA + watchedInputs() pattern
├── base-input/base-input.ts         ← Simple CVA
├── base-radio/base-radio.ts         ← Radio CVA
├── base-radio/base-enum-radio.ts    ← Enum radio (no API call)
└── base-feature/
    ├── base-feature.ts              ← DEPRECATED (use DestroyRef)
    ├── base-feature-grid.ts         ← Legacy grid base (NOT migrated)
    ├── base-feature-form.ts         ← Legacy form base
    └── base-feature-form-child.ts   ← Legacy form child base
```

## BaseDropdown — Full Signal Migration

The most complex base class. Demonstrates:
- `input()` for 7 config inputs
- `output()` for loaded event
- `signal()` for internal state (optionList, value, disabled, loading)
- `effect()` with `untracked()` for watched inputs pattern
- `watchedInputs()` override pattern for cascading dropdowns

```typescript
import { OnInit, OnDestroy, Directive, HostBinding, signal, input, effect, untracked, InputSignal, output } from '@angular/core';
import { ControlValueAccessor } from '@angular/forms';
import { finalize } from 'rxjs/operators';
import { Observable } from 'rxjs';

@Directive()
export class BaseDropdown<T> implements OnInit, OnDestroy, ControlValueAccessor {

  @HostBinding('class') hostClass = 'base-control';

  // Signal inputs
  controlClass = input<string>();
  fullWidth = input(true);
  placeholder = input<string>();
  compareWith = input<(o1: any, o2: any) => boolean>();
  lang = input<'th' | 'en'>();
  autoReset = input(false);
  showAll = input<boolean | string>();

  // Signal output
  loaded = output<T[]>();

  // Internal state signals
  optionList = signal<T[]>([]);
  value = signal<T | undefined>(undefined);
  disabled = signal(false);
  loading = signal(false);
  isPrimitive = signal(false);

  // Non-signal config (set by child classes, not from templates)
  isOptionListPrimitive = false;
  idAttr = 'id';
  textAttr = 'name';
  thTextAttr = 'nameTH';
  enTextAttr = 'nameEN';

  _showAll: boolean;
  showAllText = $localize`:@@all:ทั้งหมด`;
  showAllValue = undefined;
  showPlaceHolder = true;

  private initialized = false;
  private getOptionListSub: any;
  _compareWith: (o1: any, o2: any) => boolean;

  compareFn = (o1: any, o2: any): boolean => {
    return o1 && o2 ? o1[this.idAttr] === o2[this.idAttr] : o1 === o2;
  }

  compareFnBothPrimitive = (o1: any, o2: any): boolean => {
    return o1 === o2;
  }

  constructor() {
    // Effect 1: React to showAll input changes
    effect(() => {
      const showAllValue = this.showAll();
      if (showAllValue === undefined) return;
      this._showAll = !!showAllValue;
      if (this._showAll && typeof showAllValue === 'string') {
        this.showAllText = showAllValue;
      }
      this.showPlaceHolder = !this._showAll || (this._showAll && this.showAllValue !== undefined);
    });

    // Effect 2: watchedInputs — KEY PATTERN
    // Tracks dependent signal inputs, reloads options when they change
    effect(() => {
      const signals = this.watchedInputs();
      signals.forEach(s => s()); // Track all signal dependencies

      if (!this.initialized) return;

      untracked(() => {
        if (this.autoReset()) {
          this.reset();
        }
        this.getOptionList();
      });
    });
  }

  // Override in child classes to declare dependent inputs
  protected watchedInputs(): InputSignal<any>[] {
    return [];
  }

  ngOnInit() {
    this.initialized = true;
    this.setDefaultCompareWith();
    this.getOptionList();
  }

  ngOnDestroy() {
    if (this.getOptionListSub) {
      this.getOptionListSub.unsubscribe();
    }
  }

  // ControlValueAccessor — uses signal for value/disabled
  writeValue(value: any): void {
    if (value === undefined || value === null) {
      this.value.set(value);
    } else if (this.isOptionListPrimitive === this.isPrimitive()) {
      this.value.set(value);
    } else if (this.isPrimitive()) {
      this.value.set({ [this.idAttr]: value } as any);
    } else {
      this.value.set(value[this.idAttr] as any);
    }
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled.set(isDisabled);
  }

  onSelect() {
    const val = this.value();
    if (this.isOptionListPrimitive === this.isPrimitive()) {
      this.onChange(val);
    } else if (this.isPrimitive()) {
      this.onChange(val ? val[this.idAttr] : val);
    } else {
      this.onChange({ [this.idAttr]: val });
    }
  }

  reset() {
    if (this._showAll) {
      this.value.set(this.showAllValue);
    } else {
      this.value.set(undefined);
    }
    window.setTimeout(() => { this.onSelect(); });
  }

  // Override in child class
  protected apiOptionListGet$(): Observable<T[]> {
    throw new Error('Method not implemented.');
  }

  protected getOptionList() {
    this.loading.set(true);
    this.getOptionListSub = this.apiOptionListGet$().pipe(
      finalize(() => { this.loading.set(false); }),
    ).subscribe(data => {
      const list = [...data];
      const langValue = this.getEffectiveLang();
      if (langValue) {
        list.forEach(m => {
          m[this.textAttr] = m[this[`${langValue}TextAttr`]];
        });
      }
      this.optionList.set(list);
      this.loaded.emit(list);
    });
  }

  // ... registerOnChange, registerOnTouched, trackOption, displayOption, etc.
}
```

### Child Dropdown Example (watchedInputs pattern):

```typescript
export class SharedDistrictDropdownComponent extends BaseDropdown<DistrictDTO> {
  province = input<ProvinceDTO>();

  protected override watchedInputs() {
    return [this.province]; // Reload districts when province changes
  }

  protected apiOptionListGet$() {
    const prov = this.province();
    if (!prov) return of([]);
    return this.districtsService.apiDistrictsGet$(prov.id);
  }
}
```

## BaseRadio — Signal CVA Pattern

Similar to BaseDropdown but for radio button groups.

```typescript
@Directive()
export class BaseRadio<T> implements OnInit, OnDestroy, ControlValueAccessor {
  @HostBinding('class') hostClass = 'base-control';

  controlClass = input<string>();
  isVertical = input(false);
  showAll = input<boolean | string>();
  isPrimitive = input(false);
  lang = input<'th' | 'en'>();

  optionList = signal<T[]>([]);
  value = signal<any>(undefined);
  disabled = signal(false);
  loading = signal(false);
  loaded = output<T[]>();

  // Key CVA methods using signals:
  writeValue(value: any): void {
    if (this.isPrimitive() || value === undefined || value === null) {
      this.value.set(value);
    } else {
      this.value.set(value[this.idAttr]);
    }
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled.set(isDisabled);
  }

  onSelect() {
    if (this.isPrimitive()) {
      this.onChange(this.value());
    } else {
      const list = this.optionList();
      if (list) {
        this.onChange(list.find(option => option[this.idAttr] === this.value()));
      }
    }
  }
}
```

## BaseInput — Simple CVA

Minimal signal usage — only `input()` for config, not for value (CVA handles value):

```typescript
@Directive()
export class BaseInput implements ControlValueAccessor, AfterViewInit {
  @HostBinding('class') hostClass = 'base-control';
  @ViewChild('input') input: ElementRef<any>;

  placeholder = input('');
  maxlength = input<number>();
  autofocus = input<boolean | ''>();

  // value/disabled stay as plain properties (CVA manages them)
  value: any;
  disabled = false;

  writeValue(value: any): void {
    this.value = value;
    if (this.cdr) { this.cdr.markForCheck(); }
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
    if (this.cdr) { this.cdr.markForCheck(); }
  }
}
```

## BaseFeatureGridComponent — Legacy Pattern (NOT Migrated)

**Important**: This class is NOT migrated to signals. It uses a `rows` setter with `cdr.markForCheck()`. This means child components that extend it DON'T need to convert `rows` to signals — the base class already handles OnPush CD for `rows`.

```typescript
@Directive()
export class BaseFeatureGridComponent<PropType, FilterType = any>
  extends BaseFeatureComponent implements OnInit {

  protected cdr = inject(ChangeDetectorRef);

  @ViewChild('grid', { static: true }) grid: GridComponent<PropType>;
  @ViewChild('modal') modal: SoftModalComponent;

  // KEY: rows uses setter with markForCheck — this is WHY
  // child components don't need to convert rows to signal()
  private _rows: PropType[];
  get rows(): PropType[] { return this._rows; }
  set rows(value: PropType[]) {
    this._rows = value;
    this.cdr.markForCheck(); // ← Handles OnPush for rows
  }

  columns: GridTableColumn<PropType>[];
  page = { page: 1, perPage: 10, totalCount: 0 };
  sort = { prop: undefined, sortBy: undefined, asc: undefined };
  filter = {} as FilterType;

  // Child components: these methods DON'T need signal conversion
  // because `rows` setter already calls markForCheck
  pageChange() { this.getData(); }
  sortChange() {
    this.page = { ...this.page, page: 1 }; // Immutable update
    this.getData();
  }
  filterChange(event?) {
    if (event) { this.filter = event; }
    this.page = { ...this.page, page: 1 };
    this.getData();
  }

  protected getData() { this.rows = []; }
}
```

### Impact on Migration Scope

When auditing subscribe callbacks (RULE 4), skip these properties in child components of BaseFeatureGridComponent:
- `this.rows = data` — Already handled by setter + markForCheck
- `this.page = { ...this.page, totalCount: x }` — Triggers CD via the rows setter in the same subscribe

**BUT** other properties in subscribe callbacks DO need conversion:
```typescript
// These need signal() because they're NOT covered by the base class:
this.chartData = data;         // → this.chartData.set(data)
this.workflowCount = data;     // → this.workflowCount.set(data)
this.breadcrumbData = {...};   // → this.breadcrumbData.set({...})
```
