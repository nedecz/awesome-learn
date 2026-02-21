# Frontend Anti-Patterns

A catalogue of the most common frontend development mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a code-review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Excessive Prop Drilling](#1-excessive-prop-drilling)
- [2. Over-Engineering State Management](#2-over-engineering-state-management)
- [3. Ignoring Accessibility from the Start](#3-ignoring-accessibility-from-the-start)
- [4. Not Optimizing Bundle Size](#4-not-optimizing-bundle-size)
- [5. Direct DOM Manipulation in Framework Apps](#5-direct-dom-manipulation-in-framework-apps)
- [6. Blocking the Main Thread](#6-blocking-the-main-thread)
- [7. Inconsistent Error Handling](#7-inconsistent-error-handling)
- [8. Hardcoding Environment-Specific Values](#8-hardcoding-environment-specific-values)
- [9. Ignoring Browser Caching](#9-ignoring-browser-caching)
- [10. Not Using Semantic HTML](#10-not-using-semantic-html)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Anti-patterns are recurring practices that seem reasonable at first glance but create significant problems over time. In frontend development, the consequences include poor performance, accessibility failures, maintenance nightmares, and security vulnerabilities — often at the exact moment your application needs to work flawlessly.

Each anti-pattern documented here is:

- **Seductive** — it felt like the right approach when first implemented
- **Harmful** — it creates real problems in performance, accessibility, maintainability, or user experience
- **Fixable** — there is a well-understood better approach

### How to Use This Document

1. **Code review:** Reference specific sections when reviewing frontend PRs
2. **Pre-production review:** Use the [Quick Reference Checklist](#quick-reference-checklist) before deploying
3. **Team onboarding:** Assign this document to new engineers joining the frontend team
4. **Periodic audit:** Review existing code against these patterns quarterly

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|-------------|----------|----------|
| 1 | Excessive Prop Drilling | Architecture | 🟠 High |
| 2 | Over-Engineering State Management | Architecture | 🟠 High |
| 3 | Ignoring Accessibility from the Start | Accessibility | 🔴 Critical |
| 4 | Not Optimizing Bundle Size | Performance | 🟠 High |
| 5 | Direct DOM Manipulation in Framework Apps | Architecture | 🟠 High |
| 6 | Blocking the Main Thread | Performance | 🔴 Critical |
| 7 | Inconsistent Error Handling | Reliability | 🟠 High |
| 8 | Hardcoding Environment-Specific Values | Maintainability | 🟡 Medium |
| 9 | Ignoring Browser Caching | Performance | 🟡 Medium |
| 10 | Not Using Semantic HTML | Accessibility / SEO | 🟠 High |

---

## 1. Excessive Prop Drilling

### Problem

Passing props through many intermediate components that do not use them, just to get data to a deeply nested child.

### Why It Happens

The component tree grows organically. A piece of state that started near the consumer gets pushed up as more components need it, but the intermediate components keep forwarding it.

### Impact

- Every intermediate component is coupled to data it does not use
- Renaming or restructuring props requires changes across many files
- Adding a new prop to a deeply nested component requires edits at every level

### Solution

Use a context or dependency injection mechanism appropriate to your framework.

```typescript
// ❌ Anti-pattern: drilling through 4 levels
function App() {
  const [user, setUser] = useState(currentUser);
  return <Layout user={user}><Sidebar user={user}><UserMenu user={user} /></Sidebar></Layout>;
}

// ✅ Solution: React Context
const UserContext = createContext<User | null>(null);

function App() {
  const [user, setUser] = useState(currentUser);
  return (
    <UserContext.Provider value={user}>
      <Layout><Sidebar><UserMenu /></Sidebar></Layout>
    </UserContext.Provider>
  );
}

function UserMenu() {
  const user = useContext(UserContext); // Direct access, no drilling
  return <div>{user?.name}</div>;
}
```

---

## 2. Over-Engineering State Management

### Problem

Reaching for Redux, NgRx, or another heavy state management library for state that could be handled with local component state or a simple context.

### Why It Happens

Developers adopt a state management library early (often because a tutorial used it) and route all state through it — even form inputs, modal visibility, and local UI toggles.

### Impact

- Massive boilerplate for simple operations (action → reducer → selector for a boolean toggle)
- Slower development velocity
- Harder onboarding for new team members
- Larger bundle size

### Solution

Use the simplest approach that meets your needs.

```typescript
// ❌ Anti-pattern: Redux for a modal toggle
// actions.ts
const TOGGLE_MODAL = "TOGGLE_MODAL";
// reducer.ts
case TOGGLE_MODAL: return { ...state, isModalOpen: !state.isModalOpen };
// selector.ts
const selectIsModalOpen = (state) => state.ui.isModalOpen;
// component.tsx
const isOpen = useSelector(selectIsModalOpen);
const dispatch = useDispatch();
dispatch({ type: TOGGLE_MODAL });

// ✅ Solution: local component state
function SettingsPage() {
  const [isModalOpen, setIsModalOpen] = useState(false);
  return (
    <>
      <button onClick={() => setIsModalOpen(true)}>Open Settings</button>
      {isModalOpen && <SettingsModal onClose={() => setIsModalOpen(false)} />}
    </>
  );
}
```

**Rule of thumb:** If the state is only used by one component or its direct children, keep it local.

---

## 3. Ignoring Accessibility from the Start

### Problem

Treating accessibility as a last-minute add-on or a "nice to have" rather than building it in from the beginning.

### Why It Happens

Accessibility is not visible in demos or screenshots. Deadlines pressure teams to ship visual features first and "fix accessibility later."

### Impact

- Retrofitting accessibility is 3–10× more expensive than building it in
- Legal risk — accessibility lawsuits are increasing year over year
- Excludes users with disabilities (~15% of the global population)
- Poor SEO (search engines rely on semantic markup)

### Solution

Make accessibility part of the development workflow from day one.

```html
<!-- ❌ Anti-pattern: div soup with no semantics -->
<div class="nav">
  <div class="nav-item" onclick="navigate('/')">Home</div>
  <div class="nav-item" onclick="navigate('/about')">About</div>
</div>

<!-- ✅ Solution: semantic HTML with keyboard support -->
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>
```

See [09-ACCESSIBILITY.md](09-ACCESSIBILITY.md) for comprehensive guidance.

---

## 4. Not Optimizing Bundle Size

### Problem

Shipping large JavaScript bundles that slow down initial page load, especially on mobile devices and slow networks.

### Why It Happens

Developers install libraries without considering their size. Entire utility libraries are imported when only one function is needed. No bundle analysis is performed.

### Impact

- Slow Time to Interactive (TTI), especially on mobile
- Higher bounce rates
- Poor Core Web Vitals scores
- More data usage for users on metered connections

### Solution

```javascript
// ❌ Anti-pattern: import the entire library (70 KB gzipped)
import _ from "lodash";
const result = _.debounce(handler, 300);

// ✅ Solution: import only what you need (< 1 KB)
import debounce from "lodash/debounce";
const result = debounce(handler, 300);

// ✅ Even better: use a lightweight alternative or write your own
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

**Prevention:** Add bundle analysis to your CI pipeline and set size budgets.

---

## 5. Direct DOM Manipulation in Framework Apps

### Problem

Using `document.querySelector`, `element.innerHTML`, or jQuery alongside a framework (React, Angular, Vue) that manages the DOM.

### Why It Happens

Developers coming from jQuery or vanilla JavaScript habits continue using direct DOM APIs inside framework components.

### Impact

- Framework and manual DOM updates conflict, causing bugs and stale UI
- Breaks framework assumptions about DOM state (virtual DOM diffing, change detection)
- Memory leaks from event listeners that the framework does not clean up
- Harder to test — tests cannot interact with manually created DOM

### Solution

Use the framework's APIs for all DOM interactions.

```typescript
// ❌ Anti-pattern in React
function BadComponent() {
  useEffect(() => {
    document.getElementById("title")!.textContent = "Updated!";
  }, []);
  return <h1 id="title">Original</h1>;
}

// ✅ Solution: use React state
function GoodComponent() {
  const [title, setTitle] = useState("Original");
  useEffect(() => {
    setTitle("Updated!");
  }, []);
  return <h1>{title}</h1>;
}
```

```typescript
// ❌ Anti-pattern in Angular
@Component({ template: `<h1 id="title">Original</h1>` })
class BadComponent implements OnInit {
  ngOnInit() {
    document.getElementById("title")!.textContent = "Updated!";
  }
}

// ✅ Solution: use Angular data binding
@Component({ template: `<h1>{{ title }}</h1>` })
class GoodComponent {
  title = "Updated!";
}
```

---

## 6. Blocking the Main Thread

### Problem

Running heavy computation or synchronous operations on the main thread, freezing the UI and making the application unresponsive.

### Why It Happens

JavaScript is single-threaded. Developers write loops, sorting algorithms, or data transformations inline without considering their execution time.

### Impact

- UI freezes — buttons don't respond, animations stutter
- Poor INP (Interaction to Next Paint) scores
- Users perceive the app as broken
- Mobile devices with slower CPUs suffer most

### Solution

```javascript
// ❌ Anti-pattern: synchronous processing of large data on main thread
function processLargeDataset(data) {
  return data
    .filter((item) => item.isActive)
    .sort((a, b) => b.score - a.score)
    .map((item) => computeExpensiveMetric(item)); // Takes 500ms
}

// ✅ Solution 1: use a Web Worker
const worker = new Worker(new URL("./processor.worker.js", import.meta.url));
worker.postMessage({ data });
worker.onmessage = (event) => renderResults(event.data);

// ✅ Solution 2: break into chunks with scheduler
async function processInChunks(data, chunkSize = 100) {
  const results = [];
  for (let i = 0; i < data.length; i += chunkSize) {
    const chunk = data.slice(i, i + chunkSize);
    results.push(...chunk.map(computeExpensiveMetric));
    // Yield to the main thread between chunks
    await new Promise((resolve) => setTimeout(resolve, 0));
  }
  return results;
}
```

---

## 7. Inconsistent Error Handling

### Problem

Some API calls have `try/catch`, others swallow errors silently, some show error messages, others show blank screens. No consistent pattern.

### Why It Happens

Error handling is added reactively — only after a bug report. Each developer implements it differently.

### Impact

- Users see blank screens or broken UI with no explanation
- Errors are not logged, making debugging impossible
- Inconsistent user experience erodes trust

### Solution

Establish a centralized error handling strategy.

```typescript
// ❌ Anti-pattern: inconsistent handling
async function loadUserA() {
  const res = await fetch("/api/user"); // No error handling at all
  return res.json();
}

async function loadUserB() {
  try {
    const res = await fetch("/api/user");
    return res.json();
  } catch (e) {
    console.log(e); // Swallowed silently
  }
}

// ✅ Solution: centralized API client with consistent error handling
async function apiClient<T>(url: string, options?: RequestInit): Promise<T> {
  const response = await fetch(url, options);

  if (!response.ok) {
    const error = new ApiError(response.status, await response.text());
    reportError(error); // Always log to monitoring service
    throw error;
  }

  return response.json();
}

// Components use the client and handle the UI side consistently
function UserProfile({ userId }) {
  const { data, error, isLoading } = useQuery({
    queryKey: ["user", userId],
    queryFn: () => apiClient(`/api/users/${userId}`),
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  return <UserCard user={data} />;
}
```

---

## 8. Hardcoding Environment-Specific Values

### Problem

Embedding API URLs, feature flags, or configuration values directly in source code instead of using environment variables.

### Why It Happens

It is faster to hardcode a value during development than to set up proper configuration.

### Impact

- Deploying to staging or production requires code changes
- Secrets may end up in version control
- Feature flags cannot be toggled without redeployment

### Solution

```typescript
// ❌ Anti-pattern: hardcoded values
const API_URL = "https://api.production.example.com";
const STRIPE_KEY = "pk_live_abc123";

// ✅ Solution: environment variables
const API_URL = import.meta.env.VITE_API_URL;
const STRIPE_KEY = import.meta.env.VITE_STRIPE_PUBLIC_KEY;
```

```bash
# .env.development
VITE_API_URL=http://localhost:8080

# .env.production
VITE_API_URL=https://api.production.example.com
```

---

## 9. Ignoring Browser Caching

### Problem

Serving assets without proper cache headers, so browsers re-download unchanged files on every visit.

### Why It Happens

Caching configuration is often left to "the infrastructure team" and forgotten until a performance audit.

### Impact

- Slower repeat visits
- Higher bandwidth costs
- Unnecessary server load
- Poor user experience on slow connections

### Solution

| Asset Type | Caching Strategy |
|-----------|-----------------|
| `index.html` | `Cache-Control: no-cache` (always revalidate) |
| Hashed JS/CSS (`app-3a7f2b.js`) | `Cache-Control: public, max-age=31536000, immutable` |
| Images with hash | `Cache-Control: public, max-age=31536000, immutable` |
| API responses | `Cache-Control: max-age=60, stale-while-revalidate=300` |

Ensure your build tool produces content-hashed filenames so that cached files are automatically invalidated when content changes.

---

## 10. Not Using Semantic HTML

### Problem

Building entire page layouts with `<div>` and `<span>` elements, ignoring semantic HTML elements that convey meaning.

### Why It Happens

Developers learn `<div>` first and use it for everything. Many tutorials and framework examples use `<div>` by default.

### Impact

- Screen readers cannot navigate the page structure
- Search engines cannot understand content hierarchy
- Keyboard navigation breaks (no focusable elements without `tabindex`)
- Worse SEO rankings

### Solution

```html
<!-- ❌ Anti-pattern: div soup -->
<div class="header">
  <div class="nav">
    <div class="nav-item" onclick="go('/')">Home</div>
  </div>
</div>
<div class="content">
  <div class="title">Page Title</div>
  <div class="text">Some content...</div>
</div>

<!-- ✅ Solution: semantic HTML -->
<header>
  <nav aria-label="Main">
    <ul>
      <li><a href="/">Home</a></li>
    </ul>
  </nav>
</header>
<main>
  <h1>Page Title</h1>
  <p>Some content...</p>
</main>
```

---

## Quick Reference Checklist

| # | Check | Command / Action |
|---|-------|-----------------|
| 1 | No prop drilling deeper than 2 levels | Review component tree |
| 2 | State management matches complexity | Audit store usage |
| 3 | All interactive elements are accessible | Run axe audit, keyboard test |
| 4 | Bundle size within budget | `npx vite-bundle-visualizer` |
| 5 | No direct DOM manipulation in framework code | Search for `document.querySelector`, `innerHTML` |
| 6 | No long tasks on main thread | Chrome DevTools Performance tab |
| 7 | All API calls have error handling | Search for unhandled `fetch` / `http` calls |
| 8 | No hardcoded URLs or secrets | Search for `http://`, `https://`, API keys in source |
| 9 | Cache headers configured | Check response headers in DevTools Network tab |
| 10 | Semantic HTML used throughout | Validate with HTMLHint, axe, Lighthouse |

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Positive patterns to follow |
| 2 | [06-PERFORMANCE](06-PERFORMANCE.md) | Performance optimization in depth |
| 3 | [09-ACCESSIBILITY](09-ACCESSIBILITY.md) | Accessibility in depth |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial frontend anti-patterns documentation |
