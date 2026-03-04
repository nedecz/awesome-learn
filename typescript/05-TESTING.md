# TypeScript Testing

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Testing Fundamentals with TypeScript](#testing-fundamentals-with-typescript)
   - [Why TypeScript Changes Testing](#why-typescript-changes-testing)
   - [Test File Organization](#test-file-organization)
3. [Jest with TypeScript](#jest-with-typescript)
   - [Configuration and ts-jest](#configuration-and-ts-jest)
   - [Writing Typed Tests](#writing-typed-tests)
   - [Custom Matchers with Type Safety](#custom-matchers-with-type-safety)
4. [Vitest with TypeScript](#vitest-with-typescript)
   - [Configuration and Native TypeScript Support](#configuration-and-native-typescript-support)
   - [Vitest Features for TypeScript](#vitest-features-for-typescript)
5. [Type Testing](#type-testing)
   - [Testing Types with expect-type](#testing-types-with-expect-type)
   - [Testing Types with tsd](#testing-types-with-tsd)
   - [Compile-Time Assertion Patterns](#compile-time-assertion-patterns)
6. [Mocking Strategies](#mocking-strategies)
   - [Typed Mocks](#typed-mocks)
   - [Module Mocking](#module-mocking)
   - [Dependency Injection for Testability](#dependency-injection-for-testability)
7. [Testing Async Code](#testing-async-code)
   - [Promises and Async/Await](#promises-and-asyncawait)
   - [Timers and Intervals](#timers-and-intervals)
   - [Testing Event Emitters](#testing-event-emitters)
8. [Integration Testing with TypeScript](#integration-testing-with-typescript)
   - [API Integration Tests](#api-integration-tests)
   - [Database Integration Tests](#database-integration-tests)
9. [End-to-End Testing](#end-to-end-testing)
   - [Playwright with TypeScript](#playwright-with-typescript)
   - [Cypress with TypeScript](#cypress-with-typescript)
10. [Test Patterns and Utilities](#test-patterns-and-utilities)
    - [Test Builders and Factories](#test-builders-and-factories)
    - [Fixtures and Helpers](#fixtures-and-helpers)
11. [Code Coverage with TypeScript](#code-coverage-with-typescript)
    - [Coverage Configuration](#coverage-configuration)
    - [Coverage Thresholds and Reporting](#coverage-thresholds-and-reporting)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Testing TypeScript code requires more than verifying runtime behavior — it means ensuring that your types, interfaces, and generics work correctly at compile time as well. TypeScript's static type system eliminates many categories of bugs before tests even run, but it also introduces new testing concerns: verifying that your public API types are correct, mocking dependencies with full type safety, and configuring test runners to handle `.ts` files natively. This guide covers the complete testing landscape for TypeScript projects, from unit test configuration through end-to-end testing with type-safe page objects.

### Target Audience

- **TypeScript developers setting up testing infrastructure** — Engineers who need to configure Jest or Vitest for TypeScript projects and want type-safe test patterns from the start
- **Library authors verifying public APIs** — Developers who ship type definitions and need to ensure their exported types work correctly across versions
- **Teams migrating JavaScript tests to TypeScript** — Teams converting existing test suites to TypeScript and looking for patterns that leverage the type system effectively

### Scope

- Configuring Jest and Vitest for TypeScript projects
- Writing type-safe tests with full IntelliSense and compile-time checking
- Testing that types themselves are correct using expect-type and tsd
- Mocking strategies that preserve type safety including dependency injection
- Integration and end-to-end testing with Playwright and Cypress
- Test utility patterns including builders, factories, and fixtures
- Code coverage configuration and threshold enforcement

---

## Testing Fundamentals with TypeScript

### Why TypeScript Changes Testing

TypeScript's compiler catches many errors that would otherwise require unit tests in JavaScript. You no longer need tests to verify that a function receives the correct argument types or that an object has the expected shape. This shifts the testing focus toward behavior verification, edge cases, and integration boundaries.

```typescript
// In JavaScript, you might write a test to verify argument types:
// test("rejects non-string input", () => { expect(() => greet(42)).toThrow(); })

// In TypeScript, the compiler handles this — greet(42) is a compile error
function greet(name: string): string {
  return `Hello, ${name}!`;
}

// Tests can focus on behavior instead
function formatCurrency(amount: number, currency: string = "USD"): string {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency,
  }).format(amount);
}

// No need to test wrong argument types — focus on edge cases
// test: formatCurrency(0) → "$0.00"
// test: formatCurrency(-50.5) → "-$50.50"
// test: formatCurrency(1234.56, "EUR") → "€1,234.56"
```

### Test File Organization

A consistent file structure makes tests discoverable and keeps them close to the code they verify.

```typescript
// Recommended project structure
// src/
//   services/
//     user.service.ts
//     user.service.test.ts       ← unit tests co-located with source
//   utils/
//     validation.ts
//     validation.test.ts
// tests/
//   integration/
//     user-api.integration.test.ts  ← integration tests in separate directory
//   e2e/
//     checkout.e2e.test.ts          ← end-to-end tests
//   fixtures/
//     users.ts                      ← shared test data
//   helpers/
//     test-server.ts                ← shared test utilities

// src/services/user.service.ts
interface User {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user" | "viewer";
  createdAt: Date;
}

interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<User>;
  delete(id: string): Promise<void>;
}

interface EmailService {
  sendWelcome(to: string, name: string): Promise<void>;
}

class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailService: EmailService
  ) {}

  async register(name: string, email: string): Promise<User> {
    const existing = await this.userRepo.findByEmail(email);
    if (existing) {
      throw new Error(`User with email ${email} already exists`);
    }

    const user: User = {
      id: crypto.randomUUID(),
      name,
      email,
      role: "user",
      createdAt: new Date(),
    };

    const saved = await this.userRepo.save(user);
    await this.emailService.sendWelcome(email, name);
    return saved;
  }

  async getById(id: string): Promise<User> {
    const user = await this.userRepo.findById(id);
    if (!user) {
      throw new Error(`User not found: ${id}`);
    }
    return user;
  }
}

export { UserService, type User, type UserRepository, type EmailService };
```

---

## Jest with TypeScript

### Configuration and ts-jest

Jest does not natively understand TypeScript, so it requires a transform layer. The `ts-jest` package compiles TypeScript files on the fly during test runs, providing full type checking.

```typescript
// jest.config.ts
import type { Config } from "jest";

const config: Config = {
  preset: "ts-jest",
  testEnvironment: "node",
  roots: ["<rootDir>/src", "<rootDir>/tests"],
  testMatch: [
    "**/*.test.ts",
    "**/*.test.tsx",
    "**/*.spec.ts",
  ],
  moduleNameMapper: {
    "^@/(.*)$": "<rootDir>/src/$1",
  },
  collectCoverageFrom: [
    "src/**/*.ts",
    "!src/**/*.d.ts",
    "!src/**/index.ts",
  ],
  coverageThresholds: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  // Use ts-jest for TypeScript files
  transform: {
    "^.+\\.tsx?$": [
      "ts-jest",
      {
        tsconfig: "tsconfig.json",
        diagnostics: {
          // Report type errors as warnings during test runs
          warnOnly: true,
        },
      },
    ],
  },
};

export default config;
```

### Writing Typed Tests

With `ts-jest` configured, every test file benefits from full type checking. The compiler catches mismatched assertions and incorrect mock return types before the test even runs.

```typescript
// src/services/user.service.test.ts
import { UserService, type User, type UserRepository, type EmailService } from "./user.service";

// Create typed mocks that satisfy the interface contracts
function createMockUserRepo(): jest.Mocked<UserRepository> {
  return {
    findById: jest.fn(),
    findByEmail: jest.fn(),
    save: jest.fn(),
    delete: jest.fn(),
  };
}

function createMockEmailService(): jest.Mocked<EmailService> {
  return {
    sendWelcome: jest.fn(),
  };
}

describe("UserService", () => {
  let userRepo: jest.Mocked<UserRepository>;
  let emailService: jest.Mocked<EmailService>;
  let service: UserService;

  beforeEach(() => {
    userRepo = createMockUserRepo();
    emailService = createMockEmailService();
    service = new UserService(userRepo, emailService);
  });

  describe("register", () => {
    it("creates a new user and sends a welcome email", async () => {
      const expectedUser: User = {
        id: "test-id",
        name: "Alice",
        email: "alice@example.com",
        role: "user",
        createdAt: new Date("2025-01-01"),
      };

      userRepo.findByEmail.mockResolvedValue(null);
      userRepo.save.mockResolvedValue(expectedUser);
      emailService.sendWelcome.mockResolvedValue(undefined);

      const result = await service.register("Alice", "alice@example.com");

      expect(result).toEqual(expectedUser);
      expect(userRepo.findByEmail).toHaveBeenCalledWith("alice@example.com");
      expect(userRepo.save).toHaveBeenCalledWith(
        expect.objectContaining({
          name: "Alice",
          email: "alice@example.com",
          role: "user",
        })
      );
      expect(emailService.sendWelcome).toHaveBeenCalledWith("alice@example.com", "Alice");
    });

    it("throws when the email is already registered", async () => {
      const existingUser: User = {
        id: "existing-id",
        name: "Bob",
        email: "bob@example.com",
        role: "user",
        createdAt: new Date(),
      };
      userRepo.findByEmail.mockResolvedValue(existingUser);

      await expect(
        service.register("Bob", "bob@example.com")
      ).rejects.toThrow("User with email bob@example.com already exists");

      expect(userRepo.save).not.toHaveBeenCalled();
      expect(emailService.sendWelcome).not.toHaveBeenCalled();
    });
  });

  describe("getById", () => {
    it("returns the user when found", async () => {
      const user: User = {
        id: "user-1",
        name: "Carol",
        email: "carol@example.com",
        role: "admin",
        createdAt: new Date(),
      };
      userRepo.findById.mockResolvedValue(user);

      const result = await service.getById("user-1");

      expect(result).toEqual(user);
      expect(result.role).toBe("admin");
    });

    it("throws when the user is not found", async () => {
      userRepo.findById.mockResolvedValue(null);

      await expect(service.getById("missing")).rejects.toThrow("User not found: missing");
    });
  });
});
```

### Custom Matchers with Type Safety

Jest allows custom matchers to be added via `expect.extend`. TypeScript requires a module augmentation to make these matchers type-safe.

```typescript
// tests/helpers/custom-matchers.ts
interface CustomMatchers<R = unknown> {
  toBeWithinRange(floor: number, ceiling: number): R;
  toBeValidEmail(): R;
}

declare module "expect" {
  interface AsymmetricMatchers extends CustomMatchers {}
  interface Matchers<R> extends CustomMatchers<R> {}
}

expect.extend({
  toBeWithinRange(received: number, floor: number, ceiling: number) {
    const pass = received >= floor && received <= ceiling;
    return {
      pass,
      message: () =>
        `expected ${received} ${pass ? "not " : ""}to be within range ${floor} - ${ceiling}`,
    };
  },

  toBeValidEmail(received: string) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    const pass = emailRegex.test(received);
    return {
      pass,
      message: () =>
        `expected "${received}" ${pass ? "not " : ""}to be a valid email address`,
    };
  },
});

// Usage in tests — fully typed with IntelliSense
// expect(score).toBeWithinRange(0, 100);
// expect(user.email).toBeValidEmail();
```

---

## Vitest with TypeScript

### Configuration and Native TypeScript Support

Vitest understands TypeScript natively through Vite's built-in esbuild transform. No separate `ts-jest` plugin is needed, and test files are processed using the same pipeline as the application code.

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "src"),
    },
  },
  test: {
    globals: true,
    environment: "node",
    include: ["src/**/*.test.ts", "tests/**/*.test.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "html", "lcov"],
      include: ["src/**/*.ts"],
      exclude: ["src/**/*.d.ts", "src/**/index.ts", "src/**/*.test.ts"],
      thresholds: {
        branches: 80,
        functions: 80,
        lines: 80,
        statements: 80,
      },
    },
    // Type checking during test runs
    typecheck: {
      enabled: true,
      tsconfig: "./tsconfig.json",
    },
  },
});
```

### Vitest Features for TypeScript

Vitest provides several TypeScript-specific utilities that streamline test authoring, including built-in type assertions and a Jest-compatible mock API with improved type inference.

```typescript
// src/utils/result.ts
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

function parsePositiveInt(input: string): Result<number, string> {
  const num = parseInt(input, 10);
  if (isNaN(num)) return err(`"${input}" is not a number`);
  if (num <= 0) return err(`${num} is not positive`);
  return ok(num);
}

export { type Result, ok, err, parsePositiveInt };

// src/utils/result.test.ts
import { describe, it, expect, vi } from "vitest";
import { ok, err, parsePositiveInt, type Result } from "./result";

describe("parsePositiveInt", () => {
  it("parses valid positive integers", () => {
    const result = parsePositiveInt("42");
    expect(result).toEqual({ ok: true, value: 42 });

    // Vitest narrows the type after the check
    if (result.ok) {
      expect(result.value).toBe(42);
    }
  });

  it("returns an error for non-numeric input", () => {
    const result = parsePositiveInt("abc");
    expect(result).toEqual({ ok: false, error: '"abc" is not a number' });
  });

  it("returns an error for negative numbers", () => {
    const result = parsePositiveInt("-5");
    expect(result).toEqual({ ok: false, error: "-5 is not positive" });
  });

  it("returns an error for zero", () => {
    const result = parsePositiveInt("0");
    expect(result.ok).toBe(false);
  });
});

// Vitest in-source testing — tests live inside the source file
// Enabled via `defineConfig({ define: { "import.meta.vitest": "undefined" } })`
// if (import.meta.vitest) {
//   const { describe, it, expect } = import.meta.vitest;
//   describe("ok", () => {
//     it("wraps a value", () => {
//       expect(ok(42)).toEqual({ ok: true, value: 42 });
//     });
//   });
// }
```

---

## Type Testing

Type tests verify that your exported types behave correctly — they catch regressions in your public API's type definitions without running any code at runtime.

### Testing Types with expect-type

The `expect-type` library provides a fluent API for asserting type relationships. It works with both Vitest (which bundles it) and Jest.

```typescript
// src/types.test-d.ts (Vitest convention for type-only test files)
import { expectTypeOf } from "vitest";
import type { Result } from "./utils/result";
import { ok, err } from "./utils/result";

describe("Result type", () => {
  it("ok() returns Result with the correct value type", () => {
    const result = ok(42);
    expectTypeOf(result).toMatchTypeOf<Result<number, never>>();
  });

  it("err() returns Result with the correct error type", () => {
    const result = err("something failed");
    expectTypeOf(result).toMatchTypeOf<Result<never, string>>();
  });

  it("narrows correctly when ok is true", () => {
    const result: Result<string, Error> = ok("hello");
    if (result.ok) {
      expectTypeOf(result.value).toBeString();
    }
  });

  it("narrows correctly when ok is false", () => {
    const result: Result<string, Error> = err(new Error("fail"));
    if (!result.ok) {
      expectTypeOf(result.error).toMatchTypeOf<Error>();
    }
  });
});

// Testing generic utility types
type Prettify<T> = { [K in keyof T]: T[K] } & {};

interface UserInput {
  name: string;
  email: string;
}

interface UserDefaults {
  role: "user";
  active: true;
}

it("Prettify flattens intersection types", () => {
  type Combined = Prettify<UserInput & UserDefaults>;

  expectTypeOf<Combined>().toEqualTypeOf<{
    name: string;
    email: string;
    role: "user";
    active: true;
  }>();
});
```

### Testing Types with tsd

The `tsd` package is a standalone type testing tool commonly used by library authors. It reads `*.test-d.ts` files and reports type errors.

```typescript
// index.test-d.ts — used with the tsd package
import { expectType, expectError, expectNotType, expectAssignable } from "tsd";
import { createStore, type Store } from "./store";

// Verify that createStore returns the correct type
const store = createStore({ count: 0, name: "test" });
expectType<Store<{ count: number; name: string }>>(store);

// Verify that get returns the correct value type for a given key
expectType<number>(store.get("count"));
expectType<string>(store.get("name"));

// Verify that set enforces the correct value type
store.set("count", 10);       // should compile
expectError(store.set("count", "ten")); // should fail — wrong value type

// Verify that invalid keys are rejected
expectError(store.get("nonexistent"));
expectError(store.set("nonexistent", 42));

// Verify assignability for broader types
expectAssignable<{ get(key: string): unknown }>(store);

// Verify that the type is NOT something unexpected
expectNotType<Store<{ count: string }>>(store);
```

### Compile-Time Assertion Patterns

For projects that do not want an extra dependency, compile-time assertions can be written using TypeScript's own type system.

```typescript
// tests/type-assertions.ts — this file is only compiled, never executed

// A helper type that produces a compile error if the condition is false
type Assert<T extends true> = T;
type IsEqual<A, B> = [A] extends [B] ? ([B] extends [A] ? true : false) : false;

// Verify that a mapped type produces the expected output
type EventMap = {
  click: { x: number; y: number };
  keypress: { key: string };
};

type EventNames = keyof EventMap;
type _TestEventNames = Assert<IsEqual<EventNames, "click" | "keypress">>;

// Verify that a conditional type distributes correctly
type NonNullish<T> = T extends null | undefined ? never : T;
type _TestNonNullish = Assert<IsEqual<NonNullish<string | null | undefined>, string>>;

// Verify that a generic function's return type is inferred correctly
declare function identity<T>(value: T): T;
type _TestIdentity = Assert<IsEqual<ReturnType<typeof identity<number>>, number>>;
```

---

## Mocking Strategies

### Typed Mocks

Creating mocks that preserve the original type's contract prevents tests from silently becoming invalid when the source interface changes.

```typescript
// Vitest typed mocking
import { describe, it, expect, vi, type MockedFunction } from "vitest";

interface PaymentGateway {
  charge(amount: number, currency: string): Promise<{ transactionId: string }>;
  refund(transactionId: string): Promise<{ success: boolean }>;
  getBalance(): Promise<number>;
}

// Create a fully typed mock object using vi.fn()
function createMockPaymentGateway(): {
  [K in keyof PaymentGateway]: MockedFunction<PaymentGateway[K]>;
} {
  return {
    charge: vi.fn(),
    refund: vi.fn(),
    getBalance: vi.fn(),
  };
}

class OrderService {
  constructor(private gateway: PaymentGateway) {}

  async checkout(amount: number): Promise<string> {
    if (amount <= 0) throw new Error("Amount must be positive");
    const result = await this.gateway.charge(amount, "USD");
    return result.transactionId;
  }

  async cancelOrder(transactionId: string): Promise<boolean> {
    const result = await this.gateway.refund(transactionId);
    return result.success;
  }
}

describe("OrderService", () => {
  it("charges the payment gateway and returns the transaction ID", async () => {
    const gateway = createMockPaymentGateway();
    gateway.charge.mockResolvedValue({ transactionId: "txn-123" });

    const service = new OrderService(gateway);
    const txnId = await service.checkout(99.99);

    expect(txnId).toBe("txn-123");
    expect(gateway.charge).toHaveBeenCalledWith(99.99, "USD");
  });

  it("rejects zero or negative amounts", async () => {
    const gateway = createMockPaymentGateway();
    const service = new OrderService(gateway);

    await expect(service.checkout(0)).rejects.toThrow("Amount must be positive");
    expect(gateway.charge).not.toHaveBeenCalled();
  });
});
```

### Module Mocking

Both Jest and Vitest allow mocking entire modules. TypeScript ensures the mock shape matches the original module's exports.

```typescript
// src/services/notification.service.ts
import { sendEmail } from "../adapters/email-adapter";
import { sendSms } from "../adapters/sms-adapter";

interface NotificationPayload {
  userId: string;
  message: string;
  channel: "email" | "sms" | "both";
}

async function notify(payload: NotificationPayload): Promise<void> {
  if (payload.channel === "email" || payload.channel === "both") {
    await sendEmail(payload.userId, payload.message);
  }
  if (payload.channel === "sms" || payload.channel === "both") {
    await sendSms(payload.userId, payload.message);
  }
}

export { notify, type NotificationPayload };

// src/services/notification.service.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { notify } from "./notification.service";

// Mock entire modules — Vitest infers types from the actual module
vi.mock("../adapters/email-adapter", () => ({
  sendEmail: vi.fn().mockResolvedValue(undefined),
}));

vi.mock("../adapters/sms-adapter", () => ({
  sendSms: vi.fn().mockResolvedValue(undefined),
}));

// Import the mocked versions with correct types
import { sendEmail } from "../adapters/email-adapter";
import { sendSms } from "../adapters/sms-adapter";

const mockedSendEmail = vi.mocked(sendEmail);
const mockedSendSms = vi.mocked(sendSms);

describe("notify", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("sends only an email when channel is email", async () => {
    await notify({ userId: "u1", message: "Hello", channel: "email" });

    expect(mockedSendEmail).toHaveBeenCalledWith("u1", "Hello");
    expect(mockedSendSms).not.toHaveBeenCalled();
  });

  it("sends only an SMS when channel is sms", async () => {
    await notify({ userId: "u1", message: "Hello", channel: "sms" });

    expect(mockedSendEmail).not.toHaveBeenCalled();
    expect(mockedSendSms).toHaveBeenCalledWith("u1", "Hello");
  });

  it("sends both when channel is both", async () => {
    await notify({ userId: "u1", message: "Hello", channel: "both" });

    expect(mockedSendEmail).toHaveBeenCalledWith("u1", "Hello");
    expect(mockedSendSms).toHaveBeenCalledWith("u1", "Hello");
  });
});
```

### Dependency Injection for Testability

Designing modules around injected dependencies makes them inherently testable without any mocking framework magic. Interfaces define the contract, and tests supply lightweight implementations.

```typescript
// src/services/invoice.service.ts
interface InvoiceRepository {
  save(invoice: Invoice): Promise<Invoice>;
  findByCustomer(customerId: string): Promise<Invoice[]>;
}

interface PdfGenerator {
  generate(invoice: Invoice): Promise<Buffer>;
}

interface Logger {
  info(message: string, meta?: Record<string, unknown>): void;
  error(message: string, meta?: Record<string, unknown>): void;
}

interface Invoice {
  id: string;
  customerId: string;
  items: { description: string; amount: number }[];
  total: number;
  createdAt: Date;
}

class InvoiceService {
  constructor(
    private repo: InvoiceRepository,
    private pdfGenerator: PdfGenerator,
    private logger: Logger
  ) {}

  async createInvoice(
    customerId: string,
    items: { description: string; amount: number }[]
  ): Promise<Invoice> {
    const total = items.reduce((sum, item) => sum + item.amount, 0);
    const invoice: Invoice = {
      id: crypto.randomUUID(),
      customerId,
      items,
      total,
      createdAt: new Date(),
    };

    const saved = await this.repo.save(invoice);
    this.logger.info("Invoice created", { invoiceId: saved.id, customerId });
    return saved;
  }

  async generatePdf(invoiceId: string, customerId: string): Promise<Buffer> {
    const invoices = await this.repo.findByCustomer(customerId);
    const invoice = invoices.find((inv) => inv.id === invoiceId);
    if (!invoice) throw new Error(`Invoice ${invoiceId} not found`);

    return this.pdfGenerator.generate(invoice);
  }
}

// tests/unit/invoice.service.test.ts
import { describe, it, expect } from "vitest";

describe("InvoiceService", () => {
  // Simple fakes that satisfy the interfaces — no mocking library needed
  function setup() {
    const savedInvoices: Invoice[] = [];
    const logMessages: string[] = [];

    const repo: InvoiceRepository = {
      async save(invoice) {
        savedInvoices.push(invoice);
        return invoice;
      },
      async findByCustomer(customerId) {
        return savedInvoices.filter((inv) => inv.customerId === customerId);
      },
    };

    const pdfGenerator: PdfGenerator = {
      async generate(invoice) {
        return Buffer.from(`PDF for invoice ${invoice.id}`);
      },
    };

    const logger: Logger = {
      info(message) { logMessages.push(message); },
      error(message) { logMessages.push(message); },
    };

    const service = new InvoiceService(repo, pdfGenerator, logger);
    return { service, savedInvoices, logMessages };
  }

  it("creates an invoice with the correct total", async () => {
    const { service, savedInvoices } = setup();

    const invoice = await service.createInvoice("cust-1", [
      { description: "Widget", amount: 25 },
      { description: "Gadget", amount: 75 },
    ]);

    expect(invoice.total).toBe(100);
    expect(invoice.customerId).toBe("cust-1");
    expect(savedInvoices).toHaveLength(1);
  });

  it("logs a message when an invoice is created", async () => {
    const { service, logMessages } = setup();

    await service.createInvoice("cust-2", [{ description: "Item", amount: 50 }]);

    expect(logMessages).toContain("Invoice created");
  });

  it("generates a PDF for an existing invoice", async () => {
    const { service } = setup();

    const invoice = await service.createInvoice("cust-3", [
      { description: "Service", amount: 200 },
    ]);

    const pdf = await service.generatePdf(invoice.id, "cust-3");
    expect(pdf.toString()).toContain(`PDF for invoice ${invoice.id}`);
  });
});
```

---

## Testing Async Code

### Promises and Async/Await

TypeScript ensures that async test assertions are properly awaited, preventing false-passing tests caused by unhandled promises.

```typescript
import { describe, it, expect, vi } from "vitest";

interface CacheService {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T, ttlSeconds: number): Promise<void>;
  delete(key: string): Promise<boolean>;
}

class CachedUserLoader {
  constructor(
    private cache: CacheService,
    private fetchUser: (id: string) => Promise<User>
  ) {}

  async load(userId: string): Promise<User> {
    const cached = await this.cache.get<User>(`user:${userId}`);
    if (cached) return cached;

    const user = await this.fetchUser(userId);
    await this.cache.set(`user:${userId}`, user, 300);
    return user;
  }
}

describe("CachedUserLoader", () => {
  function createTestUser(id: string): User {
    return {
      id,
      name: "Test User",
      email: `${id}@example.com`,
      role: "user",
      createdAt: new Date(),
    };
  }

  it("returns cached data when available", async () => {
    const cachedUser = createTestUser("u1");
    const cache: CacheService = {
      get: vi.fn().mockResolvedValue(cachedUser),
      set: vi.fn().mockResolvedValue(undefined),
      delete: vi.fn().mockResolvedValue(true),
    };
    const fetchUser = vi.fn();

    const loader = new CachedUserLoader(cache, fetchUser);
    const result = await loader.load("u1");

    expect(result).toEqual(cachedUser);
    expect(fetchUser).not.toHaveBeenCalled();
  });

  it("fetches and caches when cache misses", async () => {
    const freshUser = createTestUser("u2");
    const cache: CacheService = {
      get: vi.fn().mockResolvedValue(null),
      set: vi.fn().mockResolvedValue(undefined),
      delete: vi.fn().mockResolvedValue(true),
    };
    const fetchUser = vi.fn().mockResolvedValue(freshUser);

    const loader = new CachedUserLoader(cache, fetchUser);
    const result = await loader.load("u2");

    expect(result).toEqual(freshUser);
    expect(fetchUser).toHaveBeenCalledWith("u2");
    expect(cache.set).toHaveBeenCalledWith("user:u2", freshUser, 300);
  });
});
```

### Timers and Intervals

Fake timers allow tests to control time-dependent code without waiting for real delays.

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";

class RetryWithBackoff {
  constructor(
    private maxRetries: number = 3,
    private baseDelayMs: number = 1000
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    let lastError: Error | undefined;

    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error as Error;
        if (attempt < this.maxRetries) {
          const delay = this.baseDelayMs * Math.pow(2, attempt);
          await new Promise((resolve) => setTimeout(resolve, delay));
        }
      }
    }

    throw lastError;
  }
}

describe("RetryWithBackoff", () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it("retries with exponential backoff delays", async () => {
    const fn = vi.fn()
      .mockRejectedValueOnce(new Error("fail 1"))
      .mockRejectedValueOnce(new Error("fail 2"))
      .mockResolvedValue("success");

    const retry = new RetryWithBackoff(3, 1000);
    const promise = retry.execute(fn);

    // First retry after 1000ms (1000 * 2^0)
    await vi.advanceTimersByTimeAsync(1000);
    // Second retry after 2000ms (1000 * 2^1)
    await vi.advanceTimersByTimeAsync(2000);

    const result = await promise;
    expect(result).toBe("success");
    expect(fn).toHaveBeenCalledTimes(3);
  });

  it("throws after exhausting all retries", async () => {
    const fn = vi.fn().mockRejectedValue(new Error("persistent failure"));

    const retry = new RetryWithBackoff(2, 100);
    const promise = retry.execute(fn);

    await vi.advanceTimersByTimeAsync(100);  // first retry delay
    await vi.advanceTimersByTimeAsync(200);  // second retry delay

    await expect(promise).rejects.toThrow("persistent failure");
    expect(fn).toHaveBeenCalledTimes(3); // initial + 2 retries
  });
});
```

### Testing Event Emitters

Typed event emitters require tests that verify both the event payload types and the emission behavior.

```typescript
import { EventEmitter } from "node:events";
import { describe, it, expect, vi } from "vitest";

interface AppEvents {
  "user:login": { userId: string; timestamp: Date };
  "user:logout": { userId: string };
  "error": { code: string; message: string };
}

class TypedEventEmitter<Events extends Record<string, unknown>> {
  private emitter = new EventEmitter();

  on<K extends keyof Events & string>(
    event: K,
    listener: (payload: Events[K]) => void
  ): void {
    this.emitter.on(event, listener);
  }

  emit<K extends keyof Events & string>(event: K, payload: Events[K]): void {
    this.emitter.emit(event, payload);
  }

  off<K extends keyof Events & string>(
    event: K,
    listener: (payload: Events[K]) => void
  ): void {
    this.emitter.off(event, listener);
  }
}

describe("TypedEventEmitter", () => {
  it("emits events with the correct payload type", () => {
    const bus = new TypedEventEmitter<AppEvents>();
    const handler = vi.fn();

    bus.on("user:login", handler);
    bus.emit("user:login", { userId: "u1", timestamp: new Date() });

    expect(handler).toHaveBeenCalledWith(
      expect.objectContaining({ userId: "u1" })
    );
  });

  it("supports multiple listeners for the same event", () => {
    const bus = new TypedEventEmitter<AppEvents>();
    const handler1 = vi.fn();
    const handler2 = vi.fn();

    bus.on("error", handler1);
    bus.on("error", handler2);
    bus.emit("error", { code: "E001", message: "Something broke" });

    expect(handler1).toHaveBeenCalledTimes(1);
    expect(handler2).toHaveBeenCalledTimes(1);
  });

  it("removes listeners with off()", () => {
    const bus = new TypedEventEmitter<AppEvents>();
    const handler = vi.fn();

    bus.on("user:logout", handler);
    bus.off("user:logout", handler);
    bus.emit("user:logout", { userId: "u1" });

    expect(handler).not.toHaveBeenCalled();
  });
});
```

---

## Integration Testing with TypeScript

### API Integration Tests

Integration tests verify that HTTP endpoints, middleware, and database layers work together correctly. TypeScript ensures the request and response shapes match your API contract.

```typescript
// tests/integration/user-api.integration.test.ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";

interface TestServer {
  url: string;
  close(): Promise<void>;
}

interface ApiUser {
  id: string;
  name: string;
  email: string;
  role: string;
}

interface ApiErrorResponse {
  error: string;
  statusCode: number;
}

// Typed helper for making requests to the test server
async function apiRequest<T>(
  server: TestServer,
  method: string,
  path: string,
  body?: unknown
): Promise<{ status: number; data: T }> {
  const response = await fetch(`${server.url}${path}`, {
    method,
    headers: { "Content-Type": "application/json" },
    body: body ? JSON.stringify(body) : undefined,
  });
  const data: T = await response.json();
  return { status: response.status, data };
}

describe("User API", () => {
  let server: TestServer;

  beforeAll(async () => {
    // Assume createTestServer starts an Express/Fastify server on a random port
    server = await createTestServer();
  });

  afterAll(async () => {
    await server.close();
  });

  it("POST /users creates a user and returns 201", async () => {
    const { status, data } = await apiRequest<ApiUser>(server, "POST", "/users", {
      name: "Integration Test User",
      email: "integration@example.com",
    });

    expect(status).toBe(201);
    expect(data.name).toBe("Integration Test User");
    expect(data.email).toBe("integration@example.com");
    expect(data.id).toBeDefined();
  });

  it("POST /users returns 400 for invalid input", async () => {
    const { status, data } = await apiRequest<ApiErrorResponse>(
      server, "POST", "/users", { name: "" }
    );

    expect(status).toBe(400);
    expect(data.error).toBeDefined();
  });

  it("GET /users/:id returns the created user", async () => {
    const createResult = await apiRequest<ApiUser>(server, "POST", "/users", {
      name: "Fetch Test",
      email: "fetch@example.com",
    });

    const { status, data } = await apiRequest<ApiUser>(
      server, "GET", `/users/${createResult.data.id}`
    );

    expect(status).toBe(200);
    expect(data.id).toBe(createResult.data.id);
    expect(data.name).toBe("Fetch Test");
  });
});

// Placeholder for the test server factory used in the examples above
declare function createTestServer(): Promise<TestServer>;
```

### Database Integration Tests

Database integration tests use real database connections to verify queries, migrations, and data integrity.

```typescript
// tests/integration/user-repo.integration.test.ts
import { describe, it, expect, beforeEach } from "vitest";

interface DatabaseConnection {
  query<T>(sql: string, params?: unknown[]): Promise<T[]>;
  execute(sql: string, params?: unknown[]): Promise<{ affectedRows: number }>;
}

class PostgresUserRepository {
  constructor(private db: DatabaseConnection) {}

  async findById(id: string): Promise<User | null> {
    const rows = await this.db.query<User>(
      "SELECT id, name, email, role, created_at as \"createdAt\" FROM users WHERE id = $1",
      [id]
    );
    return rows[0] ?? null;
  }

  async save(user: Omit<User, "createdAt">): Promise<User> {
    const rows = await this.db.query<User>(
      `INSERT INTO users (id, name, email, role)
       VALUES ($1, $2, $3, $4)
       RETURNING id, name, email, role, created_at as "createdAt"`,
      [user.id, user.name, user.email, user.role]
    );
    return rows[0];
  }
}

describe("PostgresUserRepository", () => {
  let db: DatabaseConnection;
  let repo: PostgresUserRepository;

  beforeEach(async () => {
    // Assume getTestDatabase returns a connection to a test database
    db = await getTestDatabase();
    await db.execute("DELETE FROM users");
    repo = new PostgresUserRepository(db);
  });

  it("saves and retrieves a user by ID", async () => {
    const saved = await repo.save({
      id: "db-test-1",
      name: "DB Test User",
      email: "dbtest@example.com",
      role: "user",
    });

    const found = await repo.findById("db-test-1");

    expect(found).not.toBeNull();
    expect(found!.name).toBe("DB Test User");
    expect(found!.email).toBe("dbtest@example.com");
  });

  it("returns null for a non-existent user", async () => {
    const found = await repo.findById("non-existent");
    expect(found).toBeNull();
  });
});

// Placeholder for the database connection factory used in the examples above
declare function getTestDatabase(): Promise<DatabaseConnection>;
```

---

## End-to-End Testing

### Playwright with TypeScript

Playwright provides first-class TypeScript support with typed page objects, locators, and assertions. Its code generator can scaffold typed tests from browser interactions.

```typescript
// tests/e2e/login.e2e.test.ts
import { test, expect, type Page } from "@playwright/test";

// Type-safe page object
class LoginPage {
  constructor(private page: Page) {}

  async navigate(): Promise<void> {
    await this.page.goto("/login");
  }

  async fillCredentials(email: string, password: string): Promise<void> {
    await this.page.getByLabel("Email").fill(email);
    await this.page.getByLabel("Password").fill(password);
  }

  async submit(): Promise<void> {
    await this.page.getByRole("button", { name: "Sign in" }).click();
  }

  async getErrorMessage(): Promise<string | null> {
    const alert = this.page.getByRole("alert");
    if (await alert.isVisible()) {
      return alert.textContent();
    }
    return null;
  }

  async login(email: string, password: string): Promise<void> {
    await this.fillCredentials(email, password);
    await this.submit();
  }
}

class DashboardPage {
  constructor(private page: Page) {}

  async getWelcomeMessage(): Promise<string> {
    return this.page.getByTestId("welcome-message").innerText();
  }

  async isVisible(): Promise<boolean> {
    return this.page.getByTestId("dashboard").isVisible();
  }
}

test.describe("Login flow", () => {
  test("successful login redirects to dashboard", async ({ page }) => {
    const loginPage = new LoginPage(page);
    const dashboard = new DashboardPage(page);

    await loginPage.navigate();
    await loginPage.login("user@example.com", "password123");

    await expect(page).toHaveURL(/\/dashboard/);
    expect(await dashboard.isVisible()).toBe(true);
  });

  test("invalid credentials show an error message", async ({ page }) => {
    const loginPage = new LoginPage(page);

    await loginPage.navigate();
    await loginPage.login("wrong@example.com", "badpassword");

    const error = await loginPage.getErrorMessage();
    expect(error).toContain("Invalid credentials");
  });
});
```

### Cypress with TypeScript

Cypress supports TypeScript through its built-in bundler. Custom commands can be typed with module augmentation to provide IntelliSense across the entire test suite.

```typescript
// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      login(email: string, password: string): Chainable<void>;
      getByTestId(testId: string): Chainable<JQuery<HTMLElement>>;
      seedDatabase(fixture: string): Chainable<void>;
    }
  }
}

Cypress.Commands.add("login", (email: string, password: string) => {
  cy.session([email, password], () => {
    cy.visit("/login");
    cy.get('[data-testid="email-input"]').type(email);
    cy.get('[data-testid="password-input"]').type(password);
    cy.get('[data-testid="submit-button"]').click();
    cy.url().should("include", "/dashboard");
  });
});

Cypress.Commands.add("getByTestId", (testId: string) => {
  return cy.get(`[data-testid="${testId}"]`);
});

Cypress.Commands.add("seedDatabase", (fixture: string) => {
  cy.request("POST", "/api/test/seed", { fixture });
});

export {};

// cypress/e2e/checkout.cy.ts
describe("Checkout flow", () => {
  beforeEach(() => {
    cy.seedDatabase("products");
    cy.login("shopper@example.com", "password123");
  });

  it("completes a purchase", () => {
    cy.visit("/products");
    cy.getByTestId("product-card").first().click();
    cy.getByTestId("add-to-cart").click();
    cy.getByTestId("cart-count").should("contain", "1");

    cy.getByTestId("checkout-button").click();
    cy.getByTestId("confirm-order").click();

    cy.url().should("include", "/order-confirmation");
    cy.getByTestId("order-status").should("contain", "confirmed");
  });
});
```

---

## Test Patterns and Utilities

### Test Builders and Factories

Test builders use the builder pattern to create domain objects with sensible defaults, overriding only the fields relevant to each test. This reduces boilerplate and makes test intent clear.

```typescript
// tests/factories/user.factory.ts
interface User {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user" | "viewer";
  createdAt: Date;
}

class UserBuilder {
  private user: User = {
    id: crypto.randomUUID(),
    name: "Default User",
    email: "default@example.com",
    role: "user",
    createdAt: new Date("2025-01-01"),
  };

  withId(id: string): this {
    this.user.id = id;
    return this;
  }

  withName(name: string): this {
    this.user.name = name;
    return this;
  }

  withEmail(email: string): this {
    this.user.email = email;
    return this;
  }

  withRole(role: User["role"]): this {
    this.user.role = role;
    return this;
  }

  asAdmin(): this {
    this.user.role = "admin";
    return this;
  }

  build(): User {
    return { ...this.user };
  }
}

function aUser(): UserBuilder {
  return new UserBuilder();
}

// Usage in tests
// const admin = aUser().asAdmin().withName("Admin Alice").build();
// const viewer = aUser().withRole("viewer").withEmail("viewer@test.com").build();

// Generic factory function for simpler cases
function createFactory<T>(defaults: T): (overrides?: Partial<T>) => T {
  return (overrides = {}) => ({ ...defaults, ...overrides });
}

const createUser = createFactory<User>({
  id: "default-id",
  name: "Default User",
  email: "default@example.com",
  role: "user",
  createdAt: new Date("2025-01-01"),
});

// Usage
// const user = createUser({ name: "Custom Name", role: "admin" });
```

### Fixtures and Helpers

Shared test helpers centralize common setup logic, reducing duplication across test files.

```typescript
// tests/helpers/test-clock.ts
class TestClock {
  private currentTime: Date;

  constructor(initialTime: Date = new Date("2025-01-01T00:00:00Z")) {
    this.currentTime = initialTime;
  }

  now(): Date {
    return new Date(this.currentTime);
  }

  advance(ms: number): void {
    this.currentTime = new Date(this.currentTime.getTime() + ms);
  }

  advanceMinutes(minutes: number): void {
    this.advance(minutes * 60 * 1000);
  }

  advanceHours(hours: number): void {
    this.advance(hours * 60 * 60 * 1000);
  }
}

// tests/helpers/assertions.ts
function expectDatesClose(actual: Date, expected: Date, toleranceMs: number = 1000): void {
  const diff = Math.abs(actual.getTime() - expected.getTime());
  expect(diff).toBeLessThanOrEqual(toleranceMs);
}

function expectArrayToContainAll<T>(actual: T[], expected: T[]): void {
  for (const item of expected) {
    expect(actual).toContainEqual(item);
  }
}

// tests/helpers/test-database.ts
interface TestDatabaseContext {
  db: DatabaseConnection;
  cleanup(): Promise<void>;
}

async function withTestDatabase(
  fn: (ctx: TestDatabaseContext) => Promise<void>
): Promise<void> {
  const db = await createTestDatabaseConnection();
  try {
    await fn({ db, cleanup: () => db.execute("DELETE FROM users") });
  } finally {
    await db.execute("ROLLBACK");
  }
}

// Usage in tests
// it("does something with the database", async () => {
//   await withTestDatabase(async ({ db }) => {
//     const repo = new UserRepository(db);
//     const user = await repo.save(aUser().build());
//     expect(user.id).toBeDefined();
//   });
// });

declare function createTestDatabaseConnection(): Promise<DatabaseConnection>;

interface DatabaseConnection {
  query<T>(sql: string, params?: unknown[]): Promise<T[]>;
  execute(sql: string, params?: unknown[]): Promise<{ affectedRows: number }>;
}
```

---

## Code Coverage with TypeScript

### Coverage Configuration

Code coverage tools measure which lines, branches, and functions are exercised by tests. When working with TypeScript, configure coverage to operate on the source `.ts` files rather than compiled output, and exclude type-only files that have no runtime code.

```typescript
// vitest.config.ts — coverage section
export default defineConfig({
  test: {
    coverage: {
      provider: "v8",               // or "istanbul"
      reporter: ["text", "html", "lcov", "json-summary"],
      reportsDirectory: "./coverage",
      include: ["src/**/*.ts"],
      exclude: [
        "src/**/*.d.ts",            // type declaration files
        "src/**/*.test.ts",         // test files
        "src/**/*.spec.ts",
        "src/**/index.ts",          // barrel exports
        "src/**/__mocks__/**",      // mock files
        "src/types/**",             // type-only modules
      ],
      thresholds: {
        branches: 80,
        functions: 85,
        lines: 85,
        statements: 85,
      },
    },
  },
});
```

### Coverage Thresholds and Reporting

Enforce minimum coverage thresholds in CI to prevent regressions. Use per-file thresholds for critical modules that require higher coverage.

```typescript
// jest.config.ts — coverage thresholds
const config: Config = {
  collectCoverageFrom: [
    "src/**/*.ts",
    "!src/**/*.d.ts",
    "!src/**/*.test.ts",
  ],
  coverageThresholds: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    // Stricter thresholds for critical modules
    "./src/services/payment.service.ts": {
      branches: 95,
      functions: 100,
      lines: 95,
      statements: 95,
    },
    "./src/utils/validation.ts": {
      branches: 90,
      functions: 100,
      lines: 95,
      statements: 95,
    },
  },
  coverageReporters: ["text", "html", "lcov"],
};
```

---

## Next Steps

Continue your TypeScript learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | TypeScript Fundamentals | Core language features, basic types, and project setup |
| [01-TYPE-SYSTEM.md](01-TYPE-SYSTEM.md) | Type System | Generics, utility types, conditional types, and type inference |
| [02-PATTERNS.md](02-PATTERNS.md) | Design Patterns | Common design patterns implemented in TypeScript |
| [03-NODE-AND-BACKEND.md](03-NODE-AND-BACKEND.md) | Node.js and Backend | Server-side TypeScript with Express, NestJS, Fastify, and databases |
| [04-FRONTEND-FRAMEWORKS.md](04-FRONTEND-FRAMEWORKS.md) | Frontend Frameworks | React, Angular, and Vue with TypeScript |
| [06-TOOLING.md](06-TOOLING.md) | Tooling | Build tools, linters, and developer experience |
| [07-BEST-PRACTICES.md](07-BEST-PRACTICES.md) | Best Practices | Coding standards and recommended approaches |
| [08-ANTI-PATTERNS.md](08-ANTI-PATTERNS.md) | Anti-Patterns | Common mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Guided progression through all TypeScript topics |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial TypeScript Testing documentation |
