# JavaScript Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [ES6+ Features Essential for Modern Frameworks](#es6-features-essential-for-modern-frameworks)
3. [Closures Prototypes and the Event Loop](#closures-prototypes-and-the-event-loop)
4. [Promises Async Await and Error Handling](#promises-async-await-and-error-handling)
5. [Modules ESM vs CommonJS](#modules-esm-vs-commonjs)
6. [DOM Manipulation and Events](#dom-manipulation-and-events)
7. [Fetch API and HTTP Requests](#fetch-api-and-http-requests)
8. [Web APIs](#web-apis)
9. [Functional Programming Patterns](#functional-programming-patterns)
10. [Common Gotchas and Pitfalls](#common-gotchas-and-pitfalls)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

JavaScript is the programming language of the web. Every modern frontend framework — Angular, React, Vue, Svelte — compiles down to JavaScript that the browser executes. A strong foundation in JavaScript is the single most important prerequisite for effective frontend development.

This document covers the JavaScript features and concepts that are essential for working with modern frameworks, beyond basic syntax.

### Scope

- ES6+ features used daily in modern codebases
- Core language concepts: closures, prototypes, the event loop
- Asynchronous programming with Promises and async/await
- Module systems: ESM and CommonJS
- DOM manipulation and event handling
- Fetch API and HTTP communication
- Key Web APIs: Storage, Workers, Intersection Observer
- Functional programming patterns
- Common gotchas and pitfalls

---

## ES6+ Features Essential for Modern Frameworks

### Destructuring

```javascript
// Object destructuring
const { name, age, role = "user" } = user;

// Array destructuring
const [first, second, ...rest] = items;

// Function parameter destructuring
function createUser({ name, email, role = "member" }) {
  return { name, email, role, createdAt: Date.now() };
}
```

### Template Literals

```javascript
const greeting = `Hello, ${user.name}! You have ${count} messages.`;

// Tagged templates (used by styled-components, GraphQL, etc.)
const query = gql`
  query GetUser($id: ID!) {
    user(id: $id) { name email }
  }
`;
```

### Spread and Rest Operators

```javascript
// Spread: expand an iterable into individual elements
const merged = { ...defaults, ...userConfig };
const combined = [...array1, ...array2];

// Rest: collect remaining elements
function log(message, ...args) {
  console.log(message, ...args);
}
```

### Optional Chaining and Nullish Coalescing

```javascript
// Optional chaining — short-circuits to undefined if any part is null/undefined
const city = user?.address?.city;
const first = items?.[0];
const result = callback?.();

// Nullish coalescing — falls back only for null/undefined (not 0, "", false)
const port = config.port ?? 3000;
const name = user.name ?? "Anonymous";
```

### Arrow Functions

```javascript
// Concise syntax with lexical `this` binding
const double = (x) => x * 2;
const greet = (name) => `Hello, ${name}`;

// Lexical `this` — crucial for class methods and callbacks
class Timer {
  start() {
    // Arrow function inherits `this` from enclosing scope
    setInterval(() => this.tick(), 1000);
  }
}
```

### Map, Set, WeakMap, WeakSet

```javascript
// Map: key-value pairs with any type as key
const cache = new Map();
cache.set(userObj, computedValue);

// Set: unique values
const unique = new Set([1, 2, 2, 3]); // Set {1, 2, 3}

// WeakMap: keys are garbage-collected when no other references exist
const metadata = new WeakMap();
metadata.set(domElement, { clickCount: 0 });
```

---

## Closures Prototypes and the Event Loop

### Closures

A closure is a function that retains access to its lexical scope even when executed outside that scope.

```javascript
function createCounter(initial = 0) {
  let count = initial; // closed over by the returned functions

  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count,
  };
}

const counter = createCounter(10);
counter.increment(); // 11
counter.increment(); // 12
counter.getCount();  // 12
```

**Why closures matter for frameworks:**
- React hooks (`useState`, `useEffect`) rely on closures to capture state
- Event handlers close over component state and props
- Module patterns use closures for encapsulation

### Prototypes

JavaScript uses prototypal inheritance. Every object has a prototype chain that it follows when looking up properties.

```javascript
// Modern class syntax (syntactic sugar over prototypes)
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    return `${this.name} makes a sound.`;
  }
}

class Dog extends Animal {
  speak() {
    return `${this.name} barks.`;
  }
}

// Under the hood: Dog.prototype.__proto__ === Animal.prototype
```

### The Event Loop

JavaScript is single-threaded. The event loop is the mechanism that enables non-blocking asynchronous behavior.

```
┌───────────────────────────┐
│        Call Stack          │  Executes synchronous code one frame at a time
└─────────────┬─────────────┘
              │ When the stack is empty, the event loop checks:
              ▼
┌───────────────────────────┐
│     Microtask Queue       │  Promises (.then), queueMicrotask, MutationObserver
│  (highest priority)       │  Fully drained before moving to macrotasks
└─────────────┬─────────────┘
              │ After all microtasks are processed:
              ▼
┌───────────────────────────┐
│     Macrotask Queue       │  setTimeout, setInterval, I/O, UI rendering
│  (one per event loop tick)│
└───────────────────────────┘
```

```javascript
console.log("1 — synchronous");

setTimeout(() => console.log("2 — macrotask"), 0);

Promise.resolve().then(() => console.log("3 — microtask"));

console.log("4 — synchronous");

// Output: 1, 4, 3, 2
```

---

## Promises Async Await and Error Handling

### Promises

```javascript
function fetchUser(id) {
  return new Promise((resolve, reject) => {
    fetch(`/api/users/${id}`)
      .then((response) => {
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return response.json();
      })
      .then(resolve)
      .catch(reject);
  });
}
```

### Async/Await

```javascript
async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }
  return response.json();
}

// Error handling with try/catch
async function loadDashboard() {
  try {
    const [user, settings, notifications] = await Promise.all([
      fetchUser(userId),
      fetchSettings(userId),
      fetchNotifications(userId),
    ]);
    render({ user, settings, notifications });
  } catch (error) {
    showErrorMessage("Failed to load dashboard. Please try again.");
    console.error("Dashboard load failed:", error);
  }
}
```

### Promise Combinators

| Method | Behavior |
|--------|----------|
| `Promise.all()` | Resolves when ALL promises resolve; rejects if ANY rejects |
| `Promise.allSettled()` | Waits for ALL to settle (resolve or reject); never rejects |
| `Promise.race()` | Resolves/rejects with the FIRST settled promise |
| `Promise.any()` | Resolves with the FIRST fulfilled; rejects only if ALL reject |

---

## Modules ESM vs CommonJS

### ES Modules (ESM) — The Standard

```javascript
// Named exports
export function formatDate(date) { /* ... */ }
export const API_URL = "/api/v1";

// Default export
export default class UserService { /* ... */ }

// Named imports
import { formatDate, API_URL } from "./utils.js";

// Default import
import UserService from "./user-service.js";

// Namespace import
import * as utils from "./utils.js";

// Dynamic import (code splitting)
const module = await import("./heavy-module.js");
```

### CommonJS (CJS) — Node.js Legacy

```javascript
// Export
module.exports = { formatDate, API_URL };
// or
exports.formatDate = formatDate;

// Import
const { formatDate, API_URL } = require("./utils");
```

### ESM vs CommonJS

| Feature | ESM | CommonJS |
|---------|-----|----------|
| **Syntax** | `import`/`export` | `require()`/`module.exports` |
| **Loading** | Static (analyzed at parse time) | Dynamic (evaluated at runtime) |
| **Tree-shaking** | ✅ Yes (static analysis) | ❌ No |
| **Browser support** | ✅ Native (`<script type="module">`) | ❌ Requires bundler |
| **Top-level await** | ✅ Supported | ❌ Not supported |

**Recommendation:** Use ESM for all new code. CommonJS is legacy.

---

## DOM Manipulation and Events

### Selecting Elements

```javascript
// Preferred: querySelector / querySelectorAll
const header = document.querySelector(".header");
const items = document.querySelectorAll(".list-item");

// By ID
const app = document.getElementById("app");
```

### Creating and Modifying Elements

```javascript
const card = document.createElement("div");
card.className = "card";
card.textContent = "Hello, world!";
card.setAttribute("data-id", "42");

document.querySelector(".container").appendChild(card);
```

### Event Handling

```javascript
// addEventListener — preferred approach
button.addEventListener("click", (event) => {
  event.preventDefault();
  handleClick(event.target);
});

// Event delegation — handle events on a parent for dynamic children
document.querySelector(".list").addEventListener("click", (event) => {
  const item = event.target.closest(".list-item");
  if (item) {
    handleItemClick(item.dataset.id);
  }
});

// Cleanup — remove listeners to prevent memory leaks
const handler = () => { /* ... */ };
element.addEventListener("click", handler);
element.removeEventListener("click", handler);
```

---

## Fetch API and HTTP Requests

```javascript
// GET request
const response = await fetch("/api/users");
const users = await response.json();

// POST request with JSON body
const response = await fetch("/api/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Alice", email: "alice@example.com" }),
});

// Handle errors — fetch does NOT reject on HTTP errors
async function fetchJSON(url, options = {}) {
  const response = await fetch(url, {
    headers: { "Content-Type": "application/json", ...options.headers },
    ...options,
  });

  if (!response.ok) {
    const error = new Error(`HTTP ${response.status}: ${response.statusText}`);
    error.status = response.status;
    error.response = response;
    throw error;
  }

  return response.json();
}

// AbortController — cancel requests
const controller = new AbortController();
const response = await fetch("/api/data", { signal: controller.signal });

// Cancel after 5 seconds
setTimeout(() => controller.abort(), 5000);
```

---

## Web APIs

### Local Storage and Session Storage

```javascript
// localStorage — persists across browser sessions
localStorage.setItem("theme", "dark");
const theme = localStorage.getItem("theme");
localStorage.removeItem("theme");

// sessionStorage — cleared when tab closes
sessionStorage.setItem("token", jwt);
```

### Web Workers

```javascript
// main.js — offload heavy computation to a background thread
const worker = new Worker("worker.js");
worker.postMessage({ data: largeDataSet });
worker.onmessage = (event) => {
  console.log("Result:", event.data);
};

// worker.js
self.onmessage = (event) => {
  const result = heavyComputation(event.data);
  self.postMessage(result);
};
```

### Intersection Observer

```javascript
// Lazy load images when they enter the viewport
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        const img = entry.target;
        img.src = img.dataset.src;
        observer.unobserve(img);
      }
    });
  },
  { rootMargin: "200px" } // Start loading 200px before visible
);

document.querySelectorAll("img[data-src]").forEach((img) => {
  observer.observe(img);
});
```

---

## Functional Programming Patterns

### Array Transformations

```javascript
const users = [
  { name: "Alice", age: 30, role: "admin" },
  { name: "Bob", age: 25, role: "user" },
  { name: "Charlie", age: 35, role: "admin" },
];

// map — transform each element
const names = users.map((user) => user.name);

// filter — select elements matching a condition
const admins = users.filter((user) => user.role === "admin");

// reduce — accumulate a single value
const totalAge = users.reduce((sum, user) => sum + user.age, 0);

// Chaining
const adminNames = users
  .filter((user) => user.role === "admin")
  .map((user) => user.name)
  .sort();
```

### Pure Functions and Immutability

```javascript
// Pure function: same input always produces same output, no side effects
const add = (a, b) => a + b;

// Immutable update patterns (critical for React/Redux)
const updatedUser = { ...user, name: "New Name" };
const updatedItems = items.map((item) =>
  item.id === targetId ? { ...item, completed: true } : item
);
const withoutItem = items.filter((item) => item.id !== targetId);
```

---

## Common Gotchas and Pitfalls

| Gotcha | Problem | Solution |
|--------|---------|----------|
| `this` in callbacks | `this` is `undefined` or `window` in regular functions | Use arrow functions or `.bind()` |
| `==` vs `===` | `==` performs type coercion (`"1" == 1` is `true`) | Always use `===` for strict equality |
| `typeof null` | Returns `"object"` (historical bug) | Use `value === null` for null checks |
| `0.1 + 0.2` | Returns `0.30000000000000004` | Use `Math.abs(a - b) < Number.EPSILON` or integer arithmetic |
| `for...in` on arrays | Iterates over all enumerable properties including prototype | Use `for...of`, `.forEach()`, or `.map()` |
| `var` hoisting | `var` is function-scoped and hoisted | Use `const` and `let` (block-scoped) |
| Mutating state | Direct mutation doesn't trigger framework reactivity | Use immutable update patterns (spread, `.map()`, `.filter()`) |
| Forgetting `await` | Promise is returned instead of the resolved value | Always `await` async function calls |
| Implicit globals | Assigning to an undeclared variable creates a global | Use `"use strict"` or ESLint `no-undef` rule |

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [03-TYPESCRIPT](03-TYPESCRIPT.md) | Type system, generics, framework integration |
| 2 | [05-STATE-MANAGEMENT](05-STATE-MANAGEMENT.md) | Flux, Redux, signals, server state |
| 3 | [06-PERFORMANCE](06-PERFORMANCE.md) | Runtime performance, avoiding layout thrashing |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial JavaScript fundamentals documentation |
