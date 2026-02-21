# Angular Best Practices

## Table of Contents

1. [Overview](#overview)
2. [Project Structure and Module Organization](#project-structure-and-module-organization)
3. [Standalone Components as Default](#standalone-components-as-default)
4. [Smart vs Presentational Components](#smart-vs-presentational-components)
5. [Strict Typing and TypeScript](#strict-typing-and-typescript)
6. [Signals for Reactive State](#signals-for-reactive-state)
7. [Functional Guards and Interceptors](#functional-guards-and-interceptors)
8. [Angular Style Guide](#angular-style-guide)
9. [Angular CLI for Code Generation](#angular-cli-for-code-generation)
10. [Subscription Management](#subscription-management)
11. [Error Handling Patterns](#error-handling-patterns)
12. [Security](#security)
13. [Internationalization](#internationalization)
14. [Checklist](#checklist)
15. [Next Steps](#next-steps)
16. [Version History](#version-history)

---

## Overview

This document consolidates production-ready best practices for Angular development. These patterns are proven in large-scale applications and align with modern Angular (17/18/19) — standalone components, signals, functional APIs, and zoneless change detection.

### Scope

- Feature-based project structure
- Standalone components as the default
- Smart vs presentational component separation
- TypeScript strict mode and strong typing
- Signal-first reactive state
- Functional guards and interceptors
- Angular style guide adherence
- CLI-based code generation
- Subscription management
- Error handling, security, and i18n
- Production readiness checklist

---

## Project Structure and Module Organization

### Recommended Feature-Based Structure

```
src/
├── app/
│   ├── core/                    # Singleton services, guards, interceptors
│   │   ├── auth/
│   │   │   ├── auth.guard.ts
│   │   │   ├── auth.interceptor.ts
│   │   │   └── auth.service.ts
│   │   ├── error-handling/
│   │   │   └── global-error-handler.ts
│   │   └── layout/
│   │       ├── header.component.ts
│   │       └── footer.component.ts
│   │
│   ├── features/                # Feature modules (domain-oriented)
│   │   ├── users/
│   │   │   ├── user-list.component.ts
│   │   │   ├── user-detail.component.ts
│   │   │   ├── user.service.ts
│   │   │   ├── user.model.ts
│   │   │   └── users.routes.ts
│   │   ├── orders/
│   │   │   ├── order-list.component.ts
│   │   │   ├── order.service.ts
│   │   │   └── orders.routes.ts
│   │   └── settings/
│   │       └── ...
│   │
│   ├── shared/                  # Reusable components, pipes, directives
│   │   ├── components/
│   │   │   ├── button.component.ts
│   │   │   ├── modal.component.ts
│   │   │   └── spinner.component.ts
│   │   ├── directives/
│   │   │   └── tooltip.directive.ts
│   │   ├── pipes/
│   │   │   ├── truncate.pipe.ts
│   │   │   └── file-size.pipe.ts
│   │   └── models/
│   │       └── api-response.model.ts
│   │
│   ├── app.component.ts
│   ├── app.config.ts
│   └── app.routes.ts
├── assets/
├── environments/
└── styles/
```

### Naming Rules

| Item | Convention | Example |
|------|-----------|---------|
| Components | `kebab-case.component.ts` | `user-list.component.ts` |
| Services | `kebab-case.service.ts` | `user.service.ts` |
| Guards | `kebab-case.guard.ts` | `auth.guard.ts` |
| Interceptors | `kebab-case.interceptor.ts` | `auth.interceptor.ts` |
| Pipes | `kebab-case.pipe.ts` | `truncate.pipe.ts` |
| Directives | `kebab-case.directive.ts` | `tooltip.directive.ts` |
| Models/interfaces | `kebab-case.model.ts` | `user.model.ts` |
| Route files | `kebab-case.routes.ts` | `users.routes.ts` |

---

## Standalone Components as Default

Since Angular 17, standalone components are the default. Every new component should be standalone unless you are maintaining a legacy NgModule-based project.

```typescript
// ✅ Modern standalone component
@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [DatePipe, RouterLink],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <article class="card">
      <h2>{{ user().name }}</h2>
      <p>Joined: {{ user().joinedAt | date:'mediumDate' }}</p>
      <a [routerLink]="['/users', user().id]">View Profile</a>
    </article>
  `,
})
export class UserCardComponent {
  user = input.required<User>();
}
```

### Key Rules

- **Always use `standalone: true`** for new components, directives, and pipes
- **Import dependencies directly** — each component declares what it needs
- **Use `bootstrapApplication()`** instead of `platformBrowserDynamic().bootstrapModule()`
- **Co-locate route files** — feature routes live next to their components

---

## Smart vs Presentational Components

### Presentational (Dumb) Components

- Receive data through `input()`, emit events through `output()`
- No injected services (except optionally for styling/animation)
- Pure rendering — easy to test, reuse, and reason about
- Always use `OnPush` change detection

### Smart (Container) Components

- Inject services and manage state
- Orchestrate presentational components
- Handle side effects (API calls, navigation)
- Connected to the store or services

```typescript
// ✅ Smart component
@Component({
  selector: 'app-user-page',
  standalone: true,
  imports: [UserListComponent, UserFilterComponent],
  template: `
    <app-user-filter (filterChange)="onFilterChange($event)" />
    <app-user-list [users]="filteredUsers()" (userClick)="navigateToUser($event)" />
  `,
})
export class UserPageComponent {
  private userService = inject(UserService);
  private router = inject(Router);

  private filter = signal('');
  allUsers = toSignal(this.userService.getAll(), { initialValue: [] });
  filteredUsers = computed(() =>
    this.allUsers().filter(u => u.name.toLowerCase().includes(this.filter().toLowerCase()))
  );

  onFilterChange(filter: string): void { this.filter.set(filter); }
  navigateToUser(user: User): void { this.router.navigate(['/users', user.id]); }
}
```

---

## Strict Typing and TypeScript

### Enable Strict Mode

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true
  },
  "angularCompilerOptions": {
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}
```

### Typing Best Practices

```typescript
// ✅ Typed interfaces for all data models
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'editor' | 'viewer';
  createdAt: Date;
}

// ✅ Typed forms
userForm = new FormGroup<{
  name: FormControl<string>;
  email: FormControl<string>;
  role: FormControl<'admin' | 'editor' | 'viewer'>;
}>({ /* ... */ });

// ✅ Typed HTTP responses
this.http.get<User[]>('/api/users');

// ❌ Avoid 'any'
this.http.get('/api/users');  // Returns Observable<Object>
```

---

## Signals for Reactive State

Prefer signals for synchronous reactive state. Use RxJS only where async stream composition is genuinely needed.

```typescript
// ✅ Signal-based state
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private _theme = signal<'light' | 'dark'>('light');
  readonly theme = this._theme.asReadonly();
  readonly isDark = computed(() => this._theme() === 'dark');

  toggle(): void {
    this._theme.update(t => t === 'light' ? 'dark' : 'light');
  }
}

// ✅ Signal inputs and outputs
@Component({ /* ... */ })
export class CounterComponent {
  value = model(0);
  step = input(1);
}
```

### When to Use Signals vs RxJS

| Scenario | Use |
|----------|-----|
| Component state (UI toggles, counters) | `signal()` |
| Derived/computed values | `computed()` |
| Side effects reacting to state | `effect()` |
| HTTP requests | RxJS `Observable` + `toSignal()` |
| WebSocket streams | RxJS `Observable` |
| Complex async composition (debounce, merge, switch) | RxJS operators |
| Form value changes → search API | RxJS + `toSignal()` at component boundary |

---

## Functional Guards and Interceptors

Prefer functional guards and interceptors over class-based ones — they are simpler, tree-shakable, and composable.

```typescript
// ✅ Functional guard
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isAuthenticated() ? true : router.createUrlTree(['/login']);
};

// ✅ Functional interceptor
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getAccessToken();
  if (token) {
    req = req.clone({ headers: req.headers.set('Authorization', `Bearer ${token}`) });
  }
  return next(req);
};

// ✅ Guard factory for role-based access
export function roleGuard(role: string): CanActivateFn {
  return () => inject(AuthService).hasRole(role);
}
```

---

## Angular Style Guide

Follow the [official Angular style guide](https://angular.dev/style-guide):

| Rule | Guideline |
|------|-----------|
| **Single responsibility** | One component/service/pipe per file |
| **File naming** | `feature-name.type.ts` (e.g., `user-list.component.ts`) |
| **Selector prefix** | Use a consistent prefix (`app-`, `admin-`, etc.) |
| **Member ordering** | Signals/inputs → injected services → public methods → private methods |
| **Max file length** | ~400 lines per file; extract if larger |
| **Template complexity** | Extract complex logic into computed signals or pipes |
| **Service scope** | Use `providedIn: 'root'` for singletons; component `providers` for scoped |

---

## Angular CLI for Code Generation

Always use `ng generate` to scaffold code — it ensures consistent naming, file placement, and boilerplate:

```bash
# Component (standalone by default)
ng generate component features/users/user-card --change-detection=OnPush

# Service
ng generate service core/auth/auth

# Guard (functional)
ng generate guard core/auth/auth --functional

# Pipe
ng generate pipe shared/pipes/truncate

# Interceptor (functional)
ng generate interceptor core/http/auth --functional

# Full feature scaffold
ng generate component features/orders/order-list --change-detection=OnPush
ng generate service features/orders/order
```

---

## Subscription Management

**Rule:** Every subscription must have a clear cleanup strategy.

| Strategy | When to Use |
|----------|-------------|
| `async` pipe | Template bindings |
| `toSignal()` | Converting Observable to Signal in component |
| `takeUntilDestroyed()` | Manual subscribe in constructor/field initializer |
| `takeUntilDestroyed(destroyRef)` | Manual subscribe in `ngOnInit` or other methods |
| `Subscription.add()` + `unsubscribe()` | Legacy code |

```typescript
// ✅ Best: no subscription at all (signals)
users = toSignal(this.userService.getAll(), { initialValue: [] });

// ✅ Good: takeUntilDestroyed in field initializer
private updates$ = this.liveService.updates$.pipe(
  takeUntilDestroyed(),
).subscribe(update => this.handleUpdate(update));

// ❌ Bad: subscribe with no cleanup
ngOnInit() {
  this.service.getData().subscribe(data => this.data = data);
}
```

---

## Error Handling Patterns

### Global Error Handler

```typescript
import { ErrorHandler, Injectable, inject } from '@angular/core';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private logger = inject(LoggingService);

  handleError(error: unknown): void {
    this.logger.error('Unhandled error:', error);
    // Report to error tracking service (Sentry, etc.)
  }
}

// Register in app.config.ts
providers: [
  { provide: ErrorHandler, useClass: GlobalErrorHandler },
],
```

### HTTP Error Handling

```typescript
// Centralized error interceptor (see 07-HTTP-AND-API-INTEGRATION.md)
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      inject(NotificationService).add(getErrorMessage(error), 'error');
      return throwError(() => error);
    }),
  );
};
```

### Component-Level Error Boundaries

```typescript
@Component({
  template: `
    @if (error()) {
      <app-error-message [error]="error()!" (retry)="reload()" />
    } @else if (loading()) {
      <app-spinner />
    } @else {
      <ng-content />
    }
  `,
})
export class DataContainerComponent {
  error = input<Error | null>(null);
  loading = input(false);
  retry = output<void>();

  reload(): void { this.retry.emit(); }
}
```

---

## Security

### Built-in Protections

| Threat | Angular's Defense |
|--------|-------------------|
| **XSS** | Automatic output sanitization — all interpolated values are escaped |
| **Script injection** | `bypassSecurityTrust*` required for trusted HTML/URLs |
| **CSRF** | Built-in XSRF token handling with `withXsrfConfiguration()` |
| **Open redirects** | Router guards validate navigation targets |

### Security Best Practices

```typescript
// ✅ Angular sanitizes by default
<p>{{ userInput }}</p>  <!-- Auto-escaped -->

// ❌ Avoid bypassing sanitization unless absolutely necessary
import { DomSanitizer } from '@angular/platform-browser';
this.sanitizer.bypassSecurityTrustHtml(trustedHtml);  // Use sparingly
```

```typescript
// ✅ Content Security Policy header
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'

// ✅ XSRF protection
provideHttpClient(
  withXsrfConfiguration({
    cookieName: 'XSRF-TOKEN',
    headerName: 'X-XSRF-TOKEN',
  }),
),
```

### Security Checklist

- [ ] Never use `innerHTML` with user input
- [ ] Minimize use of `bypassSecurityTrust*` methods
- [ ] Enable CSP headers in production
- [ ] Use `HttpOnly` and `Secure` flags for auth cookies
- [ ] Validate all input on the server (client validation is not enough)
- [ ] Keep Angular and dependencies updated

---

## Internationalization

### @angular/localize

```bash
ng add @angular/localize
```

```html
<!-- Mark text for translation -->
<h1 i18n="@@welcomeTitle">Welcome to our app</h1>
<p i18n="user count|A message showing user count@@userCount">
  { count, plural, =0 {No users} =1 {1 user} other {{{ count }} users} }
</p>
```

```bash
# Extract translation messages
ng extract-i18n --output-path=src/locale

# Build for a specific locale
ng build --localize
```

---

## Checklist

### Before Every PR

- [ ] All components use `standalone: true`
- [ ] All components use `ChangeDetectionStrategy.OnPush` (or signals + zoneless)
- [ ] Signal inputs (`input()`) and outputs (`output()`) used (not `@Input`/`@Output`)
- [ ] All subscriptions have cleanup (takeUntilDestroyed, async pipe, or toSignal)
- [ ] No `any` types — strict typing throughout
- [ ] Unit tests pass with adequate coverage
- [ ] No lint errors (`ng lint`)

### Before Production

- [ ] Routes are lazy loaded
- [ ] Images use `NgOptimizedImage`
- [ ] `@defer` used for below-the-fold content
- [ ] Bundle size within budget
- [ ] SSR enabled for SEO-critical pages
- [ ] Error handling at HTTP and component level
- [ ] Security headers configured (CSP, HSTS, X-Frame-Options)
- [ ] Accessibility audit passed (axe, Lighthouse)
- [ ] Performance audit passed (Lighthouse > 90)

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [11-ANTI-PATTERNS](11-ANTI-PATTERNS.md) | Common mistakes to avoid |
| 2 | [09-PERFORMANCE](09-PERFORMANCE.md) | Performance optimization in depth |
| 3 | [08-TESTING](08-TESTING.md) | Testing strategies |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Angular best practices documentation |
