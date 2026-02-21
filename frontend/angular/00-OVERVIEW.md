# Angular Overview

## Table of Contents

1. [Overview](#overview)
2. [What Is Angular](#what-is-angular)
3. [History and Evolution](#history-and-evolution)
4. [Angular vs Other Frameworks](#angular-vs-other-frameworks)
5. [Angular Architecture Fundamentals](#angular-architecture-fundamentals)
6. [The Angular CLI](#the-angular-cli)
7. [Angular Versioning and Release Schedule](#angular-versioning-and-release-schedule)
8. [Standalone Components](#standalone-components)
9. [Angular Signals](#angular-signals)
10. [Prerequisites](#prerequisites)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

This document provides a comprehensive introduction to Angular — Google's platform for building web applications. It covers what Angular is, its history from AngularJS through modern Angular with signals and standalone components, how it compares to other frameworks, and the foundational concepts you need before diving into specific topics.

### Target Audience

- **New developers** evaluating Angular as their framework choice
- **React or Vue developers** exploring Angular for the first time
- **AngularJS developers** migrating to modern Angular
- **Architects** comparing frameworks for enterprise projects

### Scope

- What Angular is and why it matters in modern web development
- Evolution from AngularJS (2010) to Angular 19 (2024) with signals
- Comparison with React, Vue, and Svelte — when to choose Angular
- Core architecture: components, modules, services, dependency injection
- The Angular CLI and project structure
- Standalone components as the modern default
- Angular signals: the new reactive primitive

---

## What Is Angular

Angular is a **TypeScript-based, open-source web application framework** developed and maintained by Google. It provides a complete platform for building single-page applications (SPAs) and progressive web apps (PWAs) with a batteries-included approach.

### Why Angular Matters

| Concern | Angular's Approach |
|---------|-------------------|
| **Type safety** | Built on TypeScript from the ground up — types everywhere |
| **Structure** | Opinionated project structure and conventions reduce decision fatigue |
| **Dependency injection** | First-class DI system for testable, decoupled architecture |
| **Tooling** | Powerful CLI for scaffolding, building, testing, and deploying |
| **Enterprise readiness** | Long-term support, predictable releases, Google-scale proven |
| **Full platform** | Router, forms, HTTP client, animations, i18n — all built in |

### Key Characteristics

Angular is a **framework**, not a library. It provides everything you need to build a complete application:

```
┌───────────────────────────────────────────────────────────────┐
│                    Angular Platform                            │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  Components  │  │  Services    │  │  Dependency          │ │
│  │  & Templates │  │  & DI        │  │  Injection           │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  Router      │  │  Forms       │  │  HttpClient          │ │
│  │              │  │  (reactive   │  │  & Interceptors      │ │
│  │              │  │   & template)│  │                      │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  Signals     │  │  RxJS        │  │  Angular CLI         │ │
│  │  (reactive   │  │  (async      │  │  (scaffold, build,   │ │
│  │   primitive) │  │   streams)   │  │   test, deploy)      │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

---

## History and Evolution

### Timeline

| Year | Version | Milestone |
|------|---------|-----------|
| 2010 | AngularJS 1.x | Original framework — two-way data binding, MVC, `$scope` |
| 2016 | Angular 2 | Complete rewrite — TypeScript, components, modules, RxJS |
| 2017 | Angular 4–5 | HttpClient, animations module, AOT by default |
| 2018 | Angular 6–7 | `ng update`, `ng add`, CLI Builders, virtual scrolling |
| 2019 | Angular 8–9 | Differential loading, Ivy renderer (opt-in → default) |
| 2020 | Angular 10–11 | Strict mode, ESLint migration, webpack 5 |
| 2021 | Angular 12–13 | Ivy everywhere, View Engine removed, standalone RFC |
| 2022 | Angular 14 | Standalone components (developer preview), typed forms |
| 2023 | Angular 16–17 | Signals, new control flow (@if/@for/@switch), standalone default |
| 2024 | Angular 18–19 | Zoneless change detection, resource API, signal forms exploration |

### The Big Shifts

**AngularJS → Angular 2 (2016):** A complete rewrite. AngularJS used JavaScript, two-way data binding with `$scope`, and an MVC pattern. Angular 2+ adopted TypeScript, a component-based architecture, RxJS for reactivity, and a hierarchical dependency injection system.

**Ivy Renderer (Angular 9–13):** The Ivy compilation and rendering pipeline replaced View Engine, enabling smaller bundles, faster compilation, better debugging, and tree-shakable components.

**Standalone Components (Angular 14–17):** Standalone components remove the need for NgModules, simplifying the developer experience. Since Angular 17, `ng new` generates standalone projects by default.

**Signals (Angular 16+):** Signals introduce a synchronous, fine-grained reactive primitive that complements RxJS. They enable zoneless change detection, simpler state management, and better performance.

---

## Angular vs Other Frameworks

### Comparison Table

| Feature | Angular | React | Vue | Svelte |
|---------|---------|-------|-----|--------|
| **Type** | Full framework | UI library | Progressive framework | Compiler framework |
| **Language** | TypeScript (required) | JavaScript/TypeScript | JavaScript/TypeScript | JavaScript/TypeScript |
| **Reactivity** | Signals + RxJS | useState/useReducer | Refs + reactive() | Compiler-based |
| **Templating** | HTML templates with directives | JSX | HTML templates with directives | HTML templates with special syntax |
| **Styling** | Component-scoped CSS (built-in) | CSS Modules / CSS-in-JS | Scoped CSS (built-in) | Scoped CSS (built-in) |
| **State management** | Services, Signals, NgRx | Context, Redux, Zustand | Pinia | Svelte stores |
| **DI system** | Built-in hierarchical DI | None (manual) | provide/inject | None |
| **CLI** | @angular/cli (comprehensive) | create-react-app / Vite | create-vue (Vite-based) | create-svelte |
| **SSR** | @angular/ssr | Next.js | Nuxt | SvelteKit |
| **Learning curve** | Steep (many concepts) | Moderate | Gentle | Gentle |
| **Bundle size** | Medium-large | Small (library only) | Small | Very small |
| **Backing** | Google | Meta | Community (Evan You) | Community (Rich Harris / Vercel) |

### When to Choose Angular

Angular is the strongest choice when:

- **Enterprise applications** — Opinionated structure scales well across large teams
- **Complex forms** — Built-in reactive forms with typed validation are best-in-class
- **Dependency injection** — Services and DI reduce coupling in large codebases
- **Full platform needed** — Router, HTTP, forms, i18n, animations all included
- **TypeScript-first** — The entire ecosystem is built on TypeScript
- **Long-term maintenance** — Predictable 6-month release cycle with LTS
- **Google ecosystem** — Integration with Firebase, Google Cloud, Material Design

### When Other Frameworks May Be Better

- **Small interactive widgets** → React or Svelte (lighter weight)
- **Content-heavy sites with SSG** → Next.js (React) or Nuxt (Vue)
- **Rapid prototyping** → Vue or Svelte (lower learning curve)
- **Maximum bundle size control** → Svelte (compiler, no runtime)

---

## Angular Architecture Fundamentals

### Core Building Blocks

```
┌─────────────────────────────────────────────────────┐
│                  Angular Application                 │
│                                                     │
│  ┌─────────────┐      ┌──────────────────────────┐  │
│  │  Component   │─────▶│  Template (HTML)          │  │
│  │  (TypeScript │      │  - Interpolation {{ }}    │  │
│  │   class)     │      │  - Bindings [ ] ( ) [( )] │  │
│  │              │      │  - @if @for @switch @defer │  │
│  └──────┬──────┘      └──────────────────────────┘  │
│         │                                           │
│         │ injects                                   │
│         ▼                                           │
│  ┌─────────────┐      ┌──────────────────────────┐  │
│  │  Service     │─────▶│  HttpClient / API calls   │  │
│  │  (business   │      │  State management          │  │
│  │   logic)     │      │  Shared logic              │  │
│  └──────┬──────┘      └──────────────────────────┘  │
│         │                                           │
│         │ provided by                               │
│         ▼                                           │
│  ┌─────────────────────────────────────────────────┐│
│  │  Dependency Injection (Hierarchical Injectors)  ││
│  │  root → component → element                     ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

### Components

Components are the fundamental building blocks. Each component is a TypeScript class decorated with `@Component` that controls a section of the UI:

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-greeting',
  standalone: true,
  template: `
    <h1>Hello, {{ name() }}!</h1>
    <button (click)="updateName('Angular')">Greet Angular</button>
  `,
})
export class GreetingComponent {
  name = signal('World');

  updateName(newName: string): void {
    this.name.set(newName);
  }
}
```

### Services and Dependency Injection

Services encapsulate business logic and are provided through Angular's DI system:

```typescript
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class UserService {
  private currentUser = signal<User | null>(null);

  readonly user = this.currentUser.asReadonly();

  login(credentials: Credentials): Observable<User> {
    return this.http.post<User>('/api/login', credentials).pipe(
      tap(user => this.currentUser.set(user))
    );
  }
}
```

### Modules vs Standalone

**Legacy (NgModule-based):**

```typescript
@NgModule({
  declarations: [AppComponent, HeaderComponent],
  imports: [BrowserModule, RouterModule.forRoot(routes)],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

**Modern (Standalone — recommended since Angular 17):**

```typescript
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
  ],
});
```

---

## The Angular CLI

The Angular CLI (`@angular/cli`) is the official command-line tool for creating, building, testing, and deploying Angular applications.

### Essential Commands

| Command | Purpose |
|---------|---------|
| `ng new my-app` | Create a new standalone Angular project |
| `ng serve` | Start dev server with hot reload (default port 4200) |
| `ng generate component my-comp` | Generate a standalone component |
| `ng generate service my-service` | Generate a service with `providedIn: 'root'` |
| `ng build` | Build for production (AOT, tree shaking, minification) |
| `ng test` | Run unit tests (Karma/Jasmine or Jest) |
| `ng e2e` | Run end-to-end tests |
| `ng update` | Update Angular packages to latest version |
| `ng add @angular/material` | Add a library with automatic configuration |

### Project Structure (Standalone Default)

```
my-app/
├── src/
│   ├── app/
│   │   ├── app.component.ts       # Root component
│   │   ├── app.component.html     # Root template
│   │   ├── app.component.css      # Root styles
│   │   ├── app.component.spec.ts  # Root tests
│   │   ├── app.config.ts          # Application configuration
│   │   └── app.routes.ts          # Route definitions
│   ├── assets/                    # Static assets
│   ├── environments/              # Environment configs
│   ├── index.html                 # Host page
│   ├── main.ts                    # Bootstrap entry point
│   └── styles.css                 # Global styles
├── angular.json                   # CLI workspace configuration
├── package.json                   # Dependencies
├── tsconfig.json                  # TypeScript configuration
└── tsconfig.app.json              # App-specific TS config
```

---

## Angular Versioning and Release Schedule

Angular follows a predictable **6-month major release cycle** with semantic versioning:

| Release Type | Frequency | Example | Scope |
|-------------|-----------|---------|-------|
| **Major** | Every 6 months | v17 → v18 → v19 | New features, deprecations, breaking changes |
| **Minor** | 1–3 per major | v18.1, v18.2 | New features, no breaking changes |
| **Patch** | Weekly | v18.2.1 | Bug fixes, security patches |

### Support Policy

| Version | Active Support | Long-Term Support (LTS) |
|---------|---------------|------------------------|
| Current (e.g., v19) | 6 months | 12 months after next major |
| LTS (e.g., v18) | — | 18 months total from release |

---

## Standalone Components

Standalone components are the **modern default since Angular 17**. They do not require an NgModule and import their dependencies directly:

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-nav',
  standalone: true,
  imports: [CommonModule, RouterLink],
  template: `
    <nav>
      <a routerLink="/">Home</a>
      <a routerLink="/about">About</a>
    </nav>
  `,
})
export class NavComponent {}
```

### Benefits of Standalone

- **Simpler mental model** — no NgModule boilerplate
- **Better tree shaking** — only imported dependencies are bundled
- **Easier lazy loading** — `loadComponent` in routes loads a single component
- **Gradual migration** — standalone and NgModule-based components can coexist

---

## Angular Signals

Signals are a **synchronous, fine-grained reactive primitive** introduced in Angular 16. They represent a value that changes over time and notify consumers when updated.

### Core Signal APIs

```typescript
import { signal, computed, effect } from '@angular/core';

// Writable signal
const count = signal(0);
console.log(count());  // Read: 0

count.set(5);          // Set new value
count.update(v => v + 1);  // Update based on current value

// Computed signal (derived, read-only)
const doubled = computed(() => count() * 2);
console.log(doubled()); // 12

// Effect (side effect that runs when signals change)
effect(() => {
  console.log(`Count is now: ${count()}`);
});
```

### Signals vs RxJS

| Aspect | Signals | RxJS Observables |
|--------|---------|-----------------|
| **Synchronous** | Yes — always has a current value | No — values arrive over time |
| **Subscription** | Automatic (read in template/computed/effect) | Manual (subscribe, async pipe) |
| **Use case** | UI state, template binding | Async streams, HTTP, events |
| **Memory leaks** | No (auto-tracked) | Possible (forgotten subscriptions) |
| **Interop** | `toSignal()` / `toObservable()` | `toSignal()` / `toObservable()` |

Signals and RxJS are **complementary**. Use signals for synchronous state and template binding. Use RxJS for asynchronous streams, complex event composition, and HTTP calls.

---

## Prerequisites

Before learning Angular, you should be comfortable with:

| Prerequisite | Level Needed | Why |
|-------------|-------------|-----|
| **TypeScript** | Intermediate | Angular is built entirely in TypeScript — classes, decorators, generics, interfaces |
| **HTML** | Solid | Angular templates are HTML with additional binding syntax |
| **CSS** | Solid | Component styling, ViewEncapsulation, responsive design |
| **Node.js & npm** | Basic | Angular CLI runs on Node.js; npm manages packages |
| **ES6+ JavaScript** | Solid | Destructuring, arrow functions, modules, promises, async/await |
| **Git** | Basic | Version control for any real project |

### Recommended Prior Reading

If you are new to the frontend stack, review these documents first:

| Document | What You Will Learn |
|----------|-------------------|
| [Frontend Overview](../00-OVERVIEW.md) | Frontend ecosystem, rendering models, browser pipeline |
| [JavaScript Fundamentals](../02-JAVASCRIPT-FUNDAMENTALS.md) | ES6+, closures, async/await, modules |
| [TypeScript](../03-TYPESCRIPT.md) | Type system, generics, decorators |

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-COMPONENTS-AND-TEMPLATES](01-COMPONENTS-AND-TEMPLATES.md) | Component architecture, templates, new control flow |
| 2 | [02-SERVICES-AND-DEPENDENCY-INJECTION](02-SERVICES-AND-DEPENDENCY-INJECTION.md) | Services, DI, providers |
| 3 | [LEARNING-PATH](LEARNING-PATH.md) | Structured 10–14 week curriculum |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Angular overview documentation |
