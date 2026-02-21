# State Management Patterns

## Table of Contents

1. [Overview](#overview)
2. [What Is State and Why It Matters](#what-is-state-and-why-it-matters)
3. [Local vs Global State](#local-vs-global-state)
4. [Props Drilling and Lifting State Up](#props-drilling-and-lifting-state-up)
5. [Flux Architecture and Redux Pattern](#flux-architecture-and-redux-pattern)
6. [Reactive State and Signals](#reactive-state-and-signals)
7. [Server State Management](#server-state-management)
8. [URL State and Routing](#url-state-and-routing)
9. [State Machines](#state-machines)
10. [Choosing the Right Approach](#choosing-the-right-approach)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

State management is one of the most important — and most debated — topics in frontend development. Every interactive application must track data that changes over time: user input, fetched data, UI toggles, authentication status, and more. How you organize and manage that data has a direct impact on your application's reliability, maintainability, and performance.

### Scope

- What state is and how to categorize it
- Local component state vs global application state
- Common patterns: props drilling, lifting state, context
- Flux/Redux pattern and its variants
- Reactive state with signals (Angular, Solid, Vue, Preact)
- Server state management with TanStack Query and SWR
- URL state and routing as state management
- State machines with XState
- Decision framework for choosing the right approach

---

## What Is State and Why It Matters

**State** is any data that can change over time and affects what the UI renders or how it behaves.

### Categories of State

| Category | Description | Examples |
|----------|-------------|---------|
| **UI State** | Local visual state of components | Modal open/closed, accordion expanded, form input values |
| **Application State** | Shared data across multiple components | Current user, theme, language, feature flags |
| **Server State** | Data fetched from an API | User list, product catalog, notifications |
| **URL State** | State encoded in the URL | Current page, search filters, sort order |
| **Form State** | Input values, validation, submission status | Login form, checkout form, search bar |

### Why State Management Matters

```
Simple app (few components, little shared state):
  Component state is enough. No library needed.

Medium app (shared state across features):
  Need a pattern: Context API, signals, or lightweight store.

Complex app (many features, async data, offline support):
  Need a strategy: Redux/NgRx, TanStack Query, state machines, or a combination.
```

---

## Local vs Global State

### Local State

State that belongs to a single component and is not needed elsewhere.

```javascript
// React: useState
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

```typescript
// Angular: component property
@Component({
  template: `<button (click)="count = count + 1">Count: {{ count }}</button>`,
})
export class CounterComponent {
  count = 0;
}
```

### Global State

State shared across multiple components, often unrelated in the component tree.

```
When to use global state:
✅ User authentication (needed everywhere)
✅ Theme / dark mode (affects entire app)
✅ Shopping cart (header badge + cart page + checkout)
✅ Notification queue (triggered anywhere, displayed globally)

When NOT to use global state:
❌ Form input values (local to the form)
❌ Modal open/closed (local to the component)
❌ Hover state (local to the element)
```

---

## Props Drilling and Lifting State Up

### Props Drilling

Props drilling is passing data through multiple intermediate components that don't use it — they just forward it to children.

```
   App (owns user state)
    │
    └── Layout (passes user down)
         │
         └── Sidebar (passes user down)
              │
              └── UserMenu (finally uses user) ← needs user
```

**Problem:** Every intermediate component must accept and pass the prop, even if it doesn't care about it. This creates tight coupling and maintenance burden.

### Lifting State Up

When two sibling components need the same state, move the state to their closest common parent.

```javascript
// Parent owns the state, passes it to both children
function ProductPage() {
  const [selectedColor, setSelectedColor] = useState("red");

  return (
    <>
      <ColorPicker selected={selectedColor} onChange={setSelectedColor} />
      <ProductPreview color={selectedColor} />
    </>
  );
}
```

### Solutions to Props Drilling

| Solution | Framework | Use Case |
|----------|-----------|----------|
| **Context API** | React | Low-frequency updates (theme, locale, auth) |
| **Dependency Injection** | Angular | Services shared across components |
| **Provide/Inject** | Vue | Ancestor-to-descendant data passing |
| **State management library** | Any | Complex shared state with frequent updates |

---

## Flux Architecture and Redux Pattern

### The Flux Pattern

Flux introduced **unidirectional data flow** to frontend state management:

```
┌──────────┐     ┌────────────┐     ┌─────────┐     ┌──────────┐
│  Action   │────▶│ Dispatcher │────▶│  Store  │────▶│   View   │
│ (event)   │     │ (router)   │     │ (state) │     │ (render) │
└──────────┘     └────────────┘     └─────────┘     └────┬─────┘
      ▲                                                   │
      └───────────────────────────────────────────────────┘
                         User interaction
```

### Redux Pattern

Redux simplified Flux with a single store and pure reducer functions:

```typescript
// 1. Define action types
type Action =
  | { type: "ADD_TODO"; payload: { text: string } }
  | { type: "TOGGLE_TODO"; payload: { id: string } }
  | { type: "REMOVE_TODO"; payload: { id: string } };

// 2. Define state shape
interface TodoState {
  items: Array<{ id: string; text: string; completed: boolean }>;
}

// 3. Pure reducer function: (state, action) => newState
function todoReducer(state: TodoState, action: Action): TodoState {
  switch (action.type) {
    case "ADD_TODO":
      return {
        items: [
          ...state.items,
          { id: crypto.randomUUID(), text: action.payload.text, completed: false },
        ],
      };
    case "TOGGLE_TODO":
      return {
        items: state.items.map((item) =>
          item.id === action.payload.id
            ? { ...item, completed: !item.completed }
            : item
        ),
      };
    case "REMOVE_TODO":
      return {
        items: state.items.filter((item) => item.id !== action.payload.id),
      };
  }
}
```

### Framework-Specific Implementations

| Library | Framework | Key Features |
|---------|-----------|-------------|
| **Redux Toolkit** | React | Simplified Redux with `createSlice`, RTK Query |
| **NgRx** | Angular | RxJS-based Redux for Angular with effects and selectors |
| **Pinia** | Vue | Lightweight, TypeScript-first store for Vue 3 |
| **Vuex** | Vue 2 | Flux-inspired state management (legacy, use Pinia for Vue 3) |

---

## Reactive State and Signals

Signals are a reactive primitive that has gained significant traction across frameworks. A signal holds a value and automatically notifies consumers when it changes.

### Signals Across Frameworks

```typescript
// Angular Signals
import { signal, computed, effect } from "@angular/core";

const count = signal(0);
const doubled = computed(() => count() * 2);

effect(() => {
  console.log(`Count is ${count()}, doubled is ${doubled()}`);
});

count.set(5);      // Logs: Count is 5, doubled is 10
count.update(c => c + 1); // Logs: Count is 6, doubled is 12
```

```typescript
// Vue Reactivity (ref / computed)
import { ref, computed, watchEffect } from "vue";

const count = ref(0);
const doubled = computed(() => count.value * 2);

watchEffect(() => {
  console.log(`Count is ${count.value}, doubled is ${doubled.value}`);
});

count.value = 5;
```

```typescript
// Solid.js Signals
import { createSignal, createMemo, createEffect } from "solid-js";

const [count, setCount] = createSignal(0);
const doubled = createMemo(() => count() * 2);

createEffect(() => {
  console.log(`Count is ${count()}, doubled is ${doubled()}`);
});

setCount(5);
```

### Why Signals Are Gaining Adoption

| Advantage | Description |
|-----------|-------------|
| **Fine-grained reactivity** | Only the specific DOM nodes that depend on a signal update |
| **No virtual DOM diffing** | Updates are surgical, not tree-wide comparisons |
| **Simple mental model** | Value in, value out — no reducers, actions, or middleware |
| **Composable** | Computed signals derive from other signals automatically |

---

## Server State Management

Server state (data fetched from APIs) has fundamentally different characteristics than client state:

| Characteristic | Client State | Server State |
|---------------|-------------|-------------|
| **Ownership** | Controlled by the frontend | Controlled by the backend |
| **Staleness** | Always current | May be stale the moment it's received |
| **Caching** | Usually in-memory | Needs smart caching with invalidation |
| **Deduplication** | Not an issue | Same data may be requested by multiple components |

### TanStack Query (React Query)

```typescript
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

function UserList() {
  const { data: users, isLoading, error } = useQuery({
    queryKey: ["users"],
    queryFn: () => fetch("/api/users").then((r) => r.json()),
    staleTime: 5 * 60 * 1000, // Consider data fresh for 5 minutes
  });

  const queryClient = useQueryClient();

  const createUser = useMutation({
    mutationFn: (newUser) =>
      fetch("/api/users", {
        method: "POST",
        body: JSON.stringify(newUser),
        headers: { "Content-Type": "application/json" },
      }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["users"] });
    },
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### SWR

```typescript
import useSWR from "swr";

const fetcher = (url: string) => fetch(url).then((r) => r.json());

function UserProfile({ userId }: { userId: string }) {
  const { data, error, isLoading } = useSWR(`/api/users/${userId}`, fetcher);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading user</div>;

  return <div>{data.name}</div>;
}
```

---

## URL State and Routing

The URL is a powerful state container that is often overlooked. URL state is:

- **Shareable** — users can bookmark or share a link that restores exact application state
- **Persistent** — survives page refresh without any storage code
- **Browser-integrated** — back/forward navigation works automatically

### What Belongs in the URL

| ✅ Put in URL | ❌ Keep elsewhere |
|--------------|------------------|
| Current page/route | Transient UI state (modal open/closed) |
| Search query and filters | Form input before submission |
| Sort order | Animation state |
| Pagination (`?page=3`) | Ephemeral loading indicators |
| Selected tab | Session-only data |

### URL State Example

```typescript
// React Router: read and write URL search params
import { useSearchParams } from "react-router-dom";

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();

  const category = searchParams.get("category") ?? "all";
  const sort = searchParams.get("sort") ?? "name";
  const page = Number(searchParams.get("page") ?? "1");

  function updateFilters(updates: Record<string, string>) {
    setSearchParams((prev) => {
      const next = new URLSearchParams(prev);
      Object.entries(updates).forEach(([key, value]) => next.set(key, value));
      return next;
    });
  }

  return (
    <div>
      <select
        value={category}
        onChange={(e) => updateFilters({ category: e.target.value, page: "1" })}
      >
        <option value="all">All</option>
        <option value="electronics">Electronics</option>
        <option value="books">Books</option>
      </select>
    </div>
  );
}
```

---

## State Machines

State machines model state as a finite set of states with explicit transitions between them. This eliminates impossible states and makes complex flows predictable.

### When to Use State Machines

- Multi-step forms and wizards
- Authentication flows (idle → authenticating → authenticated → error)
- Media players (stopped → playing → paused → buffering)
- Complex UI interactions with many conditional states

### XState Example

```typescript
import { createMachine, assign } from "xstate";

const authMachine = createMachine({
  id: "auth",
  initial: "idle",
  context: { user: null, error: null },
  states: {
    idle: {
      on: { LOGIN: "authenticating" },
    },
    authenticating: {
      invoke: {
        src: "loginService",
        onDone: {
          target: "authenticated",
          actions: assign({ user: (_, event) => event.data }),
        },
        onError: {
          target: "error",
          actions: assign({ error: (_, event) => event.data.message }),
        },
      },
    },
    authenticated: {
      on: { LOGOUT: "idle" },
    },
    error: {
      on: {
        RETRY: "authenticating",
        RESET: "idle",
      },
    },
  },
});
```

---

## Choosing the Right Approach

| State Type | Recommended Approach |
|-----------|---------------------|
| **Local UI state** (toggle, form input) | Component state (`useState`, signals, component properties) |
| **Shared UI state** (theme, sidebar) | Context / provide-inject / lightweight store |
| **Server data** (API responses) | TanStack Query, SWR, Apollo Client |
| **Complex async flows** (multi-step, branching) | State machines (XState) |
| **Large-scale app state** (enterprise apps) | Redux Toolkit, NgRx, Pinia |
| **URL-representable state** (filters, pagination) | URL search params + router |

### Decision Flow

```
Is the state used by a single component only?
  YES → Local component state (useState / signal / property)
  NO  ↓

Is the state server data (fetched from an API)?
  YES → TanStack Query / SWR / Apollo
  NO  ↓

Should the state persist in the URL?
  YES → URL search params
  NO  ↓

Is the state a complex flow with many transitions?
  YES → State machine (XState)
  NO  ↓

Is the state shared by a few nearby components?
  YES → Lift state up or Context / provide-inject
  NO  → Global store (Redux Toolkit / NgRx / Pinia)
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-PERFORMANCE](06-PERFORMANCE.md) | Optimizing re-renders and state updates |
| 2 | [07-TESTING](07-TESTING.md) | Testing stateful components and stores |
| 3 | [11-ANTI-PATTERNS](11-ANTI-PATTERNS.md) | Over-engineering state management |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial state management patterns documentation |
