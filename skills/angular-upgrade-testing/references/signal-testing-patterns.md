# Signal Testing Patterns

## Reading Signal Values in Tests

### input() signals

```typescript
// Set input via ComponentRef (the Angular way)
fixture.componentRef.setInput('disabled', true);
fixture.detectChanges();
expect(component.disabled()).toBe(true);

// Alternative: direct signal read without setInput
// (only for testing initial values)
expect(component.disabled()).toBe(false);  // default value
```

### signal() state

```typescript
// Read current value
expect(component.loading()).toBe(false);

// Set and verify
component.loading.set(true);
fixture.detectChanges();
expect(component.loading()).toBe(true);

// Update and verify
component.loading.update(v => !v);
expect(component.loading()).toBe(false);
```

### computed() signals

```typescript
// Set the source signal
fixture.componentRef.setInput('page', { page: 3, perPage: 10, totalCount: 100 });
fixture.detectChanges();

// Read computed values (derived from page)
expect(component.count()).toBe(100);     // page.totalCount
expect(component.offset()).toBe(2);      // page.page - 1
expect(component.limit()).toBe(10);      // page.perPage
```

### model() signals

```typescript
// Read initial value
expect(component.page()).toEqual({ page: 1, perPage: 10, totalCount: 0 });

// Set new value
component.page.set({ page: 2, perPage: 10, totalCount: 50 });
fixture.detectChanges();

// Verify the model updated
expect(component.page()).toEqual({ page: 2, perPage: 10, totalCount: 50 });

// Verify computed values updated
expect(component.offset()).toBe(1);
```

### output() signals

```typescript
// Subscribe to output before triggering
let emittedValue: Page | undefined;
component.pageChange.subscribe((page: Page) => {
  emittedValue = page;
});

// Trigger the output
component.onPageChange(3);

// Verify emission
expect(emittedValue).toEqual({ page: 3, perPage: 10, totalCount: 50 });
```

## Testing effect() Side Effects

```typescript
it('should reload options when province changes', () => {
  const apiSpy = spyOn(component, 'getOptionList');

  // Mark as initialized (effects skip first run)
  (component as any).initialized = true;

  // Change the watched input
  fixture.componentRef.setInput('province', { id: 1, name: 'Bangkok' });
  fixture.detectChanges();

  // Effect should have triggered
  expect(apiSpy).toHaveBeenCalled();
});
```

## Testing Components with OnPush

```typescript
// OnPush components need explicit detectChanges
it('should show loading spinner', () => {
  component.loading.set(true);
  fixture.detectChanges();  // REQUIRED for OnPush

  const spinner = fixture.debugElement.query(By.css('.spinner'));
  expect(spinner).toBeTruthy();
});

// After async operations
it('should display data after load', fakeAsync(() => {
  const mockData = [{ id: 1, name: 'Test' }];
  apiService.getData.and.returnValue(of(mockData));

  component.loadData();
  tick();
  fixture.detectChanges();  // REQUIRED for OnPush

  const rows = fixture.debugElement.queryAll(By.css('.data-row'));
  expect(rows.length).toBe(1);
}));
```

## Testing Signal-Based Services

```typescript
describe('NotificationsService', () => {
  let service: NotificationsService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [NotificationsService],
    });
    service = TestBed.inject(NotificationsService);
  });

  it('should update menuStat signal', () => {
    expect(service.menuStat()).toBeNull();

    const mockStat = { pendingCount: 5 } as UserMenuDTO;
    service.menuStat.set(mockStat);

    expect(service.menuStat()).toEqual(mockStat);
    expect(service.menuStat()!.pendingCount).toBe(5);
  });
});
```

## Testing Base Component Signals

```typescript
describe('BaseDropdown', () => {
  it('should have signal properties', () => {
    const dropdown = runInInjectionContext(
      TestBed.inject(EnvironmentInjector),
      () => new TestDropdown()
    );

    // Test signal defaults
    expect(dropdown.disabled()).toBe(false);
    expect(dropdown.loading()).toBe(false);
    expect(dropdown.autoReset()).toBe(false);

    // Test signal updates
    dropdown.disabled.set(true);
    expect(dropdown.disabled()).toBe(true);
  });
});
```

## Testing Navbar with Signal Menus

```typescript
describe('NavbarComponent', () => {
  it('should compute filtered menus from signal', () => {
    const authService = createMockAuthService({
      can: createFullCan(true),
    });

    // ... setup TestBed

    fixture.detectChanges();

    // Read computed signal
    const menus = component.menus();
    expect(menus.length).toBeGreaterThan(0);
    expect(menus[0].label).toBeDefined();
  });
});
```

## Testing ViewChild and Signal Queries

### Static ViewChild — Grid Template Refs

`@ViewChild({ static: true })` refs are real after `detectChanges()`. Test helpers that mock these get overwritten:

```typescript
// mockViewChildReferences creates callable spy functions for grid template refs
mockViewChildReferences(component);
// component.grid.switchTpl = jasmine.createSpy(...)

fixture.detectChanges();
// NOW: component.grid's @ViewChild({ static: true }) refs are REAL TemplateRefs
// The mocks were overwritten. Tests should verify against real template refs.

// Test actual column configuration:
const statusCol = component.columns.find(c => c.prop === 'status');
expect(statusCol?.cellTemplate).toBe(component.registerStatusTpl);
```

### Non-Static ViewChild — Modal Mocking

Modals with `@ViewChild` (non-static) need mocking before the component tries to use them:

```typescript
beforeEach(async () => {
  const testBed = await createTestBedFor(DocumentCheckComponent);
  fixture = testBed.fixture;
  component = testBed.component;

  // Mock modals BEFORE detectChanges
  component.approveModal = jasmine.createSpyObj('SoftModalComponent', ['open', 'close']);
  component.resultModal = jasmine.createSpyObj('SoftModalComponent', ['open', 'close']);

  fixture.detectChanges();
});

it('should open approve modal on approve action', () => {
  component.onApprove(mockRow);
  expect(component.approveModal.open).toHaveBeenCalled();
});
```

### E-Form Signal Input Testing

E-form base classes use `input()` signals. Tests must use `setInput()`:

```typescript
// Setup
const mockData = createMockEFormDTO();
const mockForm = new NgForm([], []);

fixture.componentRef.setInput('form', mockForm);
fixture.componentRef.setInput('data', mockData);
fixture.detectChanges();

// Read signal values
expect(component.data()).toBeDefined();
expect(component.data().truckType).toBe('expected');

// Access nested data via signal
expect(component.data().compartments.length).toBeGreaterThan(0);

// Clear input
fixture.componentRef.setInput('data', undefined);
expect(component.data()).toBeUndefined();
```

## Common Mistakes in Signal Testing

### Mistake 1: Not calling ()

```typescript
// WRONG — tests the signal function reference
expect(component.disabled).toBeFalsy();  // Functions are truthy!

// RIGHT — tests the signal value
expect(component.disabled()).toBe(false);
```

### Mistake 2: Setting input directly

```typescript
// WRONG — bypasses signal input mechanism
component.disabled = true;

// RIGHT — use setInput
fixture.componentRef.setInput('disabled', true);
fixture.detectChanges();
```

### Mistake 3: Missing detectChanges

```typescript
// WRONG — template not updated
component.data.set(newData);
expect(fixture.nativeElement.textContent).toContain('new');

// RIGHT — trigger CD first
component.data.set(newData);
fixture.detectChanges();
expect(fixture.nativeElement.textContent).toContain('new');
```

### Mistake 4: Mocking viewChild.required properties

```typescript
// WRONG — viewChild.required creates a getter, can't be overwritten
// component.modal = jasmine.createSpyObj(...);  // TypeError!

// RIGHT — use @ViewChild for mockable properties
// In the component: @ViewChild('modal') modal: SoftModalComponent;
// In the test:
component.modal = jasmine.createSpyObj('modal', ['open', 'close']);
```

### Mistake 5: Assigning to signal inputs directly

```typescript
// WRONG — signal inputs are readonly
component.data = mockData;  // TS2540: Cannot assign to 'data' because it is a read-only property

// RIGHT — use setInput
fixture.componentRef.setInput('data', mockData);
```

### Mistake 6: Mutating signal input value via component reference

```typescript
// WRONG — component.data() returns the signal value, mutating it doesn't re-trigger
component.data().truckType = 'newType';  // Mutation, no signal notification

// RIGHT — use a separate mock variable and re-set input
const mockData = createMockEFormDTO();
fixture.componentRef.setInput('data', mockData);
// Later, to change a property:
mockData.truckType = 'newType';
fixture.componentRef.setInput('data', { ...mockData });  // New reference triggers signal
```

### Mistake 7: expect() inside if

```typescript
// WRONG — silently passes if data is null
it('should have name', () => {
  if (component.data()) {
    expect(component.data()!.name).toBe('Test');
  }
});

// RIGHT — explicit assertions
it('should have name', () => {
  expect(component.data()).toBeDefined();
  expect(component.data()!.name).toBe('Test');
});
```
