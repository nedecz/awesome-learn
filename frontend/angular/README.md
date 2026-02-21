# Angular Learning Resources

A comprehensive guide to Angular framework development — covering Angular 17/18/19 with signals, standalone components, new control flow syntax, and modern reactive patterns. From core concepts through production-ready best practices.

> This is part of the [Frontend Development](../README.md) topic.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | What is Angular, history, architecture, when to choose Angular | **Start here** |
| [01-COMPONENTS-AND-TEMPLATES](01-COMPONENTS-AND-TEMPLATES.md) | Component architecture, templates, new control flow, signals I/O | When building UI components |
| [02-SERVICES-AND-DEPENDENCY-INJECTION](02-SERVICES-AND-DEPENDENCY-INJECTION.md) | DI system, providers, injection tokens, service patterns | When structuring business logic |
| [03-ROUTING-AND-NAVIGATION](03-ROUTING-AND-NAVIGATION.md) | Router config, lazy loading, guards, resolvers | When adding navigation |
| [04-FORMS](04-FORMS.md) | Template-driven vs reactive forms, typed forms, validation | When building forms |
| [05-RXJS-AND-OBSERVABLES](05-RXJS-AND-OBSERVABLES.md) | RxJS operators, subjects, async pipe, signals interop | When working with async data |
| [06-STATE-MANAGEMENT](06-STATE-MANAGEMENT.md) | Service state, signal state, NgRx, NGXS, Akita | When managing application state |
| [07-HTTP-AND-API-INTEGRATION](07-HTTP-AND-API-INTEGRATION.md) | HttpClient, interceptors, error handling, caching | When calling APIs |
| [08-TESTING](08-TESTING.md) | Unit, integration, E2E testing, TestBed, component harnesses | When writing tests |
| [09-PERFORMANCE](09-PERFORMANCE.md) | Change detection, lazy loading, SSR, hydration, profiling | When optimizing speed |
| [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Project structure, style guide, security, i18n, checklist | **Essential — production checklist** |
| [11-ANTI-PATTERNS](11-ANTI-PATTERNS.md) | Common Angular mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured 10–14 week guide with exercises | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand what Angular is and how it compares to other frameworks
   - Learn the Angular CLI and project structure
   - Review prerequisites (TypeScript, HTML, CSS, Node.js)

2. **Learn Components & Templates** ([01-COMPONENTS-AND-TEMPLATES](01-COMPONENTS-AND-TEMPLATES.md))
   - Understand component architecture and lifecycle hooks
   - Master template syntax and the new control flow (@if, @for, @switch)
   - Build your first Angular components

3. **Master Services & Routing** ([02-SERVICES-AND-DEPENDENCY-INJECTION](02-SERVICES-AND-DEPENDENCY-INJECTION.md) → [03-ROUTING-AND-NAVIGATION](03-ROUTING-AND-NAVIGATION.md))
   - Understand Angular's dependency injection system
   - Set up routing with lazy-loaded standalone components

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured curriculum with hands-on exercises
   - Progressive skill building from foundations to production

### For Experienced Developers

1. **Review Best Practices** ([10-BEST-PRACTICES](10-BEST-PRACTICES.md))
   - Modern Angular patterns: standalone components, signals, functional APIs
   - Production-ready project structure and security

2. **Avoid Anti-Patterns** ([11-ANTI-PATTERNS](11-ANTI-PATTERNS.md))
   - Common Angular mistakes across change detection, subscriptions, architecture
   - Code examples with problems and solutions

3. **Optimize Performance** ([09-PERFORMANCE](09-PERFORMANCE.md))
   - Signal-based change detection and zoneless Angular
   - SSR with @angular/ssr, hydration, @defer blocks

4. **Master State & Reactivity** ([05-RXJS-AND-OBSERVABLES](05-RXJS-AND-OBSERVABLES.md) → [06-STATE-MANAGEMENT](06-STATE-MANAGEMENT.md))
   - RxJS and signals interop (toSignal, toObservable)
   - NgRx SignalStore and signal-based state management

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Angular Application                         │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │            Standalone Components / NgModules              │  │
│  │  (entry points — declare what the app is made of)        │  │
│  └──────────────────────┬────────────────────────────────────┘  │
│                         │                                       │
│  ┌──────────────────────▼────────────────────────────────────┐  │
│  │                   Components                              │  │
│  │  TypeScript class + HTML template + CSS styles            │  │
│  │  Inputs (signals) ← Parent    Outputs → Parent            │  │
│  └──────────────────────┬────────────────────────────────────┘  │
│                         │                                       │
│  ┌──────────────────────▼────────────────────────────────────┐  │
│  │                   Templates                               │  │
│  │  @if / @for / @switch / @defer    (control flow)          │  │
│  │  {{ interpolation }}  [binding]  (event)  [(two-way)]     │  │
│  └──────────────────────┬────────────────────────────────────┘  │
│                         │                                       │
│  ┌──────────────────────▼────────────────────────────────────┐  │
│  │                   Services                                │  │
│  │  Business logic, API calls, shared state                  │  │
│  │  @Injectable({ providedIn: 'root' })                      │  │
│  └──────────────────────┬────────────────────────────────────┘  │
│                         │                                       │
│  ┌──────────────────────▼────────────────────────────────────┐  │
│  │             Dependency Injection (DI)                      │  │
│  │  Hierarchical injectors: root → component → element       │  │
│  │  inject() function or constructor injection               │  │
│  └──────────────────────┬────────────────────────────────────┘  │
│                         │                                       │
│  ┌──────────────────────▼────────────────────────────────────┐  │
│  │              Change Detection                             │  │
│  │  Zone.js (default) or Zoneless (signal-based)             │  │
│  │  Default strategy vs OnPush strategy                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
Components
──────────
Component     → Building block of Angular UI — class + template + styles
Standalone    → Self-contained component (no NgModule required, default since Angular 17)
Lifecycle     → Hooks: ngOnInit, ngOnChanges, ngOnDestroy, afterNextRender
Signal Inputs → Reactive inputs using input() and output() functions

Modules & Standalone
────────────────────
NgModule      → Legacy way to organize related components, directives, pipes
Standalone    → Modern default — each component imports its own dependencies
Bootstrapping → bootstrapApplication() with standalone, or NgModule with platformBrowserDynamic

Services & DI
─────────────
Service       → Class decorated with @Injectable for business logic
DI            → Angular creates and provides instances automatically
inject()      → Modern function-based injection (preferred over constructor injection)
providedIn    → Tree-shakable service registration ('root', 'platform', 'any')

Reactive Programming
────────────────────
RxJS          → Library for composing asynchronous streams with Observables
Signals       → Synchronous reactive primitive (Angular 16+)
toSignal()    → Convert Observable to Signal for template binding
toObservable()→ Convert Signal to Observable for RxJS pipelines

Routing
───────
Router        → Maps URL paths to components
Lazy Loading  → Load routes on demand with loadComponent / loadChildren
Guards        → Functional guards: canActivate, canMatch, canDeactivate
Resolvers     → Pre-fetch data before route activation

Forms
─────
Reactive      → FormControl / FormGroup / FormArray — explicit, testable
Template      → ngModel-based — simpler for basic forms
Typed Forms   → Strongly typed reactive forms (Angular 14+)
```

## 📋 Topics Covered

- **Overview** — Angular history, architecture, CLI, standalone components, signals
- **Components & Templates** — Lifecycle hooks, template syntax, new control flow, content projection
- **Services & DI** — Injectable services, hierarchical injectors, injection tokens
- **Routing** — Route config, lazy loading, functional guards, resolvers
- **Forms** — Reactive forms, typed forms, validation, dynamic forms
- **RxJS & Observables** — Operators, subjects, async pipe, signals interop
- **State Management** — Service state, NgRx Store, NgRx SignalStore, NGXS
- **HTTP & APIs** — HttpClient, functional interceptors, caching, testing
- **Testing** — TestBed, component testing, service testing, E2E with Playwright
- **Performance** — OnPush, zoneless, @defer, SSR, hydration, profiling
- **Best Practices** — Project structure, style guide, security, i18n, checklist
- **Anti-Patterns** — Subscription leaks, improper change detection, DOM manipulation

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to Angular?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Know Angular basics?** → Jump to [05-RXJS-AND-OBSERVABLES.md](05-RXJS-AND-OBSERVABLES.md) or [06-STATE-MANAGEMENT.md](06-STATE-MANAGEMENT.md)

**Going to production?** → Review [10-BEST-PRACTICES.md](10-BEST-PRACTICES.md) and [11-ANTI-PATTERNS.md](11-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — progressive exercises from foundations to production
