# TypeScript Learning Resources

A comprehensive guide to the TypeScript language, patterns, and ecosystem — from fundamentals and the type system to design patterns, frontend and backend frameworks, testing, tooling, best practices, and common pitfalls.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | TypeScript fundamentals, type system, compiler options | **Start here** |
| [01-TYPE-SYSTEM](01-TYPE-SYSTEM.md) | Generics, utility types, conditional types, mapped types | When working with advanced type-level programming |
| [02-PATTERNS](02-PATTERNS.md) | Design patterns in TypeScript, functional patterns | When architecting applications and libraries |
| [03-NODE-AND-BACKEND](03-NODE-AND-BACKEND.md) | Express, NestJS, server-side TypeScript | When building APIs and backend services |
| [04-FRONTEND-FRAMEWORKS](04-FRONTEND-FRAMEWORKS.md) | React, Angular, Vue with TypeScript | When building type-safe frontend applications |
| [05-TESTING](05-TESTING.md) | Jest, Vitest, type testing, mocking | When writing tests for TypeScript projects |
| [06-TOOLING](06-TOOLING.md) | ESLint, Prettier, tsconfig, build tools | When setting up or optimizing your dev environment |
| [07-BEST-PRACTICES](07-BEST-PRACTICES.md) | Strict mode, type safety, error handling | **Essential — every TypeScript developer** |
| [08-ANTI-PATTERNS](08-ANTI-PATTERNS.md) | Common TypeScript mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand TypeScript's relationship to JavaScript and why types matter
   - Learn basic types, interfaces, type aliases, and enums
   - Configure the TypeScript compiler with `tsconfig.json`

2. **Explore the Type System** ([01-TYPE-SYSTEM](01-TYPE-SYSTEM.md))
   - Work with generics to write reusable, type-safe code
   - Use utility types like `Partial`, `Pick`, `Omit`, and `Record`
   - Understand union types, intersection types, and type narrowing

3. **Learn Best Practices** ([07-BEST-PRACTICES](07-BEST-PRACTICES.md))
   - Enable strict mode from day one
   - Handle errors with discriminated unions and `Result` types
   - Write self-documenting code with expressive type annotations

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured curriculum with hands-on exercises
   - Progressive skill building from basics to production

### For Experienced Engineers

1. **Master Advanced Types** ([01-TYPE-SYSTEM](01-TYPE-SYSTEM.md))
   - Conditional types, template literal types, and type inference
   - Mapped types and recursive type transformations
   - Type-level programming and branded types

2. **Avoid Anti-Patterns** ([08-ANTI-PATTERNS](08-ANTI-PATTERNS.md))
   - Common `any` abuse and escape hatches that erode type safety
   - Overengineered generics and unnecessary type complexity
   - Misuse of type assertions and non-null assertions

3. **Build Backend Services** ([03-NODE-AND-BACKEND](03-NODE-AND-BACKEND.md))
   - Type-safe Express middleware and route handlers
   - NestJS dependency injection and decorator patterns
   - Database integration with Prisma, Drizzle, and TypeORM

4. **Optimize Your Tooling** ([06-TOOLING](06-TOOLING.md))
   - Fine-tune `tsconfig.json` for performance and strictness
   - Configure ESLint with `typescript-eslint` rules
   - Build with esbuild, swc, tsup, or Vite for fast compilation

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     TypeScript Ecosystem                         │
│                                                                  │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│   │   Fundamentals   │  │   Type System    │  │   Patterns   │  │
│   │  (Learn the      │  │  (Master the     │  │  (Design     │  │
│   │   language)      │  │   types)         │  │   well)      │  │
│   │                  │  │                  │  │              │  │
│   │  - Basic types   │  │  - Generics      │  │  - Builder   │  │
│   │  - Interfaces    │  │  - Utility types │  │  - Strategy  │  │
│   │  - Compiler opts │  │  - Conditionals  │  │  - Functional│  │
│   └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│   │  Node & Backend  │  │    Frontend      │  │   Testing    │  │
│   │  (Server-side    │  │  (Type-safe UI   │  │  (Verify     │  │
│   │   TypeScript)    │  │   frameworks)    │  │   your code) │  │
│   │                  │  │                  │  │              │  │
│   │  - Express       │  │  - React / Next  │  │  - Jest      │  │
│   │  - NestJS        │  │  - Angular       │  │  - Vitest    │  │
│   │  - Prisma / ORM  │  │  - Vue           │  │  - Type tests│  │
│   └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│   ┌──────────────────┐  ┌──────────────────┐                    │
│   │     Tooling      │  │  Best Practices  │                    │
│   │  (Dev experience)│  │  (Ship with      │                    │
│   │                  │  │   confidence)    │                    │
│   │  - ESLint        │  │  - Strict mode   │                    │
│   │  - Prettier      │  │  - Type safety   │                    │
│   │  - Build tools   │  │  - Error handling│                    │
│   └──────────────────┘  └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
TypeScript Fundamentals
───────────────────────
Type Inference       → TypeScript automatically infers types from context
Structural Typing    → Types are compatible based on shape, not name
Type Narrowing       → Refine types through control flow analysis
Declaration Files    → .d.ts files describe the shape of JavaScript libraries
Strict Mode          → Enable all strict compiler checks for maximum safety

Type System
───────────
Generics             → Parameterized types for reusable, type-safe abstractions
Utility Types        → Built-in type transformations (Partial, Pick, Omit, Record)
Conditional Types    → Types that depend on a condition (T extends U ? X : Y)
Mapped Types         → Transform properties of an existing type systematically
Template Literals    → String manipulation at the type level

Patterns & Architecture
───────────────────────
Discriminated Unions → Tagged unions for exhaustive pattern matching
Builder Pattern      → Type-safe fluent APIs with method chaining
Result Type          → Encode success/failure in the type system instead of exceptions
Dependency Injection → Decouple components with interfaces and IoC containers

Tooling & Ecosystem
───────────────────
tsconfig.json        → Compiler configuration for strictness, target, and module system
typescript-eslint    → Lint TypeScript code with type-aware rules
Bundlers             → esbuild, swc, tsup, Vite — fast TypeScript compilation
Monorepos            → Project references and composite builds for large codebases
```

## 📋 Topics Covered

- **Fundamentals** — Basic types, interfaces, type aliases, enums, type inference, compiler options
- **Type System** — Generics, utility types, conditional types, mapped types, template literal types
- **Patterns** — Design patterns, functional programming, discriminated unions, builder pattern
- **Node & Backend** — Express, NestJS, Fastify, database ORMs, API design, middleware
- **Frontend Frameworks** — React with TypeScript, Angular, Vue 3 Composition API, type-safe props
- **Testing** — Jest, Vitest, type testing with `tsd` and `expect-type`, mocking strategies
- **Tooling** — ESLint, Prettier, tsconfig, esbuild, swc, Vite, monorepo configuration
- **Best Practices** — Strict mode, error handling, type safety, code organization, documentation
- **Anti-Patterns** — `any` abuse, type assertion overuse, unnecessary complexity, unsafe patterns

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to TypeScript?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already writing TypeScript?** → Jump to [01-TYPE-SYSTEM.md](01-TYPE-SYSTEM.md) or [02-PATTERNS.md](02-PATTERNS.md)

**Going to production?** → Review [07-BEST-PRACTICES.md](07-BEST-PRACTICES.md) and [08-ANTI-PATTERNS.md](08-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — progressive exercises from basics to production
