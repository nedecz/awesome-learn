# TypeScript Anti-Patterns

A catalogue of the most common TypeScript mistakes — what they look like, why they persist, and exactly how to fix them. Use this document as a code-review checklist, a migration guide, or a team learning resource to write safer, more idiomatic TypeScript.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Using `any` Everywhere](#1-using-any-everywhere)
- [2. Type Assertions Instead of Type Narrowing](#2-type-assertions-instead-of-type-narrowing)
- [3. Not Enabling Strict Mode](#3-not-enabling-strict-mode)
- [4. Ignoring TypeScript Errors with @ts-ignore](#4-ignoring-typescript-errors-with-ts-ignore)
- [5. Overly Complex Generic Types](#5-overly-complex-generic-types)
- [6. Not Validating External Data at Runtime](#6-not-validating-external-data-at-runtime)
- [7. Using Enums When Union Types Suffice](#7-using-enums-when-union-types-suffice)
- [8. Barrel Files Causing Circular Dependencies](#8-barrel-files-causing-circular-dependencies)
- [9. Treating TypeScript as Just "JavaScript with Types"](#9-treating-typescript-as-just-javascript-with-types)
- [10. Not Using Discriminated Unions for State](#10-not-using-discriminated-unions-for-state)
- [11. Mutable Shared State Without Proper Typing](#11-mutable-shared-state-without-proper-typing)
- [12. Using the `object` Type](#12-using-the-object-type)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

TypeScript anti-patterns are recurring practices that seem reasonable — or even productive — at first but erode type safety, readability, and maintainability over time. They persist because they are often the path of least resistance: casting to `any` silences the compiler instantly, skipping strict mode avoids a wave of new errors, and a quick `@ts-ignore` unblocks the build.

The patterns documented here represent real mistakes found in production TypeScript codebases. Each one is:

- **Seductive** — it felt like the right approach or the fastest fix when first written
- **Harmful** — it creates bugs, runtime failures, or maintenance burdens that compound over time
- **Fixable** — there is a well-understood, type-safe alternative

### How to Use This Document

1. **Code review** — Reference specific anti-patterns when reviewing pull requests
2. **Migration checklist** — Use the [Quick Reference Checklist](#quick-reference-checklist) when migrating JavaScript to TypeScript
3. **Team onboarding** — Assign this document to engineers new to TypeScript before they ship to production
4. **Linting audit** — Map each anti-pattern to an ESLint rule and enforce it in CI
5. **Periodic review** — Run through the checklist quarterly to catch regressions in established codebases

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Severity | Impact |
|---|-------------|----------|--------|
| 1 | Using `any` Everywhere | 🔴 Critical | Disables type checking, defeats purpose of TypeScript |
| 2 | Type Assertions Instead of Type Narrowing | 🟠 High | Unsafe casts, runtime crashes on invalid assumptions |
| 3 | Not Enabling Strict Mode | 🔴 Critical | Loses half of TypeScript's safety guarantees |
| 4 | Ignoring Errors with `@ts-ignore` | 🟠 High | Hides real bugs, creates invisible tech debt |
| 5 | Overly Complex Generic Types | 🟡 Medium | Unreadable code, unmaintainable type signatures |
| 6 | Not Validating External Data at Runtime | 🔴 Critical | Runtime crashes, security vulnerabilities at boundaries |
| 7 | Using Enums When Union Types Suffice | 🟡 Medium | Unnecessary runtime code, bundle bloat, quirky behavior |
| 8 | Barrel Files Causing Circular Dependencies | 🟠 High | Bundle bloat, initialization errors, broken tree-shaking |
| 9 | Treating TypeScript as Just "JavaScript with Types" | 🟡 Medium | Missed patterns, shallow adoption, false confidence |
| 10 | Not Using Discriminated Unions for State | 🟠 High | Impossible states representable, boolean explosion |
| 11 | Mutable Shared State Without Proper Typing | 🟠 High | Race conditions, unpredictable mutations, stale reads |
| 12 | Using the `object` Type | 🟡 Medium | Too permissive, almost no type checking, confusing errors |

---

## 1. Using `any` Everywhere

### The Anti-Pattern

Using `any` as the default type annotation whenever the correct type is unclear, complex, or inconvenient to define. This includes function parameters, return types, variable declarations, and generic type arguments.

### Why It Happens

- Migrating a large JavaScript codebase — `any` is the fastest way to silence compiler errors
- Unfamiliarity with TypeScript's utility types and type narrowing
- Third-party libraries without type definitions
- Time pressure to ship features without learning the type system

### The Problem

`any` disables all type checking for every value it touches. Worse, it is contagious — a single `any` propagates through assignments, function calls, and property accesses, silently disabling the type checker across the codebase.

```typescript
// ❌ Bad: any disables all type safety
function processUser(user: any) {
  // No autocomplete, no error detection, no refactoring safety
  console.log(user.nmae); // Typo — no error
  user.permissions.admin = true; // May not exist — no error
  return user.age.toFixed(2); // age may be undefined — no error
}

const result: any = processUser({ name: "Alice" });
result.nonExistent.method(); // Runtime crash — TypeScript said nothing
```

### The Solution

Use `unknown` for values whose type you genuinely do not know. Use proper type definitions for values whose shape you can determine. Use utility types and generics when types are complex.

```typescript
// ✅ Good: Define the shape you expect
interface User {
  name: string;
  age: number;
  permissions: {
    admin: boolean;
  };
}

function processUser(user: User) {
  console.log(user.name); // Autocomplete works, typo caught
  user.permissions.admin = true; // Type-checked
  return user.age.toFixed(2); // Guaranteed to be a number
}

// ✅ Good: Use unknown when you truly don't know the type
function parseInput(raw: unknown): User {
  if (
    typeof raw === "object" &&
    raw !== null &&
    "name" in raw &&
    "age" in raw
  ) {
    return raw as User; // Narrowed with runtime checks first
  }
  throw new Error("Invalid user data");
}
```

### How to Detect

- ESLint rule: `@typescript-eslint/no-explicit-any`
- Search the codebase: `grep -rn ': any' src/`
- Track `any` count over time — it should trend toward zero

### Prevention

- Ban `any` in ESLint with the `no-explicit-any` rule set to `"error"`
- Use `unknown` as the default for truly unknown types
- Generate types from API schemas (OpenAPI, GraphQL codegen)
- Add a CI gate that fails builds when new `any` annotations are introduced

---

## 2. Type Assertions Instead of Type Narrowing

### The Anti-Pattern

Using `as` (type assertions) to tell the compiler a value is a specific type without performing any runtime check to verify the claim. This overrides the type checker with programmer assumptions.

### Why It Happens

- The developer "knows" what the type is and wants the compiler to agree
- TypeScript's control-flow narrowing is not understood or feels verbose
- DOM APIs return broad types (`Element`) that need narrowing to specific types (`HTMLInputElement`)
- Migration from JavaScript where casts feel natural

### The Problem

Type assertions are trust statements — you are telling the compiler to trust you, not verifying anything at runtime. If the assertion is wrong, you get a runtime crash with no compile-time warning.

```typescript
// ❌ Bad: Assertion without verification
interface ApiResponse {
  data: { id: number; name: string };
}

const response = await fetch("/api/user");
const body = (await response.json()) as ApiResponse; // No validation
console.log(body.data.name); // Runtime crash if API shape changes
```

```typescript
// ❌ Bad: Double assertion to force incompatible types
const input = someValue as unknown as SpecificType; // Red flag
```

### The Solution

Use type guards, `instanceof`, `in` checks, or validation libraries to narrow the type at runtime before using it.

```typescript
// ✅ Good: Type narrowing with runtime checks
function isApiResponse(value: unknown): value is ApiResponse {
  return (
    typeof value === "object" &&
    value !== null &&
    "data" in value &&
    typeof (value as Record<string, unknown>).data === "object"
  );
}

const response = await fetch("/api/user");
const body: unknown = await response.json();

if (isApiResponse(body)) {
  console.log(body.data.name); // Safe — type is narrowed
} else {
  throw new Error("Unexpected API response shape");
}
```

```typescript
// ✅ Good: DOM narrowing with instanceof
const element = document.getElementById("email");

if (element instanceof HTMLInputElement) {
  element.value = "user@example.com"; // Safe — narrowed to input
}
```

### How to Detect

- ESLint rule: `@typescript-eslint/consistent-type-assertions`
- Search for double assertions: `as unknown as`
- Review any `as` that is not preceded by a runtime check

### Prevention

- Prefer type predicates (`value is Type`) for custom type guards
- Use validation libraries (Zod, io-ts) at API boundaries
- Configure ESLint to warn on type assertions in non-test files
- Allow assertions only in test code where mocking demands it

---

## 3. Not Enabling Strict Mode

### The Anti-Pattern

Running the TypeScript compiler with `strict: false` (the default) or selectively disabling strict sub-options to avoid fixing errors.

### Why It Happens

- New projects created without reviewing `tsconfig.json` defaults
- Enabling strict mode on an existing codebase surfaces hundreds of errors
- Individual strict options are disabled to "fix later" and never re-enabled
- Tutorials or starter templates that ship with lenient settings

### The Problem

Without strict mode, TypeScript allows implicit `any`, skips null checks, ignores `this` typing, and permits other unsafe patterns. You lose approximately half of the type safety that TypeScript provides.

```json
// ❌ Bad: Loose tsconfig.json
{
  "compilerOptions": {
    "strict": false,
    "target": "ES2022"
  }
}
```

```typescript
// With strict disabled, all of these compile without errors:
function greet(name) {
  // name is implicitly `any` — no error
  return name.toUperCase(); // Typo — no error
}

function getUser() {
  // Return type allows null without being declared
  return null;
}

const user = getUser();
console.log(user.name); // Null dereference — no error
```

### The Solution

Enable `strict: true` and add additional strictness flags beyond the umbrella.

```json
// ✅ Good: Strict tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noPropertyAccessFromIndexSignature": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

For existing codebases, enable strict mode incrementally by turning on individual sub-options one at a time and fixing errors before moving to the next.

### How to Detect

- Check `tsconfig.json` for `"strict": true`
- Run `tsc --showConfig` and verify all strict sub-options are enabled
- Add a CI check that verifies `strict` is enabled

### Prevention

- Use a strict `tsconfig.json` template for all new projects
- Add `noUncheckedIndexedAccess` — the most commonly missed strict option
- Document the team standard and check it during code review

---

## 4. Ignoring TypeScript Errors with @ts-ignore

### The Anti-Pattern

Using `@ts-ignore` or `@ts-expect-error` comments to suppress compiler errors instead of fixing the underlying type issue.

### Why It Happens

- A library has incomplete type definitions
- A complex type error is difficult to understand or resolve
- Quick fix to unblock a build without investigating the cause
- Legacy code where fixing the type would require significant refactoring

### The Problem

`@ts-ignore` silences all errors on the following line — including future errors introduced by refactoring. It creates invisible technical debt that grows as the codebase evolves.

```typescript
// ❌ Bad: Hiding a real bug
interface Config {
  port: number;
  host: string;
}

function startServer(config: Config) {
  // @ts-ignore
  config.prot; // Typo for "port" — silenced, never caught
}

// ❌ Bad: Suppressing errors from refactoring
// @ts-ignore — TODO: fix this later
const value = someFunction(outdatedArg1, outdatedArg2);
// "Later" never comes, and the args may no longer exist
```

### The Solution

If suppression is unavoidable, use `@ts-expect-error` instead — it will error if the suppressed line stops having an error (catching stale suppressions). Better yet, fix the underlying type issue.

```typescript
// ✅ Good: Fix the type issue
function startServer(config: Config) {
  console.log(`Starting on port ${config.port}`); // Correct property
}

// ✅ Acceptable: Use @ts-expect-error with an explanation
// @ts-expect-error — library v3.2 types are missing the `timeout` option
// tracked in https://github.com/org/lib/issues/123
client.connect({ timeout: 5000 });
```

```typescript
// ✅ Good: Augment incomplete library types
declare module "incomplete-library" {
  interface ConnectOptions {
    timeout?: number;
  }
}
```

### How to Detect

- ESLint rule: `@typescript-eslint/ban-ts-comment` (configure to require descriptions)
- Search the codebase: `grep -rn '@ts-ignore\|@ts-expect-error' src/`
- Track suppression count over time — it should decrease

### Prevention

- Ban `@ts-ignore` entirely — require `@ts-expect-error` with a description
- Require a linked issue for every `@ts-expect-error` comment
- Augment or contribute to incomplete type definitions rather than suppressing errors
- Set a team budget for total allowed suppressions and reduce it each quarter

---

## 5. Overly Complex Generic Types

### The Anti-Pattern

Writing deeply nested, heavily conditional generic types that are impossible to read, debug, or maintain. Over-engineering type-level computations when simpler alternatives exist.

### Why It Happens

- Desire to encode every business rule in the type system
- "Type-level programming" enthusiasm without considering readability
- Copying complex utility types from blog posts without understanding them
- Lack of intermediate type aliases to break down complexity

### The Problem

Complex generics create a maintenance burden. When a type error occurs in a deeply nested generic, the error message is incomprehensible. New team members cannot contribute, and even the author struggles to modify the type months later.

```typescript
// ❌ Bad: Unreadable nested generics
type DeepPartialWithNullable<T> = T extends object
  ? T extends Array<infer U>
    ? Array<DeepPartialWithNullable<U>> | null
    : { [K in keyof T]?: DeepPartialWithNullable<T[K]> | null } | null
  : T extends string | number | boolean
    ? T | null
    : T extends (...args: infer A) => infer R
      ? (...args: DeepPartialWithNullable<A>) => DeepPartialWithNullable<R>
      : T | null;

// Error message when this breaks:
// Type 'X' is not assignable to type
//   'DeepPartialWithNullable<DeepPartialWithNullable<Omit<...>>>'
//   ... 14 more lines of nested types
```

### The Solution

Break complex types into named intermediate types. Prefer simple generics with clear constraints. Document non-obvious type parameters.

```typescript
// ✅ Good: Break into readable, named intermediate types
type Nullable<T> = T | null;

type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

type NullablePartial<T> = Nullable<DeepPartial<T>>;

// ✅ Good: Constrain generics with meaningful bounds
function merge<T extends Record<string, unknown>>(
  target: T,
  source: Partial<T>
): T {
  return { ...target, ...source };
}

// ✅ Good: Simple generics with clear intent
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}
```

### How to Detect

- Review any type definition longer than 5 lines of conditional types
- Check for more than 3 levels of generic nesting
- Watch for error messages that are longer than the code they reference

### Prevention

- Establish a team guideline: max 2–3 levels of generic nesting
- Use named intermediate types liberally — they cost nothing at runtime
- Prefer runtime validation over type-level encoding for complex business rules
- Document every type parameter with a JSDoc `@template` tag

---

## 6. Not Validating External Data at Runtime

### The Anti-Pattern

Trusting that data from external sources (APIs, user input, files, databases, environment variables) matches the TypeScript type definition without performing runtime validation.

### Why It Happens

- TypeScript types create a false sense of security — "the type says it's a `User`, so it must be"
- Types are erased at runtime — they cannot enforce anything after compilation
- Validation is seen as extra work when the API "always returns the right shape"
- Lack of awareness about runtime validation libraries

### The Problem

TypeScript types exist only at compile time. External data can be anything at runtime. Without validation, a missing field, wrong type, or unexpected null causes a crash far from the boundary where the data entered the system.

```typescript
// ❌ Bad: Trusting fetch response matches the type
interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

async function getProduct(id: number): Promise<Product> {
  const response = await fetch(`/api/products/${id}`);
  return response.json(); // No validation — blind trust
}

const product = await getProduct(1);
// If API returns { id: 1, name: "Widget", price: "19.99" }
// price is a string, not a number — .toFixed(2) may work by accident
// but arithmetic like price * quantity gives wrong results
```

### The Solution

Validate external data at system boundaries using a schema validation library. Parse, don't just typecast.

```typescript
// ✅ Good: Validate with Zod at the boundary
import { z } from "zod";

const ProductSchema = z.object({
  id: z.number(),
  name: z.string(),
  price: z.number(),
  category: z.string(),
});

type Product = z.infer<typeof ProductSchema>;

async function getProduct(id: number): Promise<Product> {
  const response = await fetch(`/api/products/${id}`);
  const data: unknown = await response.json();
  return ProductSchema.parse(data); // Throws with clear error if invalid
}
```

```typescript
// ✅ Good: Validate environment variables at startup
const EnvSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().int().min(1).max(65535),
  NODE_ENV: z.enum(["development", "staging", "production"]),
});

const env = EnvSchema.parse(process.env);
// App crashes immediately on startup if env vars are wrong
// rather than at 3 AM when a missing value is first accessed
```

### How to Detect

- Search for `as SomeType` after `response.json()`, `JSON.parse()`, or `fs.readFile()`
- Look for `fetch` calls whose return type is asserted without validation
- Check for `process.env.VARIABLE` without validation

### Prevention

- Establish a "parse, don't assume" rule for all external data
- Use Zod, io-ts, or Valibot at every system boundary
- Generate validation schemas from API specs (OpenAPI → Zod)
- Add integration tests that send malformed data and verify graceful failure

---

## 7. Using Enums When Union Types Suffice

### The Anti-Pattern

Defaulting to TypeScript `enum` declarations for simple sets of string or numeric constants when a string literal union type would be simpler, smaller, and more idiomatic.

### Why It Happens

- Habit from languages like Java or C# where enums are the standard pattern
- Early TypeScript tutorials promoted enums before union types were well-supported
- Perceived need for a "named constant" without understanding the trade-offs
- Unaware that string enums emit runtime JavaScript objects

### The Problem

TypeScript enums have surprising behaviors: numeric enums allow reverse mapping and accept any number, `const enum` has different behavior depending on the `--isolatedModules` flag, and all enums generate runtime JavaScript that increases bundle size. String literal unions are simpler, lighter, and work better with type narrowing.

```typescript
// ❌ Bad: Enum with quirks
enum Status {
  Active = "active",
  Inactive = "inactive",
  Pending = "pending",
}

// Enums generate runtime code:
// var Status;
// (function (Status) {
//   Status["Active"] = "active";
//   Status["Inactive"] = "inactive";
//   Status["Pending"] = "pending";
// })(Status || (Status = {}));

// Numeric enums are even worse — they accept any number
enum Direction {
  Up,
  Down,
  Left,
  Right,
}

const d: Direction = 999; // No error — any number is a valid Direction
```

### The Solution

Use string literal union types for simple sets of values. Reserve enums for cases where you genuinely need the runtime object (e.g., iteration over all values).

```typescript
// ✅ Good: String literal union — no runtime cost
type Status = "active" | "inactive" | "pending";

function setStatus(status: Status) {
  // Only the three defined values are accepted
}

setStatus("active"); // ✅ OK
setStatus("unknown"); // ❌ Compile error

// ✅ Good: If you need a runtime list of all values
const STATUSES = ["active", "inactive", "pending"] as const;
type Status = (typeof STATUSES)[number]; // "active" | "inactive" | "pending"

// Now you can iterate and type-check
STATUSES.forEach((s) => console.log(s));
```

### How to Detect

- ESLint rule: `no-restricted-syntax` targeting `TSEnumDeclaration`
- Search for `enum` declarations and evaluate if a union type would suffice
- Check for numeric enums — they almost always should be replaced

### Prevention

- Default to string literal union types for new code
- Use `as const` arrays when you need both a type and a runtime list
- Reserve `enum` for interoperability with external systems that require them
- Add an ESLint rule or code review guideline preferring unions over enums

---

## 8. Barrel Files Causing Circular Dependencies

### The Anti-Pattern

Creating `index.ts` barrel files that re-export everything from a module, leading to circular dependency chains, broken tree-shaking, and runtime initialization errors.

### Why It Happens

- Clean import paths feel good: `import { User } from "./models"` instead of `import { User } from "./models/user"`
- Barrel files are recommended in many style guides without mentioning the risks
- Circular dependencies are invisible until they cause a runtime error
- Bundlers and Node.js resolve circular imports silently — until they don't

### The Problem

When module A re-exports from module B through a barrel file, and module B imports from module A, a circular dependency is created. This can cause `undefined` at runtime when a module is accessed before it is initialized.

```typescript
// ❌ Bad: Barrel file creating circular dependency

// models/index.ts (barrel)
export { User } from "./user";
export { Order } from "./order";
export { Product } from "./product";

// models/order.ts
import { User } from "./index"; // Imports through barrel
import { Product } from "./index"; // Imports through barrel

export interface Order {
  user: User;
  products: Product[];
}

// models/user.ts
import { Order } from "./index"; // Circular! user → index → order → index → user

export interface User {
  name: string;
  orders: Order[];
}

// At runtime, one of these may be `undefined` depending on
// which module the bundler initializes first
```

### The Solution

Import directly from the source module. Use barrel files sparingly and only for public API surfaces.

```typescript
// ✅ Good: Direct imports — no circular risk
// models/order.ts
import { User } from "./user"; // Direct import
import { Product } from "./product"; // Direct import

export interface Order {
  user: User;
  products: Product[];
}

// models/user.ts
import { Order } from "./order"; // Direct import — dependency is clear

export interface User {
  name: string;
  orders: Order[];
}

// ✅ Good: Barrel file only at the package boundary
// models/index.ts — used by external consumers only
export type { User } from "./user";
export type { Order } from "./order";
export type { Product } from "./product";
```

### How to Detect

- Use `madge --circular src/` to find circular dependency chains
- Enable the `import/no-cycle` ESLint rule
- Check bundle analyzer output for unexpectedly large chunks (barrel files pull in everything)

### Prevention

- Use direct imports within a module — barrel files only for package-level public APIs
- Use `export type` in barrel files when re-exporting only types (avoids runtime cycles)
- Run circular dependency detection in CI
- Configure bundler to warn on circular imports

---

## 9. Treating TypeScript as Just "JavaScript with Types"

### The Anti-Pattern

Adding type annotations to JavaScript code without leveraging TypeScript-specific patterns — discriminated unions, branded types, exhaustive matching, type-safe error handling, and compile-time enforcement of invariants.

### Why It Happens

- Migrating from JavaScript and retaining JavaScript habits
- Learning TypeScript syntax (annotations, interfaces) but not its design patterns
- Tutorials that focus on "annotating what you have" rather than "designing with types"
- Teams that adopt TypeScript for tooling (autocomplete, refactoring) without changing how they model data

### The Problem

Using TypeScript as "JavaScript with annotations" captures only a fraction of its value. You get autocomplete but miss compile-time guarantees that prevent entire categories of bugs.

```typescript
// ❌ Bad: JavaScript patterns with type annotations bolted on
function processPayment(amount: number, currency: string, method: string) {
  // currency could be "usd", "USD", "us-dollar", or "potato"
  // method could be "card", "credit_card", "CARD", or anything
  if (method === "card") {
    // ...
  } else if (method === "bank") {
    // ...
  }
  // What if a new method is added? No compile-time safety.
}
```

### The Solution

Design with types — use literal types, discriminated unions, branded types, and exhaustive matching to make invalid states unrepresentable.

```typescript
// ✅ Good: TypeScript-native design
type Currency = "USD" | "EUR" | "GBP";

type PaymentMethod =
  | { type: "card"; cardToken: string }
  | { type: "bank"; accountNumber: string; routingNumber: string }
  | { type: "wallet"; walletId: string };

function processPayment(
  amount: number,
  currency: Currency,
  method: PaymentMethod
) {
  switch (method.type) {
    case "card":
      chargeCard(method.cardToken, amount, currency);
      break;
    case "bank":
      debitBank(method.accountNumber, method.routingNumber, amount);
      break;
    case "wallet":
      chargeWallet(method.walletId, amount, currency);
      break;
    default:
      // Exhaustive check — compile error if a new method is added
      const _exhaustive: never = method;
      throw new Error(`Unhandled payment method: ${_exhaustive}`);
  }
}
```

### How to Detect

- Look for functions with multiple `string` or `number` parameters that represent constrained values
- Check for `if/else` chains that switch on string values without exhaustive matching
- Identify boolean flags that represent mutually exclusive states

### Prevention

- Model domain concepts as types first, then write functions that consume them
- Use discriminated unions for any value that has multiple forms
- Use branded types for values that are structurally identical but semantically different (e.g., `UserId` vs `OrderId`)
- Study TypeScript-specific patterns: [02-PATTERNS.md](02-PATTERNS.md) covers these in depth

---

## 10. Not Using Discriminated Unions for State

### The Anti-Pattern

Modeling state with booleans, optional fields, and nullable values instead of discriminated unions, creating types that allow impossible combinations.

### Why It Happens

- Boolean flags feel simple and familiar
- Adding an optional field seems less work than redesigning the type
- The impossible states "won't happen in practice" — until they do
- Lack of familiarity with discriminated union patterns

### The Problem

When state is modeled with independent booleans and optional fields, the type system allows combinations that are logically impossible. Every function that consumes the type must handle these impossible states with defensive checks.

```typescript
// ❌ Bad: Boolean flags allow impossible states
interface RequestState {
  isLoading: boolean;
  isError: boolean;
  data?: User;
  error?: Error;
}

// This type allows 16 combinations (2^4), but only 3 are valid:
// 1. Loading: isLoading=true, isError=false, no data, no error
// 2. Success: isLoading=false, isError=false, data present, no error
// 3. Error: isLoading=false, isError=true, no data, error present

// Impossible but representable:
const broken: RequestState = {
  isLoading: true,
  isError: true,
  data: someUser,
  error: new Error("wat"),
};
```

### The Solution

Use discriminated unions where a single `status` field determines which other fields exist. Each variant of the union contains exactly the fields that are valid for that state.

```typescript
// ✅ Good: Discriminated union — impossible states are unrepresentable
type RequestState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: Error };

function renderUser(state: RequestState) {
  switch (state.status) {
    case "idle":
      return "Ready to load";
    case "loading":
      return "Loading...";
    case "success":
      return `Hello, ${state.data.name}`; // data is guaranteed present
    case "error":
      return `Error: ${state.error.message}`; // error is guaranteed present
  }
}

// ✅ Impossible states are now compile errors:
// const broken: RequestState = { status: "loading", data: someUser };
// Error: Object literal may only specify known properties
```

### How to Detect

- Look for interfaces with `isX` boolean pairs (`isLoading` + `isError`)
- Look for optional fields that are only valid in certain states
- Check for `if (state.isLoading && state.data)` defensive checks

### Prevention

- Default to discriminated unions for any state with multiple modes
- Ask: "Can I construct a value of this type that is logically invalid?" If yes, redesign
- Use the `never` type for exhaustive matching to catch unhandled states
- Review [07-BEST-PRACTICES.md](07-BEST-PRACTICES.md) for discriminated union patterns

---

## 11. Mutable Shared State Without Proper Typing

### The Anti-Pattern

Using mutable objects, arrays, or maps as shared state across modules without marking them as `readonly` or controlling mutation through a typed API.

### Why It Happens

- Default JavaScript objects and arrays are mutable
- `readonly` feels like extra work for internal code
- Shared state starts small ("just one config object") and grows
- No awareness that TypeScript can enforce immutability at the type level

### The Problem

Mutable shared state leads to unpredictable behavior — any module can modify the data at any time, making it difficult to track changes, reproduce bugs, or reason about state transitions. Without `readonly`, TypeScript does not warn about accidental mutations.

```typescript
// ❌ Bad: Mutable shared config — anyone can modify it
export const config = {
  apiUrl: "https://api.example.com",
  retryCount: 3,
  features: ["search", "export"],
};

// Somewhere in module A:
config.retryCount = 0; // Silent mutation

// Somewhere in module B:
config.features.push("dangerous-feature"); // Array mutated in-place

// Somewhere in module C — bug that is nearly impossible to trace:
config.apiUrl = ""; // Entire app now makes requests to empty string
```

### The Solution

Use `readonly` modifiers, `Readonly<T>`, `ReadonlyArray<T>`, and `as const` to enforce immutability at the type level. Expose mutation only through controlled, typed functions.

```typescript
// ✅ Good: Immutable config with as const
export const config = {
  apiUrl: "https://api.example.com",
  retryCount: 3,
  features: ["search", "export"],
} as const;

// config.retryCount = 0;
// Error: Cannot assign to 'retryCount' because it is a read-only property

// config.features.push("dangerous-feature");
// Error: Property 'push' does not exist on type 'readonly ["search", "export"]'
```

```typescript
// ✅ Good: Controlled mutation through a typed API
interface AppState {
  readonly users: ReadonlyArray<User>;
  readonly selectedId: string | null;
}

function addUser(state: AppState, user: User): AppState {
  return {
    ...state,
    users: [...state.users, user], // New array, no mutation
  };
}

function selectUser(state: AppState, id: string): AppState {
  return {
    ...state,
    selectedId: id,
  };
}
```

### How to Detect

- Look for exported `let` declarations or exported mutable objects
- Check for `Array.push()`, `Array.splice()`, or direct property assignment on shared state
- Search for global or module-level mutable variables

### Prevention

- Default to `as const` for all configuration objects
- Use `Readonly<T>` and `ReadonlyArray<T>` for shared data structures
- Expose state changes only through pure functions that return new state
- Enable the `prefer-readonly` and `prefer-readonly-parameter-types` ESLint rules

---

## 12. Using the `object` Type

### The Anti-Pattern

Using the `object` type (lowercase) as a type annotation, which accepts any non-primitive value and provides almost no type safety.

### Why It Happens

- Confusion between `object`, `Object`, and `{}` — they all mean different things
- Attempt to say "I want an object, not a string or number" without specifying shape
- Copied from documentation or examples without understanding the implications
- Using `object` as a "better `any`" when the actual shape is unknown

### The Problem

The `object` type accepts arrays, functions, dates, regex, class instances, and any other non-primitive — but you cannot access any properties on it. It provides almost no useful type checking.

```typescript
// ❌ Bad: object is too broad
function processData(data: object) {
  // data.name — Error: Property 'name' does not exist on type 'object'
  // data.id — Error: Property 'id' does not exist on type 'object'
  // You can't do anything useful without casting
}

// All of these are valid — not what you intended:
processData([]); // Array
processData(() => {}); // Function
processData(/regex/); // RegExp
processData(new Date()); // Date
processData({ name: "Alice" }); // Also an object, but so is everything above
```

### The Solution

Use `Record<string, unknown>` for "any object with string keys," define an interface for known shapes, or use `unknown` with type narrowing.

```typescript
// ✅ Good: Specific interface when shape is known
interface UserData {
  id: number;
  name: string;
}

function processUser(data: UserData) {
  console.log(data.name); // Autocomplete, type checking
}

// ✅ Good: Record for dynamic key-value objects
function logFields(data: Record<string, unknown>) {
  for (const [key, value] of Object.entries(data)) {
    console.log(`${key}: ${value}`);
  }
}

// ✅ Good: unknown with narrowing for truly unknown shapes
function processUnknown(data: unknown) {
  if (typeof data === "object" && data !== null && "name" in data) {
    console.log((data as { name: string }).name);
  }
}
```

### How to Detect

- ESLint rule: `@typescript-eslint/ban-types` (flags `object`, `Object`, `{}`)
- Search for `: object` annotations in function parameters and return types
- Review for `: Object` (capital O) — even more permissive, accepts primitives too

### Prevention

- Ban `object`, `Object`, and `{}` as type annotations with ESLint
- Use `Record<string, unknown>` when you need a generic object type
- Use `unknown` with type narrowing when the shape is truly unknown
- Define interfaces or type aliases for every object shape used in the codebase

---

## Quick Reference Checklist

Use this checklist for TypeScript code reviews and pre-production audits:

### Critical (Must Fix Before Production)

- [ ] No `any` types in production code (use `unknown` or proper types)
- [ ] `strict: true` enabled in `tsconfig.json`
- [ ] All external data validated at runtime (APIs, user input, env vars)
- [ ] No `@ts-ignore` comments — use `@ts-expect-error` with linked issues if unavoidable

### High (Fix Before Production or Immediately After)

- [ ] No unsafe type assertions (`as Type`) without preceding runtime checks
- [ ] Discriminated unions used for state modeling (no boolean flag pairs)
- [ ] No circular dependencies between modules (verified with `madge`)
- [ ] Shared state is `readonly` or mutated through controlled APIs

### Medium (Address Within First Sprint)

- [ ] Generic types limited to 2–3 levels of nesting with named intermediates
- [ ] String literal unions preferred over enums for simple value sets
- [ ] `object` type banned — use `Record<string, unknown>` or specific interfaces
- [ ] Domain concepts modeled as types with exhaustive matching

---

## Next Steps

1. **Audit your codebase** — Use the [Quick Reference Checklist](#quick-reference-checklist) and count violations
2. **Fix critical severity first** — Address 🔴 Critical anti-patterns (#1, #3, #6) before others
3. **Read the companion guide** — [07-BEST-PRACTICES.md](07-BEST-PRACTICES.md) describes the correct patterns in detail
4. **Study TypeScript patterns** — [02-PATTERNS.md](02-PATTERNS.md) covers discriminated unions, branded types, and type-safe design
5. **Configure tooling** — [06-TOOLING.md](06-TOOLING.md) shows how to enforce these rules with ESLint and CI
6. **Validate at boundaries** — [03-NODE-AND-BACKEND.md](03-NODE-AND-BACKEND.md) covers runtime validation with Zod and io-ts
7. **Follow the learning path** — [LEARNING-PATH.md](LEARNING-PATH.md) provides a structured curriculum for TypeScript mastery

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial TypeScript Anti-Patterns documentation |
