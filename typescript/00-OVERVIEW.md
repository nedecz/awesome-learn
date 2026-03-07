# TypeScript Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [What is TypeScript](#what-is-typescript)
3. [TypeScript vs JavaScript](#typescript-vs-javascript)
4. [The Type System](#the-type-system)
5. [Type Inference and Type Narrowing](#type-inference-and-type-narrowing)
6. [Compiler Options and tsconfig.json](#compiler-options-and-tsconfigjson)
7. [Modules and Namespaces](#modules-and-namespaces)
8. [Declaration Files](#declaration-files)
9. [The TypeScript Compilation Process](#the-typescript-compilation-process)
10. [Setting Up a TypeScript Project](#setting-up-a-typescript-project)
11. [Prerequisites](#prerequisites)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

This documentation provides a comprehensive introduction to TypeScript fundamentals. It covers what TypeScript is and why it exists, the type system that powers it, compiler configuration, type inference and narrowing, modules, declaration files, and practical guidance for setting up and working with TypeScript projects.

### Target Audience

- **JavaScript Developers** adopting TypeScript for the first time and looking to understand how types layer on top of existing JavaScript knowledge
- **Backend Engineers** working with Node.js who want to add type safety to server-side applications
- **Frontend Engineers** building applications with React, Angular, Vue, or other frameworks that benefit from TypeScript
- **Full-Stack Developers** looking for a single language with type safety across the entire stack
- **Engineers New to TypeScript** who want a solid foundation in the type system, compiler, and project setup

### Scope

- What TypeScript is, how it relates to JavaScript, and why it matters
- TypeScript vs JavaScript: syntax, type safety, tooling, and ecosystem differences
- The type system: primitives, interfaces, type aliases, unions, intersections, generics, and enums
- Type inference and type narrowing: how TypeScript reduces boilerplate while keeping safety
- Compiler options and `tsconfig.json`: configuring the TypeScript compiler for your project
- Modules and namespaces: organizing code in TypeScript
- Declaration files (`.d.ts`): typing third-party and untyped JavaScript libraries
- The TypeScript compilation process: from `.ts` to `.js`
- Setting up a TypeScript project from scratch

---

## What is TypeScript

**TypeScript** is a statically typed superset of JavaScript developed and maintained by Microsoft. Every valid JavaScript program is a valid TypeScript program, but TypeScript adds an optional type system and compile-time checks that catch errors before code runs in production.

### Why TypeScript Matters

TypeScript exists to solve real problems that JavaScript developers encounter at scale:

- **Catch errors early** — Type errors are detected at compile time rather than at runtime, preventing entire categories of bugs from reaching production
- **Improve developer experience** — Editors and IDEs use type information to provide autocompletion, inline documentation, and refactoring tools that are impossible without types
- **Enable safe refactoring** — Renaming a property, changing a function signature, or restructuring a module triggers compile errors everywhere the change has an impact
- **Document intent** — Types serve as machine-checked documentation that describes what a function accepts, what it returns, and what shape data takes
- **Scale with confidence** — As codebases grow from hundreds to thousands of files, types provide guardrails that keep teams productive and reduce onboarding time

### How TypeScript Works

TypeScript introduces a compilation step between writing code and running it. The TypeScript compiler (`tsc`) reads `.ts` files, checks them for type errors, and emits standard JavaScript (`.js`) files that run anywhere JavaScript runs — browsers, Node.js, Deno, Bun, or serverless platforms.

```
TypeScript Compilation Flow
────────────────────────────
                                    Type information
                                    is erased at compile time
                                    ↓
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  .ts     │────►│  tsc     │────►│  .js     │────►│ Runtime  │
│  source  │     │ compiler │     │  output  │     │ (browser,│
│  files   │     │          │     │  files   │     │  Node.js)│
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                      │
                      ▼
                 Type errors
                 reported at
                 compile time
```

**Key insight:** TypeScript types exist only at compile time. They are completely erased from the emitted JavaScript. This means TypeScript adds zero runtime overhead — the JavaScript that runs is exactly what you would have written by hand, but with the confidence that the type checker has validated your code.

---

## TypeScript vs JavaScript

Understanding the differences between TypeScript and JavaScript is essential for JavaScript developers adopting TypeScript.

### Syntax Comparison

```typescript
// JavaScript
function greet(name) {
  return "Hello, " + name;
}

// TypeScript — the only difference is the type annotation
function greet(name: string): string {
  return "Hello, " + name;
}
```

```typescript
// JavaScript — no way to express the shape of the parameter
function createUser(config) {
  return {
    id: Math.random().toString(36),
    name: config.name,
    email: config.email,
    role: config.role || "viewer",
  };
}

// TypeScript — the shape is explicit and enforced
interface UserConfig {
  name: string;
  email: string;
  role?: "admin" | "editor" | "viewer";
}

interface User {
  id: string;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
}

function createUser(config: UserConfig): User {
  return {
    id: Math.random().toString(36),
    name: config.name,
    email: config.email,
    role: config.role || "viewer",
  };
}
```

### Key Differences

```
TypeScript vs JavaScript
─────────────────────────
Feature               JavaScript          TypeScript
──────────────        ──────────          ──────────
Type system           Dynamic             Static (compile-time)
Type annotations      None                Optional, explicit
Error detection       Runtime             Compile-time + Runtime
Compilation step      Not required        Required (tsc)
Tooling support       Good                Excellent (autocomplete, refactoring)
Learning curve        Lower               Slightly higher
Runtime overhead      None                None (types are erased)
File extension        .js / .mjs          .ts / .mts
Ecosystem             npm / yarn          npm / yarn (same ecosystem)
Browser support       Direct              Requires compilation to JS
```

### When to Use TypeScript

TypeScript provides the most value when:

- The codebase is large or growing — types prevent regressions as the system scales
- Multiple developers collaborate — types serve as contracts between modules and teams
- The application is long-lived — types make maintenance and refactoring safer
- You need strong editor tooling — autocompletion, go-to-definition, and inline errors
- You are building libraries consumed by others — types provide a documented API surface

---

## The Type System

The TypeScript type system is structural (based on the shape of values, not their declared class) and gradual (you can adopt types incrementally). This section covers the core building blocks.

### Primitive Types

TypeScript includes type annotations for all JavaScript primitives:

```typescript
// String
let name: string = "Alice";

// Number (integers and floats are both 'number')
let age: number = 30;
let price: number = 19.99;

// Boolean
let isActive: boolean = true;

// Null and Undefined
let nothing: null = null;
let notDefined: undefined = undefined;

// BigInt
let largeNumber: bigint = 9007199254740991n;

// Symbol
let uniqueKey: symbol = Symbol("key");
```

### Arrays and Tuples

```typescript
// Arrays — two equivalent syntaxes
let numbers: number[] = [1, 2, 3];
let strings: Array<string> = ["a", "b", "c"];

// Tuples — fixed-length arrays with known types at each position
let pair: [string, number] = ["age", 30];
let rgb: [number, number, number] = [255, 128, 0];

// Readonly arrays — prevent mutation
let frozen: readonly number[] = [1, 2, 3];
// frozen.push(4); // Error: Property 'push' does not exist on type 'readonly number[]'
```

### Interfaces

Interfaces define the shape of an object. They are open (can be extended and merged) and are the primary way to describe object types in TypeScript.

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age?: number; // optional property
  readonly createdAt: Date; // cannot be reassigned after creation
}

// Extending interfaces
interface AdminUser extends User {
  permissions: string[];
  department: string;
}

// Implementing an interface
const admin: AdminUser = {
  id: "1",
  name: "Alice",
  email: "alice@example.com",
  createdAt: new Date(),
  permissions: ["users:read", "users:write"],
  department: "Engineering",
};
```

```typescript
// Interface with method signatures
interface Logger {
  info(message: string): void;
  error(message: string, error?: Error): void;
  debug(message: string, context?: Record<string, unknown>): void;
}

// Interface with index signatures
interface Dictionary {
  [key: string]: string;
}

const headers: Dictionary = {
  "Content-Type": "application/json",
  Authorization: "Bearer token123",
};
```

### Type Aliases

Type aliases create a name for any type. Unlike interfaces, they can represent primitives, unions, intersections, tuples, and other complex types.

```typescript
// Alias for a primitive
type ID = string;

// Alias for a union
type Status = "pending" | "active" | "inactive";

// Alias for a function type
type EventHandler = (event: Event) => void;

// Alias for an object type
type Point = {
  x: number;
  y: number;
};

// Alias for a complex type
type ApiResponse<T> = {
  data: T;
  status: number;
  message: string;
  timestamp: Date;
};
```

### Interfaces vs Type Aliases

```
When to Use Interfaces vs Type Aliases
───────────────────────────────────────
Use interfaces when:
  - Defining the shape of objects or classes
  - You need declaration merging (extending across files)
  - You are building a public API or library

Use type aliases when:
  - Defining union or intersection types
  - Creating aliases for primitives, tuples, or function types
  - You need mapped or conditional types
```

### Union Types

Union types represent a value that can be one of several types. They are declared with the `|` operator.

```typescript
// A value that is either a string or a number
type StringOrNumber = string | number;

function formatId(id: StringOrNumber): string {
  if (typeof id === "string") {
    return id.toUpperCase();
  }
  return id.toString().padStart(6, "0");
}

// Discriminated unions — the preferred pattern for modeling variants
interface SuccessResult {
  status: "success";
  data: unknown;
}

interface ErrorResult {
  status: "error";
  error: string;
}

type Result = SuccessResult | ErrorResult;

function handleResult(result: Result): void {
  switch (result.status) {
    case "success":
      console.log("Data:", result.data);
      break;
    case "error":
      console.error("Error:", result.error);
      break;
  }
}
```

### Intersection Types

Intersection types combine multiple types into one. The resulting type has all properties from every constituent type.

```typescript
interface HasId {
  id: string;
}

interface HasTimestamps {
  createdAt: Date;
  updatedAt: Date;
}

interface HasName {
  name: string;
}

// Intersection: the resulting type has id, name, createdAt, and updatedAt
type Entity = HasId & HasTimestamps & HasName;

const entity: Entity = {
  id: "abc-123",
  name: "Example Entity",
  createdAt: new Date("2025-01-01"),
  updatedAt: new Date("2025-06-01"),
};
```

### Generics

Generics allow you to write reusable components that work with any type while preserving type safety.

```typescript
// Generic function
function identity<T>(value: T): T {
  return value;
}

const str = identity("hello"); // type is string
const num = identity(42); // type is number

// Generic interface
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(item: Omit<T, "id">): Promise<T>;
  update(id: string, item: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}

// Generic with constraints
interface Lengthwise {
  length: number;
}

function logLength<T extends Lengthwise>(value: T): void {
  console.log(`Length: ${value.length}`);
}

logLength("hello"); // OK — string has .length
logLength([1, 2, 3]); // OK — array has .length
// logLength(42);     // Error — number does not have .length
```

### Enums

Enums define a set of named constants. TypeScript supports numeric and string enums.

```typescript
// String enum — preferred for readability and debuggability
enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT",
}

// Numeric enum
enum HttpStatus {
  OK = 200,
  Created = 201,
  BadRequest = 400,
  Unauthorized = 401,
  NotFound = 404,
  InternalServerError = 500,
}

function handleResponse(status: HttpStatus): void {
  if (status === HttpStatus.OK) {
    console.log("Request succeeded");
  }
}

// const enum — inlined at compile time for zero runtime overhead
const enum Feature {
  DarkMode = "DARK_MODE",
  BetaAccess = "BETA_ACCESS",
  NewDashboard = "NEW_DASHBOARD",
}
```

### Utility Types

TypeScript provides built-in utility types that transform existing types:

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
}

// Partial<T> — makes all properties optional
type PartialUser = Partial<User>;

// Required<T> — makes all properties required
type RequiredUser = Required<PartialUser>;

// Pick<T, K> — selects a subset of properties
type UserPreview = Pick<User, "id" | "name">;

// Omit<T, K> — removes specified properties
type CreateUserInput = Omit<User, "id">;

// Record<K, V> — constructs an object type with keys K and values V
type UserRoles = Record<string, "admin" | "editor" | "viewer">;

// Readonly<T> — makes all properties readonly
type FrozenUser = Readonly<User>;
```

---

## Type Inference and Type Narrowing

TypeScript is designed to require as few explicit type annotations as possible. The compiler infers types from context, and narrows types within control flow blocks.

### Type Inference

```typescript
// TypeScript infers the type from the assigned value
let count = 10; // inferred as number
let message = "hello"; // inferred as string
let active = true; // inferred as boolean
let items = [1, 2, 3]; // inferred as number[]

// Return type inference — the compiler knows this returns a string
function greet(name: string) {
  return `Hello, ${name}!`;
}

// Object literal inference
const config = {
  host: "localhost",
  port: 3000,
  debug: false,
}; // inferred as { host: string; port: number; debug: boolean }
```

### Type Narrowing

Type narrowing is the process by which TypeScript refines a broad type to a more specific type within a conditional block.

```typescript
// typeof narrowing
function processValue(value: string | number): string {
  if (typeof value === "string") {
    // TypeScript knows value is a string here
    return value.toUpperCase();
  }
  // TypeScript knows value is a number here
  return value.toFixed(2);
}

// instanceof narrowing
function formatError(error: unknown): string {
  if (error instanceof Error) {
    // TypeScript knows error is an Error here
    return `${error.name}: ${error.message}`;
  }
  return String(error);
}

// Truthiness narrowing
function printName(name: string | null | undefined): void {
  if (name) {
    // TypeScript knows name is a string here (not null/undefined)
    console.log(name.toUpperCase());
  } else {
    console.log("No name provided");
  }
}

// in operator narrowing
interface Cat {
  meow(): void;
}

interface Dog {
  bark(): void;
}

function makeSound(animal: Cat | Dog): void {
  if ("meow" in animal) {
    animal.meow(); // TypeScript knows this is a Cat
  } else {
    animal.bark(); // TypeScript knows this is a Dog
  }
}
```

### Type Guards

Custom type guards let you define reusable narrowing logic:

```typescript
interface ApiError {
  code: number;
  message: string;
}

interface ValidationError {
  field: string;
  message: string;
}

// Type predicate — the return type 'value is ApiError' tells TypeScript
// that if this function returns true, the value is an ApiError
function isApiError(value: unknown): value is ApiError {
  return (
    typeof value === "object" &&
    value !== null &&
    "code" in value &&
    "message" in value
  );
}

function handleError(error: ApiError | ValidationError): void {
  if (isApiError(error)) {
    console.error(`API Error ${error.code}: ${error.message}`);
  } else {
    console.error(`Validation Error on ${error.field}: ${error.message}`);
  }
}
```

---

## Compiler Options and tsconfig.json

The TypeScript compiler is configured through `tsconfig.json`, a JSON file at the root of a TypeScript project. It controls what files are compiled, how strict the type checking is, and what JavaScript version is emitted.

### Basic tsconfig.json

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### Key Compiler Options

```
Essential Compiler Options
──────────────────────────
Option                          Purpose
──────                          ───────
target                          ECMAScript version for emitted JS (ES2020, ES2022, ESNext)
module                          Module system for emitted JS (CommonJS, NodeNext, ESNext)
moduleResolution                How module specifiers are resolved (node, NodeNext, bundler)
lib                             Built-in type declarations to include (ES2022, DOM)
outDir                          Output directory for compiled .js files
rootDir                         Root directory of source .ts files
strict                          Enable all strict type-checking options
esModuleInterop                 Emit helpers for CommonJS/ESM interop
declaration                     Generate .d.ts declaration files
sourceMap                       Generate .map files for debugging
skipLibCheck                    Skip type checking of declaration files
resolveJsonModule               Allow importing .json files
```

### Strict Mode Options

The `strict` flag is a shorthand that enables all of the following individual checks:

```typescript
// When "strict": true is set, all of these are enabled:
{
  "compilerOptions": {
    "strict": true
    // Equivalent to enabling all of:
    // "noImplicitAny": true          — Error on expressions with implied 'any' type
    // "noImplicitThis": true         — Error when 'this' has an implied 'any' type
    // "strictNullChecks": true       — null and undefined are not assignable to other types
    // "strictFunctionTypes": true    — Stricter checking of function type assignments
    // "strictBindCallApply": true    — Check that bind, call, apply match the function signature
    // "strictPropertyInitialization": true — Class properties must be initialized
    // "alwaysStrict": true           — Emit 'use strict' in every file
    // "useUnknownInCatchVariables": true  — Default catch variable type is 'unknown'
  }
}
```

### Project References

For large projects, TypeScript supports project references to split a codebase into smaller, independently compilable units:

```typescript
// tsconfig.json (root)
{
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/api" },
    { "path": "./packages/web" }
  ],
  "files": []
}

// packages/core/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true
  },
  "include": ["src/**/*"]
}
```

---

## Modules and Namespaces

### ES Modules

TypeScript fully supports the standard ECMAScript module system. ES modules are the recommended way to organize code.

```typescript
// math.ts — named exports
export function add(a: number, b: number): number {
  return a + b;
}

export function multiply(a: number, b: number): number {
  return a * b;
}

export const PI = 3.14159265359;

// app.ts — named imports
import { add, multiply, PI } from "./math.js";

console.log(add(2, 3)); // 5
console.log(multiply(PI, 2)); // 6.28318530718
```

```typescript
// logger.ts — default export
export default class Logger {
  constructor(private prefix: string) {}

  info(message: string): void {
    console.log(`[${this.prefix}] INFO: ${message}`);
  }

  error(message: string): void {
    console.error(`[${this.prefix}] ERROR: ${message}`);
  }
}

// app.ts — default import
import Logger from "./logger.js";

const log = new Logger("App");
log.info("Application started");
```

### Re-exports and Barrel Files

```typescript
// models/user.ts
export interface User {
  id: string;
  name: string;
}

// models/order.ts
export interface Order {
  id: string;
  userId: string;
  total: number;
}

// models/index.ts — barrel file re-exports everything
export { User } from "./user.js";
export { Order } from "./order.js";

// app.ts — import from the barrel
import { User, Order } from "./models/index.js";
```

### Namespaces

Namespaces are a TypeScript-specific way to organize code. They are rarely used in modern TypeScript — prefer ES modules instead. They still appear in legacy code and in ambient declaration files.

```typescript
// Namespaces are primarily useful in declaration files
// and legacy codebases. Prefer ES modules for new code.
namespace Validation {
  export interface Validator {
    isValid(value: string): boolean;
  }

  export class EmailValidator implements Validator {
    isValid(value: string): boolean {
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
    }
  }
}

const validator = new Validation.EmailValidator();
console.log(validator.isValid("user@example.com")); // true
```

---

## Declaration Files

Declaration files (`.d.ts`) describe the types of JavaScript code without providing implementations. They are essential for using untyped JavaScript libraries in TypeScript.

### What Declaration Files Do

```
Declaration Files
──────────────────
.ts files    →  Contain both code and type information
.js files    →  Contain only code (no type information)
.d.ts files  →  Contain only type information (no code)

When TypeScript encounters an import:
  1. It looks for a .ts file first
  2. Then a .d.ts file next to the .js file
  3. Then a @types package in node_modules
  4. If nothing is found, the import is typed as 'any' (or errors under strict mode)
```

### Writing Declaration Files

```typescript
// types/config.d.ts — declare types for a JavaScript module
declare module "app-config" {
  interface AppConfig {
    port: number;
    host: string;
    database: {
      url: string;
      poolSize: number;
    };
    logging: {
      level: "debug" | "info" | "warn" | "error";
      format: "json" | "text";
    };
  }

  export function loadConfig(path: string): AppConfig;
  export function getConfig(): AppConfig;
}
```

### Using DefinitelyTyped (@types)

The DefinitelyTyped repository provides type declarations for thousands of JavaScript packages. These are published as `@types/*` packages on npm.

```typescript
// Install type declarations for a package
// npm install --save-dev @types/express @types/node

// Now TypeScript knows the types for express
import express, { Request, Response } from "express";

const app = express();

app.get("/api/health", (req: Request, res: Response) => {
  res.json({ status: "healthy", timestamp: new Date().toISOString() });
});
```

### Global Declarations

```typescript
// global.d.ts — extend global types
declare global {
  interface Window {
    __APP_CONFIG__: {
      apiUrl: string;
      version: string;
    };
  }
}

// This empty export makes the file a module, which is required
// for the 'declare global' augmentation to work
export {};
```

---

## The TypeScript Compilation Process

Understanding how TypeScript compiles code helps you debug build issues and optimize your workflow.

### Compilation Steps

```
TypeScript Compilation Pipeline
─────────────────────────────────
Step 1: Parse
  .ts source files → Abstract Syntax Tree (AST)
  The parser reads TypeScript source code and produces a tree
  representation of the program structure.

Step 2: Bind
  AST → Symbol Table
  The binder walks the AST and creates a symbol for each
  declaration (variables, functions, classes, interfaces).

Step 3: Check
  AST + Symbol Table → Diagnostics (errors and warnings)
  The type checker validates that every expression conforms
  to the type system rules. This is where type errors are reported.

Step 4: Emit
  AST → .js output files + .d.ts declarations + .map source maps
  The emitter transforms TypeScript AST nodes into JavaScript,
  stripping all type annotations. Types are completely erased.
```

### What Gets Erased

Everything that is TypeScript-specific is removed during compilation:

```typescript
// Before compilation (TypeScript)
interface User {
  id: string;
  name: string;
}

function greet(user: User): string {
  return `Hello, ${user.name}!`;
}

const alice: User = { id: "1", name: "Alice" };
console.log(greet(alice));

// After compilation (JavaScript) — types are erased
function greet(user) {
  return `Hello, ${user.name}!`;
}

const alice = { id: "1", name: "Alice" };
console.log(greet(alice));
```

### What Gets Transformed

Some TypeScript features emit runtime code:

```typescript
// Enums emit runtime objects
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}

// Compiles to:
// var Color;
// (function (Color) {
//     Color["Red"] = "RED";
//     Color["Green"] = "GREEN";
//     Color["Blue"] = "BLUE";
// })(Color || (Color = {}));

// Decorators (when enabled) emit runtime metadata
// Class fields with 'declare' are erased; without 'declare' they emit initialization code
```

---

## Setting Up a TypeScript Project

### New Project from Scratch

```typescript
// Step 1: Initialize the project
// mkdir my-project && cd my-project
// npm init -y

// Step 2: Install TypeScript
// npm install --save-dev typescript

// Step 3: Initialize tsconfig.json
// npx tsc --init

// Step 4: Create source directory
// mkdir src

// Step 5: Create entry point
// src/index.ts
```

### Recommended Project Structure

```
Project Structure
─────────────────
my-project/
├── src/                     # TypeScript source files
│   ├── index.ts             # Entry point
│   ├── config.ts            # Configuration
│   ├── models/              # Data models and interfaces
│   │   ├── user.ts
│   │   └── index.ts         # Barrel file
│   ├── services/            # Business logic
│   │   ├── user-service.ts
│   │   └── index.ts
│   └── utils/               # Utility functions
│       ├── logger.ts
│       └── index.ts
├── tests/                   # Test files
│   ├── user-service.test.ts
│   └── setup.ts
├── dist/                    # Compiled output (gitignored)
├── tsconfig.json            # TypeScript configuration
├── package.json             # Node.js project manifest
└── .gitignore               # Ignore dist/, node_modules/
```

### Recommended tsconfig.json for Node.js

```typescript
// tsconfig.json — modern Node.js project (Node 20+)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Package.json Scripts

```typescript
// package.json (relevant sections)
{
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "start": "node dist/index.js",
    "lint": "eslint src/",
    "typecheck": "tsc --noEmit"
  }
}
```

### A Complete Example

```typescript
// src/models/task.ts
export interface Task {
  id: string;
  title: string;
  description: string;
  status: "todo" | "in-progress" | "done";
  priority: "low" | "medium" | "high";
  createdAt: Date;
  updatedAt: Date;
}

export type CreateTaskInput = Omit<Task, "id" | "createdAt" | "updatedAt">;
export type UpdateTaskInput = Partial<Omit<Task, "id" | "createdAt" | "updatedAt">>;
```

```typescript
// src/services/task-service.ts
import { Task, CreateTaskInput, UpdateTaskInput } from "../models/task.js";
import { randomUUID } from "node:crypto";

export class TaskService {
  private tasks: Map<string, Task> = new Map();

  create(input: CreateTaskInput): Task {
    const now = new Date();
    const task: Task = {
      id: randomUUID(),
      ...input,
      createdAt: now,
      updatedAt: now,
    };
    this.tasks.set(task.id, task);
    return task;
  }

  findById(id: string): Task | undefined {
    return this.tasks.get(id);
  }

  findAll(): Task[] {
    return Array.from(this.tasks.values());
  }

  update(id: string, input: UpdateTaskInput): Task | undefined {
    const existing = this.tasks.get(id);
    if (!existing) {
      return undefined;
    }
    const updated: Task = {
      ...existing,
      ...input,
      updatedAt: new Date(),
    };
    this.tasks.set(id, updated);
    return updated;
  }

  delete(id: string): boolean {
    return this.tasks.delete(id);
  }
}
```

```typescript
// src/index.ts
import { TaskService } from "./services/task-service.js";

const service = new TaskService();

const task = service.create({
  title: "Learn TypeScript",
  description: "Complete the TypeScript fundamentals documentation",
  status: "in-progress",
  priority: "high",
});

console.log("Created task:", task);

service.update(task.id, { status: "done" });
console.log("Updated task:", service.findById(task.id));
console.log("All tasks:", service.findAll());
```

---

## Prerequisites

Before diving into TypeScript topics, you should be familiar with:

- **JavaScript fundamentals** — Variables, functions, objects, arrays, closures, promises, and async/await
- **ES6+ features** — Arrow functions, destructuring, template literals, classes, modules (import/export), and spread/rest operators
- **Node.js basics** — npm/yarn, package.json, running scripts, and module resolution
- **Command line basics** — Terminal, shell commands, file system navigation
- **A code editor** — VS Code is recommended for the best TypeScript experience (built-in language support)

---

## Next Steps

Continue your TypeScript learning journey:

| File | Topic | Description |
|---|---|---|
| [01-TYPE-SYSTEM.md](01-TYPE-SYSTEM.md) | Type System | Generics, utility types, conditional types, mapped types |
| [02-PATTERNS.md](02-PATTERNS.md) | Patterns | Design patterns in TypeScript, functional patterns |
| [07-BEST-PRACTICES.md](07-BEST-PRACTICES.md) | Best Practices | Strict mode, type safety, error handling |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured curriculum with exercises |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial TypeScript Fundamentals documentation |
