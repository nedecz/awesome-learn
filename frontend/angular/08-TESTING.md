# Angular Testing

## Table of Contents

1. [Overview](#overview)
2. [Testing Architecture](#testing-architecture)
3. [TestBed and Component Testing](#testbed-and-component-testing)
4. [Testing Components with Inputs and Outputs](#testing-components-with-inputs-and-outputs)
5. [Testing Services](#testing-services)
6. [Testing Pipes and Directives](#testing-pipes-and-directives)
7. [Testing with HttpClientTestingModule](#testing-with-httpclienttestingmodule)
8. [Testing Observables and Async Code](#testing-observables-and-async-code)
9. [Integration Testing](#integration-testing)
10. [E2E Testing](#e2e-testing)
11. [Component Harnesses](#component-harnesses)
12. [Code Coverage and Testing Strategies](#code-coverage-and-testing-strategies)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

Testing is a core part of Angular development. Angular provides a comprehensive testing infrastructure through TestBed, which configures a testing module that mimics an Angular module. This document covers unit testing with Jasmine/Jest, component and service testing, async testing patterns, E2E testing, and strategies for effective test coverage.

### Scope

- Testing architecture: Jasmine/Karma vs Jest + Angular Testing Library
- TestBed configuration and component testing
- Testing inputs, outputs, and component interactions
- Testing services with mocked dependencies
- Testing pipes and directives
- HTTP testing with `HttpTestingController`
- Async testing: `fakeAsync`, `waitForAsync`, `done`
- Integration testing for component trees
- E2E testing with Cypress or Playwright
- Component harnesses from Angular CDK
- Coverage strategies and testing best practices

---

## Testing Architecture

### Framework Options

| Framework | Type | Default | Notes |
|-----------|------|---------|-------|
| **Jasmine + Karma** | Unit/integration | Yes (legacy default) | Browser-based test runner |
| **Jest** | Unit/integration | No (opt-in since Angular 16) | Faster, Node-based, better DX |
| **Angular Testing Library** | Component testing | No (recommended add-on) | Tests from user perspective |
| **Cypress** | E2E + component | No | Real browser, great DX |
| **Playwright** | E2E | No (experimental support) | Cross-browser, fast |

### Recommended Modern Stack

```
Unit tests         → Jest + Angular Testing Library
Component tests    → Jest + Angular Testing Library
Integration tests  → Jest + TestBed
E2E tests          → Playwright or Cypress
```

### File Naming Convention

```
user.component.ts           → Component
user.component.spec.ts      → Unit/integration tests (co-located)
user.component.e2e.ts       → E2E test (separate directory)
```

---

## TestBed and Component Testing

### Basic Component Test

```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { GreetingComponent } from './greeting.component';

describe('GreetingComponent', () => {
  let component: GreetingComponent;
  let fixture: ComponentFixture<GreetingComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [GreetingComponent],  // Standalone components go in imports
    }).compileComponents();

    fixture = TestBed.createComponent(GreetingComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();  // Trigger initial data binding
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should display the greeting', () => {
    const compiled: HTMLElement = fixture.nativeElement;
    expect(compiled.querySelector('h1')?.textContent).toContain('Hello');
  });
});
```

### Testing with Dependencies

```typescript
describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let mockUserService: jasmine.SpyObj<UserService>;

  beforeEach(async () => {
    mockUserService = jasmine.createSpyObj('UserService', ['getAll']);
    mockUserService.getAll.and.returnValue(of([
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ]));

    await TestBed.configureTestingModule({
      imports: [UserListComponent],
      providers: [
        { provide: UserService, useValue: mockUserService },
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should display users', () => {
    const items = fixture.nativeElement.querySelectorAll('.user-item');
    expect(items.length).toBe(2);
    expect(items[0].textContent).toContain('Alice');
  });

  it('should call getAll on init', () => {
    expect(mockUserService.getAll).toHaveBeenCalledTimes(1);
  });
});
```

---

## Testing Components with Inputs and Outputs

### Testing Signal Inputs

```typescript
describe('UserBadgeComponent', () => {
  it('should display the user name', () => {
    const fixture = TestBed.createComponent(UserBadgeComponent);
    fixture.componentRef.setInput('name', 'Alice');
    fixture.componentRef.setInput('variant', 'primary');
    fixture.detectChanges();

    const badge = fixture.nativeElement.querySelector('.badge');
    expect(badge.textContent.trim()).toBe('Alice');
    expect(badge.classList).toContain('primary');
  });
});
```

### Testing Outputs

```typescript
describe('SearchBoxComponent', () => {
  it('should emit searchChange when user types', () => {
    const fixture = TestBed.createComponent(SearchBoxComponent);
    fixture.detectChanges();

    let emittedValue = '';
    fixture.componentInstance.searchChange.subscribe((value: string) => {
      emittedValue = value;
    });

    const input: HTMLInputElement = fixture.nativeElement.querySelector('input');
    input.value = 'angular';
    input.dispatchEvent(new Event('input'));
    fixture.detectChanges();

    expect(emittedValue).toBe('angular');
  });
});
```

### Testing with Angular Testing Library

```typescript
import { render, screen, fireEvent } from '@testing-library/angular';

describe('SearchBoxComponent', () => {
  it('should emit on input', async () => {
    const searchChangeSpy = jest.fn();
    await render(SearchBoxComponent, {
      componentOutputs: { searchChange: { emit: searchChangeSpy } as any },
    });

    const input = screen.getByPlaceholderText('Search...');
    fireEvent.input(input, { target: { value: 'angular' } });

    expect(searchChangeSpy).toHaveBeenCalledWith('angular');
  });
});
```

---

## Testing Services

### Simple Service Test

```typescript
describe('NotificationService', () => {
  let service: NotificationService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(NotificationService);
  });

  it('should start with no notifications', () => {
    expect(service.notifications()).toEqual([]);
  });

  it('should add a notification', () => {
    service.add('Test message', 'info');
    expect(service.notifications().length).toBe(1);
    expect(service.notifications()[0].message).toBe('Test message');
    expect(service.notifications()[0].type).toBe('info');
  });

  it('should dismiss a notification by id', () => {
    service.add('Message 1');
    service.add('Message 2');
    const idToRemove = service.notifications()[0].id;

    service.dismiss(idToRemove);

    expect(service.notifications().length).toBe(1);
    expect(service.notifications()[0].message).toBe('Message 2');
  });
});
```

### Service with HTTP Dependency

```typescript
describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });
    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should fetch users', () => {
    const mockUsers: User[] = [{ id: '1', name: 'Alice' }];

    service.getAll().subscribe(users => {
      expect(users).toEqual(mockUsers);
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
});
```

---

## Testing Pipes and Directives

### Testing a Pipe

```typescript
import { TruncatePipe } from './truncate.pipe';

describe('TruncatePipe', () => {
  const pipe = new TruncatePipe();

  it('should truncate strings longer than the limit', () => {
    expect(pipe.transform('Hello, World!', 5)).toBe('Hello...');
  });

  it('should not truncate short strings', () => {
    expect(pipe.transform('Hi', 5)).toBe('Hi');
  });

  it('should handle empty strings', () => {
    expect(pipe.transform('', 5)).toBe('');
  });
});
```

### Testing a Directive

```typescript
@Component({
  template: `<div appHighlight [color]="'yellow'">Highlighted</div>`,
  imports: [HighlightDirective],
})
class TestHostComponent {}

describe('HighlightDirective', () => {
  it('should apply background color', () => {
    const fixture = TestBed.configureTestingModule({
      imports: [TestHostComponent],
    }).createComponent(TestHostComponent);

    fixture.detectChanges();

    const div: HTMLElement = fixture.nativeElement.querySelector('div');
    expect(div.style.backgroundColor).toBe('yellow');
  });
});
```

---

## Testing with HttpClientTestingModule

### Testing Request Method and URL

```typescript
it('should make a DELETE request', () => {
  service.delete('123').subscribe();

  const req = httpMock.expectOne('/api/users/123');
  expect(req.request.method).toBe('DELETE');
  req.flush(null);
});
```

### Testing Request Body

```typescript
it('should send correct body on POST', () => {
  const dto: CreateUserDto = { name: 'Carol', email: 'carol@test.com' };
  service.create(dto).subscribe();

  const req = httpMock.expectOne('/api/users');
  expect(req.request.body).toEqual(dto);
  req.flush({ id: '3', ...dto });
});
```

### Testing Query Parameters

```typescript
it('should include query params in search', () => {
  service.search('angular', 2).subscribe();

  const req = httpMock.expectOne(r =>
    r.url === '/api/users' &&
    r.params.get('q') === 'angular' &&
    r.params.get('page') === '2'
  );
  req.flush({ data: [], total: 0 });
});
```

### Simulating Errors

```typescript
it('should handle server error', () => {
  service.getAll().subscribe({
    next: () => fail('Expected an error'),
    error: (err) => expect(err.status).toBe(500),
  });

  const req = httpMock.expectOne('/api/users');
  req.flush('Server Error', { status: 500, statusText: 'Internal Server Error' });
});
```

---

## Testing Observables and Async Code

### fakeAsync + tick

```typescript
it('should debounce search input', fakeAsync(() => {
  component.onSearch({ target: { value: 'ang' } } as any);
  tick(100);  // Not enough time — debounce is 300ms
  expect(searchService.search).not.toHaveBeenCalled();

  tick(200);  // Total 300ms — debounce fires
  expect(searchService.search).toHaveBeenCalledWith('ang');
}));
```

### waitForAsync

```typescript
it('should load data asynchronously', waitForAsync(() => {
  fixture.detectChanges();

  fixture.whenStable().then(() => {
    fixture.detectChanges();
    const items = fixture.nativeElement.querySelectorAll('.item');
    expect(items.length).toBeGreaterThan(0);
  });
}));
```

### Testing Signals

```typescript
it('should update computed signal', () => {
  const fixture = TestBed.createComponent(CartSummaryComponent);
  const cartService = TestBed.inject(CartService);

  cartService.addItem({ id: '1', name: 'Widget', price: 9.99 });
  fixture.detectChanges();

  expect(fixture.nativeElement.textContent).toContain('1 items');
  expect(fixture.nativeElement.textContent).toContain('$9.99');
});
```

---

## Integration Testing

Integration tests verify that multiple components work together correctly.

```typescript
describe('UserPage Integration', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserPageComponent],  // Includes child components
      providers: [
        { provide: UserService, useValue: mockUserService },
      ],
    }).compileComponents();
  });

  it('should show user detail when a user is clicked', () => {
    const fixture = TestBed.createComponent(UserPageComponent);
    fixture.detectChanges();

    // Click the first user in the list
    const userRows = fixture.nativeElement.querySelectorAll('.user-row');
    userRows[0].click();
    fixture.detectChanges();

    // Detail panel should appear
    const detail = fixture.nativeElement.querySelector('app-user-detail');
    expect(detail).toBeTruthy();
    expect(detail.textContent).toContain('Alice');
  });
});
```

### Testing with Router

```typescript
describe('Navigation', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [AppComponent],
      providers: [
        provideRouter([
          { path: '', component: HomeComponent },
          { path: 'about', component: AboutComponent },
        ]),
      ],
    }).compileComponents();
  });

  it('should navigate to about page', fakeAsync(() => {
    const fixture = TestBed.createComponent(AppComponent);
    const router = TestBed.inject(Router);

    fixture.detectChanges();
    router.navigate(['/about']);
    tick();
    fixture.detectChanges();

    expect(fixture.nativeElement.textContent).toContain('About');
  }));
});
```

---

## E2E Testing

### Playwright (Recommended)

```typescript
// e2e/user-list.spec.ts
import { test, expect } from '@playwright/test';

test.describe('User List Page', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/users');
  });

  test('should display the user list', async ({ page }) => {
    await expect(page.locator('.user-item')).toHaveCount(10);
  });

  test('should search for users', async ({ page }) => {
    await page.fill('[data-testid="search-input"]', 'Alice');
    await expect(page.locator('.user-item')).toHaveCount(1);
    await expect(page.locator('.user-item')).toContainText('Alice');
  });

  test('should navigate to user detail', async ({ page }) => {
    await page.click('.user-item >> text=Alice');
    await expect(page).toHaveURL(/\/users\/\d+/);
    await expect(page.locator('h1')).toContainText('Alice');
  });
});
```

### Cypress

```typescript
// cypress/e2e/user-list.cy.ts
describe('User List Page', () => {
  beforeEach(() => {
    cy.visit('/users');
  });

  it('should display users', () => {
    cy.get('.user-item').should('have.length', 10);
  });

  it('should filter users by search', () => {
    cy.get('[data-testid="search-input"]').type('Alice');
    cy.get('.user-item').should('have.length', 1);
    cy.get('.user-item').should('contain', 'Alice');
  });
});
```

---

## Component Harnesses

Angular CDK provides **component harnesses** — a testing abstraction that shields tests from component implementation details.

### Using a Material Harness

```typescript
import { HarnessLoader } from '@angular/cdk/testing';
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { MatButtonHarness } from '@angular/material/button/testing';
import { MatInputHarness } from '@angular/material/input/testing';

describe('LoginComponent (Harness)', () => {
  let loader: HarnessLoader;

  beforeEach(async () => {
    const fixture = TestBed.createComponent(LoginComponent);
    loader = TestbedHarnessEnvironment.loader(fixture);
    fixture.detectChanges();
  });

  it('should submit the form with valid credentials', async () => {
    const emailInput = await loader.getHarness(MatInputHarness.with({ selector: '#email' }));
    const passwordInput = await loader.getHarness(MatInputHarness.with({ selector: '#password' }));
    const submitButton = await loader.getHarness(MatButtonHarness.with({ text: 'Log In' }));

    await emailInput.setValue('user@example.com');
    await passwordInput.setValue('password123');
    await submitButton.click();

    // Assert login was called
  });
});
```

### Creating a Custom Harness

```typescript
import { ComponentHarness } from '@angular/cdk/testing';

export class UserCardHarness extends ComponentHarness {
  static hostSelector = 'app-user-card';

  private nameEl = this.locatorFor('.user-name');
  private deleteButton = this.locatorFor('button.delete');

  async getName(): Promise<string> {
    return (await this.nameEl()).text();
  }

  async clickDelete(): Promise<void> {
    return (await this.deleteButton()).click();
  }
}
```

---

## Code Coverage and Testing Strategies

### Coverage Targets

| Metric | Target | Rationale |
|--------|--------|-----------|
| **Statements** | 80%+ | Covers most code paths |
| **Branches** | 75%+ | Ensures conditional logic is tested |
| **Functions** | 85%+ | Ensures all public APIs are tested |
| **Lines** | 80%+ | General coverage metric |

### What to Test (Priority Order)

| Priority | What | How |
|----------|------|-----|
| 🔴 High | Services with business logic | Unit tests with mocked dependencies |
| 🔴 High | Components with complex interactions | Component tests with TestBed |
| 🟠 Medium | Pipes with transformation logic | Pure unit tests (no TestBed needed) |
| 🟠 Medium | Guards and interceptors | Unit tests with mocked services |
| 🟡 Low | Simple presentational components | Snapshot or smoke tests |
| 🟡 Low | Models and interfaces | TypeScript compiler handles this |

### Testing Strategy

```
                        ┌─────────────┐
                        │   E2E Tests │  ← Few, slow, high confidence
                        │  (Playwright)│
                       ┌┴─────────────┴┐
                       │  Integration   │  ← Component trees, routing
                       │    Tests       │
                      ┌┴───────────────┴┐
                      │   Component      │  ← TestBed, harnesses
                      │     Tests        │
                     ┌┴─────────────────┴┐
                     │    Unit Tests      │  ← Fast, isolated, many
                     │  (Services, Pipes) │
                     └───────────────────┘
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-PERFORMANCE](09-PERFORMANCE.md) | Performance optimization strategies |
| 2 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Best practices including testing patterns |
| 3 | [07-HTTP-AND-API-INTEGRATION](07-HTTP-AND-API-INTEGRATION.md) | HTTP testing in depth |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Angular testing documentation |
