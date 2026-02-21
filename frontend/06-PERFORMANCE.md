# Frontend Performance Optimization

## Table of Contents

1. [Overview](#overview)
2. [Core Web Vitals](#core-web-vitals)
3. [Code Splitting and Lazy Loading](#code-splitting-and-lazy-loading)
4. [Tree Shaking and Dead Code Elimination](#tree-shaking-and-dead-code-elimination)
5. [Image Optimization](#image-optimization)
6. [Caching Strategies](#caching-strategies)
7. [Critical Rendering Path Optimization](#critical-rendering-path-optimization)
8. [Bundle Analysis and Size Budgets](#bundle-analysis-and-size-budgets)
9. [Runtime Performance](#runtime-performance)
10. [Web Workers for Heavy Computation](#web-workers-for-heavy-computation)
11. [Performance Monitoring and Profiling](#performance-monitoring-and-profiling)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Frontend performance directly impacts user experience, conversion rates, and search engine rankings. Google uses Core Web Vitals as a ranking signal, and studies consistently show that users abandon sites that take more than 3 seconds to load.

This document covers the techniques, tools, and strategies for optimizing both load-time and runtime performance of frontend applications.

### Scope

- Core Web Vitals: LCP, INP, CLS
- Code splitting, lazy loading, and tree shaking
- Image optimization: formats, lazy loading, responsive images
- Caching: HTTP caching, service workers
- Critical rendering path optimization
- Bundle analysis and size budgets
- Runtime performance: avoiding layout thrashing, `requestAnimationFrame`
- Web Workers for offloading heavy computation
- Monitoring and profiling tools

---

## Core Web Vitals

Core Web Vitals are Google's metrics for measuring real-world user experience. They are a ranking signal for search results.

### The Three Metrics

| Metric | Full Name | Measures | Good | Needs Improvement | Poor |
|--------|-----------|----------|------|-------------------|------|
| **LCP** | Largest Contentful Paint | Loading speed — when the largest visible element renders | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **INP** | Interaction to Next Paint | Responsiveness — delay between user input and visual update | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS** | Cumulative Layout Shift | Visual stability — how much the page layout shifts unexpectedly | ≤ 0.1 | ≤ 0.25 | > 0.25 |

### Improving Each Metric

**LCP — Largest Contentful Paint:**
- Optimize the critical rendering path (preload key resources)
- Use `<link rel="preload">` for hero images and fonts
- Server-side render above-the-fold content
- Use a CDN for static assets
- Compress images and use modern formats (AVIF, WebP)

**INP — Interaction to Next Paint:**
- Break long tasks (> 50ms) into smaller chunks
- Use `requestAnimationFrame` for visual updates
- Defer non-critical JavaScript with `defer` or dynamic `import()`
- Avoid blocking the main thread with heavy computation (use Web Workers)
- Minimize DOM size — smaller DOM means faster updates

**CLS — Cumulative Layout Shift:**
- Always set `width` and `height` on images and videos
- Reserve space for dynamic content (ads, embeds, lazy-loaded content)
- Avoid inserting content above existing content
- Use `font-display: swap` with proper font fallback sizing
- Use CSS `contain` to limit layout recalculation scope

---

## Code Splitting and Lazy Loading

Code splitting breaks your JavaScript bundle into smaller chunks that are loaded on demand, reducing initial load time.

### Route-Based Splitting

```javascript
// React: lazy loading routes
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));
const Analytics = lazy(() => import("./pages/Analytics"));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}
```

```typescript
// Angular: lazy loading routes
const routes: Routes = [
  {
    path: "dashboard",
    loadComponent: () =>
      import("./pages/dashboard.component").then((m) => m.DashboardComponent),
  },
  {
    path: "settings",
    loadChildren: () =>
      import("./pages/settings/settings.routes").then((m) => m.SETTINGS_ROUTES),
  },
];
```

### Component-Level Splitting

```javascript
// Load a heavy component only when needed
const HeavyChart = lazy(() => import("./components/HeavyChart"));

function AnalyticsPanel({ showChart }) {
  return (
    <div>
      <h2>Analytics</h2>
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
```

---

## Tree Shaking and Dead Code Elimination

Tree shaking removes unused exports from your final bundle. It relies on ES module static `import`/`export` syntax.

### How It Works

```javascript
// math.js — exports multiple functions
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }
export function multiply(a, b) { return a * b; }
export function divide(a, b) { return a / b; }

// app.js — only imports add
import { add } from "./math.js";
console.log(add(1, 2));

// After tree shaking: subtract, multiply, divide are removed from the bundle
```

### Best Practices for Tree-Shakeable Code

| Practice | Why It Helps |
|----------|-------------|
| Use ES modules (`import`/`export`) | Static analysis enables tree shaking |
| Avoid side effects in module scope | Bundlers can't remove modules with side effects |
| Set `"sideEffects": false` in `package.json` | Tells bundlers the package is safe to tree-shake |
| Import specific functions, not entire libraries | `import { debounce } from "lodash-es"` not `import _ from "lodash"` |
| Use barrel files carefully | Re-exporting everything can prevent tree shaking |

---

## Image Optimization

Images typically account for 50%+ of a page's total weight. Optimizing images is one of the highest-impact performance improvements.

### Optimization Checklist

| Technique | Impact | Implementation |
|-----------|--------|---------------|
| **Modern formats** | 25–50% smaller files | Use AVIF with WebP fallback via `<picture>` |
| **Responsive images** | Serve appropriate size | Use `srcset` and `sizes` attributes |
| **Lazy loading** | Faster initial load | `loading="lazy"` on below-fold images |
| **Compression** | 10–40% size reduction | Use tools like `sharp`, `imagemin`, or CDN auto-optimization |
| **Width/height attributes** | Prevents CLS | Always set explicit dimensions |
| **Async decoding** | Unblocks main thread | `decoding="async"` |

```html
<img
  src="photo-800.webp"
  srcset="photo-400.webp 400w, photo-800.webp 800w, photo-1200.webp 1200w"
  sizes="(width >= 1024px) 50vw, 100vw"
  alt="Descriptive alt text"
  width="800"
  height="600"
  loading="lazy"
  decoding="async"
/>
```

---

## Caching Strategies

### HTTP Caching

| Header | Strategy | Best For |
|--------|----------|----------|
| `Cache-Control: public, max-age=31536000, immutable` | Long-term cache with hashed filenames | JS bundles, CSS, images with content hashes |
| `Cache-Control: public, max-age=0, must-revalidate` | Always revalidate | `index.html`, service worker |
| `Cache-Control: public, max-age=3600, stale-while-revalidate=86400` | Serve stale while fetching fresh | API responses, non-critical data |
| `ETag` / `Last-Modified` | Conditional requests | Resources that change occasionally |

### Content-Hashed Filenames

```
// Vite/Webpack output with content hashes
dist/
  index.html              ← no cache (always fetch latest)
  assets/
    app-3a7f2b.js         ← immutable cache (hash changes on content change)
    styles-9c4e1d.css     ← immutable cache
    logo-b2f8a0.svg       ← immutable cache
```

### Service Worker Caching

```javascript
// Cache static assets with a cache-first strategy
self.addEventListener("fetch", (event) => {
  if (event.request.destination === "image" ||
      event.request.destination === "style" ||
      event.request.destination === "script") {
    event.respondWith(
      caches.match(event.request).then((cached) => {
        return cached || fetch(event.request).then((response) => {
          const clone = response.clone();
          caches.open("static-v1").then((cache) => cache.put(event.request, clone));
          return response;
        });
      })
    );
  }
});
```

---

## Critical Rendering Path Optimization

### Optimizing Resource Loading

```html
<head>
  <!-- Preconnect to critical origins -->
  <link rel="preconnect" href="https://api.example.com" />
  <link rel="preconnect" href="https://fonts.googleapis.com" crossorigin />

  <!-- Preload critical resources (hero image, main font) -->
  <link rel="preload" href="/fonts/inter-var.woff2" as="font" type="font/woff2" crossorigin />
  <link rel="preload" href="/images/hero.avif" as="image" />

  <!-- CSS is render-blocking — keep it small and load critical CSS inline -->
  <style>
    /* Inline critical CSS for above-the-fold content */
    body { margin: 0; font-family: system-ui, sans-serif; }
    .hero { min-height: 100dvh; display: grid; place-items: center; }
  </style>

  <!-- Non-critical CSS loaded asynchronously -->
  <link rel="stylesheet" href="/styles/main.css" media="print" onload="this.media='all'" />

  <!-- JavaScript: defer for non-critical, async for independent scripts -->
  <script src="/js/app.js" defer></script>
  <script src="/js/analytics.js" async></script>
</head>
```

### Resource Loading Priority

| Attribute | Behavior | Use For |
|-----------|----------|---------|
| Default `<script>` | Blocks HTML parsing | Never use without `defer`/`async` |
| `<script defer>` | Downloads in parallel, executes after HTML parsing | Main application bundle |
| `<script async>` | Downloads in parallel, executes immediately when ready | Independent scripts (analytics) |
| `<script type="module">` | Deferred by default, strict mode | ES module entry points |
| `<link rel="preload">` | High-priority fetch, doesn't block rendering | Fonts, hero images, critical assets |

---

## Bundle Analysis and Size Budgets

### Analyzing Bundle Size

```bash
# Vite: build with rollup-plugin-visualizer
npx vite build

# Webpack: use webpack-bundle-analyzer
npx webpack --profile --json > stats.json
npx webpack-bundle-analyzer stats.json

# Quick check: show sizes of output files
du -sh dist/assets/* | sort -rh
```

### Performance Budgets

| Resource Type | Budget | Rationale |
|--------------|--------|-----------|
| **Total JavaScript** | ≤ 200 KB (compressed) | Parse + compile time dominates on mobile |
| **Total CSS** | ≤ 50 KB (compressed) | Render-blocking resource |
| **Largest image** | ≤ 200 KB | Dominant contributor to page weight |
| **Total page weight** | ≤ 500 KB (compressed) | Good experience on 3G connections |
| **Time to Interactive** | ≤ 3.5s on 4G | User patience threshold |

---

## Runtime Performance

### Avoiding Layout Thrashing

Layout thrashing occurs when you read and write to the DOM alternately, forcing the browser to recalculate layout on every read.

```javascript
// ❌ Bad: causes layout thrashing (read-write-read-write loop)
items.forEach((item) => {
  const height = item.offsetHeight;     // Force layout read
  item.style.height = height * 2 + "px"; // Write triggers layout
});

// ✅ Good: batch reads, then batch writes
const heights = items.map((item) => item.offsetHeight);
items.forEach((item, i) => {
  item.style.height = heights[i] * 2 + "px";
});
```

### Using `requestAnimationFrame`

```javascript
// Synchronize visual updates with the browser's paint cycle
function animateScroll(targetY, duration) {
  const startY = window.scrollY;
  const startTime = performance.now();

  function step(currentTime) {
    const elapsed = currentTime - startTime;
    const progress = Math.min(elapsed / duration, 1);
    const eased = 1 - Math.pow(1 - progress, 3); // ease-out cubic

    window.scrollTo(0, startY + (targetY - startY) * eased);

    if (progress < 1) {
      requestAnimationFrame(step);
    }
  }

  requestAnimationFrame(step);
}
```

### Debouncing and Throttling

```javascript
// Debounce: wait until user stops typing before firing
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

const handleSearch = debounce((query) => fetchResults(query), 300);

// Throttle: fire at most once per interval
function throttle(fn, interval) {
  let lastTime = 0;
  return (...args) => {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn(...args);
    }
  };
}

const handleScroll = throttle(() => updateScrollPosition(), 100);
```

---

## Web Workers for Heavy Computation

Web Workers run JavaScript on a separate thread, keeping the main thread free for UI updates.

```javascript
// main.js
const worker = new Worker(new URL("./worker.js", import.meta.url));

worker.postMessage({ type: "PROCESS_DATA", payload: largeDataset });

worker.onmessage = (event) => {
  const { result } = event.data;
  renderChart(result);
};

// worker.js
self.onmessage = (event) => {
  const { type, payload } = event.data;

  if (type === "PROCESS_DATA") {
    const result = processHeavyComputation(payload);
    self.postMessage({ result });
  }
};
```

### When to Use Web Workers

| ✅ Good Use Cases | ❌ Not Appropriate |
|-------------------|-------------------|
| CSV/JSON parsing of large datasets | Simple DOM updates |
| Image processing / filtering | Network requests (use main thread fetch) |
| Complex sorting or filtering | Small computations (< 16ms) |
| Markdown / syntax highlighting | Anything needing DOM access |
| Cryptographic operations | |

---

## Performance Monitoring and Profiling

### Tools

| Tool | What It Measures | When to Use |
|------|-----------------|-------------|
| **Lighthouse** | Lab metrics (LCP, CLS, TBT, FCP) | During development, CI/CD gates |
| **Chrome DevTools Performance** | Runtime timeline, flame chart, paint flashing | Debugging runtime jank |
| **WebPageTest** | Real-world loading on real devices and networks | Pre-release testing |
| **web-vitals library** | Real User Metrics (RUM) in production | Production monitoring |
| **Bundle analyzer** | JavaScript and CSS bundle composition | Build optimization |

### Measuring in Production

```javascript
import { onLCP, onINP, onCLS } from "web-vitals";

function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
  });

  // Use sendBeacon for reliable delivery during page unload
  navigator.sendBeacon("/analytics/vitals", body);
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [08-BUILD-TOOLS](08-BUILD-TOOLS.md) | Bundler configuration for optimal performance |
| 2 | [04-RESPONSIVE-DESIGN](04-RESPONSIVE-DESIGN.md) | Responsive images, viewport optimization |
| 3 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Production performance checklist |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial frontend performance optimization documentation |
