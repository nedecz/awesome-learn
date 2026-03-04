# TypeScript Type System

## Table of Contents

1. [Overview](#overview)
   - [What Is the TypeScript Type System?](#what-is-the-typescript-type-system)
   - [Why It Matters](#why-it-matters)
   - [Scope](#scope)
   - [Target Audience](#target-audience)
2. [Generics](#generics)
   - [Generic Functions](#generic-functions)
   - [Generic Interfaces](#generic-interfaces)
   - [Generic Classes](#generic-classes)
   - [Generic Constraints](#generic-constraints)
   - [Default Type Parameters](#default-type-parameters)
3. [Utility Types](#utility-types)
   - [Partial and Required](#partial-and-required)
   - [Pick and Omit](#pick-and-omit)
   - [Record](#record)
   - [Exclude and Extract](#exclude-and-extract)
   - [ReturnType and Parameters](#returntype-and-parameters)
   - [NonNullable and Awaited](#nonnullable-and-awaited)
4. [Conditional Types](#conditional-types)
   - [Basic Conditional Types](#basic-conditional-types)
   - [The infer Keyword](#the-infer-keyword)
   - [Distributive Conditional Types](#distributive-conditional-types)
5. [Mapped Types](#mapped-types)
   - [Basic Mapped Types](#basic-mapped-types)
   - [Key Remapping with as](#key-remapping-with-as)
   - [Modifiers in Mapped Types](#modifiers-in-mapped-types)
6. [Index Types and keyof Operator](#index-types-and-keyof-operator)
   - [The keyof Operator](#the-keyof-operator)
   - [Indexed Access Types](#indexed-access-types)
7. [Type Guards and Type Predicates](#type-guards-and-type-predicates)
   - [Built-in Type Guards](#built-in-type-guards)
   - [Custom Type Predicates](#custom-type-predicates)
   - [Assertion Functions](#assertion-functions)
8. [Discriminated Unions](#discriminated-unions)
   - [Building Discriminated Unions](#building-discriminated-unions)
   - [Exhaustive Checking](#exhaustive-checking)
9. [Template Literal Types](#template-literal-types)
   - [Basic Template Literals](#basic-template-literals)
   - [Intrinsic String Manipulation Types](#intrinsic-string-manipulation-types)
10. [Recursive Types](#recursive-types)
    - [Recursive Type Aliases](#recursive-type-aliases)
    - [Recursive Conditional Types](#recursive-conditional-types)
11. [Variance and Covariance](#variance-and-covariance)
    - [Covariance](#covariance)
    - [Contravariance](#contravariance)
    - [The Variance Annotations](#the-variance-annotations)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

### What Is the TypeScript Type System?

TypeScript's type system is a **structural type system** that operates at compile time to catch errors before code runs. Unlike nominal type systems found in languages like Java or C#, TypeScript determines type compatibility based on the shape (structure) of types rather than their declared names. The type system is also **gradually typed**, meaning you can adopt it incrementally — from loose `any` types to precise, deeply constrained generic types.

The advanced features of the type system — generics, conditional types, mapped types, and more — form a powerful meta-programming language that lets you express complex relationships between types, build type-safe abstractions, and eliminate entire categories of runtime errors.

### Why It Matters

- **Catch bugs at compile time** — Prevent `undefined is not a function` and similar runtime errors before they reach production
- **Self-documenting code** — Types serve as living documentation that stays in sync with the implementation
- **Refactoring confidence** — Change a type and the compiler shows you every affected location
- **Better tooling** — Autocompletion, inline documentation, and go-to-definition all rely on the type system
- **Library design** — Advanced types let library authors provide precise, ergonomic APIs

### Scope

- Generics and generic constraints
- Built-in utility types and how to build custom ones
- Conditional types and the `infer` keyword
- Mapped types and key remapping
- Index types, type guards, and discriminated unions
- Template literal types and recursive types
- Variance annotations and type-level programming patterns

### Target Audience

- **Intermediate TypeScript developers** — Developers comfortable with basic types who want to understand the full power of the type system
- **Library authors** — Anyone designing public APIs that need to be type-safe and ergonomic
- **Tech leads and architects** — Engineers making decisions about type safety strategies for their teams

---

## Generics

Generics allow you to write reusable components that work with a variety of types rather than a single one. They are the foundation of type-safe abstractions in TypeScript.

### Generic Functions

A generic function declares one or more **type parameters** that are determined when the function is called.

```typescript
// Without generics — loses type information
function firstElement(arr: any[]): any {
  return arr[0];
}

// With generics — preserves the relationship between input and output
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

const num = firstElement([1, 2, 3]);       // type: number | undefined
const str = firstElement(["a", "b", "c"]); // type: string | undefined

// Multiple type parameters capture relationships between arguments
function map<Input, Output>(
  arr: Input[],
  fn: (item: Input) => Output
): Output[] {
  return arr.map(fn);
}

const lengths = map(["hello", "world"], (s) => s.length);
// type: number[] — TypeScript infers both Input = string and Output = number
```

### Generic Interfaces

Interfaces can be parameterized to describe generic data structures.

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
  timestamp: Date;
  errors?: string[];
}

interface PaginatedResponse<T> extends ApiResponse<T[]> {
  page: number;
  pageSize: number;
  totalCount: number;
  hasNextPage: boolean;
}

// Usage — the generic parameter flows through the entire structure
type UserResponse = ApiResponse<{ id: string; name: string }>;
type UserListResponse = PaginatedResponse<{ id: string; name: string }>;

// Generic interface for a repository pattern
interface Repository<T, ID = string> {
  findById(id: ID): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(entity: Omit<T, "id">): Promise<T>;
  update(id: ID, entity: Partial<T>): Promise<T>;
  delete(id: ID): Promise<boolean>;
}
```

### Generic Classes

Classes can use generics to create type-safe containers and abstractions.

```typescript
class TypedEventEmitter<Events extends Record<string, unknown[]>> {
  private listeners = new Map<keyof Events, Set<Function>>();

  on<K extends keyof Events>(
    event: K,
    listener: (...args: Events[K]) => void
  ): void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(listener);
  }

  emit<K extends keyof Events>(event: K, ...args: Events[K]): void {
    const handlers = this.listeners.get(event);
    if (handlers) {
      handlers.forEach((handler) => handler(...args));
    }
  }
}

// Define events as a type map
interface AppEvents {
  userLogin: [userId: string, timestamp: Date];
  pageView: [path: string];
  error: [error: Error, context: string];
}

const emitter = new TypedEventEmitter<AppEvents>();

emitter.on("userLogin", (userId, timestamp) => {
  // userId: string, timestamp: Date — fully typed
  console.log(`User ${userId} logged in at ${timestamp}`);
});

// emitter.emit("userLogin", 123); // Error: number is not assignable to string
```

### Generic Constraints

Use `extends` to constrain what types a generic parameter can accept.

```typescript
// Constraint ensures T has a 'length' property
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}

longest("hello", "world");     // OK: string has length
longest([1, 2], [1, 2, 3]);   // OK: array has length
// longest(10, 20);            // Error: number has no length

// keyof constraint — K must be a key of T
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 30, email: "alice@example.com" };
const name = getProperty(user, "name");  // type: string
const age = getProperty(user, "age");    // type: number
// getProperty(user, "phone");           // Error: "phone" is not a key of user

// Mutual constraints between type parameters
function merge<T extends object, U extends object>(
  target: T,
  source: U
): T & U {
  return { ...target, ...source };
}
```

### Default Type Parameters

Generic parameters can have defaults, similar to default function arguments.

```typescript
interface FetchOptions<TBody = undefined> {
  url: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  body?: TBody;
  headers?: Record<string, string>;
}

// TBody defaults to undefined when not specified
const getOptions: FetchOptions = {
  url: "/api/users",
  method: "GET",
};

const postOptions: FetchOptions<{ name: string }> = {
  url: "/api/users",
  method: "POST",
  body: { name: "Alice" },
};
```

---

## Utility Types

TypeScript provides a set of built-in utility types that transform existing types. Understanding them is essential — and knowing how they are implemented helps you build your own.

### Partial and Required

`Partial<T>` makes all properties optional. `Required<T>` makes all properties required.

```typescript
interface Config {
  host: string;
  port: number;
  debug?: boolean;
  logLevel: "info" | "warn" | "error";
}

// Partial<Config> makes every property optional
function updateConfig(current: Config, updates: Partial<Config>): Config {
  return { ...current, ...updates };
}

const config: Config = { host: "localhost", port: 3000, logLevel: "info" };
updateConfig(config, { port: 8080 }); // Only override what you need

// Required<Config> makes every property required, including debug
type StrictConfig = Required<Config>;
// { host: string; port: number; debug: boolean; logLevel: "info" | "warn" | "error" }

// How Partial is implemented under the hood
type MyPartial<T> = {
  [P in keyof T]?: T[P];
};
```

### Pick and Omit

`Pick<T, K>` selects specific properties. `Omit<T, K>` removes specific properties.

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
  updatedAt: Date;
}

// Pick only the fields safe to expose in an API response
type PublicUser = Pick<User, "id" | "name" | "email">;
// { id: string; name: string; email: string }

// Omit sensitive fields — equivalent result, different approach
type SafeUser = Omit<User, "password">;
// { id: string; name: string; email: string; createdAt: Date; updatedAt: Date }

// Composing utility types to create a "create user" payload
type CreateUserPayload = Omit<User, "id" | "createdAt" | "updatedAt">;
// { name: string; email: string; password: string }
```

### Record

`Record<K, V>` constructs a type with keys of type `K` and values of type `V`.

```typescript
// Simple lookup table
type HttpStatusMessages = Record<number, string>;

const statusMessages: HttpStatusMessages = {
  200: "OK",
  404: "Not Found",
  500: "Internal Server Error",
};

// Using a union for keys gives you exhaustive checking
type Role = "admin" | "editor" | "viewer";

type RolePermissions = Record<Role, string[]>;

const permissions: RolePermissions = {
  admin: ["read", "write", "delete", "manage"],
  editor: ["read", "write"],
  viewer: ["read"],
  // Removing any role here causes a compile error
};

// Record is implemented as a mapped type
type MyRecord<K extends keyof any, T> = {
  [P in K]: T;
};
```

### Exclude and Extract

`Exclude<T, U>` removes types from a union. `Extract<T, U>` keeps only matching types.

```typescript
type AllEvents = "click" | "scroll" | "mousemove" | "keypress" | "keyup";

type MouseEvents = Extract<AllEvents, "click" | "scroll" | "mousemove">;
// "click" | "scroll" | "mousemove"

type KeyboardEvents = Exclude<AllEvents, "click" | "scroll" | "mousemove">;
// "keypress" | "keyup"

// Practical example: filtering union members by shape
type EventPayload =
  | { type: "user"; userId: string }
  | { type: "system"; code: number }
  | { type: "error"; message: string };

type UserOrSystemPayload = Extract<EventPayload, { type: "user" | "system" }>;
// { type: "user"; userId: string } | { type: "system"; code: number }
```

### ReturnType and Parameters

`ReturnType<T>` extracts the return type of a function. `Parameters<T>` extracts its parameter types as a tuple.

```typescript
function createUser(name: string, email: string, role: "admin" | "user") {
  return {
    id: crypto.randomUUID(),
    name,
    email,
    role,
    createdAt: new Date(),
  };
}

type NewUser = ReturnType<typeof createUser>;
// { id: string; name: string; email: string; role: "admin" | "user"; createdAt: Date }

type CreateUserParams = Parameters<typeof createUser>;
// [name: string, email: string, role: "admin" | "user"]

// Useful for wrapping functions without duplicating types
function withLogging<T extends (...args: any[]) => any>(
  fn: T
): (...args: Parameters<T>) => ReturnType<T> {
  return (...args) => {
    console.log("Calling with:", args);
    const result = fn(...args);
    console.log("Result:", result);
    return result;
  };
}
```

### NonNullable and Awaited

```typescript
// NonNullable removes null and undefined from a union
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>; // string

// Awaited unwraps Promise types, including nested promises
type A = Awaited<Promise<string>>;              // string
type B = Awaited<Promise<Promise<number>>>;     // number
type C = Awaited<string | Promise<boolean>>;    // string | boolean

// Practical use: typing async function return values
async function fetchUser(id: string) {
  const response = await fetch(`/api/users/${id}`);
  return response.json() as Promise<{ id: string; name: string }>;
}

type FetchedUser = Awaited<ReturnType<typeof fetchUser>>;
// { id: string; name: string }
```

---

## Conditional Types

Conditional types let you express type-level `if/else` logic using the syntax `T extends U ? X : Y`.

### Basic Conditional Types

```typescript
// Conditional type: if T is a string, return true; otherwise false
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;       // true
type B = IsString<number>;       // false
type C = IsString<"hello">;      // true (string literal extends string)

// Flatten arrays or return the type as-is
type Flatten<T> = T extends Array<infer Item> ? Item : T;

type Str = Flatten<string[]>;    // string
type Num = Flatten<number>;      // number

// Practical: extract the resolved type from a Promise or return as-is
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
```

### The infer Keyword

The `infer` keyword declares a type variable within a conditional type's `extends` clause, letting you capture and extract parts of a type.

```typescript
// Extract the return type of a function type
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// Extract the first argument type
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any
  ? F
  : never;

type FA = FirstArg<(name: string, age: number) => void>; // string

// Extract element type from an array
type ElementType<T> = T extends (infer E)[] ? E : never;

type El = ElementType<string[]>;    // string
type El2 = ElementType<number[]>;   // number

// Extract the type a Promise resolves to
type Unwrap<T> = T extends Promise<infer U>
  ? U extends Promise<any>
    ? Unwrap<U>     // recursively unwrap nested promises
    : U
  : T;

// Multiple infer positions in a single conditional
type ParseRoute<T extends string> =
  T extends `${infer Start}/${infer Rest}`
    ? [Start, ...ParseRoute<Rest>]
    : [T];

type Segments = ParseRoute<"api/users/profile">;
// ["api", "users", "profile"]
```

### Distributive Conditional Types

When a conditional type acts on a **naked type parameter** that receives a union, it distributes over each member of the union.

```typescript
type ToArray<T> = T extends any ? T[] : never;

// Distribution: applies to each member independently
type Result = ToArray<string | number>;
// string[] | number[]  (NOT (string | number)[])

// Preventing distribution with a tuple wrapper
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type Result2 = ToArrayNonDist<string | number>;
// (string | number)[]

// Practical use: filter union members
type Filter<T, U> = T extends U ? never : T;

type WithoutFunctions = Filter<string | number | (() => void), Function>;
// string | number
```

---

## Mapped Types

Mapped types iterate over the keys of a type and transform each property. They are the `for...in` loop of the type system.

### Basic Mapped Types

```typescript
// Make all properties of T into getter functions
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface Person {
  name: string;
  age: number;
}

type PersonGetters = Getters<Person>;
// { getName: () => string; getAge: () => number }

// Create a read-only version of a type (this is how Readonly<T> works)
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Create a nullable version of all properties
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

type NullablePerson = Nullable<Person>;
// { name: string | null; age: number | null }
```

### Key Remapping with as

TypeScript 4.1+ lets you remap keys in mapped types using the `as` clause.

```typescript
// Prefix all keys with "on" and capitalize them
type EventMap<T> = {
  [K in keyof T as `on${Capitalize<string & K>}`]: (value: T[K]) => void;
};

interface State {
  count: number;
  name: string;
  active: boolean;
}

type StateEvents = EventMap<State>;
// {
//   onCount: (value: number) => void;
//   onName: (value: string) => void;
//   onActive: (value: boolean) => void;
// }

// Filter keys by remapping to never
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

type StringProps = OnlyStrings<State>;
// { name: string }

// Create setters alongside getters
type Accessors<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
} & {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};
```

### Modifiers in Mapped Types

You can add or remove `readonly` and `?` modifiers using `+` and `-` prefixes.

```typescript
// Remove readonly from all properties
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

interface FrozenConfig {
  readonly host: string;
  readonly port: number;
}

type MutableConfig = Mutable<FrozenConfig>;
// { host: string; port: number } — no longer readonly

// Remove optional modifier — make everything required
type Concrete<T> = {
  [K in keyof T]-?: T[K];
};

// This is equivalent to Required<T>
```

---

## Index Types and keyof Operator

### The keyof Operator

`keyof` produces a union of the known public property names of a type.

```typescript
interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
}

type ProductKeys = keyof Product;
// "id" | "name" | "price" | "inStock"

// keyof with index signatures
type StringMap = { [key: string]: unknown };
type SM = keyof StringMap; // string | number (number keys are auto-coerced)

// Practical: type-safe object access
function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map((item) => item[key]);
}

const products: Product[] = [
  { id: "1", name: "Widget", price: 9.99, inStock: true },
  { id: "2", name: "Gadget", price: 19.99, inStock: false },
];

const names = pluck(products, "name");    // string[]
const prices = pluck(products, "price");  // number[]
```

### Indexed Access Types

Use `T[K]` syntax to look up a specific property type on another type.

```typescript
type ProductName = Product["name"];       // string
type ProductPrice = Product["price"];     // number

// Access multiple properties at once using a union
type IdOrName = Product["id" | "name"];   // string

// All property value types
type ProductValues = Product[keyof Product];
// string | number | boolean

// Nested indexed access
interface ApiSchema {
  users: {
    list: { params: { page: number }; response: User[] };
    get: { params: { id: string }; response: User };
  };
  posts: {
    list: { params: { authorId: string }; response: Post[] };
  };
}

// Drill into nested types
type UserListResponse = ApiSchema["users"]["list"]["response"];
// User[]
```

---

## Type Guards and Type Predicates

Type guards narrow a type within a conditional block, giving you safe access to more specific properties.

### Built-in Type Guards

```typescript
function processValue(value: string | number | boolean) {
  if (typeof value === "string") {
    // TypeScript knows value is string here
    console.log(value.toUpperCase());
  } else if (typeof value === "number") {
    // TypeScript knows value is number here
    console.log(value.toFixed(2));
  } else {
    // TypeScript knows value is boolean here
    console.log(value ? "yes" : "no");
  }
}

// instanceof guard for class instances
class ApiError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

function handleError(error: Error) {
  if (error instanceof ApiError) {
    // TypeScript knows error is ApiError — statusCode is accessible
    console.log(`API Error ${error.statusCode}: ${error.message}`);
  } else {
    console.log(`Unexpected error: ${error.message}`);
  }
}

// "in" operator guard
interface Fish { swim: () => void; }
interface Bird { fly: () => void; }

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim(); // TypeScript knows animal is Fish
  } else {
    animal.fly();  // TypeScript knows animal is Bird
  }
}
```

### Custom Type Predicates

A **type predicate** is a return type annotation of the form `paramName is Type`.

```typescript
interface Cat { meow: () => void; lives: number; }
interface Dog { bark: () => void; breed: string; }

// The return type "animal is Cat" tells TypeScript to narrow the type
function isCat(animal: Cat | Dog): animal is Cat {
  return "meow" in animal;
}

function handleAnimal(animal: Cat | Dog) {
  if (isCat(animal)) {
    animal.meow();          // Safe — TypeScript knows it's a Cat
    console.log(animal.lives);
  } else {
    animal.bark();          // Safe — TypeScript knows it's a Dog
    console.log(animal.breed);
  }
}

// Type predicates with arrays — filter narrows the type
function isNonNull<T>(value: T | null | undefined): value is T {
  return value != null;
}

const values: (string | null)[] = ["hello", null, "world", null];
const strings: string[] = values.filter(isNonNull);
// Without the predicate, filter would return (string | null)[]
```

### Assertion Functions

Assertion functions throw if a condition isn't met, and TypeScript narrows the type after the call.

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error(`Expected string, got ${typeof value}`);
  }
}

function processInput(input: unknown) {
  assertIsString(input);
  // After the assertion, TypeScript knows input is string
  console.log(input.toUpperCase());
}

// Assert non-null
function assertDefined<T>(
  value: T | null | undefined,
  message?: string
): asserts value is T {
  if (value == null) {
    throw new Error(message ?? "Value is null or undefined");
  }
}
```

---

## Discriminated Unions

Discriminated unions combine union types with a **discriminant property** — a literal type that TypeScript uses to narrow the union.

### Building Discriminated Unions

```typescript
// Each variant has a "kind" property with a unique literal type
interface Circle {
  kind: "circle";
  radius: number;
}

interface Rectangle {
  kind: "rectangle";
  width: number;
  height: number;
}

interface Triangle {
  kind: "triangle";
  base: number;
  height: number;
}

type Shape = Circle | Rectangle | Triangle;

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
  }
}

// Real-world example: action types for a reducer
type Action =
  | { type: "INCREMENT"; amount: number }
  | { type: "DECREMENT"; amount: number }
  | { type: "RESET" }
  | { type: "SET"; value: number };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT":
      return state + action.amount;
    case "DECREMENT":
      return state - action.amount;
    case "RESET":
      return 0;
    case "SET":
      return action.value;
  }
}
```

### Exhaustive Checking

Use the `never` type to ensure all variants are handled.

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}

function describeShape(shape: Shape): string {
  switch (shape.kind) {
    case "circle":
      return `Circle with radius ${shape.radius}`;
    case "rectangle":
      return `Rectangle ${shape.width}x${shape.height}`;
    case "triangle":
      return `Triangle with base ${shape.base}`;
    default:
      // If you add a new shape variant but forget to handle it,
      // TypeScript will error here because shape won't be 'never'
      return assertNever(shape);
  }
}
```

---

## Template Literal Types

Template literal types build new string literal types by concatenating existing ones, mirroring JavaScript template literal syntax at the type level.

### Basic Template Literals

```typescript
type Color = "red" | "green" | "blue";
type Size = "small" | "medium" | "large";

// Generates all combinations: "small-red" | "small-green" | ... | "large-blue"
type ColorSize = `${Size}-${Color}`;

// Practical: CSS property types
type CSSUnit = "px" | "em" | "rem" | "%";
type CSSValue = `${number}${CSSUnit}`;

const width: CSSValue = "100px";   // OK
const height: CSSValue = "2.5em";  // OK
// const bad: CSSValue = "auto";   // Error

// Event handler naming pattern
type DOMEvent = "click" | "focus" | "blur" | "input";
type EventHandler = `on${Capitalize<DOMEvent>}`;
// "onClick" | "onFocus" | "onBlur" | "onInput"

// Route parameter extraction
type ExtractParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractParams<Rest>
    : T extends `${string}:${infer Param}`
      ? Param
      : never;

type Params = ExtractParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"
```

### Intrinsic String Manipulation Types

TypeScript provides built-in types for string transformation.

```typescript
type Upper = Uppercase<"hello">;       // "HELLO"
type Lower = Lowercase<"HELLO">;       // "hello"
type Cap = Capitalize<"hello">;        // "Hello"
type Uncap = Uncapitalize<"Hello">;    // "hello"

// Combine with mapped types for powerful transformations
type CamelToSnake<S extends string> =
  S extends `${infer Head}${infer Tail}`
    ? Tail extends Uncapitalize<Tail>
      ? `${Lowercase<Head>}${CamelToSnake<Tail>}`
      : `${Lowercase<Head>}_${CamelToSnake<Tail>}`
    : S;

type Snake = CamelToSnake<"getUserName">; // "get_user_name"

// Convert object keys from camelCase to snake_case at the type level
type SnakeCaseKeys<T> = {
  [K in keyof T as K extends string ? CamelToSnake<K> : K]: T[K];
};

interface ApiInput {
  userName: string;
  emailAddress: string;
}

type SnakeInput = SnakeCaseKeys<ApiInput>;
// { user_name: string; email_address: string }
```

---

## Recursive Types

Recursive types reference themselves, allowing you to model tree structures, deeply nested data, and recursive algorithms at the type level.

### Recursive Type Aliases

```typescript
// JSON-compatible types
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

// Deeply nested object type
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

interface NestedConfig {
  server: {
    host: string;
    port: number;
    ssl: {
      enabled: boolean;
      cert: string;
    };
  };
  logging: {
    level: string;
  };
}

// Every property at every level becomes optional
type PartialConfig = DeepPartial<NestedConfig>;

// Tree structures
interface TreeNode<T> {
  value: T;
  children: TreeNode<T>[];
}

const fileSystem: TreeNode<string> = {
  value: "root",
  children: [
    {
      value: "src",
      children: [
        { value: "index.ts", children: [] },
        { value: "utils.ts", children: [] },
      ],
    },
    { value: "README.md", children: [] },
  ],
};

// Deep readonly — makes every nested property readonly
type DeepReadonly<T> = T extends (infer E)[]
  ? ReadonlyArray<DeepReadonly<E>>
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;
```

### Recursive Conditional Types

```typescript
// Flatten a deeply nested array type
type DeepFlatten<T> = T extends Array<infer Inner>
  ? DeepFlatten<Inner>
  : T;

type Flat = DeepFlatten<number[][][]>; // number

// Get all possible dot-notation paths of an object
type DotPaths<T, Prefix extends string = ""> = T extends object
  ? {
      [K in keyof T & string]: T[K] extends object
        ? DotPaths<T[K], `${Prefix}${K}.`> | `${Prefix}${K}`
        : `${Prefix}${K}`;
    }[keyof T & string]
  : never;

interface Settings {
  theme: {
    color: string;
    fontSize: number;
  };
  notifications: {
    email: boolean;
    sms: boolean;
  };
}

type SettingPaths = DotPaths<Settings>;
// "theme" | "theme.color" | "theme.fontSize" | "notifications" | "notifications.email" | "notifications.sms"
```

---

## Variance and Covariance

Variance describes how subtype relationships between complex types relate to subtype relationships between their component types.

```
┌──────────────────────────────────────────────────────────┐
│                   Variance Summary                       │
├──────────────────┬───────────────────────────────────────┤
│ Covariant        │ Preserves subtype direction           │
│                  │ Cat → Animal means Box<Cat> → Box<Animal>│
├──────────────────┼───────────────────────────────────────┤
│ Contravariant    │ Reverses subtype direction            │
│                  │ Cat → Animal means Fn<Animal> → Fn<Cat>│
├──────────────────┼───────────────────────────────────────┤
│ Invariant        │ No subtype relationship               │
│                  │ Mutable containers are typically invariant│
├──────────────────┼───────────────────────────────────────┤
│ Bivariant        │ Goes both directions                  │
│                  │ TypeScript method params (legacy)      │
└──────────────────┴───────────────────────────────────────┘
```

### Covariance

A type is covariant in a position if subtypes are preserved. Return types and readonly properties are covariant.

```typescript
interface Animal { name: string; }
interface Cat extends Animal { purr: () => void; }

// Function return types are covariant
type Producer<T> = () => T;

// Cat extends Animal, so Producer<Cat> extends Producer<Animal>
const catProducer: Producer<Cat> = () => ({ name: "Whiskers", purr: () => {} });
const animalProducer: Producer<Animal> = catProducer; // OK

// ReadonlyArray is covariant
const cats: readonly Cat[] = [{ name: "Whiskers", purr: () => {} }];
const animals: readonly Animal[] = cats; // OK — can't mutate, so it's safe
```

### Contravariance

A type is contravariant in a position if subtypes are reversed. Function parameter types are contravariant (with `strictFunctionTypes` enabled).

```typescript
// Function parameter types are contravariant
type Consumer<T> = (item: T) => void;

const feedAnimal: Consumer<Animal> = (animal) => {
  console.log(`Feeding ${animal.name}`);
};

// Animal is a supertype of Cat, but Consumer<Animal> is a subtype of Consumer<Cat>
const feedCat: Consumer<Cat> = feedAnimal; // OK — anything that can handle any Animal can handle a Cat
// const feedAnimalFromCat: Consumer<Animal> = feedCat; // Error — a Cat feeder might call purr()
```

### The Variance Annotations

TypeScript 4.7+ introduced explicit variance annotations using `in` and `out` keywords.

```typescript
// out = covariant (used in output positions)
// in = contravariant (used in input positions)

interface ReadonlyBox<out T> {
  get(): T;
}

interface WriteOnlyBox<in T> {
  set(value: T): void;
}

interface MutableBox<in out T> {
  get(): T;
  set(value: T): void;
}

// Variance annotations help TypeScript check type correctness faster
// and document the intended relationship for library consumers
```

---

## Next Steps

Continue your TypeScript learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | TypeScript Fundamentals | Core language features, basic types, and project setup |
| [02-PATTERNS.md](02-PATTERNS.md) | Design Patterns | Common design patterns implemented in TypeScript |
| [03-NODE-AND-BACKEND.md](03-NODE-AND-BACKEND.md) | Server-side TypeScript | Building backend services with Node.js and TypeScript |
| [04-FRONTEND-FRAMEWORKS.md](04-FRONTEND-FRAMEWORKS.md) | Frontend Frameworks | Using TypeScript with React, Angular, and Vue |
| [05-TESTING.md](05-TESTING.md) | Testing | Testing strategies and tools for TypeScript projects |
| [06-TOOLING.md](06-TOOLING.md) | Tooling | Compilers, bundlers, linters, and developer experience |
| [07-BEST-PRACTICES.md](07-BEST-PRACTICES.md) | Best Practices | Conventions and patterns for production TypeScript |
| [08-ANTI-PATTERNS.md](08-ANTI-PATTERNS.md) | Anti-Patterns | Common mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured guide through all TypeScript topics |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial TypeScript Type System documentation |
