# Services & Dependency Injection

## Table of Contents

1. [Overview](#overview)
2. [Angular DI System Fundamentals](#angular-di-system-fundamentals)
3. [Creating and Providing Services](#creating-and-providing-services)
4. [Injectable Decorators and providedIn Options](#injectable-decorators-and-providedin-options)
5. [Hierarchical Injectors](#hierarchical-injectors)
6. [Injection Tokens](#injection-tokens)
7. [Multi Providers](#multi-providers)
8. [Factory Providers](#factory-providers)
9. [Tree-Shakable Providers](#tree-shakable-providers)
10. [Service Design Patterns](#service-design-patterns)
11. [Testing Services](#testing-services)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Dependency injection (DI) is one of Angular's most powerful features. It provides a way to create and deliver services — classes that encapsulate business logic, data access, and shared state — to components and other services without tight coupling. This document covers the DI system fundamentals, provider configuration, hierarchical injectors, and service design patterns.

### Scope

- How Angular's DI system works (injectors, providers, tokens)
- Creating services with `@Injectable`
- `providedIn` options: `'root'`, `'platform'`, `'any'`, and component-level
- Hierarchical injectors and injection resolution
- `InjectionToken` for non-class dependencies
- Multi providers and factory providers
- Tree-shakable provider patterns
- Service design patterns (facade, adapter, repository)
- Testing services with TestBed

---

## Angular DI System Fundamentals

### What Is Dependency Injection?

Dependency injection is a design pattern where a class receives its dependencies from an external source rather than creating them itself. Angular's DI system automates this process.

```
┌─────────────────────────────────────────────────────────────┐
│                    DI Resolution Flow                        │
│                                                             │
│  Component requests       Injector looks up        Provider │
│  a dependency        ──▶  the provider         ──▶ creates  │
│  (inject() or             in the injector          or reuses│
│   constructor)            hierarchy                instance │
│                                                             │
│  ┌──────────┐        ┌──────────────┐        ┌───────────┐ │
│  │Component │──────▶ │  Injector    │──────▶ │  Service   │ │
│  │ inject() │        │  Hierarchy   │        │  Instance  │ │
│  └──────────┘        └──────────────┘        └───────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Three Key Concepts

| Concept | Description | Example |
|---------|-------------|---------|
| **Provider** | Tells the injector how to create a dependency | `{ provide: UserService, useClass: UserService }` |
| **Injector** | Container that maintains a list of providers and creates instances | Root injector, component injector |
| **Token** | Key used to look up a provider | Class reference (`UserService`) or `InjectionToken` |

### inject() vs Constructor Injection

**Modern approach (preferred since Angular 14+):**

```typescript
import { Component, inject } from '@angular/core';

@Component({ /* ... */ })
export class UserListComponent {
  private userService = inject(UserService);
  private router = inject(Router);
}
```

**Legacy approach (constructor injection):**

```typescript
@Component({ /* ... */ })
export class UserListComponent {
  constructor(
    private userService: UserService,
    private router: Router,
  ) {}
}
```

The `inject()` function is preferred because:

- Works in any injection context (constructors, field initializers, factory functions)
- No need for constructor parameter decorators
- Better tree shaking
- Works with functional guards, interceptors, and resolvers

---

## Creating and Providing Services

### Basic Service

```typescript
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class NotificationService {
  private messages = signal<Notification[]>([]);

  readonly notifications = this.messages.asReadonly();

  add(message: string, type: 'info' | 'error' | 'success' = 'info'): void {
    this.messages.update(current => [
      ...current,
      { id: crypto.randomUUID(), message, type, timestamp: Date.now() },
    ]);
  }

  dismiss(id: string): void {
    this.messages.update(current => current.filter(n => n.id !== id));
  }
}
```

### Using a Service in a Component

```typescript
import { Component, inject } from '@angular/core';
import { NotificationService } from './notification.service';

@Component({
  selector: 'app-notification-panel',
  standalone: true,
  template: `
    @for (notification of notificationService.notifications(); track notification.id) {
      <div class="notification" [class]="notification.type">
        {{ notification.message }}
        <button (click)="notificationService.dismiss(notification.id)">×</button>
      </div>
    }
  `,
})
export class NotificationPanelComponent {
  protected notificationService = inject(NotificationService);
}
```

---

## Injectable Decorators and providedIn Options

### providedIn Options

| Value | Scope | Instance Count | Tree Shakable |
|-------|-------|---------------|---------------|
| `'root'` | Application-wide singleton | 1 | ✅ Yes |
| `'platform'` | Shared across multiple Angular apps on the page | 1 per platform | ✅ Yes |
| `'any'` | One instance per lazy-loaded module boundary | Multiple | ✅ Yes |
| Not set (component `providers`) | Component-level scope | 1 per component instance | ❌ No |

### Application-Wide Singleton (Most Common)

```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  // Single instance shared across the entire application
}
```

### Component-Level Provider

Each component instance gets its own service instance:

```typescript
@Component({
  selector: 'app-editor',
  standalone: true,
  providers: [EditorStateService],  // New instance per EditorComponent
  template: `<!-- ... -->`,
})
export class EditorComponent {
  private state = inject(EditorStateService);
}
```

### Route-Level Provider

Provide services scoped to a route and its children:

```typescript
export const editorRoutes: Routes = [
  {
    path: 'editor',
    providers: [EditorStateService],  // Scoped to editor route tree
    children: [
      { path: '', component: EditorComponent },
      { path: 'preview', component: PreviewComponent },
    ],
  },
];
```

---

## Hierarchical Injectors

Angular's injectors form a hierarchy that mirrors the component tree. When a component requests a dependency, Angular walks up the injector hierarchy until it finds a provider.

### Injector Hierarchy

```
┌──────────────────────────────────────────────┐
│            Platform Injector                  │
│  (shared across apps: PlatformRef, etc.)     │
├──────────────────────────────────────────────┤
│            Root Injector                      │
│  (providedIn: 'root' services, app config)   │
├──────────────────────────────────────────────┤
│        Route Injector (lazy loaded)           │
│  (providers in route config)                 │
├──────────────────────────────────────────────┤
│        Component Injector                     │
│  (providers in @Component)                   │
├──────────────────────────────────────────────┤
│        Element Injector                       │
│  (directive providers on the same element)   │
└──────────────────────────────────────────────┘
        ▲ Resolution walks UP this chain
```

### Resolution Modifiers

| Decorator / Flag | Behavior |
|-----------------|----------|
| `@Optional()` / `{ optional: true }` | Return `null` if provider not found (instead of throwing) |
| `@Self()` / `{ self: true }` | Only look in the current injector |
| `@SkipSelf()` / `{ skipSelf: true }` | Skip the current injector, start from parent |
| `@Host()` / `{ host: true }` | Stop at the host component's injector |

```typescript
// Using inject() with options
const logger = inject(LoggerService, { optional: true });
const parentState = inject(FormStateService, { skipSelf: true });
```

---

## Injection Tokens

### InjectionToken for Non-Class Dependencies

When you need to inject a value that is not a class (e.g., a configuration object, a string, a function), use `InjectionToken`:

```typescript
import { InjectionToken } from '@angular/core';

// Define the token
export interface AppConfig {
  apiUrl: string;
  maxRetries: number;
  featureFlags: Record<string, boolean>;
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');
```

### Providing a Token Value

```typescript
// In application config
bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: APP_CONFIG,
      useValue: {
        apiUrl: 'https://api.example.com',
        maxRetries: 3,
        featureFlags: { darkMode: true, betaFeatures: false },
      },
    },
  ],
});
```

### Injecting a Token

```typescript
@Injectable({ providedIn: 'root' })
export class ApiService {
  private config = inject(APP_CONFIG);

  getBaseUrl(): string {
    return this.config.apiUrl;
  }
}
```

### Token with Factory

```typescript
export const API_BASE_URL = new InjectionToken<string>('api.base.url', {
  providedIn: 'root',
  factory: () => {
    const isProduction = inject(PLATFORM_ID) === 'browser' &&
                         window.location.hostname !== 'localhost';
    return isProduction
      ? 'https://api.example.com'
      : 'http://localhost:3000';
  },
});
```

---

## Multi Providers

Multi providers allow multiple values to be registered for the same token. All values are injected as an array.

```typescript
export const HTTP_INTERCEPTOR = new InjectionToken<HttpInterceptorFn[]>('http.interceptors');

// Provide multiple values
bootstrapApplication(AppComponent, {
  providers: [
    { provide: HTTP_INTERCEPTOR, useValue: authInterceptor, multi: true },
    { provide: HTTP_INTERCEPTOR, useValue: loggingInterceptor, multi: true },
    { provide: HTTP_INTERCEPTOR, useValue: errorInterceptor, multi: true },
  ],
});

// Inject all of them
@Injectable({ providedIn: 'root' })
export class HttpService {
  private interceptors = inject(HTTP_INTERCEPTOR); // HttpInterceptorFn[]
}
```

### Common Multi Provider Use Cases

| Use Case | Token Example |
|----------|--------------|
| HTTP interceptors | `HTTP_INTERCEPTORS` |
| Route guards | Multiple guards on a route |
| Validators | Custom form validators |
| Plugin systems | Feature-specific handlers |

---

## Factory Providers

Factory providers let you create a dependency using a factory function:

```typescript
// Factory function
function loggerFactory(): Logger {
  const environment = inject(APP_CONFIG);
  if (environment.featureFlags.verboseLogging) {
    return new VerboseLogger();
  }
  return new SimpleLogger();
}

// Register with useFactory
bootstrapApplication(AppComponent, {
  providers: [
    { provide: Logger, useFactory: loggerFactory },
  ],
});
```

### Provider Types Summary

| Provider Type | Syntax | Use Case |
|--------------|--------|----------|
| `useClass` | `{ provide: Logger, useClass: ConsoleLogger }` | Substitute one class for another |
| `useValue` | `{ provide: API_URL, useValue: 'https://...' }` | Provide a static value |
| `useFactory` | `{ provide: Logger, useFactory: loggerFactory }` | Create with dynamic logic |
| `useExisting` | `{ provide: Logger, useExisting: ConsoleLogger }` | Alias — same instance as another token |

---

## Tree-Shakable Providers

A tree-shakable provider is removed from the production bundle if no component or service injects it. Angular achieves this with `providedIn`:

```typescript
// ✅ Tree-shakable: removed if never injected
@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  track(event: string): void { /* ... */ }
}

// ❌ NOT tree-shakable: always in the bundle
@Injectable()
export class LegacyService {
  // Must be listed in a providers array somewhere
}
```

### InjectionToken with Factory (Tree-Shakable)

```typescript
export const WINDOW = new InjectionToken<Window>('window', {
  providedIn: 'root',
  factory: () => {
    if (typeof window !== 'undefined') {
      return window;
    }
    // SSR: return a mock or throw
    throw new Error('Window is not available on the server');
  },
});
```

---

## Service Design Patterns

### Facade Pattern

A facade service provides a simplified API over multiple services:

```typescript
@Injectable({ providedIn: 'root' })
export class CheckoutFacade {
  private cartService = inject(CartService);
  private paymentService = inject(PaymentService);
  private orderService = inject(OrderService);

  // Expose only what the component needs
  readonly cartItems = this.cartService.items;
  readonly total = this.cartService.total;
  readonly isProcessing = signal(false);

  async checkout(paymentMethod: PaymentMethod): Promise<Order> {
    this.isProcessing.set(true);
    try {
      const payment = await firstValueFrom(
        this.paymentService.process(paymentMethod, this.total())
      );
      const order = await firstValueFrom(
        this.orderService.create(this.cartItems(), payment.id)
      );
      this.cartService.clear();
      return order;
    } finally {
      this.isProcessing.set(false);
    }
  }
}
```

### Adapter Pattern

Wrap third-party libraries behind an Angular service interface:

```typescript
// Abstract interface
export abstract class StorageService {
  abstract get<T>(key: string): T | null;
  abstract set<T>(key: string, value: T): void;
  abstract remove(key: string): void;
}

// Concrete implementation
@Injectable({ providedIn: 'root' })
export class LocalStorageService extends StorageService {
  get<T>(key: string): T | null {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : null;
  }

  set<T>(key: string, value: T): void {
    localStorage.setItem(key, JSON.stringify(value));
  }

  remove(key: string): void {
    localStorage.removeItem(key);
  }
}

// In tests, provide a mock
{ provide: StorageService, useClass: InMemoryStorageService }
```

### Repository Pattern

Centralize data access behind a consistent interface:

```typescript
@Injectable({ providedIn: 'root' })
export class UserRepository {
  private http = inject(HttpClient);
  private baseUrl = '/api/users';

  getAll(): Observable<User[]> {
    return this.http.get<User[]>(this.baseUrl);
  }

  getById(id: string): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/${id}`);
  }

  create(user: CreateUserDto): Observable<User> {
    return this.http.post<User>(this.baseUrl, user);
  }

  update(id: string, changes: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.baseUrl}/${id}`, changes);
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

---

## Testing Services

### Testing a Simple Service

```typescript
describe('NotificationService', () => {
  let service: NotificationService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(NotificationService);
  });

  it('should add a notification', () => {
    service.add('Hello', 'info');
    expect(service.notifications().length).toBe(1);
    expect(service.notifications()[0].message).toBe('Hello');
  });

  it('should dismiss a notification', () => {
    service.add('Test');
    const id = service.notifications()[0].id;
    service.dismiss(id);
    expect(service.notifications().length).toBe(0);
  });
});
```

### Testing a Service with Dependencies

```typescript
describe('UserRepository', () => {
  let service: UserRepository;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClientTesting()],
    });
    service = TestBed.inject(UserRepository);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify(); // Ensure no outstanding requests
  });

  it('should fetch all users', () => {
    const mockUsers: User[] = [
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ];

    service.getAll().subscribe(users => {
      expect(users).toEqual(mockUsers);
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
});
```

### Testing with Mock Services

```typescript
// Create a mock
const mockAuth: jasmine.SpyObj<AuthService> = jasmine.createSpyObj('AuthService', ['login', 'logout']);
mockAuth.login.and.returnValue(of({ id: '1', name: 'Test User' }));

TestBed.configureTestingModule({
  providers: [
    ProfileService,
    { provide: AuthService, useValue: mockAuth },
  ],
});
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [03-ROUTING-AND-NAVIGATION](03-ROUTING-AND-NAVIGATION.md) | Router config, lazy loading, guards |
| 2 | [07-HTTP-AND-API-INTEGRATION](07-HTTP-AND-API-INTEGRATION.md) | HttpClient, interceptors, API patterns |
| 3 | [08-TESTING](08-TESTING.md) | Testing services, components, and more |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial services and dependency injection documentation |
