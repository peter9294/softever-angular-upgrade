# Karma/Jasmine Patterns for This Project

## Configuration

### Karma Setup (karma.conf.js)

- Browser: Chrome (not headless by default)
- Single Run: `false` (watch mode)
- Coverage: Istanbul with HTML + lcovonly + text-summary
- Timeouts: 60000ms for disconnect and no-activity
- Reporters: progress + kjhtml

### Running Tests

```bash
# Main app
yarn test
ng test --watch=false          # Single run

# Libraries
ng test @base
ng test @lib

# With coverage
ng test --code-coverage

# Specific test file
ng test --include="**/auth.service.spec.ts"
```

### tsconfig.spec.json

Tests use separate TypeScript configuration. Key settings:
- `compilerOptions.types: ["jasmine"]`
- `include: ["src/**/*.spec.ts", "src/**/*.d.ts"]`

## Jasmine Patterns

### Spy Creation

```typescript
// Create a standalone spy
const spy = jasmine.createSpy('methodName');
spy.and.returnValue('result');
spy.and.callFake((arg) => arg + 1);
spy.and.throwError(new Error('fail'));

// Spy on existing object
spyOn(service, 'method').and.returnValue(of(data));

// Verify calls
expect(spy).toHaveBeenCalled();
expect(spy).toHaveBeenCalledWith('arg1', 'arg2');
expect(spy).toHaveBeenCalledTimes(2);
```

### Async Testing

```typescript
// fakeAsync + tick
it('should handle async', fakeAsync(() => {
  component.loadData();
  tick(500);  // Advance time
  fixture.detectChanges();
  expect(component.data()).toBeDefined();
}));

// waitForAsync
it('should handle promise', waitForAsync(() => {
  component.loadData().then(() => {
    expect(component.data()).toBeDefined();
  });
}));

// done callback
it('should handle observable', (done) => {
  service.getData().subscribe(data => {
    expect(data.length).toBe(3);
    done();
  });
});
```

### TestBed Configuration

```typescript
// Full module configuration
TestBed.configureTestingModule({
  declarations: [ComponentName],
  imports: [FormsModule, ReactiveFormsModule],
  schemas: [CUSTOM_ELEMENTS_SCHEMA],  // Ignore unknown elements
  providers: [
    { provide: ServiceA, useValue: mockServiceA },
    { provide: ServiceB, useClass: MockServiceB },
    { provide: ServiceC, useFactory: () => createMockServiceC() },
  ],
});

// Override component template (for isolated testing)
TestBed.overrideComponent(ComponentName, {
  set: { template: '<div>simplified</div>' },
});
```

### Component Fixture

```typescript
const fixture = TestBed.createComponent(ComponentName);
const component = fixture.componentInstance;
const element = fixture.nativeElement;
const debugElement = fixture.debugElement;

// Query elements
const button = debugElement.query(By.css('.submit-btn'));
const allItems = debugElement.queryAll(By.css('.list-item'));

// Trigger events
button.triggerEventHandler('click', null);
fixture.detectChanges();

// Read text content
expect(element.querySelector('.title').textContent).toContain('Expected');
```

### beforeEach Patterns

```typescript
// Async beforeEach (compile components)
beforeEach(async () => {
  await TestBed.configureTestingModule({ /* ... */ }).compileComponents();
});

// Sync beforeEach (create fixture)
beforeEach(() => {
  fixture = TestBed.createComponent(ComponentName);
  component = fixture.componentInstance;
  fixture.detectChanges();
});
```

## Service Testing

```typescript
describe('AuthService', () => {
  let service: AuthService;
  let storage: ReturnType<typeof createMockSoftStorageService>;

  beforeEach(() => {
    storage = createMockSoftStorageService();

    TestBed.configureTestingModule({
      providers: [
        AuthService,
        { provide: SoftStorageService, useValue: storage },
      ],
    });

    service = TestBed.inject(AuthService);
  });

  it('should store token on login', () => {
    service.setAuthorize('token-123');
    expect(storage.setItem).toHaveBeenCalledWith(
      jasmine.stringContaining('token'),
      'token-123'
    );
  });
});
```

## Pipe Testing

```typescript
describe('ReadablePipe', () => {
  let pipe: ReadablePipe;

  beforeEach(() => {
    pipe = new ReadablePipe();  // No TestBed needed for pure pipes
  });

  it('should transform camelCase to readable', () => {
    expect(pipe.transform('helloWorld')).toBe('Hello World');
  });
});
```

## Guard Testing

```typescript
describe('AuthGuard', () => {
  let guard: AuthGuard;
  let authService: ReturnType<typeof createMockAuthService>;
  let router: ReturnType<typeof createMockRouter>;

  beforeEach(() => {
    authService = createMockAuthService();
    router = createMockRouter();

    TestBed.configureTestingModule({
      providers: [
        AuthGuard,
        { provide: AuthService, useValue: authService },
        { provide: Router, useValue: router },
      ],
    });

    guard = TestBed.inject(AuthGuard);
  });

  it('should allow access when logged in with permission', () => {
    authService.isLoggedIn = true;
    const route = { data: { can: 'viewContractList' } } as any;
    expect(guard.canActivate(route, {} as any)).toBe(true);
  });

  it('should redirect to login when not logged in', () => {
    authService.isLoggedIn = false;
    guard.canActivate({} as any, {} as any);
    expect(router.navigate).toHaveBeenCalledWith(['/auth/login']);
  });
});
```
