# RxJS & Reactive Programming in Angular

## Table of Contents

1. [Overview](#overview)
2. [Observable Fundamentals](#observable-fundamentals)
3. [Essential RxJS Operators](#essential-rxjs-operators)
4. [Subjects](#subjects)
5. [The AsyncPipe](#the-asyncpipe)
6. [Subscription Management](#subscription-management)
7. [takeUntilDestroyed](#takeuntildestroyed)
8. [RxJS Interop with Signals](#rxjs-interop-with-signals)
9. [Common Reactive Patterns](#common-reactive-patterns)
10. [Error Handling in Observable Streams](#error-handling-in-observable-streams)
11. [Testing Observables](#testing-observables)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

RxJS (Reactive Extensions for JavaScript) is a library for composing asynchronous and event-based programs using observable sequences. It is deeply integrated into Angular — HTTP requests, router events, form changes, and many core APIs return Observables. This document covers essential operators, subscription management, signal interop, and testing patterns.

### Scope

- Observable creation and the Observer pattern
- Essential operators: `map`, `filter`, `switchMap`, `mergeMap`, `catchError`, `tap`, `debounceTime`, `distinctUntilChanged`, `combineLatest`, `forkJoin`
- Subject types: `Subject`, `BehaviorSubject`, `ReplaySubject`, `AsyncSubject`
- The `AsyncPipe` for template subscriptions
- Subscription management and preventing memory leaks
- `takeUntilDestroyed` (Angular 16+)
- Signal interop: `toSignal()`, `toObservable()`
- Common reactive patterns in Angular applications
- Error handling and retry strategies
- Testing Observables with `TestScheduler` and `fakeAsync`

---

## Observable Fundamentals

### What Is an Observable?

An Observable is a **lazy push collection** that can emit zero or more values over time and optionally complete or error.

```
Observable lifecycle:
────────────────────

  subscribe()
      │
      ▼
  ┌──────────────────────────────────────────────────┐
  │  next(value)   next(value)   next(value)  ...    │
  └───────────────────────────────────────┬──────────┘
                                          │
                              complete() or error(err)
```

### Creating Observables

```typescript
import { Observable, of, from, interval, fromEvent, timer } from 'rxjs';

// From static values
const numbers$ = of(1, 2, 3);

// From an array or iterable
const items$ = from([10, 20, 30]);

// From a promise
const data$ = from(fetch('/api/data').then(r => r.json()));

// Interval (emits incrementing numbers)
const tick$ = interval(1000); // 0, 1, 2, 3, ...

// From DOM events
const clicks$ = fromEvent(document, 'click');

// Timer (emit once after delay, or repeatedly)
const delayed$ = timer(2000); // Emit 0 after 2 seconds

// Custom Observable
const custom$ = new Observable<string>(subscriber => {
  subscriber.next('Hello');
  subscriber.next('World');
  subscriber.complete();

  return () => {
    // Teardown logic (cleanup)
  };
});
```

### Subscribing

```typescript
const subscription = numbers$.subscribe({
  next: value => console.log('Value:', value),
  error: err => console.error('Error:', err),
  complete: () => console.log('Done'),
});

// Unsubscribe when no longer needed
subscription.unsubscribe();
```

---

## Essential RxJS Operators

### Transformation Operators

```typescript
import { map, switchMap, mergeMap, concatMap, exhaustMap } from 'rxjs';

// map — transform each emitted value
const doubled$ = numbers$.pipe(
  map(n => n * 2)
);

// switchMap — cancel previous inner Observable on new emission
const search$ = searchInput$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.searchService.search(query))
);

// mergeMap — run inner Observables concurrently
const downloads$ = fileIds$.pipe(
  mergeMap(id => this.fileService.download(id), 3)  // max 3 concurrent
);

// concatMap — run inner Observables sequentially
const sequential$ = actions$.pipe(
  concatMap(action => this.processAction(action))
);

// exhaustMap — ignore new emissions while inner Observable is active
const submit$ = submitClicks$.pipe(
  exhaustMap(() => this.formService.submit(this.form.value))
);
```

### Higher-Order Mapping Summary

| Operator | Behavior | Use Case |
|----------|----------|----------|
| `switchMap` | Cancel previous, use latest | Search, autocomplete, navigation |
| `mergeMap` | Run all concurrently | Parallel downloads, logging |
| `concatMap` | Queue and run sequentially | Ordered writes, sequential API calls |
| `exhaustMap` | Ignore until current completes | Form submission, login |

### Filtering Operators

```typescript
import { filter, distinctUntilChanged, debounceTime, take, first, skip, takeUntil } from 'rxjs';

// filter — emit only values that pass a predicate
const adults$ = users$.pipe(filter(user => user.age >= 18));

// distinctUntilChanged — skip consecutive duplicates
const unique$ = values$.pipe(distinctUntilChanged());

// debounceTime — wait for silence before emitting
const debounced$ = input$.pipe(debounceTime(300));

// take — emit only the first N values, then complete
const firstThree$ = numbers$.pipe(take(3));

// first — emit only the first value (or first matching), then complete
const firstAdmin$ = users$.pipe(first(u => u.role === 'admin'));
```

### Combination Operators

```typescript
import { combineLatest, forkJoin, merge, zip, withLatestFrom } from 'rxjs';

// combineLatest — emit when ANY source emits (requires all to have emitted at least once)
const viewModel$ = combineLatest([user$, settings$, notifications$]).pipe(
  map(([user, settings, notifications]) => ({ user, settings, notifications }))
);

// forkJoin — emit once when ALL sources complete (like Promise.all)
const initialData$ = forkJoin({
  users: this.http.get<User[]>('/api/users'),
  roles: this.http.get<Role[]>('/api/roles'),
  config: this.http.get<Config>('/api/config'),
});

// merge — interleave emissions from multiple sources
const allEvents$ = merge(clicks$, keyPresses$, touches$);

// withLatestFrom — combine with the latest value from another source
const enriched$ = saveClicks$.pipe(
  withLatestFrom(currentUser$),
  map(([_, user]) => ({ savedBy: user.id }))
);
```

### Utility Operators

```typescript
import { tap, catchError, retry, finalize, delay, timeout } from 'rxjs';

// tap — side effect without modifying the stream
const logged$ = data$.pipe(
  tap(data => console.log('Received:', data))
);

// catchError — handle errors and return a fallback
const safe$ = data$.pipe(
  catchError(err => {
    console.error('Error:', err);
    return of(defaultValue);
  })
);

// retry — re-subscribe on error
const resilient$ = this.http.get('/api/data').pipe(
  retry({ count: 3, delay: 1000 })
);

// finalize — run cleanup when Observable completes or errors
const withCleanup$ = data$.pipe(
  finalize(() => this.isLoading.set(false))
);
```

---

## Subjects

Subjects are both an Observable and an Observer — they can emit values and be subscribed to.

### Subject Types

| Type | Initial Value | Replay | Use Case |
|------|--------------|--------|----------|
| `Subject` | None | None | Event bus, fire-and-forget |
| `BehaviorSubject` | Required | Last value (1) | Current state, always has a value |
| `ReplaySubject` | None | N values | Cache recent emissions |
| `AsyncSubject` | None | Last value on complete | One-shot async result |

### BehaviorSubject (Most Common in Angular)

```typescript
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private themeSubject = new BehaviorSubject<'light' | 'dark'>('light');

  readonly theme$ = this.themeSubject.asObservable();

  get currentTheme(): 'light' | 'dark' {
    return this.themeSubject.value;
  }

  toggleTheme(): void {
    const next = this.themeSubject.value === 'light' ? 'dark' : 'light';
    this.themeSubject.next(next);
  }
}
```

### ReplaySubject

```typescript
// Cache the last 3 messages
const messages$ = new ReplaySubject<Message>(3);
messages$.next({ text: 'Hello' });
messages$.next({ text: 'World' });
messages$.next({ text: '!' });

// Late subscriber receives all 3
messages$.subscribe(msg => console.log(msg));
```

---

## The AsyncPipe

The `AsyncPipe` subscribes to an Observable in the template, unwraps the emitted value, and **automatically unsubscribes** when the component is destroyed.

```typescript
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    @if (users$ | async; as users) {
      @for (user of users; track user.id) {
        <div>{{ user.name }}</div>
      }
    } @else {
      <p>Loading users...</p>
    }
  `,
})
export class UserListComponent {
  private userService = inject(UserService);
  users$ = this.userService.getAll();
}
```

### Benefits of AsyncPipe

- **Automatic subscription management** — no manual `subscribe()` / `unsubscribe()`
- **Works with OnPush change detection** — triggers change detection when values arrive
- **Template-driven data flow** — keeps the component class cleaner

---

## Subscription Management

Forgotten subscriptions cause memory leaks. Angular provides several patterns to manage them.

### Pattern 1: AsyncPipe (Preferred for Templates)

```typescript
// No manual subscription needed
users$ = this.userService.getAll();
// Template: {{ users$ | async }}
```

### Pattern 2: DestroyRef + takeUntilDestroyed (Preferred for Logic)

```typescript
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class DashboardComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    this.dataService.getUpdates().pipe(
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(data => {
      this.processData(data);
    });
  }
}
```

### Pattern 3: Subscription Collection (Manual)

```typescript
@Component({ /* ... */ })
export class LegacyComponent implements OnDestroy {
  private subscriptions = new Subscription();

  ngOnInit(): void {
    this.subscriptions.add(
      this.service.getData().subscribe(data => this.handleData(data))
    );
    this.subscriptions.add(
      this.route.params.subscribe(params => this.loadItem(params['id']))
    );
  }

  ngOnDestroy(): void {
    this.subscriptions.unsubscribe();
  }
}
```

---

## takeUntilDestroyed

`takeUntilDestroyed` (Angular 16+) automatically completes an Observable when the component or service is destroyed.

### In Constructor / Field Initializer (Injection Context)

```typescript
@Component({ /* ... */ })
export class NotificationsComponent {
  private notificationService = inject(NotificationService);

  // Works directly in field initializer (injection context)
  notifications = toSignal(
    this.notificationService.getStream().pipe(
      takeUntilDestroyed(),
    ),
    { initialValue: [] },
  );
}
```

### Outside Injection Context (Pass DestroyRef)

```typescript
@Component({ /* ... */ })
export class ChartComponent {
  private destroyRef = inject(DestroyRef);
  private dataService = inject(DataService);

  ngOnInit(): void {
    // Must pass destroyRef when not in injection context
    this.dataService.liveData$.pipe(
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(data => this.updateChart(data));
  }
}
```

---

## RxJS Interop with Signals

Angular provides interop functions to bridge Signals and Observables.

### toSignal — Observable → Signal

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class UserListComponent {
  private userService = inject(UserService);

  // Convert Observable to Signal
  users = toSignal(this.userService.getAll(), { initialValue: [] });
  // users() returns User[] — always has a value

  // Without initial value (signal may be undefined)
  currentUser = toSignal(this.authService.currentUser$);
  // currentUser() returns User | undefined
}
```

### toObservable — Signal → Observable

```typescript
import { toObservable } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class SearchComponent {
  query = signal('');

  // Convert signal changes to an Observable stream
  private query$ = toObservable(this.query);

  results = toSignal(
    this.query$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      filter(q => q.length > 2),
      switchMap(q => this.searchService.search(q)),
    ),
    { initialValue: [] },
  );
}
```

### When to Use Each

| Scenario | Use |
|----------|-----|
| Display Observable data in template | `toSignal()` or `async` pipe |
| Derive computed values from Observable | `toSignal()` + `computed()` |
| Apply RxJS operators to signal changes | `toObservable()` + operators |
| HTTP requests, router events | Keep as Observable, convert at component boundary |
| UI state (counters, toggles, form state) | Use `signal()` directly |

---

## Common Reactive Patterns

### Search with Debounce

```typescript
@Component({
  selector: 'app-search',
  standalone: true,
  template: `
    <input (input)="onSearch($event)" placeholder="Search..." />
    @for (result of results(); track result.id) {
      <div>{{ result.name }}</div>
    }
  `,
})
export class SearchComponent {
  private searchService = inject(SearchService);
  private searchSubject = new Subject<string>();

  results = toSignal(
    this.searchSubject.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      filter(query => query.length >= 2),
      switchMap(query => this.searchService.search(query)),
      catchError(() => of([])),
    ),
    { initialValue: [] },
  );

  onSearch(event: Event): void {
    const query = (event.target as HTMLInputElement).value;
    this.searchSubject.next(query);
  }
}
```

### Polling

```typescript
@Injectable({ providedIn: 'root' })
export class StatusService {
  private http = inject(HttpClient);

  // Poll every 30 seconds
  status$ = timer(0, 30_000).pipe(
    switchMap(() => this.http.get<SystemStatus>('/api/status')),
    retry({ count: 3, delay: 5000 }),
    shareReplay(1),
  );
}
```

### Combine Multiple Data Sources

```typescript
@Component({ /* ... */ })
export class DashboardComponent {
  private userService = inject(UserService);
  private statsService = inject(StatsService);
  private notificationService = inject(NotificationService);

  viewModel = toSignal(
    combineLatest({
      user: this.userService.currentUser$,
      stats: this.statsService.getDashboardStats(),
      notifications: this.notificationService.unread$,
    }),
  );
}
```

---

## Error Handling in Observable Streams

### catchError with Fallback

```typescript
const users$ = this.http.get<User[]>('/api/users').pipe(
  catchError(error => {
    console.error('Failed to load users:', error);
    return of([]);  // Return empty array as fallback
  }),
);
```

### Retry with Backoff

```typescript
import { retry, timer } from 'rxjs';

const resilient$ = this.http.get('/api/data').pipe(
  retry({
    count: 3,
    delay: (error, retryCount) => timer(retryCount * 1000),  // 1s, 2s, 3s
  }),
  catchError(error => {
    this.notificationService.add('Failed to load data', 'error');
    return EMPTY;
  }),
);
```

### Error State Pattern

```typescript
interface DataState<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
}

function toDataState<T>(source$: Observable<T>): Observable<DataState<T>> {
  return source$.pipe(
    map(data => ({ data, loading: false, error: null })),
    startWith({ data: null, loading: true, error: null }),
    catchError(err => of({ data: null, loading: false, error: err.message })),
  );
}

// Usage
state = toSignal(
  toDataState(this.userService.getAll()),
  { initialValue: { data: null, loading: true, error: null } },
);
```

---

## Testing Observables

### Basic Observable Testing

```typescript
describe('UserService', () => {
  it('should return users', (done) => {
    service.getAll().subscribe(users => {
      expect(users.length).toBeGreaterThan(0);
      done();
    });
  });
});
```

### Testing with fakeAsync

```typescript
it('should debounce search', fakeAsync(() => {
  const results: string[][] = [];
  component.results$.subscribe(r => results.push(r));

  component.searchSubject.next('an');   // Too short, filtered out
  tick(300);

  component.searchSubject.next('ang');  // Long enough
  tick(300);

  expect(results.length).toBe(1);
}));
```

### Testing with Marble Diagrams

```typescript
import { TestScheduler } from 'rxjs/testing';

describe('doubleValues', () => {
  let scheduler: TestScheduler;

  beforeEach(() => {
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should double each value', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const source$ = cold(' -a-b-c|', { a: 1, b: 2, c: 3 });
      const expected = '     -a-b-c|';
      const values = { a: 2, b: 4, c: 6 };

      const result$ = source$.pipe(map(v => v * 2));
      expectObservable(result$).toBe(expected, values);
    });
  });
});
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-STATE-MANAGEMENT](06-STATE-MANAGEMENT.md) | State management with RxJS and Signals |
| 2 | [07-HTTP-AND-API-INTEGRATION](07-HTTP-AND-API-INTEGRATION.md) | HTTP requests return Observables |
| 3 | [08-TESTING](08-TESTING.md) | Testing async code and Observables |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial RxJS and reactive programming documentation |
