# TypeScript for Frontend Development

## Table of Contents

1. [Overview](#overview)
2. [Why TypeScript for Frontend Development](#why-typescript-for-frontend-development)
3. [Type System Fundamentals](#type-system-fundamentals)
4. [Generics and Utility Types](#generics-and-utility-types)
5. [TypeScript with Frameworks](#typescript-with-frameworks)
6. [Declaration Files and DefinitelyTyped](#declaration-files-and-definitelytyped)
7. [Strict Mode and Compiler Options](#strict-mode-and-compiler-options)
8. [Common Patterns and Idioms](#common-patterns-and-idioms)
9. [Migration Strategies](#migration-strategies)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

---

## Overview

TypeScript is a strict superset of JavaScript that adds static type checking. It compiles to plain JavaScript and runs anywhere JavaScript runs. TypeScript has become the standard for medium-to-large frontend projects, providing compile-time safety, better tooling, and self-documenting code.

### Target Audience

- **JavaScript developers** ready to add type safety to their projects
- **Backend developers** familiar with typed languages (C#, Java, Go) learning frontend
- **Team leads** evaluating TypeScript adoption for their projects

### Scope

- Why TypeScript is valuable for frontend development
- Core type system: basic types, interfaces, type aliases, unions, intersections
- Generics and built-in utility types
- Framework-specific TypeScript usage (Angular, React, Vue)
- Declaration files and the DefinitelyTyped ecosystem
- Compiler options and strict mode
- Common patterns, idioms, and migration strategies

---

## Why TypeScript for Frontend Development

### The Case for TypeScript

| Benefit | Description |
|---------|-------------|
| **Catch bugs early** | Type errors are caught at compile time, not in production |
| **Improved IDE experience** | Autocompletion, inline docs, refactoring, go-to-definition |
| **Self-documenting code** | Types serve as living documentation for function signatures and data shapes |
| **Safer refactoring** | Rename a property and the compiler finds every usage that needs updating |
| **Team scalability** | Types define contracts between modules, reducing miscommunication |
| **Ecosystem support** | Major frameworks and libraries ship TypeScript types or are written in TypeScript |

### When TypeScript Adds the Most Value

- Projects with **more than one developer** — types are a communication tool
- **Long-lived projects** — types prevent regression during refactoring
- **Complex state management** — typed actions, reducers, and selectors prevent shape mismatches
- **API integration** — typed API responses ensure frontend and backend agree on data shapes

---

## Type System Fundamentals

### Basic Types

```typescript
// Primitives
const name: string = "Alice";
const age: number = 30;
const isActive: boolean = true;

// Arrays
const scores: number[] = [95, 87, 92];
const names: Array<string> = ["Alice", "Bob"];

// Tuple: fixed-length array with specific types per position
const coordinates: [number, number] = [40.7128, -74.006];
const entry: [string, number] = ["Alice", 30];

// Enum
enum Status {
  Pending = "PENDING",
  Active = "ACTIVE",
  Inactive = "INACTIVE",
}

// Literal types
type Direction = "north" | "south" | "east" | "west";
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
```

### Interfaces

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user" | "guest";
  createdAt: Date;
  avatar?: string; // optional property
}

// Extending interfaces
interface AdminUser extends User {
  role: "admin";
  permissions: string[];
}

// Implementing interfaces
class UserService implements UserRepository {
  async findById(id: string): Promise<User | null> {
    // implementation
  }
}
```

### Type Aliases

```typescript
// Object types
type Point = {
  x: number;
  y: number;
};

// Union types: value can be one of several types
type Result<T> = { success: true; data: T } | { success: false; error: string };

// Intersection types: value must satisfy all types
type Timestamped<T> = T & { createdAt: Date; updatedAt: Date };
type TimestampedUser = Timestamped<User>;

// Function types
type EventHandler = (event: MouseEvent) => void;
type Comparator<T> = (a: T, b: T) => number;
```

### Interfaces vs Type Aliases

| Feature | Interface | Type Alias |
|---------|-----------|-----------|
| **Object shapes** | ✅ Primary use | ✅ Works too |
| **Extends / inherits** | `interface A extends B` | `type A = B & { ... }` |
| **Declaration merging** | ✅ Interfaces merge automatically | ❌ Types don't merge |
| **Union types** | ❌ Not supported | ✅ `type A = B \| C` |
| **Mapped types** | ❌ Not supported | ✅ `type A = { [K in keyof B]: ... }` |
| **Recommendation** | Use for public APIs and class contracts | Use for unions, intersections, utility compositions |

---

## Generics and Utility Types

### Generics

```typescript
// Generic function
function first<T>(items: T[]): T | undefined {
  return items[0];
}

// Generic interface
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

// Generic constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Generic with default
interface PaginatedList<T = unknown> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
}
```

### Built-in Utility Types

| Utility Type | Description | Example |
|-------------|-------------|---------|
| `Partial<T>` | All properties optional | `Partial<User>` — for update payloads |
| `Required<T>` | All properties required | `Required<Config>` — ensure no missing fields |
| `Pick<T, K>` | Select specific properties | `Pick<User, "id" \| "name">` |
| `Omit<T, K>` | Remove specific properties | `Omit<User, "password">` |
| `Record<K, V>` | Map keys to values | `Record<string, number>` |
| `Readonly<T>` | All properties readonly | `Readonly<Config>` — immutable config |
| `ReturnType<T>` | Extract function return type | `ReturnType<typeof fetchUser>` |
| `Parameters<T>` | Extract function parameter types | `Parameters<typeof createUser>` |
| `NonNullable<T>` | Remove null and undefined | `NonNullable<string \| null>` → `string` |
| `Awaited<T>` | Unwrap Promise type | `Awaited<Promise<User>>` → `User` |

### Practical Examples

```typescript
// API payload types derived from the base model
type CreateUserPayload = Omit<User, "id" | "createdAt">;
type UpdateUserPayload = Partial<Omit<User, "id">>;
type UserSummary = Pick<User, "id" | "name" | "avatar">;

// Type-safe event map
type EventMap = {
  click: MouseEvent;
  keydown: KeyboardEvent;
  submit: SubmitEvent;
};

function on<K extends keyof EventMap>(event: K, handler: (e: EventMap[K]) => void) {
  // type-safe event handling
}
```

---

## TypeScript with Frameworks

### Angular (Native TypeScript)

Angular is written in TypeScript and uses it natively. TypeScript is required, not optional.

```typescript
import { Component, Input, Output, EventEmitter } from "@angular/core";

@Component({
  selector: "app-user-card",
  template: `
    <div class="card">
      <h2>{{ user.name }}</h2>
      <button (click)="onSelect.emit(user)">Select</button>
    </div>
  `,
})
export class UserCardComponent {
  @Input() user!: User;
  @Output() onSelect = new EventEmitter<User>();
}
```

### React with TypeScript

```typescript
import { useState, useEffect } from "react";

interface UserListProps {
  roleFilter?: string;
  onUserSelect: (user: User) => void;
}

function UserList({ roleFilter, onUserSelect }: UserListProps) {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUsers(roleFilter).then(setUsers).finally(() => setLoading(false));
  }, [roleFilter]);

  if (loading) return <div>Loading...</div>;

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id} onClick={() => onUserSelect(user)}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}
```

### Vue with TypeScript

```typescript
<script setup lang="ts">
import { ref, computed } from "vue";

interface Props {
  title: string;
  items: string[];
}

const props = defineProps<Props>();
const searchQuery = ref("");

const filteredItems = computed(() =>
  props.items.filter((item) =>
    item.toLowerCase().includes(searchQuery.value.toLowerCase())
  )
);
</script>
```

---

## Declaration Files and DefinitelyTyped

### Declaration Files (`.d.ts`)

Declaration files describe the shape of JavaScript libraries for TypeScript without containing implementation code.

```typescript
// types/analytics.d.ts
declare module "analytics-lib" {
  interface AnalyticsEvent {
    name: string;
    properties?: Record<string, string | number | boolean>;
  }

  export function track(event: AnalyticsEvent): void;
  export function identify(userId: string, traits?: Record<string, unknown>): void;
}
```

### DefinitelyTyped

The `@types/*` packages on npm come from the [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) repository — community-maintained type declarations for JavaScript packages.

```bash
# Install types for libraries that don't ship their own
npm install --save-dev @types/lodash @types/express
```

### Type Resolution Order

1. Types bundled with the package (`"types"` field in `package.json`)
2. `@types/*` package from DefinitelyTyped
3. Local declaration files in the project

---

## Strict Mode and Compiler Options

### Recommended `tsconfig.json` for Frontend

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "jsx": "react-jsx",
    "isolatedModules": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "outDir": "dist",
    "declaration": true,
    "sourceMap": true
  },
  "include": ["src"]
}
```

### What `strict: true` Enables

| Flag | What It Does |
|------|-------------|
| `strictNullChecks` | `null` and `undefined` are separate types; must be checked before use |
| `noImplicitAny` | Error when TypeScript cannot infer a type and falls back to `any` |
| `strictFunctionTypes` | Enforce contravariant function parameter types |
| `strictBindCallApply` | Type-check `bind`, `call`, and `apply` |
| `strictPropertyInitialization` | Class properties must be initialized in the constructor |
| `alwaysStrict` | Emit `"use strict"` in every output file |

**Recommendation:** Always enable `strict: true`. It is non-negotiable for production TypeScript.

---

## Common Patterns and Idioms

### Discriminated Unions (Tagged Unions)

```typescript
type LoadingState = { status: "loading" };
type SuccessState<T> = { status: "success"; data: T };
type ErrorState = { status: "error"; error: string };

type AsyncState<T> = LoadingState | SuccessState<T> | ErrorState;

function renderState(state: AsyncState<User[]>) {
  switch (state.status) {
    case "loading":
      return "Loading...";
    case "success":
      return `Found ${state.data.length} users`; // data is narrowed to User[]
    case "error":
      return `Error: ${state.error}`; // error is narrowed to string
  }
}
```

### Type Guards

```typescript
// Custom type guard
function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value
  );
}

// Using the guard
if (isUser(response.data)) {
  console.log(response.data.name); // TypeScript knows this is a User
}
```

### Branded Types

```typescript
// Prevent mixing up primitive types that represent different concepts
type UserId = string & { __brand: "UserId" };
type OrderId = string & { __brand: "OrderId" };

function createUserId(id: string): UserId {
  return id as UserId;
}

function fetchUser(id: UserId): Promise<User> { /* ... */ }
function fetchOrder(id: OrderId): Promise<Order> { /* ... */ }

// fetchUser(orderId) — Type error! OrderId is not assignable to UserId
```

---

## Migration Strategies

### Incremental Migration from JavaScript to TypeScript

| Strategy | Effort | Safety |
|----------|--------|--------|
| **Rename `.js` → `.ts`** with `allowJs: true` | Low | Low (no types yet) |
| **Add JSDoc types** to JavaScript files | Medium | Medium (type checking without renaming) |
| **Convert file by file** starting from leaf modules | Medium | High |
| **Strict mode progressively** — start with `strict: false`, enable flags one by one | Medium | Gradual |

### Step-by-Step Migration

1. **Add TypeScript** — Install `typescript`, create `tsconfig.json` with `allowJs: true`
2. **Rename entry point** — Convert one file to `.ts`, fix errors
3. **Convert leaf modules first** — Utilities, types, constants (fewest dependencies)
4. **Work inward** — Convert files that depend only on already-converted modules
5. **Enable strict flags** one at a time: `noImplicitAny` → `strictNullChecks` → `strict: true`
6. **Remove `allowJs`** when all files are converted

```bash
# Initial setup
npm install --save-dev typescript
npx tsc --init

# Convert one file at a time
mv src/utils/format.js src/utils/format.ts
npx tsc --noEmit  # check for errors
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-STATE-MANAGEMENT](05-STATE-MANAGEMENT.md) | Typed state management patterns |
| 2 | [07-TESTING](07-TESTING.md) | Testing TypeScript code |
| 3 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Code style, linting with TypeScript ESLint |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial TypeScript for frontend development documentation |
