# Angular Anti-Patterns

A catalogue of the most common Angular development mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a code-review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Not Unsubscribing from Observables](#1-not-unsubscribing-from-observables)
- [2. Using Default Change Detection Everywhere](#2-using-default-change-detection-everywhere)
- [3. Putting Logic in Components Instead of Services](#3-putting-logic-in-components-instead-of-services)
- [4. Over-Using NgModules](#4-over-using-ngmodules)
- [5. Not Using Track for Loops](#5-not-using-track-for-loops)
- [6. Direct DOM Manipulation](#6-direct-dom-manipulation)
- [7. Circular Dependencies](#7-circular-dependencies)
- [8. Heavy Computations in Templates](#8-heavy-computations-in-templates)
- [9. Not Using Typed Forms](#9-not-using-typed-forms)
- [10. Ignoring the Angular Style Guide](#10-ignoring-the-angular-style-guide)
- [11. Over-Fetching Data Without Caching](#11-over-fetching-data-without-caching)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Anti-patterns are recurring practices that seem reasonable at first but create significant problems over time. In Angular, the consequences include memory leaks, poor rendering performance, unmaintainable code, and security vulnerabilities.

Each anti-pattern documented here is:

- **Seductive** — it felt like the right approach when first implemented
- **Harmful** — it creates real problems in performance, maintainability, or correctness
- **Fixable** — there is a well-understood better approach

### How to Use This Document

1. **Code review:** Reference specific sections when reviewing Angular PRs
2. **Pre-production review:** Use the [Quick Reference Checklist](#quick-reference-checklist) before deploying
3. **Team onboarding:** Assign this document to new engineers joining the Angular team
4. **Periodic audit:** Review existing code against these patterns quarterly

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|-------------|----------|----------|
| 1 | Not unsubscribing from Observables | Memory | 🔴 Critical |
| 2 | Using Default change detection everywhere | Performance | 🟠 High |
| 3 | Putting logic in components instead of services | Architecture | 🟠 High |
| 4 | Over-using NgModules | Architecture | 🟡 Medium |
| 5 | Not using track for loops | Performance | 🟠 High |
| 6 | Direct DOM manipulation | Architecture | 🟠 High |
| 7 | Circular dependencies | Architecture | 🟠 High |
| 8 | Heavy computations in templates | Performance | 🔴 Critical |
| 9 | Not using typed forms | Type Safety | 🟡 Medium |
| 10 | Ignoring the Angular style guide | Maintainability | 🟡 Medium |
| 11 | Over-fetching data without caching | Performance | 🟠 High |

---

## 1. Not Unsubscribing from Observables

### Problem

Subscribing to Observables without cleaning up leads to memory leaks, stale callbacks, and unexpected behavior when components are destroyed.

### Why It Happens

Developers subscribe in `ngOnInit` and forget that the component can be destroyed before the Observable completes. Short-lived HTTP requests mask the problem because they auto-complete.

### Impact

- Memory leaks — subscriptions hold references to destroyed components
- Stale callbacks — updates to a destroyed component's state
- Performance degradation over time
- Console errors: "Cannot read property of undefined"

### Solution

```typescript
// ❌ Anti-pattern: subscribe with no cleanup
@Component({ /* ... */ })
export class BadComponent implements OnInit {
  data: Data[] = [];

  ngOnInit(): void {
    this.dataService.getLiveUpdates().subscribe(data => {
      this.data = data;  // Still runs after component is destroyed!
    });
  }
}

// ✅ Solution 1: toSignal (best — no subscription needed)
@Component({ /* ... */ })
export class GoodComponent {
  data = toSignal(inject(DataService).getLiveUpdates(), { initialValue: [] });
}

// ✅ Solution 2: takeUntilDestroyed
@Component({ /* ... */ })
export class GoodComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    this.dataService.getLiveUpdates().pipe(
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(data => this.handleData(data));
  }
}

// ✅ Solution 3: async pipe in template
@Component({
  template: `
    @if (data$ | async; as data) {
      @for (item of data; track item.id) {
        <div>{{ item.name }}</div>
      }
    }
  `,
})
export class GoodComponent {
  data$ = inject(DataService).getLiveUpdates();
}
```

---

## 2. Using Default Change Detection Everywhere

### Problem

Leaving all components on the `Default` change detection strategy causes Angular to check every component on every event — clicks, timers, HTTP responses, even `mousemove`.

### Why It Happens

`Default` is the default. It works without extra effort, and the performance cost is invisible during development with small data sets.

### Impact

- Exponential performance degradation as the component tree grows
- Unnecessary template re-evaluations
- Jank and dropped frames in complex UIs
- Wasted CPU cycles on unchanged components

### Solution

```typescript
// ❌ Anti-pattern: no change detection strategy (defaults to Default)
@Component({
  selector: 'app-user-card',
  template: `<h2>{{ user.name }}</h2>`,
})
export class UserCardComponent {
  @Input() user!: User;
}

// ✅ Solution: OnPush with signal inputs
@Component({
  selector: 'app-user-card',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<h2>{{ user().name }}</h2>`,
})
export class UserCardComponent {
  user = input.required<User>();
}
```

**Rule:** Every component should use `ChangeDetectionStrategy.OnPush` unless there is a specific reason not to.

---

## 3. Putting Logic in Components Instead of Services

### Problem

Business logic, data access, and state management embedded directly in components rather than extracted into services.

### Why It Happens

It is faster to write logic directly in the component, especially during prototyping. As the app grows, this logic is never refactored out.

### Impact

- Logic cannot be reused across components
- Components become large and hard to test
- Business rules are scattered across the codebase
- Tight coupling between UI and logic

### Solution

```typescript
// ❌ Anti-pattern: logic in the component
@Component({ /* ... */ })
export class OrderComponent {
  orders: Order[] = [];

  ngOnInit(): void {
    this.http.get<Order[]>('/api/orders').subscribe(orders => {
      this.orders = orders.filter(o => o.status !== 'cancelled');
      this.orders.sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
    });
  }

  calculateTotal(): number {
    return this.orders.reduce((sum, o) => sum + o.total, 0);
  }
}

// ✅ Solution: extract to a service
@Injectable({ providedIn: 'root' })
export class OrderService {
  private http = inject(HttpClient);

  getActiveOrders(): Observable<Order[]> {
    return this.http.get<Order[]>('/api/orders').pipe(
      map(orders => orders
        .filter(o => o.status !== 'cancelled')
        .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
      ),
    );
  }
}

@Component({ /* ... */ })
export class OrderComponent {
  orders = toSignal(inject(OrderService).getActiveOrders(), { initialValue: [] });
  total = computed(() => this.orders().reduce((sum, o) => sum + o.total, 0));
}
```

---

## 4. Over-Using NgModules

### Problem

Creating NgModules for every feature, shared module, and component group when standalone components handle this more simply.

### Why It Happens

NgModules were the only option before Angular 14. Existing tutorials and documentation still reference them. Teams continue patterns from legacy projects.

### Impact

- Boilerplate: every component needs a module declaration
- Confusing imports: where does this component live?
- Harder lazy loading: must lazy-load an entire module
- Larger bundles: modules pull in everything they declare

### Solution

```typescript
// ❌ Anti-pattern: NgModule for a simple feature
@NgModule({
  declarations: [UserListComponent, UserCardComponent, UserFilterComponent],
  imports: [CommonModule, RouterModule],
  exports: [UserListComponent],
})
export class UsersModule {}

// ✅ Solution: standalone components
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [UserCardComponent, UserFilterComponent, RouterLink],
  template: `<!-- ... -->`,
})
export class UserListComponent {}

// Lazy load directly
{ path: 'users', loadComponent: () => import('./user-list.component').then(m => m.UserListComponent) }
```

---

## 5. Not Using Track for Loops

### Problem

Rendering lists without a `track` expression (or `trackBy` in legacy `*ngFor`), causing Angular to destroy and recreate all DOM elements on every change.

### Why It Happens

Developers forget or do not understand the performance implication. Small lists work fine without it, masking the problem.

### Impact

- All DOM elements are destroyed and recreated on every update
- Flashing/flickering in the UI
- Lost input state (focus, selection, scroll position)
- 10–100x slower rendering for large lists

### Solution

```typescript
// ❌ Anti-pattern: no track expression
@for (user of users(); track $index) {
  // Using $index is a poor track — reordering/filtering breaks it
}

// ❌ Anti-pattern: legacy *ngFor without trackBy
<div *ngFor="let user of users">{{ user.name }}</div>

// ✅ Solution: track by unique identifier
@for (user of users(); track user.id) {
  <app-user-card [user]="user" />
} @empty {
  <p>No users found.</p>
}
```

**Rule:** Always use `track item.id` (or another stable unique identifier) in `@for` loops.

---

## 6. Direct DOM Manipulation

### Problem

Using `document.querySelector`, `document.getElementById`, `innerHTML`, or `nativeElement` directly instead of Angular's template binding and renderer APIs.

### Why It Happens

Developers with jQuery or vanilla JS backgrounds reach for familiar DOM APIs. Some third-party libraries require direct DOM access.

### Impact

- Bypasses Angular's change detection — UI gets out of sync
- Breaks server-side rendering (no `document` on the server)
- Bypasses Angular's XSS sanitization
- Makes testing harder

### Solution

```typescript
// ❌ Anti-pattern: direct DOM manipulation
@Component({ /* ... */ })
export class BadComponent {
  ngAfterViewInit(): void {
    document.getElementById('title')!.textContent = 'Hello';
    document.getElementById('content')!.innerHTML = this.htmlContent;
  }
}

// ✅ Solution: use Angular bindings
@Component({
  template: `
    <h1>{{ title() }}</h1>
    <div [innerHTML]="sanitizedHtml()"></div>
  `,
})
export class GoodComponent {
  title = signal('Hello');
  private sanitizer = inject(DomSanitizer);
  sanitizedHtml = computed(() =>
    this.sanitizer.bypassSecurityTrustHtml(this.htmlContent())
  );
}

// ✅ For third-party library integration, use afterNextRender + viewChild
@Component({ template: `<div #chart></div>` })
export class ChartComponent {
  chartEl = viewChild.required<ElementRef>('chart');

  constructor() {
    afterNextRender(() => {
      new ThirdPartyChart(this.chartEl().nativeElement);
    });
  }
}
```

---

## 7. Circular Dependencies

### Problem

Service A depends on Service B, which depends on Service A — creating a circular import chain.

### Why It Happens

Services grow organically. A notification service needs the auth service (to show user info), and the auth service needs the notification service (to show login errors).

### Impact

- Runtime errors or `undefined` imports
- Hard-to-debug initialization order issues
- Indicates poor architectural boundaries

### Solution

```typescript
// ❌ Anti-pattern: circular dependency
// auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private notification = inject(NotificationService); // ← depends on NotificationService
}

// notification.service.ts
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private auth = inject(AuthService); // ← depends on AuthService (circular!)
}

// ✅ Solution: introduce a mediator or event bus
@Injectable({ providedIn: 'root' })
export class EventBus {
  private events = new Subject<AppEvent>();
  readonly events$ = this.events.asObservable();
  emit(event: AppEvent): void { this.events.next(event); }
}

// AuthService emits events instead of calling NotificationService directly
@Injectable({ providedIn: 'root' })
export class AuthService {
  private eventBus = inject(EventBus);

  login(credentials: Credentials): Observable<User> {
    return this.http.post<User>('/api/login', credentials).pipe(
      tap(user => this.eventBus.emit({ type: 'LOGIN_SUCCESS', payload: user })),
      catchError(err => {
        this.eventBus.emit({ type: 'LOGIN_ERROR', payload: err.message });
        return throwError(() => err);
      }),
    );
  }
}
```

---

## 8. Heavy Computations in Templates

### Problem

Calling methods or performing complex expressions in templates that run on every change detection cycle.

### Why It Happens

It is convenient to call a method directly in the template. The performance cost is invisible with small data sets.

### Impact

- Method called on every change detection cycle (potentially hundreds of times per second)
- Jank and dropped frames
- CPU waste on unchanged data

### Solution

```typescript
// ❌ Anti-pattern: method call in template
@Component({
  template: `<p>Total: {{ calculateTotal() }}</p>`,
})
export class BadComponent {
  calculateTotal(): number {
    // This runs on EVERY change detection cycle
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}

// ✅ Solution 1: computed signal (best)
@Component({
  template: `<p>Total: {{ total() }}</p>`,
})
export class GoodComponent {
  items = signal<CartItem[]>([]);
  total = computed(() =>
    this.items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );
}

// ✅ Solution 2: pure pipe
@Pipe({ name: 'total', standalone: true, pure: true })
export class TotalPipe implements PipeTransform {
  transform(items: CartItem[]): number {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}

// Template: {{ items() | total }}
```

---

## 9. Not Using Typed Forms

### Problem

Using untyped reactive forms where `form.value` returns `any`, losing all compile-time type checking.

### Why It Happens

Typed forms were introduced in Angular 14. Older code and tutorials use untyped forms. The migration requires effort.

### Impact

- No compile-time validation of form structure
- Runtime errors from accessing wrong property names
- Refactoring forms is error-prone
- Cannot rely on IDE autocompletion

### Solution

```typescript
// ❌ Anti-pattern: untyped form
const form = new FormGroup({
  name: new FormControl(''),
  email: new FormControl(''),
});
const value = form.value; // { name: any; email: any } — all 'any'

// ✅ Solution: typed form with NonNullableFormBuilder
const fb = inject(NonNullableFormBuilder);
const form = fb.group({
  name: ['', Validators.required],
  email: ['', [Validators.required, Validators.email]],
});
const value = form.getRawValue();
// { name: string; email: string } — fully typed!
```

---

## 10. Ignoring the Angular Style Guide

### Problem

Inconsistent naming, file structure, and coding patterns across the project.

### Why It Happens

Teams do not establish conventions early. Developers bring habits from other frameworks. No linting enforcement.

### Impact

- Harder onboarding for new team members
- Inconsistent code is harder to navigate and maintain
- Merge conflicts from structural disagreements
- Reduced productivity

### Solution

- Follow the [official Angular style guide](https://angular.dev/style-guide)
- Use `ng lint` with Angular ESLint rules
- Enforce conventions in CI

```bash
# Install Angular ESLint
ng add @angular-eslint/schematics

# Run linting
ng lint
```

| Rule | Convention |
|------|-----------|
| File names | `kebab-case.type.ts` |
| Component selectors | `app-feature-name` (prefix + kebab) |
| Service names | `FeatureNameService` |
| One class per file | Always |
| Barrel exports | Use `index.ts` for public API of features |

---

## 11. Over-Fetching Data Without Caching

### Problem

Making the same HTTP request every time a component initializes, without any caching or deduplication.

### Why It Happens

Each component independently fetches its data in `ngOnInit`. No shared cache or state management layer exists.

### Impact

- Redundant API calls waste bandwidth and server resources
- Slower navigation (every route change triggers new requests)
- Flickering loading states
- Poor user experience

### Solution

```typescript
// ❌ Anti-pattern: fetch on every init
@Component({ /* ... */ })
export class UserProfileComponent implements OnInit {
  user?: User;

  ngOnInit(): void {
    this.http.get<User>('/api/me').subscribe(user => this.user = user);
  }
}

// ✅ Solution: cache in a service with shareReplay
@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);

  readonly currentUser$ = this.http.get<User>('/api/me').pipe(
    shareReplay(1),  // Cache the last emission, share across subscribers
  );

  readonly currentUser = toSignal(this.currentUser$);
}

// Component just reads the cached signal
@Component({ /* ... */ })
export class UserProfileComponent {
  user = inject(AuthService).currentUser;
}
```

---

## Quick Reference Checklist

| # | Check | How to Verify |
|---|-------|--------------|
| 1 | All subscriptions have cleanup | Search for `.subscribe(` without `takeUntilDestroyed`, `toSignal`, or `async` pipe |
| 2 | All components use OnPush | Search for components missing `changeDetection: ChangeDetectionStrategy.OnPush` |
| 3 | Business logic in services, not components | Review component files for HTTP calls, complex logic |
| 4 | Standalone components (no unnecessary NgModules) | Check for `@NgModule` declarations |
| 5 | All @for loops have track by ID | Search for `@for` without `track *.id` |
| 6 | No direct DOM manipulation | Search for `document.querySelector`, `document.getElementById`, `innerHTML` |
| 7 | No circular dependencies | Run `npx madge --circular src/` |
| 8 | No method calls in templates | Search for `{{ method()` patterns that are not signal reads |
| 9 | All forms are typed | Search for `new FormControl(` without type parameter |
| 10 | Consistent naming and structure | Run `ng lint` |
| 11 | HTTP responses cached where appropriate | Review services for `shareReplay` or signal-based caching |

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Positive patterns to follow |
| 2 | [09-PERFORMANCE](09-PERFORMANCE.md) | Performance optimization in depth |
| 3 | [08-TESTING](08-TESTING.md) | Testing strategies |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Angular anti-patterns documentation |
