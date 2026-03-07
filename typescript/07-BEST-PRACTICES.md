# TypeScript Best Practices

A comprehensive guide to writing safe, maintainable, and production-ready TypeScript — covering strict mode, type safety, error handling, state modeling, API boundaries, naming conventions, module organization, and a production checklist.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Enable Strict Mode](#enable-strict-mode)
   - [The strict Flag and Its Sub-Options](#the-strict-flag-and-its-sub-options)
   - [Additional Strictness Beyond strict](#additional-strictness-beyond-strict)
3. [Prefer Interfaces Over Type Aliases for Object Shapes](#prefer-interfaces-over-type-aliases-for-object-shapes)
   - [When to Use Interfaces](#when-to-use-interfaces)
   - [When to Use Type Aliases](#when-to-use-type-aliases)
4. [Use unknown Instead of any](#use-unknown-instead-of-any)
5. [Use readonly and const Assertions](#use-readonly-and-const-assertions)
6. [Proper Error Handling](#proper-error-handling)
   - [Result Types](#result-types)
   - [Never Throw Untyped Errors](#never-throw-untyped-errors)
7. [Exhaustive Pattern Matching with never](#exhaustive-pattern-matching-with-never)
8. [Avoid Type Assertions — Prefer Type Narrowing](#avoid-type-assertions--prefer-type-narrowing)
9. [Use Discriminated Unions for State Modeling](#use-discriminated-unions-for-state-modeling)
10. [Proper Null Handling](#proper-null-handling)
    - [strictNullChecks](#strictnullchecks)
    - [Optional Chaining and Nullish Coalescing](#optional-chaining-and-nullish-coalescing)
11. [Type-Safe API Boundaries](#type-safe-api-boundaries)
    - [Runtime Validation with Zod](#runtime-validation-with-zod)
    - [Runtime Validation with io-ts](#runtime-validation-with-io-ts)
12. [Naming Conventions for Types and Interfaces](#naming-conventions-for-types-and-interfaces)
13. [Module Organization and Barrel Exports](#module-organization-and-barrel-exports)
    - [Recommended Structure](#recommended-structure)
    - [Barrel Export Pitfalls](#barrel-export-pitfalls)
14. [Avoid Premature Abstraction with Generics](#avoid-premature-abstraction-with-generics)
15. [Production Checklist](#production-checklist)
16. [Next Steps](#next-steps)
17. [Version History](#version-history)

---

## Overview

TypeScript gives you a powerful type system, but the type system alone does not guarantee safe or maintainable code. How you configure the compiler, model your data, handle errors, and organize your modules determines whether TypeScript acts as a safety net or a false sense of security.

This guide distills the most impactful best practices into actionable directives with code examples. Each section explains what to do, why it matters, and shows the concrete TypeScript code to adopt. Practices are ordered from foundational (compiler configuration) to advanced (runtime validation, production checklists).

### Target Audience

- **TypeScript developers** — looking to move beyond "it compiles" toward truly safe, idiomatic code
- **Tech leads and architects** — establishing coding standards and review guidelines for TypeScript projects
- **Teams migrating to TypeScript** — needing a clear set of rules to adopt from day one rather than retrofitting later
- **Backend and frontend engineers** — applying best practices across Node.js APIs, React apps, and shared libraries

### Scope

- Compiler strictness configuration and additional safety flags
- Type design — interfaces vs. type aliases, discriminated unions, readonly data
- Error handling patterns — Result types, exhaustive matching, typed errors
- Null safety — strictNullChecks, optional chaining, nullish coalescing
- Runtime validation at API boundaries with Zod and io-ts
- Naming conventions, module organization, and barrel exports
- Avoiding common pitfalls with generics, type assertions, and `any`
- Production readiness checklist for TypeScript projects

---

## Enable Strict Mode

### The strict Flag and Its Sub-Options

The single most impactful thing you can do for a TypeScript project is enable `strict` mode. The `strict` flag is an umbrella that turns on all of the following sub-options:

```json
{
  "compilerOptions": {
    "strict": true
  }
}
```

This is equivalent to enabling every one of these individually:

```json
{
  "compilerOptions": {
    "noImplicitAny": true,
    "noImplicitThis": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "alwaysStrict": true,
    "useUnknownInCatchVariables": true
  }
}
```

**Why it matters:** Without `strict`, TypeScript silently allows `null` where you expect a string, implicit `any` types that bypass all checking, and uninitialized class properties. These are the exact categories of bugs that TypeScript was designed to catch.

> **Rule:** Every new project should start with `"strict": true`. Every existing project should have a migration plan to enable it.

### Additional Strictness Beyond strict

The `strict` flag does not cover everything. Add these additional flags for maximum safety:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true
  }
}
```

**Key additions explained:**

- **`noUncheckedIndexedAccess`** — Array and object index access returns `T | undefined` instead of `T`, forcing you to handle the case where the element does not exist.
- **`noImplicitOverride`** — Requires the `override` keyword when overriding a base class method, preventing accidental method name collisions.
- **`exactOptionalPropertyTypes`** — Distinguishes between a property that is `undefined` and a property that is missing entirely.

```typescript
// With noUncheckedIndexedAccess enabled
const items = ["a", "b", "c"];
const first = items[0]; // type is string | undefined — you MUST check before using

if (first !== undefined) {
  console.log(first.toUpperCase()); // safe
}
```

---

## Prefer Interfaces Over Type Aliases for Object Shapes

### When to Use Interfaces

Use `interface` for object shapes that represent entities, contracts, or any structure that other code will implement or extend:

```typescript
// Good — use interface for object shapes
interface User {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
}

interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<boolean>;
}

// Interfaces support declaration merging and extension
interface User {
  role: "admin" | "member"; // merged into the User interface above
}

interface AdminUser extends User {
  permissions: string[];
}
```

**Why interfaces are preferred for objects:**

- **Declaration merging** — Interfaces can be extended across files, which is essential for augmenting third-party types.
- **Better error messages** — TypeScript reports interface names directly; type alias errors can be opaque.
- **Extends is explicit** — `extends` clearly communicates inheritance; intersection types (`&`) can produce confusing results with conflicting properties.

### When to Use Type Aliases

Use `type` for unions, intersections, mapped types, conditional types, tuples, and function signatures:

```typescript
// Good — use type for unions and computed types
type Status = "pending" | "active" | "suspended";
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

// Good — use type for function signatures
type EventHandler<T> = (event: T) => void;

// Good — use type for intersections and mapped types
type WithTimestamps<T> = T & {
  createdAt: Date;
  updatedAt: Date;
};

// Good — use type for tuples
type Coordinate = [latitude: number, longitude: number];

// Good — use type for conditional types
type NonNullableFields<T> = {
  [K in keyof T]: NonNullable<T[K]>;
};
```

> **Rule:** Use `interface` for object shapes and contracts. Use `type` for everything else — unions, tuples, mapped types, function types, and computed types.

---

## Use unknown Instead of any

The `any` type disables all type checking. The `unknown` type is the type-safe counterpart — it accepts any value but requires you to narrow the type before using it:

```typescript
// Bad — any disables all checking
function parseJSON(input: string): any {
  return JSON.parse(input);
}
const data = parseJSON('{"name": "Alice"}');
data.nonExistent.method(); // no error at compile time, runtime crash

// Good — unknown forces you to validate
function parseJSON(input: string): unknown {
  return JSON.parse(input);
}
const data = parseJSON('{"name": "Alice"}');

// You must narrow before using
if (typeof data === "object" && data !== null && "name" in data) {
  console.log((data as { name: string }).name); // safe after validation
}
```

**Common patterns for narrowing `unknown`:**

```typescript
// Type guard function
function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "email" in value &&
    typeof (value as User).id === "string" &&
    typeof (value as User).email === "string"
  );
}

// Usage
const parsed: unknown = JSON.parse(rawInput);
if (isUser(parsed)) {
  // parsed is narrowed to User here
  console.log(parsed.email);
}
```

> **Rule:** Never use `any` in application code. Use `unknown` and narrow with type guards, `typeof`, `instanceof`, or runtime validation libraries.

---

## Use readonly and const Assertions

Immutable data is easier to reason about and less prone to accidental mutation bugs. TypeScript provides `readonly` properties, `ReadonlyArray`, and `as const` assertions:

```typescript
// readonly properties — prevent mutation after construction
interface Config {
  readonly apiUrl: string;
  readonly timeout: number;
  readonly retries: number;
}

const config: Config = {
  apiUrl: "https://api.example.com",
  timeout: 5000,
  retries: 3,
};
// config.timeout = 10000; // Error: Cannot assign to 'timeout' because it is a read-only property

// ReadonlyArray — prevent push, pop, splice
function processItems(items: ReadonlyArray<string>): void {
  // items.push("new"); // Error: Property 'push' does not exist
  items.forEach((item) => console.log(item)); // reading is fine
}

// const assertions — infer literal types and make everything readonly
const HTTP_METHODS = ["GET", "POST", "PUT", "DELETE"] as const;
// type is readonly ["GET", "POST", "PUT", "DELETE"]
type HttpMethod = (typeof HTTP_METHODS)[number]; // "GET" | "POST" | "PUT" | "DELETE"

const ERROR_CODES = {
  NOT_FOUND: 404,
  UNAUTHORIZED: 401,
  INTERNAL: 500,
} as const;
// All properties are readonly with literal types
type ErrorCode = (typeof ERROR_CODES)[keyof typeof ERROR_CODES]; // 404 | 401 | 500
```

> **Rule:** Default to `readonly` for interface properties. Use `as const` for configuration objects and literal value arrays. Only make properties mutable when mutation is explicitly needed.

---

## Proper Error Handling

### Result Types

Instead of throwing exceptions, return errors as values. This makes error handling explicit and type-safe:

```typescript
// Define a Result type
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

// Use it in functions
async function fetchUser(id: string): Promise<Result<User, "NOT_FOUND" | "NETWORK_ERROR">> {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (response.status === 404) {
      return { success: false, error: "NOT_FOUND" };
    }
    if (!response.ok) {
      return { success: false, error: "NETWORK_ERROR" };
    }
    const data = await response.json();
    return { success: true, data: data as User };
  } catch {
    return { success: false, error: "NETWORK_ERROR" };
  }
}

// Callers are forced to handle both cases
const result = await fetchUser("123");
if (result.success) {
  console.log(result.data.name); // type-safe access to User
} else {
  console.error(result.error); // type is "NOT_FOUND" | "NETWORK_ERROR"
}
```

### Never Throw Untyped Errors

When you must throw, use typed error classes:

```typescript
// Define domain-specific error classes
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number,
  ) {
    super(message);
    this.name = "AppError";
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`, "NOT_FOUND", 404);
    this.name = "NotFoundError";
  }
}

class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly fields: Record<string, string>,
  ) {
    super(message, "VALIDATION_ERROR", 400);
    this.name = "ValidationError";
  }
}

// Use instanceof for type-safe catch blocks
try {
  await updateUser(userId, data);
} catch (error) {
  if (error instanceof NotFoundError) {
    return res.status(404).json({ error: error.message });
  }
  if (error instanceof ValidationError) {
    return res.status(400).json({ error: error.message, fields: error.fields });
  }
  throw error; // re-throw unknown errors
}
```

> **Rule:** Prefer Result types for expected failures. When throwing is necessary, throw typed error classes — never throw raw strings or plain objects.

---

## Exhaustive Pattern Matching with never

Use the `never` type to ensure that switch statements and if-else chains handle every variant of a union type. If a new variant is added, the compiler will report an error at every unhandled location:

```typescript
type PaymentMethod =
  | { type: "credit_card"; cardNumber: string; expiry: string }
  | { type: "bank_transfer"; accountNumber: string; routingNumber: string }
  | { type: "crypto"; walletAddress: string };

function processPayment(method: PaymentMethod): string {
  switch (method.type) {
    case "credit_card":
      return `Charging card ending in ${method.cardNumber.slice(-4)}`;
    case "bank_transfer":
      return `Transferring from account ${method.accountNumber}`;
    case "crypto":
      return `Sending to wallet ${method.walletAddress}`;
    default: {
      // Exhaustiveness check — if a new variant is added to PaymentMethod
      // and not handled above, this line produces a compile error
      const _exhaustive: never = method;
      throw new Error(`Unhandled payment method: ${JSON.stringify(_exhaustive)}`);
    }
  }
}

// Helper function for reuse
function assertNever(value: never, message?: string): never {
  throw new Error(message ?? `Unexpected value: ${JSON.stringify(value)}`);
}

// Usage with the helper
function getPaymentIcon(method: PaymentMethod): string {
  switch (method.type) {
    case "credit_card":
      return "💳";
    case "bank_transfer":
      return "🏦";
    case "crypto":
      return "₿";
    default:
      return assertNever(method);
  }
}
```

> **Rule:** Every switch or if-else chain over a union type should include an exhaustiveness check using `never`. This turns forgotten cases into compile-time errors instead of runtime bugs.

---

## Avoid Type Assertions — Prefer Type Narrowing

Type assertions (`as`) tell the compiler "trust me" — they bypass type checking and can hide bugs. Prefer type narrowing, which proves correctness:

```typescript
// Bad — type assertion hides potential bugs
const input: unknown = getExternalData();
const user = input as User; // no runtime check — crashes if input is not a User

// Good — type guard narrows safely
function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) return false;
  const obj = value as Record<string, unknown>;
  return (
    typeof obj.id === "string" &&
    typeof obj.email === "string" &&
    typeof obj.name === "string"
  );
}

const input: unknown = getExternalData();
if (isUser(input)) {
  console.log(input.name); // safely narrowed to User
}

// Good — discriminated union narrows automatically
type ApiResponse =
  | { status: "success"; data: User }
  | { status: "error"; message: string };

function handleResponse(response: ApiResponse): void {
  if (response.status === "success") {
    // TypeScript narrows to { status: "success"; data: User }
    console.log(response.data.name);
  } else {
    // TypeScript narrows to { status: "error"; message: string }
    console.error(response.message);
  }
}
```

**When type assertions are acceptable:**

```typescript
// Acceptable — test setup where you control the data
const mockUser = { id: "1", email: "test@test.com", name: "Test" } as User;

// Acceptable — DOM elements where you know the structure
const canvas = document.getElementById("canvas") as HTMLCanvasElement;

// Acceptable — after a runtime check that TypeScript cannot infer
const element = event.target;
if (element instanceof HTMLInputElement) {
  // already narrowed — no assertion needed
  console.log(element.value);
}
```

> **Rule:** Avoid `as` in application code. Use type guards, `instanceof`, `typeof`, discriminated unions, or runtime validation to narrow types safely. Reserve assertions for tests and well-understood DOM interactions.

---

## Use Discriminated Unions for State Modeling

Discriminated unions model states that are mutually exclusive, preventing impossible states at the type level:

```typescript
// Bad — boolean flags create impossible states
interface RequestState {
  isLoading: boolean;
  isError: boolean;
  data: User | null;
  error: string | null;
}
// Nothing prevents isLoading: true AND isError: true simultaneously

// Good — discriminated union makes impossible states unrepresentable
type RequestState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: string };

function renderUserProfile(state: RequestState): string {
  switch (state.status) {
    case "idle":
      return "Click to load profile";
    case "loading":
      return "Loading...";
    case "success":
      return `Welcome, ${state.data.name}`;
    case "error":
      return `Error: ${state.error}`;
    default:
      return assertNever(state);
  }
}
```

**More complex example — modeling a multi-step form:**

```typescript
type FormState =
  | { step: "personal"; firstName: string; lastName: string }
  | { step: "address"; street: string; city: string; zipCode: string }
  | { step: "payment"; cardNumber: string; expiry: string }
  | { step: "confirmation"; orderId: string };

function getStepTitle(state: FormState): string {
  switch (state.step) {
    case "personal":
      return "Personal Information";
    case "address":
      return "Shipping Address";
    case "payment":
      return "Payment Details";
    case "confirmation":
      return `Order ${state.orderId} Confirmed`;
    default:
      return assertNever(state);
  }
}
```

> **Rule:** Whenever you find yourself using boolean flags or nullable fields to represent different states, refactor to a discriminated union. The compiler will enforce that each state only has the fields that make sense for it.

---

## Proper Null Handling

### strictNullChecks

With `strictNullChecks` enabled (included in `strict`), `null` and `undefined` are not assignable to other types unless explicitly included in the type:

```typescript
// With strictNullChecks
function getUser(id: string): User | null {
  // explicitly declares that null is a possible return value
  const user = database.find(id);
  return user ?? null;
}

const user = getUser("123");
// user.name; // Error: 'user' is possibly null
if (user !== null) {
  console.log(user.name); // safe
}
```

### Optional Chaining and Nullish Coalescing

Use optional chaining (`?.`) and nullish coalescing (`??`) for concise null handling:

```typescript
// Optional chaining — short-circuit on null/undefined
interface Company {
  name: string;
  address?: {
    street: string;
    city: string;
    country?: string;
  };
}

function getCountry(company: Company): string {
  // Returns undefined if address or country is missing — no runtime error
  return company.address?.country ?? "Unknown";
}

// Nullish coalescing — default only for null/undefined, not for 0 or ""
function getTimeout(config: { timeout?: number }): number {
  // Bad — || treats 0 as falsy
  // const timeout = config.timeout || 5000; // if timeout is 0, returns 5000

  // Good — ?? only defaults for null/undefined
  const timeout = config.timeout ?? 5000; // if timeout is 0, returns 0
  return timeout;
}

// Combining with assignment
interface Settings {
  theme?: string;
  fontSize?: number;
}

function applyDefaults(settings: Settings): Required<Settings> {
  return {
    theme: settings.theme ?? "light",
    fontSize: settings.fontSize ?? 14,
  };
}
```

> **Rule:** Always enable `strictNullChecks`. Use `?.` for safe property access, `??` for defaults (never `||` for values where `0`, `""`, or `false` are valid), and explicit null checks for control flow.

---

## Type-Safe API Boundaries

TypeScript types are erased at runtime. At the boundary between your application and the outside world — API requests, file reads, environment variables, database queries — you must validate data at runtime.

### Runtime Validation with Zod

Zod is the most popular runtime validation library for TypeScript. It infers TypeScript types from schemas, eliminating the need to define types separately:

```typescript
import { z } from "zod";

// Define a schema — this is BOTH the runtime validator and the TypeScript type
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(["admin", "member", "guest"]),
  createdAt: z.string().datetime(),
});

// Infer the TypeScript type from the schema
type User = z.infer<typeof UserSchema>;

// Validate incoming data
function parseUser(data: unknown): Result<User, z.ZodError> {
  const result = UserSchema.safeParse(data);
  if (result.success) {
    return { success: true, data: result.data };
  }
  return { success: false, error: result.error };
}

// Use in an API handler
app.post("/users", (req, res) => {
  const result = UserSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({
      error: "Validation failed",
      details: result.error.flatten().fieldErrors,
    });
  }
  // result.data is fully typed as User
  createUser(result.data);
});
```

### Runtime Validation with io-ts

io-ts takes a functional programming approach using codecs that can both decode and encode:

```typescript
import * as t from "io-ts";
import { isRight } from "fp-ts/Either";

const UserCodec = t.type({
  id: t.string,
  email: t.string,
  name: t.string,
  role: t.union([t.literal("admin"), t.literal("member"), t.literal("guest")]),
});

type User = t.TypeOf<typeof UserCodec>;

function decodeUser(input: unknown): User | null {
  const result = UserCodec.decode(input);
  if (isRight(result)) {
    return result.right;
  }
  return null;
}
```

> **Rule:** Never trust data from external sources. Validate at every boundary — API request bodies, query parameters, environment variables, file reads, and third-party API responses — using a runtime validation library like Zod or io-ts.

---

## Naming Conventions for Types and Interfaces

Consistent naming helps teams read and navigate code quickly. Follow these conventions:

| Category | Convention | Example |
|----------|-----------|---------|
| **Interfaces** | PascalCase, noun or noun phrase | `User`, `OrderRepository`, `HttpClient` |
| **Type aliases** | PascalCase, noun or noun phrase | `Status`, `EventHandler`, `Coordinate` |
| **Enums** | PascalCase name, PascalCase members | `Direction.North`, `HttpStatus.Ok` |
| **Generic parameters** | Single uppercase letter or short PascalCase | `T`, `TKey`, `TValue`, `TResult` |
| **Boolean properties** | Prefix with `is`, `has`, `can`, `should` | `isActive`, `hasPermission`, `canEdit` |
| **Constants** | UPPER_SNAKE_CASE for true constants | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |

```typescript
// Good — clear, consistent naming
interface UserProfile {
  id: string;
  displayName: string;
  isVerified: boolean;
  hasActiveSubscription: boolean;
}

type CreateUserInput = Omit<UserProfile, "id" | "isVerified">;

// Avoid — Hungarian notation prefixes
// interface IUser { }        // Don't prefix interfaces with I
// type TUserInput = { };     // Don't prefix types with T
// enum EDirection { }        // Don't prefix enums with E
```

> **Rule:** Do not use Hungarian notation prefixes (`I` for interfaces, `T` for types, `E` for enums). The TypeScript ecosystem convention is to use PascalCase without prefixes.

---

## Module Organization and Barrel Exports

### Recommended Structure

Organize code by feature or domain, not by technical role:

```
src/
├── users/
│   ├── index.ts              # barrel export
│   ├── user.model.ts         # User interface and related types
│   ├── user.repository.ts    # data access
│   ├── user.service.ts       # business logic
│   ├── user.controller.ts    # HTTP handlers
│   └── user.test.ts          # tests co-located with code
├── orders/
│   ├── index.ts
│   ├── order.model.ts
│   ├── order.service.ts
│   └── order.test.ts
├── shared/
│   ├── index.ts
│   ├── result.ts             # Result type
│   ├── errors.ts             # error classes
│   └── types.ts              # shared type utilities
└── index.ts                  # application entry point
```

**Barrel exports** re-export from a single `index.ts` so consumers import from the module, not from internal files:

```typescript
// src/users/index.ts — barrel export
export { User, CreateUserInput } from "./user.model";
export { UserService } from "./user.service";
export { UserRepository } from "./user.repository";

// Consumer imports from the module
import { User, UserService } from "./users";
```

### Barrel Export Pitfalls

Barrel exports can cause problems in large codebases:

- **Circular dependencies** — If module A re-exports from module B, and module B imports from module A, you get runtime errors or `undefined` values.
- **Bundle size** — Barrel exports can defeat tree-shaking in some bundlers, pulling in the entire module when only one export is needed.
- **Slow IDE performance** — Large barrel files slow down auto-import suggestions.

```typescript
// Problem — circular dependency through barrels
// src/users/index.ts exports UserService
// src/orders/index.ts exports OrderService
// UserService imports from "../orders" and OrderService imports from "../users"

// Solution — import directly from the specific file when breaking cycles
import { OrderService } from "../orders/order.service"; // bypass barrel
```

> **Rule:** Use barrel exports for clean public APIs of feature modules. Import directly from specific files when you encounter circular dependencies or bundle size issues.

---

## Avoid Premature Abstraction with Generics

Generics are powerful, but unnecessary generics add complexity without value. Start concrete and generalize only when you have multiple concrete use cases:

```typescript
// Bad — premature generic abstraction
class Repository<T, ID = string> {
  private items: Map<ID, T> = new Map();

  async findById(id: ID): Promise<T | null> {
    return this.items.get(id) ?? null;
  }

  async save(id: ID, item: T): Promise<void> {
    this.items.set(id, item);
  }
}
// If you only ever use Repository<User, string>, the generic serves no purpose

// Good — start concrete
class UserRepository {
  private users: Map<string, User> = new Map();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }

  async save(user: User): Promise<void> {
    this.users.set(user.id, user);
  }
}

// Only extract a generic when you have multiple concrete implementations
// and the pattern is proven
```

**When generics are appropriate:**

```typescript
// Good — generic utility that is genuinely reused
function groupBy<T, K extends string>(items: T[], keyFn: (item: T) => K): Record<K, T[]> {
  const result = {} as Record<K, T[]>;
  for (const item of items) {
    const key = keyFn(item);
    (result[key] ??= []).push(item);
  }
  return result;
}

// Good — generic constraint that enforces a contract
function sortByDate<T extends { createdAt: Date }>(items: T[]): T[] {
  return [...items].sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
}
```

> **Rule:** Write the concrete implementation first. Extract generics only when you have two or three concrete cases that share the same pattern. A generic used once is a premature abstraction.

---

## Production Checklist

Use this checklist before deploying a TypeScript project to production:

| Category | Check | Details |
|----------|-------|---------|
| **Compiler** | `strict: true` enabled | All strict sub-options active |
| **Compiler** | `noUncheckedIndexedAccess` enabled | Array/object index access returns `T \| undefined` |
| **Compiler** | No `// @ts-ignore` or `@ts-expect-error` without explanation | Every suppression has a comment explaining why |
| **Types** | Zero `any` in application code | Use `unknown`, generics, or proper types instead |
| **Types** | External data validated at runtime | Zod, io-ts, or equivalent at every API boundary |
| **Types** | Discriminated unions for state | No boolean flags for mutually exclusive states |
| **Errors** | Typed error classes or Result types | No throwing raw strings or untyped objects |
| **Errors** | Exhaustive switch/if-else over unions | `never` check in default/else branch |
| **Null safety** | No non-null assertions (`!`) without justification | Prefer narrowing over `!` operator |
| **Immutability** | `readonly` on interface properties where appropriate | Mutable only when explicitly needed |
| **Linting** | ESLint with `@typescript-eslint/recommended` | Plus `strict-type-checked` for stricter rules |
| **Build** | `isolatedModules: true` | Required for esbuild, SWC, and other transpilers |
| **Build** | Source maps configured for production debugging | `"sourceMap": true` or `"inlineSourceMap"` |
| **Dependencies** | `@types/*` packages match library versions | Mismatched versions cause phantom type errors |
| **CI/CD** | Type checking runs in CI pipeline | `tsc --noEmit` as a required check |
| **CI/CD** | Lint and test run on every PR | Automated quality gates before merge |

---

## Next Steps

Continue your TypeScript learning journey:

| File | Topic | Description |
|------|-------|-------------|
| [00-OVERVIEW.md](00-OVERVIEW.md) | TypeScript Fundamentals | Core language features, basic types, and project setup |
| [01-TYPE-SYSTEM.md](01-TYPE-SYSTEM.md) | Type System | Generics, utility types, conditional types, and type inference |
| [02-PATTERNS.md](02-PATTERNS.md) | Design Patterns | Common design patterns implemented in TypeScript |
| [03-NODE-AND-BACKEND.md](03-NODE-AND-BACKEND.md) | Node.js and Backend | Server-side TypeScript with Express, NestJS, Fastify, and databases |
| [04-FRONTEND-FRAMEWORKS.md](04-FRONTEND-FRAMEWORKS.md) | Frontend Frameworks | React, Angular, and Vue with TypeScript |
| [05-TESTING.md](05-TESTING.md) | Testing | Unit testing, integration testing, and E2E testing with TypeScript |
| [06-TOOLING.md](06-TOOLING.md) | Tooling | Build tools, linters, formatters, and CI/CD integration |
| [08-ANTI-PATTERNS.md](08-ANTI-PATTERNS.md) | Anti-Patterns | Common mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Guided progression through all TypeScript topics |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial TypeScript Best Practices documentation |
