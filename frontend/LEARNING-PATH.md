# Frontend Development Learning Path

A structured, self-paced training guide to mastering modern frontend development — from HTML, CSS, and JavaScript foundations through frameworks, state management, testing, performance optimization, and production-ready patterns. Each phase builds on the previous one, progressing from core concepts to advanced production skills.

> **Time Estimate:** 12–16 weeks at ~5 hours/week. Adjust pace to your experience level. Developers with existing HTML/CSS/JS knowledge may skip or accelerate Phase 1.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how frontend concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all seven phases and produces a portfolio artifact

---

## Phase 1: Foundations (Weeks 1–3)

### Learning Objectives

- Write semantic, accessible HTML5 documents
- Build layouts with CSS Flexbox and Grid
- Understand the CSS Box Model, specificity, and the cascade
- Apply responsive design principles with a mobile-first approach
- Use CSS Custom Properties for theming

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Frontend ecosystem, browser rendering pipeline, rendering strategies |
| 2 | [01-HTML-CSS](01-HTML-CSS.md) | Semantic HTML5, CSS Box Model, Flexbox, Grid, modern CSS |
| 3 | [04-RESPONSIVE-DESIGN](04-RESPONSIVE-DESIGN.md) | Mobile-first, media queries, fluid typography, responsive images |

### Exercises

**1. Semantic HTML Page:**

Build a blog article page using only semantic HTML (no `<div>` except where no semantic element applies):

- Use `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, `<footer>`
- Include a navigation bar, article with headings (h1–h3), figures with captions, and a sidebar
- Validate with the W3C HTML Validator
- Test with a screen reader (VoiceOver on macOS, NVDA on Windows)

**2. CSS Layout Challenge:**

Create a responsive dashboard layout with:

- A fixed sidebar navigation (collapses to bottom nav on mobile)
- A header with a search bar and user avatar
- A main content area with a responsive card grid (`repeat(auto-fill, minmax(280px, 1fr))`)
- Use CSS Grid for the overall layout, Flexbox for component internals
- Mobile-first: start with a single-column layout, add complexity at `768px` and `1024px`
- Use CSS Custom Properties for all colors and spacing

**3. Typography and Theming:**

Create a dark/light theme toggle using only CSS Custom Properties:

```css
:root {
  --bg: #ffffff;
  --text: #1a1a1a;
  --primary: #3b82f6;
}

[data-theme="dark"] {
  --bg: #1a1a1a;
  --text: #f0f0f0;
  --primary: #60a5fa;
}
```

- Implement fluid typography using `clamp()`
- Include a button that toggles the `data-theme` attribute with JavaScript

### Knowledge Check

- [ ] Can you explain the difference between `display: flex` and `display: grid` and when to use each?
- [ ] Can you describe the CSS specificity of `.card .title` vs `#card .title` vs `div.card .title`?
- [ ] Can you build a responsive layout without any media queries using CSS Grid `auto-fill`?
- [ ] Can you name 5 semantic HTML elements and explain when to use each?

---

## Phase 2: Modern JavaScript & TypeScript (Weeks 4–5)

### Learning Objectives

- Use ES6+ features fluently: destructuring, spread, arrow functions, template literals
- Understand closures, the event loop, and prototypal inheritance
- Write asynchronous code with Promises and async/await
- Understand ES Modules and dynamic imports
- Write basic TypeScript with interfaces, types, and generics

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-JAVASCRIPT-FUNDAMENTALS](02-JAVASCRIPT-FUNDAMENTALS.md) | ES6+, closures, event loop, async/await, DOM APIs |
| 2 | [03-TYPESCRIPT](03-TYPESCRIPT.md) | Type system, interfaces, generics, utility types |

### Exercises

**1. JavaScript Fundamentals Kata:**

Write solutions (with tests) for:

- A `debounce(fn, delay)` function using closures
- A `deepClone(obj)` function using recursion (handle objects, arrays, dates, and null)
- A `retry(fn, maxAttempts, delay)` function using async/await
- A simple `EventEmitter` class with `on(event, handler)`, `off(event, handler)`, and `emit(event, ...args)`

**2. TypeScript Migration:**

Take the JavaScript solutions from Exercise 1 and convert them to TypeScript:

- Add proper types for all function parameters and return values
- Use generics where appropriate (e.g., `deepClone<T>(obj: T): T`)
- Enable `strict: true` and fix all type errors
- Create a discriminated union type for `AsyncState<T>` with loading, success, and error variants

**3. Async Data Fetching:**

Build a script that:

- Fetches data from a public API (e.g., JSONPlaceholder)
- Implements error handling with a typed `ApiError` class
- Uses `Promise.all` to fetch users and their posts in parallel
- Implements request cancellation with `AbortController`

### Knowledge Check

- [ ] Can you explain what a closure is and give a practical example?
- [ ] Can you describe the event loop and predict the output of a `console.log` / `setTimeout` / `Promise` sequence?
- [ ] Can you write a generic TypeScript function with a constraint?
- [ ] Can you explain the difference between `interface` and `type` in TypeScript?

---

## Phase 3: Framework Fundamentals (Weeks 6–8)

### Learning Objectives

- Understand component-based architecture
- Build components with props, state, and lifecycle hooks
- Handle user events and form inputs
- Implement client-side routing
- Choose a framework (Angular, React, or Vue) and build a small application

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Architecture patterns, component-based architecture |
| 2 | Framework documentation | Angular: angular.dev, React: react.dev, Vue: vuejs.org |

### Exercises

**1. Component Library:**

Build a set of reusable components in your chosen framework:

- `Button` — variants (primary, secondary, danger), sizes (sm, md, lg), loading state
- `Input` — label, error message, helper text, disabled state
- `Card` — composable sub-components (Header, Body, Footer)
- `Modal` — opens/closes, focus trapping, closes on Escape

**2. Todo Application:**

Build a full-featured todo application:

- Add, complete, delete todos
- Filter by status (all, active, completed)
- Persist to `localStorage`
- Client-side routing: `/` for all, `/active`, `/completed`
- TypeScript throughout

**3. API Integration:**

Extend the todo app to sync with a REST API:

- Fetch todos from the API on load
- Create, update, and delete via API calls
- Show loading states during API calls
- Handle errors gracefully with user-friendly messages

### Knowledge Check

- [ ] Can you explain the component lifecycle in your chosen framework?
- [ ] Can you pass data from parent to child and from child to parent?
- [ ] Can you implement client-side routing with lazy-loaded routes?
- [ ] Can you handle form input binding and validation?

---

## Phase 4: State Management & Routing (Weeks 9–10)

### Learning Objectives

- Understand when to use local vs global state
- Implement the Redux/Flux pattern (or framework equivalent)
- Manage server state with TanStack Query or SWR
- Use URL state for shareable application state
- Understand reactive state with signals

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-STATE-MANAGEMENT](05-STATE-MANAGEMENT.md) | All sections |

### Exercises

**1. State Management Refactor:**

Take the todo app from Phase 3 and refactor the state management:

- Extract server state into TanStack Query (React) / similar (Angular/Vue)
- Keep UI state (filter, modal open) as local component state
- Implement optimistic updates for todo creation and completion
- Add URL-based filtering (`?status=active&sort=date`)

**2. Complex State Machine:**

Build a multi-step form wizard (e.g., checkout flow) using XState or a similar state machine:

- Steps: Shipping → Payment → Review → Confirmation
- Validate each step before allowing progression
- Allow going back to previous steps without losing data
- Handle the submission flow: idle → submitting → success/error

### Knowledge Check

- [ ] Can you explain the difference between client state and server state?
- [ ] Can you describe when to use Context vs a global store vs TanStack Query?
- [ ] Can you implement URL-based state that is shareable and bookmarkable?
- [ ] Can you model a complex flow with a state machine?

---

## Phase 5: Testing & Quality (Weeks 11–12)

### Learning Objectives

- Write unit tests for utility functions and state logic
- Write component tests that verify behavior from the user's perspective
- Write E2E tests for critical user journeys
- Mock API calls with MSW
- Integrate accessibility testing into the test suite

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-TESTING](07-TESTING.md) | All sections |
| 2 | [09-ACCESSIBILITY](09-ACCESSIBILITY.md) | Testing tools and workflows |

### Exercises

**1. Unit Testing:**

Write tests for the utility functions from Phase 2 (debounce, deepClone, retry, EventEmitter):

- Achieve 100% branch coverage
- Use `vi.useFakeTimers()` for time-dependent tests
- Test error cases and edge cases

**2. Component Testing:**

Write component tests for the component library from Phase 3:

- Test rendering with different props
- Test user interactions (click, type, keyboard navigation)
- Test accessibility with jest-axe
- Use Testing Library queries by role and text (not test IDs)

**3. E2E Testing:**

Write Playwright E2E tests for the todo application:

- Test the full CRUD flow (create, read, update, delete)
- Test filtering and URL state
- Test error handling (mock API failures)
- Add an accessibility scan to each test

### Knowledge Check

- [ ] Can you write a component test that queries by role instead of test ID?
- [ ] Can you mock an API endpoint with MSW?
- [ ] Can you write a Playwright test that verifies a multi-step user flow?
- [ ] Can you explain what code coverage metrics mean and their limitations?

---

## Phase 6: Performance & Optimization (Weeks 13–14)

### Learning Objectives

- Measure and improve Core Web Vitals (LCP, INP, CLS)
- Implement code splitting and lazy loading
- Optimize images and fonts
- Configure caching strategies
- Analyze and reduce bundle size

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-PERFORMANCE](06-PERFORMANCE.md) | All sections |
| 2 | [08-BUILD-TOOLS](08-BUILD-TOOLS.md) | Bundler configuration, code splitting |

### Exercises

**1. Performance Audit:**

Run a Lighthouse audit on your todo application and fix all issues:

- Achieve a Lighthouse performance score ≥ 90
- Ensure all Core Web Vitals are in the "Good" range
- Document every optimization you make and its impact

**2. Bundle Optimization:**

- Add a bundle analyzer to your build
- Identify the largest dependencies
- Implement route-based code splitting
- Set a performance budget and add it to CI (fail the build if exceeded)

**3. Image Optimization:**

Create a photo gallery page that demonstrates:

- Responsive images with `srcset` and `sizes`
- Lazy loading with `loading="lazy"`
- Modern formats (WebP/AVIF) with `<picture>` fallback
- Proper `width`/`height` attributes to prevent CLS

### Knowledge Check

- [ ] Can you explain what LCP, INP, and CLS measure?
- [ ] Can you implement route-based code splitting in your framework?
- [ ] Can you configure cache headers for different asset types?
- [ ] Can you use Chrome DevTools Performance tab to identify a long task?

---

## Phase 7: Advanced Topics & Production (Weeks 15–16)

### Learning Objectives

- Apply frontend best practices and avoid anti-patterns
- Configure build tools for production
- Implement security measures (CSP, XSS prevention)
- Set up a design system with documented components
- Prepare an application for production deployment

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | All sections, especially the checklist |
| 2 | [11-ANTI-PATTERNS](11-ANTI-PATTERNS.md) | All sections |

### Exercises

**1. Production Readiness Review:**

Audit your todo application against the [10-BEST-PRACTICES checklist](10-BEST-PRACTICES.md#checklist):

- Fix every failing item
- Document decisions and trade-offs

**2. Design System:**

Extract the component library from Phase 3 into a documented design system:

- Define design tokens (colors, spacing, typography) as CSS Custom Properties
- Document each component with Storybook
- Add visual regression tests with Playwright screenshots
- Include accessibility documentation for each component

**3. Capstone: Full Application:**

Build a complete application (e.g., project management board, recipe app, or bookmarks manager) that demonstrates:

- Semantic, accessible HTML with proper ARIA where needed
- Responsive, mobile-first CSS with a design token system
- TypeScript throughout with strict mode
- Framework components with proper state management
- Server state management with TanStack Query or equivalent
- Unit, component, and E2E tests
- Performance optimizations (code splitting, lazy loading, image optimization)
- Error boundaries and consistent error handling
- Environment-based configuration
- CI pipeline with lint, test, build, and bundle size check

### Knowledge Check

- [ ] Can you identify and fix 5 anti-patterns in a code review?
- [ ] Can you set up a complete CI pipeline for a frontend application?
- [ ] Can you explain your state management choices and justify them?
- [ ] Can you walk through the pre-launch checklist and verify every item?

---

## Recommended Learning Resources

| Resource | Type | Focus |
|----------|------|-------|
| [MDN Web Docs](https://developer.mozilla.org) | Reference | HTML, CSS, JavaScript, Web APIs |
| [web.dev](https://web.dev) | Guides | Performance, accessibility, best practices |
| [Angular.dev](https://angular.dev) | Docs | Angular framework |
| [React.dev](https://react.dev) | Docs | React framework |
| [Vue.js Guide](https://vuejs.org/guide) | Docs | Vue framework |
| [TypeScript Handbook](https://www.typescriptlang.org/docs) | Docs | TypeScript language |
| [Testing Library](https://testing-library.com) | Docs | Component testing |
| [Playwright](https://playwright.dev) | Docs | E2E testing |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial frontend development learning path |
