# Angular State Management

## Table of Contents

1. [Overview](#overview)
2. [State Management Approaches](#state-management-approaches)
3. [Service-Based State with BehaviorSubject](#service-based-state-with-behaviorsubject)
4. [Signal-Based State Management](#signal-based-state-management)
5. [NgRx Store](#ngrx-store)
6. [NgRx ComponentStore](#ngrx-componentstore)
7. [NgRx SignalStore](#ngrx-signalstore)
8. [NGXS](#ngxs)
9. [Akita](#akita)
10. [Choosing the Right Approach](#choosing-the-right-approach)
11. [Server State with HTTP Caching](#server-state-with-http-caching)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

State management is how an application tracks, updates, and shares data across components. Angular offers multiple approaches — from simple service-based state to full-featured stores like NgRx. This document covers each approach with practical examples and guidance on choosing the right one.

### Scope

- When and why you need state management
- Service-based state with `BehaviorSubject`
- Signal-based state management (Angular 17+)
- NgRx Store (Redux pattern for Angular)
- NgRx ComponentStore (local/feature state)
- NgRx SignalStore (signal-based store — newest approach)
- NGXS and Akita as alternatives
- Decision framework for choosing an approach
- Server state caching patterns

### Types of State

| Type | Description | Example |
|------|-------------|---------|
| **UI state** | Visual state of components | Selected tab, modal open/closed, sidebar collapsed |
| **Client state** | Application data not persisted on server | Form drafts, user preferences, filter selections |
| **Server state** | Data fetched from APIs | User profiles, product lists, order history |
| **URL state** | State encoded in the URL | Route params, query params, fragments |
| **Form state** | Current values and validity of forms | Input values, validation errors, dirty/pristine |

---

## State Management Approaches

```
┌─────────────────────────────────────────────────────────────────┐
│              State Management Spectrum                           │
│                                                                 │
│  Simple ◄──────────────────────────────────────────► Complex    │
│                                                                 │
│  Signals     Service +        NgRx            NgRx Store        │
│  (local)     BehaviorSubject  ComponentStore  (global Redux)    │
│              or Signal        / SignalStore                     │
│                                                                 │
│  1-5          5-20             10-50           50+ components    │
│  components   components       components      large teams      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Service-Based State with BehaviorSubject

The simplest approach for shared state — a service with a `BehaviorSubject` that exposes an Observable.

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

export interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
}

@Injectable({ providedIn: 'root' })
export class CartService {
  private itemsSubject = new BehaviorSubject<CartItem[]>([]);

  readonly items$: Observable<CartItem[]> = this.itemsSubject.asObservable();

  get currentItems(): CartItem[] {
    return this.itemsSubject.value;
  }

  addItem(product: { id: string; name: string; price: number }): void {
    const items = [...this.currentItems];
    const existing = items.find(i => i.productId === product.id);

    if (existing) {
      existing.quantity += 1;
    } else {
      items.push({ productId: product.id, name: product.name, price: product.price, quantity: 1 });
    }

    this.itemsSubject.next(items);
  }

  removeItem(productId: string): void {
    const items = this.currentItems.filter(i => i.productId !== productId);
    this.itemsSubject.next(items);
  }

  getTotal(): Observable<number> {
    return this.items$.pipe(
      map(items => items.reduce((sum, item) => sum + item.price * item.quantity, 0))
    );
  }

  clear(): void {
    this.itemsSubject.next([]);
  }
}
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Simple and familiar | No built-in devtools |
| No extra dependencies | State mutations are implicit |
| Easy to test | No action history or time-travel debugging |
| Works with `async` pipe | Can become unwieldy for complex state |

---

## Signal-Based State Management

Signals (Angular 16+) provide a simpler reactive primitive for state management — no subscriptions, no memory leak risk.

### Simple Signal State

```typescript
import { Injectable, signal, computed } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CartService {
  private items = signal<CartItem[]>([]);

  readonly cartItems = this.items.asReadonly();
  readonly itemCount = computed(() => this.items().reduce((sum, i) => sum + i.quantity, 0));
  readonly total = computed(() => this.items().reduce((sum, i) => sum + i.price * i.quantity, 0));
  readonly isEmpty = computed(() => this.items().length === 0);

  addItem(product: { id: string; name: string; price: number }): void {
    this.items.update(items => {
      const existing = items.find(i => i.productId === product.id);
      if (existing) {
        return items.map(i =>
          i.productId === product.id ? { ...i, quantity: i.quantity + 1 } : i
        );
      }
      return [...items, { productId: product.id, name: product.name, price: product.price, quantity: 1 }];
    });
  }

  removeItem(productId: string): void {
    this.items.update(items => items.filter(i => i.productId !== productId));
  }

  clear(): void {
    this.items.set([]);
  }
}
```

### Using Signal State in Components

```typescript
@Component({
  selector: 'app-cart-summary',
  standalone: true,
  template: `
    <div class="cart-summary">
      <span>{{ cartService.itemCount() }} items</span>
      <span>{{ cartService.total() | currency }}</span>
    </div>
  `,
})
export class CartSummaryComponent {
  protected cartService = inject(CartService);
}
```

---

## NgRx Store

NgRx implements the **Redux pattern** for Angular — a single source of truth with unidirectional data flow. Best for large, complex applications with multiple developers.

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    NgRx Data Flow                         │
│                                                          │
│  Component ──dispatch──▶ Action ──▶ Reducer ──▶ Store    │
│       ▲                                          │       │
│       │                                          │       │
│       └──────── Selector ◄───────────────────────┘       │
│                                                          │
│  Component ──dispatch──▶ Action ──▶ Effect ──▶ Service   │
│                                       │                  │
│                                       └──▶ Action (new)  │
└──────────────────────────────────────────────────────────┘
```

### Actions

```typescript
import { createActionGroup, props, emptyProps } from '@ngrx/store';

export const UserActions = createActionGroup({
  source: 'User',
  events: {
    'Load Users': emptyProps(),
    'Load Users Success': props<{ users: User[] }>(),
    'Load Users Failure': props<{ error: string }>(),
    'Select User': props<{ userId: string }>(),
  },
});
```

### Reducer

```typescript
import { createReducer, on } from '@ngrx/store';

export interface UserState {
  users: User[];
  selectedUserId: string | null;
  loading: boolean;
  error: string | null;
}

const initialState: UserState = {
  users: [],
  selectedUserId: null,
  loading: false,
  error: null,
};

export const userReducer = createReducer(
  initialState,
  on(UserActions.loadUsers, state => ({ ...state, loading: true, error: null })),
  on(UserActions.loadUsersSuccess, (state, { users }) => ({ ...state, users, loading: false })),
  on(UserActions.loadUsersFailure, (state, { error }) => ({ ...state, error, loading: false })),
  on(UserActions.selectUser, (state, { userId }) => ({ ...state, selectedUserId: userId })),
);
```

### Selectors

```typescript
import { createFeatureSelector, createSelector } from '@ngrx/store';

export const selectUserState = createFeatureSelector<UserState>('users');
export const selectAllUsers = createSelector(selectUserState, state => state.users);
export const selectUsersLoading = createSelector(selectUserState, state => state.loading);
export const selectSelectedUser = createSelector(
  selectUserState,
  state => state.users.find(u => u.id === state.selectedUserId) ?? null,
);
```

### Effects

```typescript
import { Actions, createEffect, ofType } from '@ngrx/effects';

@Injectable()
export class UserEffects {
  private actions$ = inject(Actions);
  private userService = inject(UserService);

  loadUsers$ = createEffect(() =>
    this.actions$.pipe(
      ofType(UserActions.loadUsers),
      switchMap(() =>
        this.userService.getAll().pipe(
          map(users => UserActions.loadUsersSuccess({ users })),
          catchError(error => of(UserActions.loadUsersFailure({ error: error.message }))),
        )
      ),
    )
  );
}
```

### Component Usage

```typescript
@Component({
  selector: 'app-user-list',
  standalone: true,
  template: `
    @if (loading()) {
      <app-spinner />
    } @else {
      @for (user of users(); track user.id) {
        <div (click)="selectUser(user.id)">{{ user.name }}</div>
      }
    }
  `,
})
export class UserListComponent implements OnInit {
  private store = inject(Store);

  users = this.store.selectSignal(selectAllUsers);
  loading = this.store.selectSignal(selectUsersLoading);

  ngOnInit(): void {
    this.store.dispatch(UserActions.loadUsers());
  }

  selectUser(userId: string): void {
    this.store.dispatch(UserActions.selectUser({ userId }));
  }
}
```

---

## NgRx ComponentStore

ComponentStore is for **local/feature-level state** — simpler than the global Store, scoped to a component or feature.

```typescript
import { Injectable } from '@angular/core';
import { ComponentStore } from '@ngrx/component-store';
import { tapResponse } from '@ngrx/operators';

interface TodoState {
  todos: Todo[];
  loading: boolean;
}

@Injectable()
export class TodoStore extends ComponentStore<TodoState> {
  constructor() {
    super({ todos: [], loading: false });
  }

  // Selectors
  readonly todos = this.selectSignal(state => state.todos);
  readonly loading = this.selectSignal(state => state.loading);
  readonly completedCount = this.selectSignal(state => state.todos.filter(t => t.done).length);

  // Updaters (synchronous state changes)
  readonly addTodo = this.updater((state, todo: Todo) => ({
    ...state,
    todos: [...state.todos, todo],
  }));

  readonly toggleTodo = this.updater((state, id: string) => ({
    ...state,
    todos: state.todos.map(t => t.id === id ? { ...t, done: !t.done } : t),
  }));

  // Effects (async operations)
  readonly loadTodos = this.effect<void>(trigger$ =>
    trigger$.pipe(
      tap(() => this.patchState({ loading: true })),
      switchMap(() =>
        this.todoService.getAll().pipe(
          tapResponse(
            todos => this.patchState({ todos, loading: false }),
            error => {
              console.error(error);
              this.patchState({ loading: false });
            },
          ),
        ),
      ),
    ),
  );

  private todoService = inject(TodoService);
}
```

```typescript
// Component scoped to this store
@Component({
  selector: 'app-todo-list',
  standalone: true,
  providers: [TodoStore],  // Each instance gets its own store
  template: `<!-- ... -->`,
})
export class TodoListComponent {
  protected store = inject(TodoStore);
}
```

---

## NgRx SignalStore

NgRx SignalStore is the **newest approach** (NgRx 17+), combining the power of NgRx with Angular signals.

```typescript
import { signalStore, withState, withComputed, withMethods, patchState } from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';

interface TodoState {
  todos: Todo[];
  loading: boolean;
  filter: 'all' | 'active' | 'done';
}

const initialState: TodoState = {
  todos: [],
  loading: false,
  filter: 'all',
};

export const TodoStore = signalStore(
  { providedIn: 'root' },
  withState(initialState),

  withComputed(({ todos, filter }) => ({
    filteredTodos: computed(() => {
      const all = todos();
      switch (filter()) {
        case 'active': return all.filter(t => !t.done);
        case 'done': return all.filter(t => t.done);
        default: return all;
      }
    }),
    totalCount: computed(() => todos().length),
    doneCount: computed(() => todos().filter(t => t.done).length),
  })),

  withMethods((store, todoService = inject(TodoService)) => ({
    addTodo(title: string): void {
      const todo: Todo = { id: crypto.randomUUID(), title, done: false };
      patchState(store, { todos: [...store.todos(), todo] });
    },

    toggleTodo(id: string): void {
      patchState(store, {
        todos: store.todos().map(t => t.id === id ? { ...t, done: !t.done } : t),
      });
    },

    setFilter(filter: 'all' | 'active' | 'done'): void {
      patchState(store, { filter });
    },

    loadTodos: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { loading: true })),
        switchMap(() =>
          todoService.getAll().pipe(
            tapResponse(
              todos => patchState(store, { todos, loading: false }),
              () => patchState(store, { loading: false }),
            ),
          ),
        ),
      ),
    ),
  })),
);
```

```typescript
@Component({
  selector: 'app-todo-page',
  standalone: true,
  template: `
    <h1>Todos ({{ store.doneCount() }}/{{ store.totalCount() }})</h1>
    @for (todo of store.filteredTodos(); track todo.id) {
      <div (click)="store.toggleTodo(todo.id)" [class.done]="todo.done">
        {{ todo.title }}
      </div>
    }
  `,
})
export class TodoPageComponent implements OnInit {
  protected store = inject(TodoStore);

  ngOnInit(): void {
    this.store.loadTodos();
  }
}
```

---

## NGXS

NGXS is a simpler alternative to NgRx that uses classes and decorators:

```typescript
import { State, Action, StateContext, Selector } from '@ngxs/store';

// Actions
export class LoadUsers { static readonly type = '[Users] Load'; }
export class LoadUsersSuccess {
  static readonly type = '[Users] Load Success';
  constructor(public users: User[]) {}
}

// State
export interface UsersStateModel {
  users: User[];
  loading: boolean;
}

@State<UsersStateModel>({
  name: 'users',
  defaults: { users: [], loading: false },
})
@Injectable()
export class UsersState {
  @Selector()
  static users(state: UsersStateModel): User[] {
    return state.users;
  }

  @Action(LoadUsers)
  loadUsers(ctx: StateContext<UsersStateModel>): Observable<void> {
    ctx.patchState({ loading: true });
    return this.userService.getAll().pipe(
      tap(users => ctx.dispatch(new LoadUsersSuccess(users))),
    );
  }

  @Action(LoadUsersSuccess)
  loadUsersSuccess(ctx: StateContext<UsersStateModel>, { users }: LoadUsersSuccess): void {
    ctx.patchState({ users, loading: false });
  }

  constructor(private userService: UserService) {}
}
```

---

## Akita

Akita uses an entity-store pattern with less boilerplate:

```typescript
import { EntityState, EntityStore, QueryEntity } from '@datorama/akita';

// Store
export interface UsersState extends EntityState<User, string> {
  loading: boolean;
}

@Injectable({ providedIn: 'root' })
export class UsersStore extends EntityStore<UsersState> {
  constructor() {
    super({ loading: false });
  }
}

// Query
@Injectable({ providedIn: 'root' })
export class UsersQuery extends QueryEntity<UsersState> {
  loading$ = this.select('loading');
  constructor(protected override store: UsersStore) {
    super(store);
  }
}

// Service
@Injectable({ providedIn: 'root' })
export class UsersService {
  private http = inject(HttpClient);
  private store = inject(UsersStore);

  loadUsers(): Observable<User[]> {
    this.store.update({ loading: true });
    return this.http.get<User[]>('/api/users').pipe(
      tap(users => {
        this.store.set(users);
        this.store.update({ loading: false });
      }),
    );
  }
}
```

---

## Choosing the Right Approach

| Criteria | Signals / Service | NgRx ComponentStore | NgRx SignalStore | NgRx Store | NGXS |
|----------|-------------------|--------------------|--------------------|------------|------|
| **Complexity** | Low | Low-Medium | Medium | High | Medium |
| **Learning curve** | Minimal | Low | Low | Steep | Moderate |
| **Boilerplate** | Minimal | Low | Low | High | Moderate |
| **DevTools** | None | Limited | Limited | Full (Redux DevTools) | Full |
| **Best for** | Small-medium apps | Feature/component state | Medium-large apps | Large enterprise apps | Medium-large apps |
| **Team size** | 1-3 devs | 1-5 devs | 2-8 devs | 5+ devs | 3-8 devs |
| **Testing** | Easy | Easy | Easy | Comprehensive | Easy |

### Decision Tree

```
Do you have complex, shared state across many components?
├── No → Use signal-based service state
└── Yes
    ├── Is the state scoped to a feature/component?
    │   ├── Yes → Use NgRx ComponentStore or SignalStore
    │   └── No → Is it a large enterprise app with strict patterns needed?
    │       ├── Yes → Use NgRx Store (full Redux)
    │       └── No → Use NgRx SignalStore or signal-based services
```

---

## Server State with HTTP Caching

### Simple Cache with shareReplay

```typescript
@Injectable({ providedIn: 'root' })
export class ConfigService {
  private http = inject(HttpClient);

  // Cached — all subscribers share the same HTTP response
  readonly config$ = this.http.get<AppConfig>('/api/config').pipe(
    shareReplay(1),
  );
}
```

### Signal-Based Cache with Refresh

```typescript
@Injectable({ providedIn: 'root' })
export class UserCacheService {
  private http = inject(HttpClient);
  private cache = signal<Map<string, User>>(new Map());

  getUser(id: string): Signal<User | undefined> {
    if (!this.cache().has(id)) {
      this.fetchUser(id);
    }
    return computed(() => this.cache().get(id));
  }

  private fetchUser(id: string): void {
    this.http.get<User>(`/api/users/${id}`).subscribe(user => {
      this.cache.update(map => new Map(map).set(id, user));
    });
  }

  invalidate(id: string): void {
    this.cache.update(map => {
      const next = new Map(map);
      next.delete(id);
      return next;
    });
  }
}
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-RXJS-AND-OBSERVABLES](05-RXJS-AND-OBSERVABLES.md) | RxJS fundamentals for state management |
| 2 | [07-HTTP-AND-API-INTEGRATION](07-HTTP-AND-API-INTEGRATION.md) | Server state and API integration |
| 3 | [09-PERFORMANCE](09-PERFORMANCE.md) | Performance implications of state management |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Angular state management documentation |
