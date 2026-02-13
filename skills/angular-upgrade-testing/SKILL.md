---
name: Angular Upgrade Testing
description: Use this skill when the user asks to "write upgrade tests", "create baseline tests", "test signals", "test OnPush", "verify migration", or when writing tests during an Angular upgrade. Covers baseline test suite creation, mock factory patterns, signal-aware testing, and the project's Karma/Jasmine setup. Also activates when discussing test verification for upgrade correctness.
version: 1.0.0
---

# Angular Upgrade Testing

## Purpose

Guide the creation of tests that verify Angular upgrade correctness. Focus on behavioral tests that catch real migration bugs, not trivial "does it render" tests.

## Testing Framework

This project uses **Karma + Jasmine** (NOT Jest).

```bash
yarn test              # Run main app tests
ng test @base          # Run @base library tests
ng test @lib           # Run @lib library tests
```

**Note:** `angular.json` has `skipTests: true` for code generation. Test files must be created manually.

## Baseline Test Suite

Before starting any migration, create a baseline test suite that captures current behavior. This was done in commit `d2df7a2` with 152 tests.

### What to Test

| Area | Priority | What to Verify |
|------|----------|---------------|
| Auth service | Critical | Login flows, role checks, token management |
| Route guards | Critical | Permission-based access control |
| Core services | High | Storage, locale, notifications |
| Layout components | High | Navbar menus, header signals, footer |
| Pipes | Medium | Data transformation correctness |
| Base components | Medium | Dropdown lifecycle, radio bindings |

### Test File Structure

```typescript
import { TestBed } from '@angular/core/testing';
import { CUSTOM_ELEMENTS_SCHEMA } from '@angular/core';

describe('ComponentName', () => {
  let component: ComponentName;
  let fixture: ComponentFixture<ComponentName>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ComponentName],
      schemas: [CUSTOM_ELEMENTS_SCHEMA],
      providers: [
        { provide: AuthService, useValue: createMockAuthService() },
        { provide: Router, useValue: createMockRouter() },
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(ComponentName);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should verify behavioral contract', () => {
    // Test actual behavior, not "renders without crashing"
  });
});
```

## Mock Factory Pattern

Use the established mock factories from `src/testing/test-helpers.ts`:

```typescript
import {
  createMockAuthService,
  createMockRouter,
  createMockActivatedRoute,
  createMockSoftStorageService,
  createMockOAuthService,
  createMockSoftPopupService,
  createMockLocaleService,
  createMockNotificationsService,
} from '../testing/test-helpers';
```

### Customizing Mocks

```typescript
// Override specific properties
const authService = createMockAuthService({
  isAdmin: true,
  displayName: 'Admin User',
});

// Override with spies
const storage = createMockSoftStorageService();
storage.getItem.and.returnValue('cached-value');
```

### API Service Mocks

From `src/testing/mock-api-services.ts`:

```typescript
import {
  createMockUsersService,
  createMockAdminsService,
  createMockHomesService,
} from '../testing/mock-api-services';
```

## Signal-Aware Testing

### CRITICAL: Read Signals with ()

```typescript
// WRONG — testing the signal function itself, always truthy
expect(component.disabled).toBeTruthy();

// RIGHT — read the signal value
expect(component.disabled()).toBe(false);
```

### Testing input() Signals

```typescript
// Use ComponentRef.setInput() for signal inputs
fixture.componentRef.setInput('disabled', true);
fixture.detectChanges();
expect(component.disabled()).toBe(true);
```

### Testing model() Signals

```typescript
// Read model value
expect(component.page()).toEqual({ page: 1, perPage: 10, totalCount: 0 });

// Update model
component.page.set({ page: 2, perPage: 10, totalCount: 50 });
fixture.detectChanges();
expect(component.page()).toEqual({ page: 2, perPage: 10, totalCount: 50 });
```

### Testing computed() Signals

```typescript
// Set the source signal, then check computed
fixture.componentRef.setInput('page', { page: 3, perPage: 10, totalCount: 100 });
fixture.detectChanges();
expect(component.offset()).toBe(2);  // (page - 1)
expect(component.count()).toBe(100);
```

### Testing effect() Side Effects

```typescript
// Effects run after change detection
fixture.componentRef.setInput('province', mockProvince);
fixture.detectChanges();
// The effect should have triggered the API call
expect(apiService.loadDistricts).toHaveBeenCalledWith(mockProvince.id);
```

### Testing output() Signals

```typescript
// Subscribe to output
let emittedValue: any;
component.save.subscribe((v: any) => emittedValue = v);

// Trigger the emit
component.onSave();

expect(emittedValue).toEqual(expectedData);
```

## ViewChild and Signal Query Testing

### @ViewChild({ static: true }) — Grid Template Refs

Static ViewChild refs resolve before `detectChanges()`. Tests using `mockViewChildReferences()` create signal-style mocks, but `detectChanges()` overwrites them with real template refs from the DOM.

```typescript
// In test helpers — mock creates callable spy functions
mockViewChildReferences(component) {
  component.grid = {
    switchTpl: jasmine.createSpy('switchTpl').and.returnValue({}),
    nameTpl: jasmine.createSpy('nameTpl').and.returnValue({}),
    // ...
  };
}

// After detectChanges(), grid's @ViewChild({ static: true }) refs
// get their REAL values from the template, overwriting the mocks.
// This is correct behavior — tests verify real template binding.
```

### @ViewChild (non-static) — Modal Mocking

Non-static ViewChild refs are `undefined` before `detectChanges()`. Mock them AFTER creating the component but BEFORE calling code that uses them:

```typescript
beforeEach(async () => {
  const testBed = await createTestBedFor(MyComponent);
  fixture = testBed.fixture;
  component = testBed.component;

  // Mock BEFORE detectChanges (modals aren't in template with CUSTOM_ELEMENTS_SCHEMA)
  component.approveModal = jasmine.createSpyObj('SoftModalComponent', ['open', 'close']);
  component.resultModal = jasmine.createSpyObj('SoftModalComponent', ['open', 'close']);

  fixture.detectChanges();
});

it('should open approve modal', () => {
  component.approve();
  expect(component.approveModal.open).toHaveBeenCalled();
});
```

### viewChild.required — NOT Testable

`viewChild.required` creates getter-backed properties that tests cannot mock:

```typescript
// Component with viewChild.required:
private readonly _modal = viewChild.required<SoftModalComponent>('modal');
get modal() { return this._modal(); }

// Test FAILS:
component.modal = jasmine.createSpyObj(...);
// TypeError: Cannot set property which has only a getter

// Solution: Use plain @ViewChild instead of viewChild.required
// for properties that tests need to mock
```

### MockDropdownComponent — @Input() Required

Mock components used in tests with `fixture.componentRef.setInput()` MUST have `@Input()` decorators:

```typescript
// WRONG — setInput() throws NG0303
@Component({ selector: 'shared-dropdown', template: '' })
class MockDropdownComponent {
  optionList: any[] = [];  // Plain property — NOT an Angular input
}

// RIGHT — setInput() works
@Component({ selector: 'shared-dropdown', template: '' })
class MockDropdownComponent {
  @Input() optionList: any[] = [];  // Angular input
  @Input() value: any = null;
  @Input() disabled = false;
}
```

## CLAUDE.md Rule: No expect() Inside if Statements

```typescript
// WRONG — if condition fails, test silently passes
it('should have data', () => {
  if (component.data()) {
    expect(component.data().name).toBe('expected');
  }
});

// RIGHT — explicit assertion
it('should have data', () => {
  const data = component.data();
  expect(data).toBeDefined();
  expect(data!.name).toBe('expected');
});
```

## Testing OnPush Components

OnPush components need explicit change detection triggers:

```typescript
// For OnPush components, always call detectChanges after state changes
it('should update template after signal change', () => {
  component.loading.set(true);
  fixture.detectChanges();

  const spinner = fixture.debugElement.query(By.css('.spinner'));
  expect(spinner).toBeTruthy();
});
```

## Testing with runInInjectionContext

For services or base classes that require injection context:

```typescript
import { runInInjectionContext } from '@angular/core';

it('should create instance with DI', () => {
  const instance = runInInjectionContext(TestBed.inject(EnvironmentInjector), () => {
    return new MyBaseClass();
  });
  expect(instance).toBeTruthy();
});
```

## Post-Migration Test Verification

After any migration phase, run the full test suite:

```bash
yarn test --watch=false    # Single run, no watch
yarn build:dev             # Build verification
yarn lint                  # Lint check
```

### Signal Migration Test Checklist

After migrating a component to signals, verify:

- [ ] All `component.property` reads now use `component.property()`
- [ ] `fixture.componentRef.setInput()` used for signal inputs (not direct assignment)
- [ ] `fixture.detectChanges()` called after signal changes
- [ ] Output subscriptions use `.subscribe()` on `OutputRef`
- [ ] Computed values tested by changing source signals

## References

- `references/karma-jasmine-patterns.md` — Karma/Jasmine configuration and patterns
- `references/signal-testing-patterns.md` — Complete signal testing examples
