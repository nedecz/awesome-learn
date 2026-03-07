# TypeScript Design Patterns

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Creational Patterns](#creational-patterns)
   - [Singleton](#singleton)
   - [Factory](#factory)
   - [Abstract Factory](#abstract-factory)
   - [Builder](#builder)
3. [Structural Patterns](#structural-patterns)
   - [Adapter](#adapter)
   - [Decorator](#decorator)
   - [Facade](#facade)
   - [Proxy](#proxy)
4. [Behavioral Patterns](#behavioral-patterns)
   - [Observer](#observer)
   - [Strategy](#strategy)
   - [Command](#command)
   - [State](#state)
5. [Functional Patterns](#functional-patterns)
   - [Function Composition](#function-composition)
   - [Currying and Partial Application](#currying-and-partial-application)
   - [Pipeline](#pipeline)
   - [Monads and the Result Type](#monads-and-the-result-type)
6. [TypeScript-Specific Patterns](#typescript-specific-patterns)
   - [Branded Types](#branded-types)
   - [Phantom Types](#phantom-types)
   - [Type-Safe Builder](#type-safe-builder)
7. [Module Patterns and Dependency Injection](#module-patterns-and-dependency-injection)
   - [Module Patterns](#module-patterns)
   - [Dependency Injection](#dependency-injection)
8. [Error Handling Patterns](#error-handling-patterns)
   - [Discriminated Union Errors](#discriminated-union-errors)
   - [The Result Type](#the-result-type)
   - [The Either Type](#the-either-type)
9. [Next Steps](#next-steps)
10. [Version History](#version-history)

---

## Overview

Design patterns are reusable solutions to common problems in software design. TypeScript's type system — with generics, union types, and structural typing — makes many classic patterns more expressive and safer than their plain JavaScript counterparts. This document covers the Gang of Four patterns adapted for TypeScript, functional programming patterns that leverage the type system, and TypeScript-specific patterns that have no equivalent in other languages.

### Target Audience

- **JavaScript Developers** — looking to apply well-known OOP and functional patterns with full type safety
- **Backend Engineers** — building scalable Node.js services and needing robust architectural patterns
- **Frontend Developers** — structuring React, Angular, or Vue applications with maintainable patterns
- **Software Architects** — evaluating which patterns translate well to TypeScript's structural type system

### Scope

- Creational, structural, and behavioral design patterns with TypeScript implementations
- Functional programming patterns including composition, currying, and pipelines
- TypeScript-specific patterns like branded types, phantom types, and type-safe builders
- Module organization and dependency injection strategies
- Error handling patterns using discriminated unions, Result, and Either types

---

## Creational Patterns

Creational patterns deal with object creation mechanisms, aiming to create objects in a manner suitable to the situation.

### Singleton

The Singleton pattern ensures a class has only one instance and provides a global point of access to it. In TypeScript, you can enforce this at compile time.

```typescript
class DatabaseConnection {
  private static instance: DatabaseConnection;
  private connectionString: string;

  private constructor(connectionString: string) {
    this.connectionString = connectionString;
  }

  static getInstance(connectionString?: string): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      if (!connectionString) {
        throw new Error("Connection string required for first initialization");
      }
      DatabaseConnection.instance = new DatabaseConnection(connectionString);
    }
    return DatabaseConnection.instance;
  }

  query(sql: string): void {
    console.log(`Executing on ${this.connectionString}: ${sql}`);
  }
}

// Usage
const db1 = DatabaseConnection.getInstance("postgres://localhost:5432/mydb");
const db2 = DatabaseConnection.getInstance();
console.log(db1 === db2); // true
```

A more idiomatic approach in TypeScript uses module-scoped instances, since ES modules are singletons by default:

```typescript
// config.ts — the module itself acts as the singleton
class AppConfig {
  readonly port: number;
  readonly host: string;
  readonly debug: boolean;

  constructor() {
    this.port = Number(process.env.PORT) || 3000;
    this.host = process.env.HOST || "localhost";
    this.debug = process.env.DEBUG === "true";
  }
}

export const config = new AppConfig();
```

### Factory

The Factory pattern provides an interface for creating objects without specifying their concrete classes. TypeScript's discriminated unions make this pattern especially clean.

```typescript
interface Notification {
  send(message: string): void;
}

class EmailNotification implements Notification {
  constructor(private recipient: string) {}

  send(message: string): void {
    console.log(`Email to ${this.recipient}: ${message}`);
  }
}

class SmsNotification implements Notification {
  constructor(private phoneNumber: string) {}

  send(message: string): void {
    console.log(`SMS to ${this.phoneNumber}: ${message}`);
  }
}

class PushNotification implements Notification {
  constructor(private deviceToken: string) {}

  send(message: string): void {
    console.log(`Push to ${this.deviceToken}: ${message}`);
  }
}

type NotificationChannel = "email" | "sms" | "push";

interface NotificationConfig {
  email: { recipient: string };
  sms: { phoneNumber: string };
  push: { deviceToken: string };
}

// Type-safe factory — config type depends on the channel
function createNotification<T extends NotificationChannel>(
  channel: T,
  config: NotificationConfig[T]
): Notification {
  switch (channel) {
    case "email":
      return new EmailNotification((config as NotificationConfig["email"]).recipient);
    case "sms":
      return new SmsNotification((config as NotificationConfig["sms"]).phoneNumber);
    case "push":
      return new PushNotification((config as NotificationConfig["push"]).deviceToken);
    default:
      throw new Error(`Unknown channel: ${channel}`);
  }
}

// Usage — TypeScript enforces the correct config shape
const email = createNotification("email", { recipient: "user@example.com" });
const sms = createNotification("sms", { phoneNumber: "+1234567890" });
```

### Abstract Factory

The Abstract Factory pattern provides an interface for creating families of related objects. TypeScript interfaces make the contract explicit.

```typescript
interface Button {
  render(): string;
  onClick(handler: () => void): void;
}

interface TextInput {
  render(): string;
  getValue(): string;
}

interface UIFactory {
  createButton(label: string): Button;
  createTextInput(placeholder: string): TextInput;
}

// Light theme family
class LightButton implements Button {
  constructor(private label: string) {}
  render(): string {
    return `<button class="light">${this.label}</button>`;
  }
  onClick(handler: () => void): void {
    handler();
  }
}

class LightTextInput implements TextInput {
  private value = "";
  constructor(private placeholder: string) {}
  render(): string {
    return `<input class="light" placeholder="${this.placeholder}" />`;
  }
  getValue(): string {
    return this.value;
  }
}

class LightUIFactory implements UIFactory {
  createButton(label: string): Button {
    return new LightButton(label);
  }
  createTextInput(placeholder: string): TextInput {
    return new LightTextInput(placeholder);
  }
}

// Dark theme family
class DarkButton implements Button {
  constructor(private label: string) {}
  render(): string {
    return `<button class="dark">${this.label}</button>`;
  }
  onClick(handler: () => void): void {
    handler();
  }
}

class DarkTextInput implements TextInput {
  private value = "";
  constructor(private placeholder: string) {}
  render(): string {
    return `<input class="dark" placeholder="${this.placeholder}" />`;
  }
  getValue(): string {
    return this.value;
  }
}

class DarkUIFactory implements UIFactory {
  createButton(label: string): Button {
    return new DarkButton(label);
  }
  createTextInput(placeholder: string): TextInput {
    return new DarkTextInput(placeholder);
  }
}

// Client code works with any theme
function buildForm(factory: UIFactory): string {
  const nameInput = factory.createTextInput("Enter your name");
  const submitButton = factory.createButton("Submit");
  return `${nameInput.render()}\n${submitButton.render()}`;
}
```

### Builder

The Builder pattern constructs complex objects step by step. TypeScript's fluent interfaces and method chaining make this pattern natural.

```typescript
interface HttpRequest {
  url: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
  headers: Record<string, string>;
  body?: string;
  timeout: number;
}

class HttpRequestBuilder {
  private request: Partial<HttpRequest> = {};

  setUrl(url: string): this {
    this.request.url = url;
    return this;
  }

  setMethod(method: HttpRequest["method"]): this {
    this.request.method = method;
    return this;
  }

  addHeader(key: string, value: string): this {
    this.request.headers = { ...this.request.headers, [key]: value };
    return this;
  }

  setBody(body: object): this {
    this.request.body = JSON.stringify(body);
    this.request.headers = {
      ...this.request.headers,
      "Content-Type": "application/json",
    };
    return this;
  }

  setTimeout(ms: number): this {
    this.request.timeout = ms;
    return this;
  }

  build(): HttpRequest {
    if (!this.request.url) throw new Error("URL is required");
    return {
      url: this.request.url,
      method: this.request.method ?? "GET",
      headers: this.request.headers ?? {},
      body: this.request.body,
      timeout: this.request.timeout ?? 30000,
    };
  }
}

// Usage
const request = new HttpRequestBuilder()
  .setUrl("https://api.example.com/users")
  .setMethod("POST")
  .addHeader("Authorization", "Bearer token123")
  .setBody({ name: "Alice", email: "alice@example.com" })
  .setTimeout(5000)
  .build();
```

---

## Structural Patterns

Structural patterns deal with object composition, creating relationships between objects to form larger structures.

### Adapter

The Adapter pattern converts the interface of a class into another interface that clients expect. This is common when integrating third-party libraries.

```typescript
// Legacy payment processor with an incompatible interface
interface LegacyPayment {
  makePayment(amount: number, currency: string, cardNumber: string): boolean;
}

// Modern payment interface your application expects
interface PaymentProcessor {
  charge(request: ChargeRequest): Promise<ChargeResult>;
}

interface ChargeRequest {
  amount: number;
  currency: string;
  paymentMethod: { type: "card"; cardNumber: string };
}

interface ChargeResult {
  success: boolean;
  transactionId: string;
}

// Adapter wraps the legacy interface
class LegacyPaymentAdapter implements PaymentProcessor {
  constructor(private legacyProcessor: LegacyPayment) {}

  async charge(request: ChargeRequest): Promise<ChargeResult> {
    const success = this.legacyProcessor.makePayment(
      request.amount,
      request.currency,
      request.paymentMethod.cardNumber
    );

    return {
      success,
      transactionId: success ? crypto.randomUUID() : "",
    };
  }
}
```

### Decorator

The Decorator pattern attaches additional responsibilities to an object dynamically. TypeScript supports both class-based decorators and functional wrapping.

```typescript
// Functional decorator approach — composable and type-safe
interface Logger {
  log(message: string): void;
  error(message: string): void;
}

class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(message);
  }
  error(message: string): void {
    console.error(message);
  }
}

// Decorator that adds timestamps
function withTimestamp(logger: Logger): Logger {
  return {
    log(message: string): void {
      logger.log(`[${new Date().toISOString()}] ${message}`);
    },
    error(message: string): void {
      logger.error(`[${new Date().toISOString()}] ${message}`);
    },
  };
}

// Decorator that adds a prefix
function withPrefix(logger: Logger, prefix: string): Logger {
  return {
    log(message: string): void {
      logger.log(`[${prefix}] ${message}`);
    },
    error(message: string): void {
      logger.error(`[${prefix}] ${message}`);
    },
  };
}

// Compose decorators — order matters
const logger = withTimestamp(withPrefix(new ConsoleLogger(), "APP"));
logger.log("Server started");
// Output: [2025-01-15T10:30:00.000Z] [APP] Server started
```

### Facade

The Facade pattern provides a simplified interface to a complex subsystem. This is especially useful when multiple services need to coordinate.

```typescript
// Complex subsystem classes
class InventoryService {
  checkStock(productId: string): boolean {
    return true; // simplified
  }

  reserveStock(productId: string, quantity: number): string {
    return `reservation-${productId}`;
  }
}

class PaymentService {
  processPayment(amount: number, method: string): string {
    return `payment-${Date.now()}`;
  }
}

class ShippingService {
  calculateShipping(address: string): number {
    return 5.99;
  }

  createShipment(orderId: string, address: string): string {
    return `shipment-${orderId}`;
  }
}

// Facade simplifies the entire checkout flow
class CheckoutFacade {
  constructor(
    private inventory: InventoryService,
    private payment: PaymentService,
    private shipping: ShippingService
  ) {}

  async placeOrder(order: {
    productId: string;
    quantity: number;
    address: string;
    paymentMethod: string;
    price: number;
  }): Promise<{ orderId: string; shipmentId: string }> {
    // Step 1: Check inventory
    if (!this.inventory.checkStock(order.productId)) {
      throw new Error("Product out of stock");
    }

    // Step 2: Reserve stock
    const reservationId = this.inventory.reserveStock(order.productId, order.quantity);

    // Step 3: Calculate total with shipping
    const shippingCost = this.shipping.calculateShipping(order.address);
    const total = order.price * order.quantity + shippingCost;

    // Step 4: Process payment
    const paymentId = this.payment.processPayment(total, order.paymentMethod);

    // Step 5: Create shipment
    const shipmentId = this.shipping.createShipment(reservationId, order.address);

    return { orderId: reservationId, shipmentId };
  }
}
```

### Proxy

The Proxy pattern provides a surrogate or placeholder for another object to control access to it. Common uses include caching, lazy loading, and access control.

```typescript
interface DataStore {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
}

class RemoteDataStore implements DataStore {
  async get(key: string): Promise<string | null> {
    // Simulate expensive remote call
    console.log(`Fetching ${key} from remote store...`);
    return `value-for-${key}`;
  }

  async set(key: string, value: string): Promise<void> {
    console.log(`Saving ${key} to remote store...`);
  }
}

// Caching proxy — intercepts reads and caches results
class CachingProxy implements DataStore {
  private cache = new Map<string, { value: string; expires: number }>();

  constructor(
    private store: DataStore,
    private ttlMs: number = 60000
  ) {}

  async get(key: string): Promise<string | null> {
    const cached = this.cache.get(key);
    if (cached && cached.expires > Date.now()) {
      console.log(`Cache hit for ${key}`);
      return cached.value;
    }

    const value = await this.store.get(key);
    if (value !== null) {
      this.cache.set(key, { value, expires: Date.now() + this.ttlMs });
    }
    return value;
  }

  async set(key: string, value: string): Promise<void> {
    this.cache.delete(key);
    await this.store.set(key, value);
  }
}

// Usage — client code works with the same interface
const store: DataStore = new CachingProxy(new RemoteDataStore(), 30000);
```

---

## Behavioral Patterns

Behavioral patterns define how objects interact and distribute responsibility.

### Observer

The Observer pattern defines a one-to-many dependency so that when one object changes state, all its dependents are notified. TypeScript generics make the event types fully type-safe.

```typescript
type EventMap = Record<string, unknown>;

class TypedEventEmitter<T extends EventMap> {
  private listeners = new Map<keyof T, Set<(data: any) => void>>();

  on<K extends keyof T>(event: K, handler: (data: T[K]) => void): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);

    // Return an unsubscribe function
    return () => {
      this.listeners.get(event)?.delete(handler);
    };
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    this.listeners.get(event)?.forEach((handler) => handler(data));
  }
}

// Define event types
interface AppEvents {
  userLogin: { userId: string; timestamp: Date };
  userLogout: { userId: string };
  orderPlaced: { orderId: string; total: number };
}

// Usage — fully type-checked
const emitter = new TypedEventEmitter<AppEvents>();

const unsubscribe = emitter.on("userLogin", (data) => {
  // data is typed as { userId: string; timestamp: Date }
  console.log(`User ${data.userId} logged in at ${data.timestamp}`);
});

emitter.emit("userLogin", { userId: "123", timestamp: new Date() });
unsubscribe();
```

### Strategy

The Strategy pattern defines a family of algorithms and makes them interchangeable. Functions as first-class citizens in TypeScript make this pattern lightweight.

```typescript
// Strategy as a function type
type SortStrategy<T> = (items: T[]) => T[];

// Concrete strategies
const sortByPrice: SortStrategy<{ price: number }> = (items) =>
  [...items].sort((a, b) => a.price - b.price);

const sortByName: SortStrategy<{ name: string }> = (items) =>
  [...items].sort((a, b) => a.name.localeCompare(b.name));

const sortByRating: SortStrategy<{ rating: number }> = (items) =>
  [...items].sort((a, b) => b.rating - a.rating);

// Context that uses a strategy
interface Product {
  name: string;
  price: number;
  rating: number;
}

class ProductCatalog {
  private products: Product[] = [];

  add(product: Product): void {
    this.products.push(product);
  }

  display(sortBy: SortStrategy<Product>): Product[] {
    return sortBy(this.products);
  }
}

// Usage — swap strategies at runtime
const catalog = new ProductCatalog();
catalog.add({ name: "Widget", price: 25.0, rating: 4.5 });
catalog.add({ name: "Gadget", price: 15.0, rating: 4.8 });
catalog.add({ name: "Doohickey", price: 35.0, rating: 3.9 });

const cheapFirst = catalog.display(sortByPrice);
const bestRated = catalog.display(sortByRating);
```

### Command

The Command pattern encapsulates a request as an object, allowing you to parameterize operations, queue them, and support undo.

```typescript
interface Command {
  execute(): void;
  undo(): void;
}

class TextEditor {
  private content = "";

  getContent(): string {
    return this.content;
  }

  insert(text: string, position: number): void {
    this.content = this.content.slice(0, position) + text + this.content.slice(position);
  }

  delete(position: number, length: number): string {
    const deleted = this.content.slice(position, position + length);
    this.content = this.content.slice(0, position) + this.content.slice(position + length);
    return deleted;
  }
}

class InsertCommand implements Command {
  constructor(
    private editor: TextEditor,
    private text: string,
    private position: number
  ) {}

  execute(): void {
    this.editor.insert(this.text, this.position);
  }

  undo(): void {
    this.editor.delete(this.position, this.text.length);
  }
}

class DeleteCommand implements Command {
  private deletedText = "";

  constructor(
    private editor: TextEditor,
    private position: number,
    private length: number
  ) {}

  execute(): void {
    this.deletedText = this.editor.delete(this.position, this.length);
  }

  undo(): void {
    this.editor.insert(this.deletedText, this.position);
  }
}

// Command history for undo/redo
class CommandHistory {
  private undoStack: Command[] = [];
  private redoStack: Command[] = [];

  execute(command: Command): void {
    command.execute();
    this.undoStack.push(command);
    this.redoStack = [];
  }

  undo(): void {
    const command = this.undoStack.pop();
    if (command) {
      command.undo();
      this.redoStack.push(command);
    }
  }

  redo(): void {
    const command = this.redoStack.pop();
    if (command) {
      command.execute();
      this.undoStack.push(command);
    }
  }
}
```

### State

The State pattern allows an object to alter its behavior when its internal state changes. The object appears to change its class.

```typescript
interface ConnectionState {
  connect(): void;
  disconnect(): void;
  send(data: string): void;
}

class WebSocketClient {
  private state: ConnectionState;

  constructor(private url: string) {
    this.state = new DisconnectedState(this);
  }

  setState(state: ConnectionState): void {
    this.state = state;
  }

  getUrl(): string {
    return this.url;
  }

  connect(): void {
    this.state.connect();
  }

  disconnect(): void {
    this.state.disconnect();
  }

  send(data: string): void {
    this.state.send(data);
  }
}

class DisconnectedState implements ConnectionState {
  constructor(private client: WebSocketClient) {}

  connect(): void {
    console.log(`Connecting to ${this.client.getUrl()}...`);
    this.client.setState(new ConnectingState(this.client));
  }

  disconnect(): void {
    console.log("Already disconnected");
  }

  send(_data: string): void {
    console.log("Cannot send — not connected");
  }
}

class ConnectingState implements ConnectionState {
  constructor(private client: WebSocketClient) {
    // Simulate async connection
    setTimeout(() => {
      console.log("Connected!");
      this.client.setState(new ConnectedState(this.client));
    }, 1000);
  }

  connect(): void {
    console.log("Already connecting...");
  }

  disconnect(): void {
    console.log("Aborting connection");
    this.client.setState(new DisconnectedState(this.client));
  }

  send(_data: string): void {
    console.log("Cannot send — still connecting");
  }
}

class ConnectedState implements ConnectionState {
  constructor(private client: WebSocketClient) {}

  connect(): void {
    console.log("Already connected");
  }

  disconnect(): void {
    console.log("Disconnecting...");
    this.client.setState(new DisconnectedState(this.client));
  }

  send(data: string): void {
    console.log(`Sending: ${data}`);
  }
}
```

---

## Functional Patterns

TypeScript's support for first-class functions, generics, and type inference makes functional patterns both practical and type-safe.

### Function Composition

Composition combines simple functions to build more complex ones. Each function takes the output of the previous as input.

```typescript
// Type-safe compose for two functions
function compose<A, B, C>(f: (b: B) => C, g: (a: A) => B): (a: A) => C {
  return (a: A) => f(g(a));
}

// Variadic pipe — applies functions left to right
function pipe<T>(value: T, ...fns: Array<(arg: any) => any>): any {
  return fns.reduce((acc, fn) => fn(acc), value);
}

// Practical example
const trim = (s: string): string => s.trim();
const toLowerCase = (s: string): string => s.toLowerCase();
const replaceSpaces = (s: string): string => s.replace(/\s+/g, "-");

const slugify = (input: string): string =>
  pipe(input, trim, toLowerCase, replaceSpaces);

console.log(slugify("  Hello World  ")); // "hello-world"

// Composing data transformations
interface User {
  name: string;
  email: string;
  age: number;
}

const filterAdults = (users: User[]): User[] =>
  users.filter((u) => u.age >= 18);

const sortByName = (users: User[]): User[] =>
  [...users].sort((a, b) => a.name.localeCompare(b.name));

const toEmails = (users: User[]): string[] =>
  users.map((u) => u.email);

// Compose a pipeline
const getAdultEmails = (users: User[]): string[] =>
  pipe(users, filterAdults, sortByName, toEmails);
```

### Currying and Partial Application

Currying transforms a function that takes multiple arguments into a sequence of functions each taking a single argument. Partial application fixes some arguments of a function, producing another function with fewer parameters.

```typescript
// Manual currying with type safety
function curry2<A, B, R>(fn: (a: A, b: B) => R): (a: A) => (b: B) => R {
  return (a: A) => (b: B) => fn(a, b);
}

function curry3<A, B, C, R>(
  fn: (a: A, b: B, c: C) => R
): (a: A) => (b: B) => (c: C) => R {
  return (a: A) => (b: B) => (c: C) => fn(a, b, c);
}

// Practical curried functions
const multiply = curry2((a: number, b: number) => a * b);
const double = multiply(2);
const triple = multiply(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15

// Curried configuration
const createLogger = curry3(
  (level: string, prefix: string, message: string) =>
    console.log(`[${level}] [${prefix}] ${message}`)
);

const errorLog = createLogger("ERROR");
const appError = errorLog("APP");
appError("Something went wrong"); // [ERROR] [APP] Something went wrong

// Partial application using a utility
function partial<A, B extends unknown[], R>(
  fn: (a: A, ...rest: B) => R,
  a: A
): (...rest: B) => R {
  return (...rest: B) => fn(a, ...rest);
}

const addTax = (rate: number, price: number): number => price * (1 + rate);
const addVAT = partial(addTax, 0.2);
console.log(addVAT(100)); // 120
```

### Pipeline

The pipeline pattern chains operations in a readable, sequential flow. TypeScript can enforce type safety across each step.

```typescript
class Pipeline<T> {
  private constructor(private value: T) {}

  static of<T>(value: T): Pipeline<T> {
    return new Pipeline(value);
  }

  pipe<U>(fn: (value: T) => U): Pipeline<U> {
    return new Pipeline(fn(this.value));
  }

  result(): T {
    return this.value;
  }
}

// Usage — each step is type-checked
const processed = Pipeline.of("  Hello, World!  ")
  .pipe((s) => s.trim())
  .pipe((s) => s.toLowerCase())
  .pipe((s) => s.split(", "))
  .pipe((arr) => arr.map((word) => word.charAt(0).toUpperCase() + word.slice(1)))
  .pipe((arr) => arr.join(" "))
  .result();

console.log(processed); // "Hello World!"

// Async pipeline for data processing
class AsyncPipeline<T> {
  private constructor(private promise: Promise<T>) {}

  static of<T>(value: T | Promise<T>): AsyncPipeline<T> {
    return new AsyncPipeline(Promise.resolve(value));
  }

  pipe<U>(fn: (value: T) => U | Promise<U>): AsyncPipeline<U> {
    return new AsyncPipeline(this.promise.then(fn));
  }

  async result(): Promise<T> {
    return this.promise;
  }
}

// Usage with async operations
const userData = await AsyncPipeline.of("/api/users/123")
  .pipe((url) => fetch(url))
  .pipe((response) => response.json())
  .pipe((data) => ({ ...data, fetchedAt: new Date() }))
  .result();
```

### Monads and the Result Type

Monads are a pattern for chaining operations on wrapped values. The Result type (also called `Either`) is the most practical monad in everyday TypeScript — it represents a computation that can succeed or fail without throwing exceptions.

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

// Constructor helpers
function Ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function Err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

// Functor — transform the success value
function map<T, U, E>(result: Result<T, E>, fn: (value: T) => U): Result<U, E> {
  return result.ok ? Ok(fn(result.value)) : result;
}

// Monad — chain operations that return Result
function flatMap<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => Result<U, E>
): Result<U, E> {
  return result.ok ? fn(result.value) : result;
}

// Practical example: validating user input
interface UserInput {
  name: string;
  email: string;
  age: string;
}

function validateName(input: UserInput): Result<UserInput, string> {
  return input.name.length >= 2
    ? Ok(input)
    : Err("Name must be at least 2 characters");
}

function validateEmail(input: UserInput): Result<UserInput, string> {
  return input.email.includes("@")
    ? Ok(input)
    : Err("Invalid email address");
}

function parseAge(input: UserInput): Result<{ name: string; email: string; age: number }, string> {
  const age = parseInt(input.age, 10);
  if (isNaN(age) || age < 0 || age > 150) {
    return Err("Invalid age");
  }
  return Ok({ name: input.name, email: input.email, age });
}

// Chain validations — stops at the first error
function validateUser(input: UserInput) {
  return flatMap(flatMap(validateName(input), validateEmail), parseAge);
}

const result = validateUser({ name: "Alice", email: "alice@example.com", age: "30" });
if (result.ok) {
  console.log(`Valid user: ${result.value.name}, age ${result.value.age}`);
} else {
  console.log(`Validation error: ${result.error}`);
}
```

---

## TypeScript-Specific Patterns

These patterns leverage TypeScript's type system in ways that have no direct counterpart in plain JavaScript.

### Branded Types

Branded types (also called **opaque types** or **nominal types**) prevent accidental mixing of structurally identical types. They use an intersection with a unique symbol to make types nominally distinct.

```typescript
// Declare a unique brand using a symbol
declare const brand: unique symbol;

type Brand<T, B extends string> = T & { readonly [brand]: B };

// Create distinct ID types — all are strings at runtime
type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
type ProductId = Brand<string, "ProductId">;

// Smart constructors validate and brand the value
function UserId(id: string): UserId {
  if (!id.startsWith("usr_")) throw new Error("Invalid user ID format");
  return id as UserId;
}

function OrderId(id: string): OrderId {
  if (!id.startsWith("ord_")) throw new Error("Invalid order ID format");
  return id as OrderId;
}

// Type safety — cannot mix up IDs even though they are all strings
function getUser(id: UserId): void {
  console.log(`Fetching user ${id}`);
}

function getOrder(id: OrderId): void {
  console.log(`Fetching order ${id}`);
}

const userId = UserId("usr_abc123");
const orderId = OrderId("ord_xyz789");

getUser(userId);    // OK
getOrder(orderId);  // OK
// getUser(orderId); // Compile error — OrderId is not assignable to UserId

// Branded primitives for units
type Kilometers = Brand<number, "Kilometers">;
type Miles = Brand<number, "Miles">;

function kmToMiles(km: Kilometers): Miles {
  return (km * 0.621371) as Miles;
}
```

### Phantom Types

Phantom types use type parameters that do not appear in the runtime value. They encode state or capabilities at the type level, so the compiler enforces correct usage with zero runtime cost.

```typescript
// A form builder that tracks whether required fields have been set
interface FormState {
  hasName: boolean;
  hasEmail: boolean;
}

type FormBuilder<S extends FormState> = {
  _state: S; // phantom — never accessed at runtime
  data: Record<string, string>;
};

function createForm(): FormBuilder<{ hasName: false; hasEmail: false }> {
  return { _state: undefined as any, data: {} };
}

function setName<S extends FormState>(
  form: FormBuilder<S>,
  name: string
): FormBuilder<S & { hasName: true }> {
  return { ...form, data: { ...form.data, name } };
}

function setEmail<S extends FormState>(
  form: FormBuilder<S>,
  email: string
): FormBuilder<S & { hasEmail: true }> {
  return { ...form, data: { ...form.data, email } };
}

// Submit only accepts forms where both name and email are set
function submit(
  form: FormBuilder<{ hasName: true; hasEmail: true }>
): void {
  console.log("Submitting:", form.data);
}

// Usage — compiler enforces the required fields
const form = setEmail(setName(createForm(), "Alice"), "alice@example.com");
submit(form); // OK

// const incomplete = setName(createForm(), "Alice");
// submit(incomplete); // Compile error — hasEmail is false
```

### Type-Safe Builder

Combining the builder pattern with TypeScript's type system ensures that required fields are set before building, catching missing fields at compile time rather than runtime.

```typescript
type RequiredFields = "host" | "port";

type ServerConfig = {
  host: string;
  port: number;
  maxConnections?: number;
  timeout?: number;
};

class ServerConfigBuilder<Set extends string = never> {
  private config: Partial<ServerConfig> = {};

  setHost(host: string): ServerConfigBuilder<Set | "host"> {
    this.config.host = host;
    return this as any;
  }

  setPort(port: number): ServerConfigBuilder<Set | "port"> {
    this.config.port = port;
    return this as any;
  }

  setMaxConnections(n: number): ServerConfigBuilder<Set> {
    this.config.maxConnections = n;
    return this as any;
  }

  setTimeout(ms: number): ServerConfigBuilder<Set> {
    this.config.timeout = ms;
    return this as any;
  }

  // build() is only available when all required fields are set
  build(this: ServerConfigBuilder<RequiredFields>): ServerConfig {
    return this.config as ServerConfig;
  }
}

// Usage
const config = new ServerConfigBuilder()
  .setHost("localhost")
  .setPort(3000)
  .setMaxConnections(100)
  .build(); // OK — host and port are set

// new ServerConfigBuilder()
//   .setHost("localhost")
//   .build(); // Compile error — port is missing
```

---

## Module Patterns and Dependency Injection

### Module Patterns

TypeScript's ES module system provides natural encapsulation. Use barrel files and explicit exports to control your public API surface.

```typescript
// internal/validator.ts — internal module, not exported from barrel
export function isValidEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

// internal/formatter.ts
export function formatUserName(first: string, last: string): string {
  return `${first} ${last}`.trim();
}

// index.ts — barrel file, public API
export { UserService } from "./user-service";
export type { User, CreateUserRequest } from "./types";
// internal modules are NOT exported — they remain encapsulation details
```

Use the **namespace object** pattern to group related functions without a class:

```typescript
// math-utils.ts
export const MathUtils = {
  clamp(value: number, min: number, max: number): number {
    return Math.min(Math.max(value, min), max);
  },

  lerp(a: number, b: number, t: number): number {
    return a + (b - a) * t;
  },

  roundTo(value: number, decimals: number): number {
    const factor = 10 ** decimals;
    return Math.round(value * factor) / factor;
  },
} as const;
```

### Dependency Injection

Dependency injection decouples components from their dependencies, making code testable and flexible. TypeScript interfaces define the contracts.

```typescript
// Define contracts
interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

interface EmailService {
  send(to: string, subject: string, body: string): Promise<void>;
}

interface User {
  id: string;
  name: string;
  email: string;
}

// Service depends on abstractions, not concrete implementations
class UserRegistrationService {
  constructor(
    private userRepo: UserRepository,
    private emailService: EmailService
  ) {}

  async register(name: string, email: string): Promise<User> {
    const user: User = { id: crypto.randomUUID(), name, email };
    await this.userRepo.save(user);
    await this.emailService.send(email, "Welcome!", `Hello ${name}`);
    return user;
  }
}

// Simple DI container using a Map and type-safe tokens
type Token<T> = symbol & { __type: T };

function createToken<T>(name: string): Token<T> {
  return Symbol(name) as Token<T>;
}

class Container {
  private bindings = new Map<symbol, () => any>();

  bind<T>(token: Token<T>, factory: () => T): void {
    this.bindings.set(token, factory);
  }

  get<T>(token: Token<T>): T {
    const factory = this.bindings.get(token);
    if (!factory) throw new Error(`No binding for ${token.toString()}`);
    return factory() as T;
  }
}

// Register dependencies
const Tokens = {
  UserRepo: createToken<UserRepository>("UserRepo"),
  EmailService: createToken<EmailService>("EmailService"),
  Registration: createToken<UserRegistrationService>("Registration"),
};

const container = new Container();
container.bind(Tokens.UserRepo, () => new InMemoryUserRepository());
container.bind(Tokens.EmailService, () => new ConsoleEmailService());
container.bind(
  Tokens.Registration,
  () =>
    new UserRegistrationService(
      container.get(Tokens.UserRepo),
      container.get(Tokens.EmailService)
    )
);
```

---

## Error Handling Patterns

Effective error handling in TypeScript goes beyond `try/catch`. Discriminated unions and result types make errors explicit in the type system.

### Discriminated Union Errors

Use discriminated unions to define a closed set of possible errors. The compiler ensures every error case is handled.

```typescript
type AppError =
  | { kind: "NotFound"; resource: string; id: string }
  | { kind: "Validation"; field: string; message: string }
  | { kind: "Unauthorized"; reason: string }
  | { kind: "RateLimit"; retryAfterMs: number };

function handleError(error: AppError): string {
  switch (error.kind) {
    case "NotFound":
      return `${error.resource} with ID ${error.id} was not found`;
    case "Validation":
      return `Validation error on ${error.field}: ${error.message}`;
    case "Unauthorized":
      return `Access denied: ${error.reason}`;
    case "RateLimit":
      return `Too many requests. Retry after ${error.retryAfterMs}ms`;
  }
}

// Functions return explicit error types
function findUser(id: string): User | AppError {
  if (!id) {
    return { kind: "Validation", field: "id", message: "ID is required" };
  }
  // ...lookup logic
  return { kind: "NotFound", resource: "User", id };
}
```

### The Result Type

The Result type encodes success and failure in the type system. This pattern eliminates unchecked exceptions and makes error flows explicit.

```typescript
// A class-based Result with chainable methods
class Result<T, E> {
  private constructor(
    private readonly _ok: boolean,
    private readonly _value?: T,
    private readonly _error?: E
  ) {}

  static ok<T>(value: T): Result<T, never> {
    return new Result(true, value);
  }

  static err<E>(error: E): Result<never, E> {
    return new Result(false, undefined, error);
  }

  isOk(): this is Result<T, never> {
    return this._ok;
  }

  isErr(): this is Result<never, E> {
    return !this._ok;
  }

  map<U>(fn: (value: T) => U): Result<U, E> {
    return this._ok ? Result.ok(fn(this._value!)) : Result.err(this._error!);
  }

  flatMap<U>(fn: (value: T) => Result<U, E>): Result<U, E> {
    return this._ok ? fn(this._value!) : Result.err(this._error!);
  }

  unwrapOr(defaultValue: T): T {
    return this._ok ? this._value! : defaultValue;
  }

  match<U>(handlers: { ok: (value: T) => U; err: (error: E) => U }): U {
    return this._ok ? handlers.ok(this._value!) : handlers.err(this._error!);
  }
}

// Usage
function parseJSON(input: string): Result<unknown, string> {
  try {
    return Result.ok(JSON.parse(input));
  } catch {
    return Result.err(`Invalid JSON: ${input.slice(0, 50)}`);
  }
}

function extractField(data: unknown, field: string): Result<string, string> {
  if (typeof data === "object" && data !== null && field in data) {
    return Result.ok(String((data as Record<string, unknown>)[field]));
  }
  return Result.err(`Missing field: ${field}`);
}

// Chain operations — errors propagate automatically
const name = parseJSON('{"name": "Alice"}')
  .flatMap((data) => extractField(data, "name"))
  .map((name) => name.toUpperCase())
  .match({
    ok: (value) => `Found: ${value}`,
    err: (error) => `Error: ${error}`,
  });
```

### The Either Type

The Either type generalizes Result by representing any disjunction of two types — not necessarily success and failure. It is useful for branching logic.

```typescript
type Either<L, R> =
  | { tag: "left"; value: L }
  | { tag: "right"; value: R };

function Left<L>(value: L): Either<L, never> {
  return { tag: "left", value };
}

function Right<R>(value: R): Either<never, R> {
  return { tag: "right", value };
}

function mapEither<L, R, U>(
  either: Either<L, R>,
  fn: (value: R) => U
): Either<L, U> {
  return either.tag === "right" ? Right(fn(either.value)) : either;
}

// Practical use: parsing that returns descriptive errors
interface ParseError {
  line: number;
  column: number;
  message: string;
}

interface Config {
  host: string;
  port: number;
}

function parseConfig(raw: string): Either<ParseError, Config> {
  try {
    const parsed = JSON.parse(raw);
    if (typeof parsed.host !== "string") {
      return Left({ line: 1, column: 0, message: "host must be a string" });
    }
    if (typeof parsed.port !== "number") {
      return Left({ line: 1, column: 0, message: "port must be a number" });
    }
    return Right({ host: parsed.host, port: parsed.port });
  } catch {
    return Left({ line: 1, column: 0, message: "Invalid JSON" });
  }
}

// Pattern match on the result
const configResult = parseConfig('{"host": "localhost", "port": 3000}');
if (configResult.tag === "right") {
  console.log(`Connecting to ${configResult.value.host}:${configResult.value.port}`);
} else {
  console.error(`Parse error at ${configResult.value.line}:${configResult.value.column}`);
}
```

---

## Next Steps

Continue your TypeScript learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | TypeScript Fundamentals | Core language features, basic types, and project setup |
| [01-TYPE-SYSTEM.md](01-TYPE-SYSTEM.md) | Type System | Generics, utility types, conditional types, and advanced type-level programming |
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
| 1.0 | 2025 | Initial TypeScript Design Patterns documentation |
