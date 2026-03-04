# TypeScript Learning Path

A structured, self-paced training guide to mastering TypeScript — from foundational types and the type system through advanced patterns, full-stack development, testing, tooling, and production best practices. Each phase builds on the previous one, progressing from core concepts to advanced production patterns.

> **Time Estimate:** 8–10 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers with prior TypeScript experience may complete Phases 1–2 in half the time.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how TypeScript concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all phases and produces a portfolio artifact you can reference in technical conversations

---

## Phase 1: TypeScript Foundations (Week 1–2)

### Learning Objectives

- Understand TypeScript's role in the JavaScript ecosystem and why it exists
- Learn primitive types, literal types, union types, and intersection types
- Understand structural typing and how it differs from nominal typing
- Master type narrowing, type guards, and discriminated unions
- Use generics to write reusable, type-safe abstractions
- Understand utility types (`Partial`, `Required`, `Pick`, `Omit`, `Record`)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | TypeScript's purpose, compiler overview, project setup, `tsconfig.json` essentials, migration strategies |
| 2 | [01-TYPE-SYSTEM](01-TYPE-SYSTEM.md) | Primitive types, unions, intersections, type narrowing, generics, utility types, conditional types, mapped types |

### Exercises

**1. TypeScript Project Initialization:**

Set up a TypeScript project from scratch to build muscle memory for project configuration:

- Initialize a new project with `npm init` and install TypeScript as a dev dependency
- Create a `tsconfig.json` with `strict: true` and target `ES2022`
- Configure `rootDir`, `outDir`, `declaration`, and `sourceMap` options
- Add build and type-check scripts to `package.json`
- Verify the setup compiles a simple `index.ts` file without errors

**2. Type System Deep Dive:**

Create a file called `type-exercises.ts` that demonstrates mastery of the type system:

- Define a discriminated union for `Shape` (circle, rectangle, triangle) with the appropriate fields for each variant
- Write a `calculateArea` function that uses type narrowing via `switch` on the discriminant to compute the area
- Create a generic `Result<T, E>` type representing either `{ ok: true; value: T }` or `{ ok: false; error: E }`
- Implement a generic `map` function that transforms `Result<T, E>` into `Result<U, E>`
- Use conditional types to create a `NonNullableProperties<T>` type that removes `null` and `undefined` from all properties of `T`

**3. Utility Type Practice:**

Build a set of custom utility types to internalize how mapped and conditional types work:

- Implement your own version of `Pick<T, K>` using mapped types
- Implement your own version of `Readonly<T>` using mapped types
- Create a `Mutable<T>` type that removes `readonly` from all properties
- Create a `DeepPartial<T>` type that recursively makes all nested properties optional
- Write test cases using `type Expect<T extends true> = T` and `type Equal<X, Y>` to verify each type works correctly

### Knowledge Check

- [ ] Can you explain the difference between `type` and `interface` and when to use each?
- [ ] Can you describe structural typing and give an example where it differs from nominal typing?
- [ ] Can you narrow a union type using `in`, `typeof`, `instanceof`, and discriminant checks?
- [ ] Can you write a generic function with constraints using `extends`?
- [ ] Can you explain how `keyof`, `typeof`, and indexed access types work together?

---

## Phase 2: Patterns & Architecture (Week 3–4)

### Learning Objectives

- Apply common TypeScript design patterns (factory, builder, strategy, observer)
- Use branded types and opaque types for domain modeling
- Implement type-safe event systems and state machines
- Design module boundaries with explicit public APIs
- Apply dependency injection patterns in TypeScript

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-PATTERNS](02-PATTERNS.md) | Design patterns in TypeScript, branded types, type-safe state machines, module design, dependency injection, error handling patterns |

### Exercises

**1. Branded Types for Domain Safety:**

Create a domain model for a financial application using branded types to prevent value misuse:

- Define branded types for `USD`, `EUR`, and `AccountId` so they cannot be accidentally interchanged
- Implement constructor functions (`createUSD`, `createEUR`) that validate input and return the branded type
- Write an `addCurrency` function that only accepts two values of the same currency type
- Demonstrate that the TypeScript compiler rejects mixing `USD` and `EUR` at compile time
- Add a `convertCurrency` function that accepts a `from` amount and an exchange rate and returns the target currency type

**2. Type-Safe State Machine:**

Implement a type-safe order state machine for an e-commerce system:

- Define the states: `Draft`, `Submitted`, `Paid`, `Shipped`, `Delivered`, `Cancelled`
- Define valid transitions (e.g., `Draft → Submitted`, `Submitted → Paid`, `Submitted → Cancelled`)
- Use the type system to make invalid transitions a compile-time error
- Implement transition functions that accept the current state and return the next state
- Add event payloads for each transition (e.g., `Paid` carries a `paymentId`, `Shipped` carries a `trackingNumber`)

**3. Dependency Injection Container:**

Build a lightweight type-safe dependency injection container:

- Create a `Container` class that registers and resolves dependencies by token
- Use generics so that `container.resolve<UserService>('userService')` returns the correct type
- Support singleton and transient lifetimes
- Implement constructor injection where dependencies are automatically resolved
- Write tests that verify resolution works correctly and that circular dependencies are detected

### Knowledge Check

- [ ] Can you explain the builder pattern in TypeScript and when it improves API ergonomics?
- [ ] Can you implement branded types and explain why they are useful for domain modeling?
- [ ] Can you design a type-safe state machine where invalid transitions are compile-time errors?
- [ ] Can you explain the difference between singleton and transient dependency lifetimes?
- [ ] Can you describe three strategies for error handling in TypeScript and their trade-offs?

---

## Phase 3: Full-Stack TypeScript (Week 5–6)

### Learning Objectives

- Build type-safe REST and GraphQL APIs with Node.js
- Share type definitions between client and server
- Use React with TypeScript effectively (props, hooks, context)
- Implement end-to-end type safety from database to UI
- Understand server-side rendering and static generation with TypeScript

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [03-NODE-AND-BACKEND](03-NODE-AND-BACKEND.md) | Express/Fastify with TypeScript, type-safe middleware, ORM integration, API validation with Zod, error handling |
| 2 | [04-FRONTEND-FRAMEWORKS](04-FRONTEND-FRAMEWORKS.md) | React + TypeScript, component typing, hooks typing, state management, type-safe routing, form handling |

### Exercises

**1. Type-Safe REST API:**

Build a REST API for a task management system using Node.js and TypeScript:

- Set up an Express or Fastify server with full TypeScript configuration
- Define request and response types for each endpoint (`GET /tasks`, `POST /tasks`, `PUT /tasks/:id`, `DELETE /tasks/:id`)
- Use Zod schemas for runtime validation and infer TypeScript types from the schemas (`z.infer<typeof TaskSchema>`)
- Implement type-safe middleware for authentication that adds a `user` property to the request object
- Create a typed error handler that returns consistent error response shapes
- Add OpenAPI documentation generated from the Zod schemas

**2. React Frontend with TypeScript:**

Build a React frontend that consumes the task management API:

- Create typed component props using `interface` for each component
- Implement a custom `useFetch<T>` hook that returns `{ data: T | null; loading: boolean; error: Error | null }`
- Use discriminated unions for component state (e.g., `Idle | Loading | Success<T> | Error`)
- Implement a form with type-safe form handling (controlled inputs with typed state)
- Create a typed React context for authentication state with a custom `useAuth` hook
- Use generic components (e.g., a `<DataTable<T>>` component that accepts typed column definitions)

**3. Shared Types Across Stack:**

Establish a shared type package between the API and frontend:

- Create a shared `types` package (or directory) with the domain models used by both client and server
- Define API contract types: `ApiResponse<T>`, `PaginatedResponse<T>`, `ErrorResponse`
- Ensure changes to the shared types produce compile-time errors in both the API and frontend
- Set up path aliases in `tsconfig.json` to import shared types cleanly
- Document the workflow for updating shared types and verifying compatibility

### Knowledge Check

- [ ] Can you explain how to infer TypeScript types from Zod schemas?
- [ ] Can you type Express middleware that extends the request object?
- [ ] Can you write a generic React component with properly typed props?
- [ ] Can you explain the trade-offs between `interface` and `type` for React component props?
- [ ] Can you describe how to share types between a Node.js API and a React frontend in a monorepo?

---

## Phase 4: Testing & Tooling (Week 7–8)

### Learning Objectives

- Write type-safe unit tests, integration tests, and end-to-end tests
- Use testing utilities effectively (mocking, stubbing, fixtures)
- Configure ESLint, Prettier, and the TypeScript compiler for team consistency
- Set up CI/CD pipelines that enforce type safety
- Use performance profiling and bundle analysis tools

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-TESTING](05-TESTING.md) | Testing strategies, type-safe mocks, test utilities, integration testing, E2E testing with Playwright |
| 2 | [06-TOOLING](06-TOOLING.md) | ESLint + typescript-eslint, Prettier, build tools (esbuild, SWC), monorepo tooling, CI/CD type checking |

### Exercises

**1. Comprehensive Test Suite:**

Write a full test suite for the task management API from Phase 3:

- Write unit tests for business logic functions using Vitest or Jest with TypeScript
- Create type-safe mock factories using generics: `createMock<UserService>()` that returns a typed mock
- Write integration tests that test API endpoints with a real (in-memory) database
- Implement test fixtures using the builder pattern for complex test data setup
- Achieve at least 80% code coverage and identify which uncovered lines are acceptable vs. risky
- Use `satisfies` and `as const` in test data to ensure test objects match expected types

**2. Tooling Configuration from Scratch:**

Set up a complete TypeScript development toolchain for a new project:

- Configure `tsconfig.json` with `strict: true`, `noUncheckedIndexedAccess: true`, and `exactOptionalPropertyTypes: true`
- Set up `typescript-eslint` with the `strict-type-checked` configuration
- Configure Prettier with format-on-save integration
- Set up a pre-commit hook (using Husky + lint-staged) that runs type checking, linting, and formatting
- Configure a CI pipeline (GitHub Actions) that runs `tsc --noEmit`, ESLint, tests, and build in parallel
- Add a `tsconfig.build.json` that extends the base config but excludes test files

**3. E2E Testing with Type Safety:**

Write end-to-end tests for the task management frontend:

- Set up Playwright with TypeScript configuration
- Create page object models with typed locators and actions
- Write E2E tests for the complete task workflow (create, read, update, delete)
- Implement visual regression testing for key pages
- Configure the E2E tests to run in CI against a test environment

### Knowledge Check

- [ ] Can you explain the testing pyramid and how it applies to a TypeScript full-stack project?
- [ ] Can you create a type-safe mock that preserves the interface contract?
- [ ] Can you explain the difference between `tsc --noEmit` and running ESLint for type checking?
- [ ] Can you configure `typescript-eslint` rules to catch common type safety mistakes?
- [ ] Can you describe a CI pipeline that enforces type safety at every stage?

---

## Phase 5: Production Readiness (Week 9–10)

### Learning Objectives

- Apply TypeScript best practices for maintainable, scalable codebases
- Identify and refactor common TypeScript anti-patterns
- Optimize TypeScript compilation performance
- Design library APIs with strong type contracts
- Migrate JavaScript codebases to TypeScript incrementally

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-BEST-PRACTICES](07-BEST-PRACTICES.md) | Strict configuration, naming conventions, module organization, API design, performance optimization |
| 2 | [08-ANTI-PATTERNS](08-ANTI-PATTERNS.md) | Overuse of `any`, type assertion abuse, incorrect generics, barrel file pitfalls, enum misuse |

### Exercises

**1. Anti-Pattern Refactoring:**

Take a TypeScript codebase (your own or an open-source project) and identify and fix anti-patterns:

- Search for all uses of `any` and replace each with a proper type (`unknown`, generics, or specific types)
- Find type assertions (`as`) and determine if each is justified or hiding a type error
- Identify functions with more than two generic parameters and simplify them
- Check for barrel files (`index.ts` re-exports) that may cause circular dependencies or bloated bundles
- Review `enum` usage and determine if string literal unions would be a better fit
- Document each finding with the anti-pattern name, location, fix applied, and reasoning

**2. Performance Optimization Audit:**

Audit the TypeScript compilation and runtime performance of a project:

- Measure `tsc` compilation time with `--diagnostics` and `--extendedDiagnostics`
- Identify the slowest files using `--generateTrace` and the TypeScript trace analyzer
- Check for deeply recursive conditional types that slow down the compiler
- Review bundle size using a bundle analyzer and identify TypeScript-specific bloat (e.g., enum runtime code, decorator metadata)
- Implement project references (`composite` and `references` in `tsconfig.json`) for faster incremental builds
- Document before-and-after metrics for each optimization

**3. JavaScript-to-TypeScript Migration Plan:**

Create a migration plan for converting a medium-sized JavaScript project (~50 files) to TypeScript:

- Enable `allowJs: true` and `checkJs: true` as the starting configuration
- Prioritize files for migration by dependency order (leaf modules first)
- Write type declarations (`.d.ts` files) for any untyped third-party libraries
- Convert five representative files from `.js` to `.ts`, demonstrating the migration workflow
- Set up a CI check that prevents new `.js` files from being added
- Create a progress tracker that shows the percentage of files migrated

### Knowledge Check

- [ ] Can you list five TypeScript anti-patterns and explain why each is harmful?
- [ ] Can you explain when `unknown` is preferable to `any` and how to narrow `unknown` safely?
- [ ] Can you describe three strategies for improving `tsc` compilation performance?
- [ ] Can you design a public library API with strong type contracts and backward compatibility?
- [ ] Can you create a migration plan for a JavaScript project that minimizes risk and disruption?

---

## Capstone Project

### Type-Safe Full-Stack Task Management Application

Build a production-ready, type-safe, full-stack TypeScript application that demonstrates every concept from Phases 1–5. The project is a **task management platform** with a REST API, a React frontend, and end-to-end type safety from database to UI.

#### Project Requirements

**Domain Model:**

- Tasks with title, description, status (todo, in-progress, done), priority (low, medium, high), assignee, due date, and tags
- Users with authentication (email/password) and roles (admin, member)
- Projects that group tasks with team membership and permissions
- Activity log tracking all changes to tasks (who changed what, when)

**Backend (Node.js + TypeScript):**

- Express or Fastify REST API with fully typed request/response handlers
- Zod schemas for all request validation with inferred TypeScript types
- Type-safe database access using Prisma or Drizzle ORM
- Authentication middleware with JWT and typed user context
- Role-based authorization using a type-safe permission model
- Structured error handling with discriminated union error types
- OpenAPI specification auto-generated from Zod schemas

**Frontend (React + TypeScript):**

- React application with strictly typed components, props, and state
- Custom hooks for data fetching, authentication, and form management
- Type-safe routing with typed route parameters
- Generic reusable components: `DataTable<T>`, `Form<T>`, `Select<T>`
- Discriminated union state management for async operations (`Idle | Loading | Success<T> | Error`)
- Shared type definitions imported from a common package

**Shared Types:**

- Domain models shared between client and server in a dedicated package or directory
- API contract types: `ApiResponse<T>`, `PaginatedResponse<T>`, `ApiError`
- Validate that changes to shared types produce compile-time errors on both sides

**Testing:**

- Unit tests for business logic with type-safe mocks and test factories
- Integration tests for API endpoints with an in-memory database
- Component tests for React components using React Testing Library
- E2E tests using Playwright with typed page object models
- At least 80% code coverage across the project

**Tooling & CI/CD:**

- `tsconfig.json` with `strict: true` and all recommended strict flags enabled
- `typescript-eslint` with the `strict-type-checked` configuration
- Pre-commit hooks running type checking, linting, and formatting
- GitHub Actions CI pipeline: type check → lint → test → build → E2E
- Zero `any` types in the codebase (enforce with ESLint rule `@typescript-eslint/no-explicit-any`)

**Production Patterns:**

- No uses of `any` — use `unknown`, generics, or specific types everywhere
- No unnecessary type assertions — use type guards and narrowing instead
- Branded types for IDs (`TaskId`, `UserId`, `ProjectId`) to prevent accidental misuse
- Discriminated unions for all domain states
- Proper error boundaries in React and structured error handling in the API

#### Deliverables

1. **Source code** — complete, compiling, and passing all tests with zero `any` types
2. **README** — project setup instructions, architecture overview, and design decisions
3. **Architecture diagram** — showing the type flow from database schema through API to frontend components
4. **Test report** — coverage summary and a brief explanation of the testing strategy
5. **Anti-pattern audit** — a checklist confirming none of the anti-patterns from [08-ANTI-PATTERNS](08-ANTI-PATTERNS.md) are present in the codebase

---

## Completion Criteria

You have completed this learning path when you can:

1. **Configure TypeScript** with strict settings and explain every compiler option you enable
2. **Design type-safe domain models** using discriminated unions, branded types, and generics
3. **Apply design patterns** (factory, builder, strategy, state machine) idiomatically in TypeScript
4. **Build full-stack applications** with end-to-end type safety from database to UI
5. **Share types across boundaries** between backend, frontend, and external consumers
6. **Write comprehensive tests** with type-safe mocks, fixtures, and E2E coverage
7. **Configure tooling** (ESLint, Prettier, CI/CD) to enforce type safety across a team
8. **Identify and refactor anti-patterns** including `any` abuse, type assertion overuse, and incorrect generics
9. **Optimize TypeScript performance** at both compile time and runtime
10. **Migrate JavaScript codebases** to TypeScript incrementally with a clear, repeatable strategy
