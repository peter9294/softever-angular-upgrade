# Softever Library Components — Signal-Migrated Reference

Production-migrated library components from `projects/@lib/`. These demonstrate advanced signal patterns for complex reusable components.

## GridComponent — Complex Signal Architecture

The core data table (wraps `@swimlane/ngx-datatable`). Demonstrates every signal pattern together:

```typescript
@Component({
  selector: 'lib-grid',
  changeDetection: ChangeDetectionStrategy.OnPush,
  standalone: false
})
export class GridComponent<PropType> implements OnInit, AfterViewInit {

  // === SIMPLE INPUTS ===
  gridClass = input('');
  scrollbarV = input(false);
  rowHeight = input<number | 'auto'>('auto');
  headerHeight = input(40);
  footerHeight = input(50);
  externalPaging = input(false);
  externalSorting = input(false);
  rowClass = input<(row: PropType) => any>();
  selectionType = input<SelectionType>();
  // ... more config inputs

  // === COMPLEX INPUTS (processed in effects) ===
  dataSub = input<Subscription | null>(null);
  rows = input<PropType[]>([]);
  columns = input<GridTableColumn<any>[]>([]);

  // === TWO-WAY BINDINGS (model) ===
  page = model<Page>({ page: 1, perPage: 10, totalCount: 0 });
  sort = model<Sort>({ sortBy: '', asc: true });
  filter = model<{ [key: string]: any }>({});
  selected = model<PropType[]>([]);

  // === COMPUTED DERIVED STATE ===
  count = computed(() => this.page()?.totalCount ?? 0);
  offset = computed(() => (this.page()?.page ?? 1) - 1);
  limit = computed(() => this.page()?.perPage ?? 10);
  sorts = computed(() => {
    const sortVal = this.sort();
    if (sortVal?.sortBy) {
      return [{ prop: sortVal.prop, dir: sortVal.asc ? 'asc' : 'desc' }];
    }
    return undefined;
  });

  // === INTERNAL STATE SIGNAL ===
  loading = signal(false);

  // === EFFECTS ===
  constructor() {
    let currentSubId = 0;

    // Effect 1: Loading spinner with debounce
    effect((onCleanup) => {
      const sub = this.dataSub();
      if (!sub) {
        this.loading.set(false);
        return;
      }
      const thisSubId = ++currentSubId;
      const timeout = window.setTimeout(() => {
        if (thisSubId === currentSubId) {
          this.loading.set(true);
        }
      }, 300);

      sub.add(() => {
        if (thisSubId === currentSubId) {
          this.loading.set(false);
        }
      });

      onCleanup(() => {
        window.clearTimeout(timeout);
        if (thisSubId === currentSubId) {
          this.loading.set(false);
        }
      });
    });

    // Effect 2: Rows processing
    effect(() => {
      const rows = this.rows();
      if (!rows || rows.length === 0) return;
      // Reset expansion state, apply transforms
    });

    // Effect 3: Column enrichment
    effect(() => {
      const columns = this.columns();
      if (!columns || columns.length === 0) return;
      // Assign default templates, process sortBy mappings
    });
  }

  // === OUTPUTS ===
  activate = output<any>();
  reorder = output<any>();
  // ... more outputs

  // === METHODS (use signal reads) ===
  onPage(event: { offset: number }) {
    const newPage = event.offset + 1;
    this.page.update(p => ({ ...p, page: newPage }));
  }

  onSort(event: { sorts: { prop: string, dir: string }[] }) {
    const sortEvent = event.sorts[0];
    const col = this.columns().find(c => c.prop === sortEvent.prop);
    this.sort.set({
      prop: sortEvent.prop,
      sortBy: col?.sortBy ?? sortEvent.prop,
      asc: sortEvent.dir === 'asc',
    });
  }
}
```

### Key Patterns from GridComponent:

1. **`model()` for two-way bindings** — page, sort, filter, selected
2. **`computed()` for derived values** — count, offset, limit, sorts
3. **`effect()` with `onCleanup`** — loading spinner with timeout cleanup
4. **Multiple effects in constructor** — each handles one concern
5. **`update()` for partial model updates** — `this.page.update(p => ({ ...p, page: newPage }))`

## TabComponent — model() Two-Way Pattern

```typescript
@Component({
  selector: 'lib-tab',
  changeDetection: ChangeDetectionStrategy.OnPush,
  standalone: false
})
export class TabComponent {
  tabs = input<{ id: string, icon?: string, name: string, disabled?: () => boolean, stat?: number }[]>();
  activeTabId = model<string>();
  width = input<string>();
  isInsideModal = input(false);
  parentTab = input<string>();

  readonly showTabs = computed(() => (this.tabs()?.length ?? 0) > 1);

  changeTab(tab: { id: string, name: string }) {
    this.activeTabId.set(tab.id); // model.set() notifies parent
    if (!this.isInsideModal()) {
      const parentTabVal = this.parentTab();
      const fragment = parentTabVal
        ? `${parentTabVal}#${tab.id}`
        : tab.id;
      this.router.navigate([], { replaceUrl: true, fragment });
    }
  }
}
```

**Usage in parent template:**
```html
<lib-tab
  [tabs]="tabs"
  [(activeTabId)]="activeTabId"
  (activeTabIdChange)="getData()"
></lib-tab>
```

## PaginationComponent — computed() Array Generation

```typescript
@Component({
  selector: 'lib-pagination',
  changeDetection: ChangeDetectionStrategy.OnPush,
  standalone: false
})
export class PaginationComponent {
  readonly page = input(1);
  readonly pageCount = input(0);
  readonly pageChange = output<number>();
  readonly maxSize = 7;

  // Derived page array — recomputes when page or pageCount changes
  readonly pages = computed(() => {
    return this.createPageArray(this.page(), this.pageCount(), this.maxSize);
  });

  setPage(pageItem: Pagination) {
    if (pageItem.label !== '...') {
      this.pageChange.emit(pageItem.value);
    }
  }

  private createPageArray(currentPage: number, totalPages: number, range: number): Pagination[] {
    // Ellipsis logic for [1 ... 4 5 6 ... 20] style pagination
    // ...
  }
}
```

## FileUploadButtonComponent — linkedSignal() Pattern

```typescript
@Component({
  selector: 'lib-file-upload-button',
  changeDetection: ChangeDetectionStrategy.OnPush,
  standalone: false
})
export class FileUploadButtonComponent extends BaseInput {
  @ViewChild('fileInput') fileInput: SoftFileModelDirective;

  buttonClass = input<string>();
  icon = input<string>();
  text = input('Browse');
  accept = input<string>();

  // Input from parent
  uploadSub = input<Subscription>();

  // linkedSignal: creates a writable signal derived from input
  // Resets when uploadSub changes, but can be modified locally
  loadingSub = linkedSignal(() => this.uploadSub());

  value: File;
  wasChanged = false;

  fileChange(value) {
    if (value) {
      this.wasChanged = true;
      this.onValueChange(value);
    }
  }
}
```

## LibSkeletonDirective — Structural Directive with Signals

Shows how to use signals in a structural directive (replaces content with skeleton loading):

```typescript
@Directive({
  selector: '[libSkeleton]',
  standalone: false
})
export class LibSkeletonDirective {
  libSkeleton = input<Subscription>();
  libSkeletonType = input<string>();
  libSkeletonTemplate = input<TemplateRef<any>>();
  libSkeletonContext = input<any>();

  private previousSub: Subscription | undefined;
  private isLoading = signal(false);
  private cdr = inject(ChangeDetectorRef);

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
  ) {
    effect(() => {
      const sub = this.libSkeleton();
      const skeletonType = this.libSkeletonType();
      const skeletonTemplate = this.libSkeletonTemplate();
      const skeletonContext = this.libSkeletonContext();
      const isLoading = this.isLoading();

      // Track new subscriptions
      if (sub && sub !== this.previousSub) {
        this.previousSub = sub;
        untracked(() => this.isLoading.set(true));
        sub.add(() => {
          this.isLoading.set(false);
          this.cdr.markForCheck();
        });
      }

      // Render skeleton or content
      this.viewContainer.clear();
      if (isLoading && sub) {
        if (skeletonType) {
          this.viewContainer.createComponent(this.builtInTypes[skeletonType]);
        } else {
          this.viewContainer.createComponent(LibDefaultSkeletonComponent);
        }
      } else if (sub && !isLoading) {
        this.viewContainer.createEmbeddedView(this.templateRef);
      }
    });
  }
}
```

**Usage:**
```html
<div *libSkeleton="dataSub">
  <!-- Content shown after dataSub completes -->
</div>
```

## DatePickerComponent — effect() for Config Sync

```typescript
@Component({
  selector: 'lib-date-picker',
  changeDetection: ChangeDetectionStrategy.OnPush,
  standalone: false
})
export class DatePickerComponent implements OnInit, ControlValueAccessor {
  format = input<string>();
  min = input<Date>();
  max = input<Date>();
  delayValidate = input(false);
  autoTime = input(true);
  placeholder = input('');
  monthOnly = input<boolean>();

  bsConfig: Partial<BsDatepickerConfig> = {};

  constructor(private cdr: ChangeDetectorRef) {
    // Sync format input to third-party config object
    effect(() => {
      const fmt = this.format();
      if (fmt) {
        this.bsConfig.dateInputFormat = fmt;
      }
    });
  }

  // Signal reads in methods
  minDate() {
    if (this.delayValidate() && !this.isOpen) return undefined;
    return this.min();
  }
}
```

## Pattern Summary

| Component | input() | output() | model() | computed() | effect() | signal() | linkedSignal() |
|-----------|---------|----------|---------|------------|----------|----------|----------------|
| GridComponent | 17 | 10 | 4 | 4 | 3 | 1 | - |
| TabComponent | 4 | - | 1 | 1 | - | - | - |
| PaginationComponent | 2 | 1 | - | 1 | - | - | - |
| FileUploadButton | 4 | - | - | - | - | - | 1 |
| LibSkeletonDirective | 4 | - | - | - | 1 | 1 | - |
| DatePickerComponent | 7 | - | - | - | 1 | - | - |
