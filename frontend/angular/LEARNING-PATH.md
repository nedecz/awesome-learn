# Angular Learning Path

A structured, self-paced training guide to mastering modern Angular development — from components and TypeScript foundations through signals, state management, testing, performance optimization, and production-ready patterns. Each phase builds on the previous one, progressing from core concepts to advanced production skills.

> **Time Estimate:** 10–14 weeks at ~5 hours/week. Adjust pace to your experience level. Developers with existing TypeScript and framework experience may accelerate Phases 1–2.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how Angular concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all seven phases and produces a portfolio artifact

---

## Phase 1: Angular Foundations (Weeks 1–2)

### Learning Objectives

- Set up an Angular development environment with the Angular CLI
- Understand Angular project structure and configuration files
- Create standalone components with templates and styles
- Use template syntax: interpolation, property binding, event binding
- Understand the new control flow syntax (@if, @for, @switch)
- Use the Angular CLI for code generation

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | What is Angular, history, architecture, prerequisites |
| 2 | [01-COMPONENTS-AND-TEMPLATES](01-COMPONENTS-AND-TEMPLATES.md) | Component architecture, template syntax, new control flow |

### Exercises

**1. Project Setup and First Component:**

Create a new Angular project and build a simple task list:

- Run `ng new task-manager` (accept defaults for standalone)
- Generate a `TaskListComponent` and a `TaskItemComponent`
- Display a list of tasks using `@for` with `track task.id`
- Add a button to toggle task completion using `(click)` and `@if`
- Style components with scoped CSS

**2. Template Syntax Practice:**

Build a user profile card component:

- Use `input()` to accept a `User` object from a parent
- Use interpolation `{{ }}` for displaying name and email
- Use property binding `[src]` for the avatar image
- Use event binding `(click)` for an edit button
- Use `@switch` to display different badges based on user role
- Use `@if` with `@else` for conditional content

**3. Component Composition:**

Build a dashboard with multiple components:

- Create `HeaderComponent`, `SidebarComponent`, `MainContentComponent`
- Use content projection (`ng-content`) in a `CardComponent`
- Create a `TabGroupComponent` with multi-slot projection
- Practice smart vs presentational component separation

### Knowledge Check

- [ ] Can you explain the difference between standalone components and NgModule-based components?
- [ ] Can you write a component with `@if`, `@for`, and `@switch` in the template?
- [ ] Can you use `input()` and `output()` for parent-child communication?
- [ ] Can you explain Angular's component lifecycle and name the most important hooks?

---

## Phase 2: Core Concepts (Weeks 3–4)

### Learning Objectives

- Create and inject services using Angular's DI system
- Use `inject()` function for dependency injection
- Configure the Angular Router with lazy-loaded routes
- Implement functional route guards
- Understand hierarchical injectors and provider scopes
- Use `providedIn: 'root'` for singleton services

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-SERVICES-AND-DEPENDENCY-INJECTION](02-SERVICES-AND-DEPENDENCY-INJECTION.md) | DI system, services, providers, injection tokens |
| 2 | [03-ROUTING-AND-NAVIGATION](03-ROUTING-AND-NAVIGATION.md) | Router setup, lazy loading, guards, resolvers |

### Exercises

**1. Service and DI:**

Build an authentication system:

- Create an `AuthService` with `providedIn: 'root'`
- Implement `login()`, `logout()`, and `isAuthenticated()` methods using signals
- Create an `InjectionToken` for app configuration
- Inject the auth service in a `LoginComponent` and `NavComponent`
- Test that both components share the same service instance

**2. Routing:**

Set up a multi-page application:

- Configure routes for Home, Dashboard, Settings, and Login pages
- Implement lazy loading with `loadComponent` for Dashboard
- Create nested routes under Settings (Profile, Security, Notifications)
- Implement a functional `authGuard` that redirects to `/login`
- Add `withComponentInputBinding()` and use `input()` for route params

**3. Route Guard and Resolver:**

Add guards and data pre-fetching:

- Create a `roleGuard` factory that accepts a role parameter
- Implement a `canDeactivate` guard for unsaved form changes
- Create a resolver that fetches user data before the profile page loads
- Handle resolver errors with a redirect to a 404 page

### Knowledge Check

- [ ] Can you explain the difference between `inject()` and constructor injection?
- [ ] Can you describe the hierarchical injector resolution process?
- [ ] Can you implement a functional route guard with `inject()`?
- [ ] Can you lazy-load a route with `loadComponent` and `loadChildren`?

---

## Phase 3: Forms & Data (Weeks 5–6)

### Learning Objectives

- Build reactive forms with typed `FormControl`, `FormGroup`, `FormArray`
- Implement built-in and custom validators (sync and async)
- Use `NonNullableFormBuilder` for typed forms
- Make HTTP requests with `HttpClient` and typed responses
- Implement functional HTTP interceptors
- Handle errors in HTTP responses

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-FORMS](04-FORMS.md) | Reactive forms, typed forms, validation, dynamic forms |
| 2 | [07-HTTP-AND-API-INTEGRATION](07-HTTP-AND-API-INTEGRATION.md) | HttpClient, interceptors, error handling |

### Exercises

**1. Registration Form:**

Build a user registration form:

- Use `NonNullableFormBuilder` with typed controls
- Fields: name, email, password, confirmPassword, role (select), skills (FormArray)
- Add validators: required, email, minLength, pattern
- Implement a cross-field validator for password confirmation
- Implement an async validator that checks if the email is already taken
- Display error messages using a reusable `FieldErrorComponent`

**2. API Integration:**

Connect the task manager to a REST API:

- Set up `provideHttpClient()` with interceptors
- Create a `TaskService` that wraps CRUD operations
- Implement a functional `authInterceptor` that adds JWT tokens
- Implement an `errorInterceptor` for centralized error handling
- Add retry with exponential backoff for GET requests
- Test HTTP calls with `HttpTestingController`

**3. Dynamic Form:**

Build a form generated from configuration:

- Accept a JSON config that defines form fields (type, label, validators)
- Dynamically generate `FormControl` instances
- Support text, number, email, select, and checkbox fields
- Handle form submission and display the result

### Knowledge Check

- [ ] Can you build a typed reactive form with `NonNullableFormBuilder`?
- [ ] Can you write a custom async validator?
- [ ] Can you implement a functional HTTP interceptor?
- [ ] Can you test HTTP requests with `HttpTestingController`?

---

## Phase 4: Reactive Programming (Weeks 7–8)

### Learning Objectives

- Use essential RxJS operators: `switchMap`, `debounceTime`, `combineLatest`, `catchError`
- Understand the difference between `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`
- Use `toSignal()` and `toObservable()` for RxJS/Signal interop
- Implement signal-based and Observable-based state management
- Evaluate state management approaches (services, NgRx, SignalStore)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-RXJS-AND-OBSERVABLES](05-RXJS-AND-OBSERVABLES.md) | Operators, subjects, async pipe, signal interop |
| 2 | [06-STATE-MANAGEMENT](06-STATE-MANAGEMENT.md) | Service state, signal state, NgRx, SignalStore |

### Exercises

**1. Reactive Search:**

Build an autocomplete search feature:

- Use a `Subject` to capture input events
- Apply `debounceTime(300)`, `distinctUntilChanged()`, `filter(q => q.length >= 2)`
- Use `switchMap` to call the search API (cancel previous requests)
- Convert the result to a signal with `toSignal()`
- Display results with loading and error states

**2. Signal-Based State:**

Refactor the task manager to use signal-based state:

- Create a `TaskStore` service using `signal()`, `computed()`, and `effect()`
- Implement actions: load, add, update, delete, filter
- Use `computed()` for filtered lists and counts
- Use `toSignal()` to bridge HTTP Observables to signals

**3. NgRx SignalStore (Optional Advanced):**

Implement the same task store using NgRx SignalStore:

- Define state with `withState()`
- Add computed properties with `withComputed()`
- Add methods with `withMethods()`
- Use `rxMethod()` for async operations
- Compare the signal store approach with the manual signal service

### Knowledge Check

- [ ] Can you explain when to use `switchMap` vs `exhaustMap`?
- [ ] Can you convert an Observable to a Signal with `toSignal()`?
- [ ] Can you implement a service-based state store with signals?
- [ ] Can you explain the tradeoffs between service state and NgRx?

---

## Phase 5: Testing (Weeks 9–10)

### Learning Objectives

- Set up and run Angular tests with Jest or Jasmine
- Test components with inputs, outputs, and injected services
- Test services with mocked dependencies
- Test async code with `fakeAsync`, `tick`, and `waitForAsync`
- Write integration tests for component trees
- Understand E2E testing with Playwright

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [08-TESTING](08-TESTING.md) | TestBed, component tests, service tests, E2E |

### Exercises

**1. Unit Testing Services:**

Write comprehensive tests for the `TaskService`:

- Test each CRUD method with `HttpTestingController`
- Verify request method, URL, and body
- Test error handling (simulate 404, 500 responses)
- Test the signal-based `TaskStore` state transitions

**2. Component Testing:**

Test the `TaskListComponent`:

- Provide a mock `TaskService` with `jasmine.createSpyObj`
- Test that tasks are rendered correctly
- Test that clicking a task emits the correct output
- Test conditional rendering (@if, @for, @empty)
- Use `fixture.componentRef.setInput()` to test signal inputs

**3. Integration Testing:**

Test the task manager feature end-to-end in TestBed:

- Render the `TaskPageComponent` (smart component) with children
- Verify that adding a task updates the list
- Verify that filtering works across components
- Test routing with `provideRouter()`

**4. E2E Testing (Optional):**

Write Playwright tests for the task manager:

- Test the full user flow: create, edit, complete, delete a task
- Test navigation between pages
- Test form validation error display

### Knowledge Check

- [ ] Can you configure TestBed for a standalone component with mocked services?
- [ ] Can you test async code with `fakeAsync` and `tick`?
- [ ] Can you write an E2E test that navigates and interacts with the UI?
- [ ] Can you explain the testing pyramid for Angular apps?

---

## Phase 6: Performance & Advanced (Weeks 11–12)

### Learning Objectives

- Apply `OnPush` change detection to all components
- Use `@defer` blocks for lazy loading content
- Understand zoneless Angular with signals
- Optimize images with `NgOptimizedImage`
- Set up Server-Side Rendering with `@angular/ssr`
- Profile performance with Angular DevTools
- Analyze and optimize bundle size

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-PERFORMANCE](09-PERFORMANCE.md) | Change detection, @defer, SSR, hydration, profiling |

### Exercises

**1. Performance Audit:**

Audit the task manager for performance:

- Ensure all components use `OnPush`
- Add `track` to all `@for` loops
- Replace template methods with `computed()` signals
- Add `@defer (on viewport)` for below-the-fold content
- Optimize images with `NgOptimizedImage`

**2. SSR Setup:**

Add server-side rendering:

- Run `ng add @angular/ssr`
- Fix any `document` or `window` references with `isPlatformBrowser()`
- Enable hydration with `provideClientHydration()`
- Build and test the SSR application

**3. Bundle Analysis:**

Optimize the bundle:

- Generate bundle stats with `ng build --stats-json`
- Analyze with `webpack-bundle-analyzer`
- Identify and eliminate large unused dependencies
- Verify all routes are lazy loaded
- Set a bundle budget in `angular.json`

### Knowledge Check

- [ ] Can you explain how `OnPush` change detection differs from `Default`?
- [ ] Can you describe what zoneless Angular is and how signals enable it?
- [ ] Can you set up SSR with `@angular/ssr` and handle platform-specific code?
- [ ] Can you analyze a bundle and identify optimization opportunities?

---

## Phase 7: Production & Best Practices (Weeks 13–14)

### Learning Objectives

- Apply the Angular style guide to project structure
- Implement security best practices (CSP, XSRF, sanitization)
- Set up error handling (global error handler, error interceptor)
- Avoid common anti-patterns
- Prepare an application for production deployment
- Complete the capstone project

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Project structure, security, i18n, checklist |
| 2 | [11-ANTI-PATTERNS](11-ANTI-PATTERNS.md) | Common mistakes and how to fix them |

### Exercises

**1. Code Quality Audit:**

Review the task manager against best practices:

- Run through the [Best Practices Checklist](10-BEST-PRACTICES.md#checklist)
- Run through the [Anti-Patterns Checklist](11-ANTI-PATTERNS.md#quick-reference-checklist)
- Fix any issues found
- Set up `ng lint` with Angular ESLint rules

**2. Security Hardening:**

Secure the task manager:

- Configure XSRF protection with `withXsrfConfiguration()`
- Review all uses of `innerHTML` and ensure proper sanitization
- Add Content Security Policy headers
- Implement a token refresh interceptor

**3. Capstone Project — Full-Stack Task Manager:**

Build a complete, production-ready Angular application:

- **Features:** User auth, task CRUD, real-time updates, filtering/sorting
- **Architecture:** Feature-based structure, standalone components, signal-based state
- **Forms:** Typed reactive forms with validation
- **Routing:** Lazy-loaded routes, functional guards, resolvers
- **HTTP:** Interceptors for auth and error handling
- **State:** Signal-based service store (or NgRx SignalStore)
- **Testing:** 80%+ coverage, unit + integration + E2E
- **Performance:** OnPush, @defer, lazy loading, NgOptimizedImage
- **SSR:** Server-side rendering with hydration
- **Deployment:** Build optimized bundle, configure caching headers

### Knowledge Check

- [ ] Can you explain Angular's built-in XSS protections?
- [ ] Can you name 5 anti-patterns from the anti-patterns document and their solutions?
- [ ] Can you describe a production deployment checklist for an Angular app?
- [ ] Have you completed the capstone project with all required features?

---

## Summary

| Phase | Weeks | Focus | Key Documents |
|-------|-------|-------|--------------|
| 1. Foundations | 1–2 | Components, templates, CLI | 00-OVERVIEW, 01-COMPONENTS |
| 2. Core Concepts | 3–4 | Services, DI, routing | 02-SERVICES, 03-ROUTING |
| 3. Forms & Data | 5–6 | Reactive forms, HTTP, interceptors | 04-FORMS, 07-HTTP |
| 4. Reactive Programming | 7–8 | RxJS, signals, state management | 05-RXJS, 06-STATE |
| 5. Testing | 9–10 | Unit, integration, E2E testing | 08-TESTING |
| 6. Performance | 11–12 | OnPush, SSR, hydration, @defer | 09-PERFORMANCE |
| 7. Production | 13–14 | Best practices, security, capstone | 10-BEST-PRACTICES, 11-ANTI-PATTERNS |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Angular learning path documentation |
