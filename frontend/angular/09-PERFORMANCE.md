# Angular Performance Optimization

## Table of Contents

1. [Overview](#overview)
2. [Change Detection Strategies](#change-detection-strategies)
3. [Signal-Based Change Detection](#signal-based-change-detection)
4. [Defer Blocks for Lazy Loading Content](#defer-blocks-for-lazy-loading-content)
5. [Lazy Loading Routes and Components](#lazy-loading-routes-and-components)
6. [TrackBy and Track Expressions](#trackby-and-track-expressions)
7. [Pure Pipes](#pure-pipes)
8. [Ahead-of-Time Compilation](#ahead-of-time-compilation)
9. [Bundle Optimization and Tree Shaking](#bundle-optimization-and-tree-shaking)
10. [Image Optimization](#image-optimization)
11. [Server-Side Rendering](#server-side-rendering)
12. [Hydration](#hydration)
13. [Web Workers](#web-workers)
14. [Performance Profiling](#performance-profiling)
15. [Next Steps](#next-steps)
16. [Version History](#version-history)

---

## Overview

Performance is critical for user experience, SEO, and business metrics. Angular provides multiple optimization strategies — from change detection tuning to server-side rendering and incremental hydration. This document covers each strategy with practical implementation guidance.

### Scope

- Change detection: Default vs OnPush vs Zoneless
- Signal-based reactivity for fine-grained updates
- `@defer` blocks for lazy loading template content
- Route and component lazy loading
- `track` expressions for efficient list rendering
- Pure pipes for memoized computations
- AOT compilation and bundle optimization
- Image optimization with `NgOptimizedImage`
- Server-side rendering with `@angular/ssr`
- Full and incremental hydration
- Web Workers for heavy computation
- Profiling with Angular DevTools

### Performance Budget

| Metric | Target | Measurement |
|--------|--------|-------------|
| **LCP** (Largest Contentful Paint) | < 2.5s | Core Web Vitals |
| **INP** (Interaction to Next Paint) | < 200ms | Core Web Vitals |
| **CLS** (Cumulative Layout Shift) | < 0.1 | Core Web Vitals |
| **Initial bundle size** | < 200 KB (gzipped) | Build output |
| **Time to Interactive** | < 3.5s | Lighthouse |

---

## Change Detection Strategies

### Default Change Detection

Angular's default strategy checks **every component** in the tree when any event occurs (click, timer, HTTP response). This is simple but can be slow in large applications.

```
Event occurs (click, HTTP response, setTimeout)
    │
    ▼
Zone.js captures the event
    │
    ▼
Angular runs change detection on the ENTIRE component tree
    │
    ▼
Every component template is re-evaluated
```

### OnPush Change Detection

With `OnPush`, Angular only checks a component when:
1. An `@Input` reference changes (or signal input updates)
2. An event handler in the component fires
3. An Observable bound with `async` pipe emits
4. `markForCheck()` is called explicitly
5. A signal read in the template updates

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h2>{{ user().name }}</h2>
    <p>{{ user().email }}</p>
  `,
})
export class UserCardComponent {
  user = input.required<User>();
}
```

### Comparison

| Aspect | Default | OnPush |
|--------|---------|--------|
| **Check frequency** | Every CD cycle | Only when inputs/signals/events change |
| **Performance** | Slower for large trees | Faster — skips unchanged subtrees |
| **Complexity** | Simple — just works | Requires immutable data patterns |
| **Recommended** | Small apps, prototypes | Production applications |

---

## Signal-Based Change Detection

### Zoneless Angular (Experimental)

Angular 18+ introduces **zoneless change detection** powered by signals. No more Zone.js — Angular only re-renders components that read changed signals.

```typescript
// app.config.ts
import { provideExperimentalZonelessChangeDetection } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideExperimentalZonelessChangeDetection(),
    provideRouter(routes),
    provideHttpClient(),
  ],
};
```

### How Zoneless Works

```
Signal value changes
    │
    ▼
Angular marks components that read the signal as dirty
    │
    ▼
Only those components are re-rendered
    │
    ▼
No Zone.js overhead — smaller bundle, faster execution
```

### Benefits of Zoneless

| Benefit | Impact |
|---------|--------|
| **Smaller bundle** | No Zone.js (~15 KB gzipped) |
| **Faster change detection** | Only affected components re-render |
| **Better debugging** | No Zone.js stack frames in stack traces |
| **Coexistence with third-party code** | No Zone.js patching of setTimeout, Promise, etc. |
| **Predictable updates** | Changes happen when signals change, not on every async event |

### Migration Path

1. Use `OnPush` change detection on all components
2. Replace `BehaviorSubject` state with `signal()` state
3. Use `toSignal()` for Observable → Signal conversion
4. Remove `ChangeDetectorRef.markForCheck()` calls (signals handle this)
5. Enable `provideExperimentalZonelessChangeDetection()`
6. Remove Zone.js from `polyfills` in `angular.json`

---

## Defer Blocks for Lazy Loading Content

`@defer` blocks (Angular 17+) lazily load sections of a template, reducing the initial bundle and speeding up first paint.

### Viewport Trigger (Most Common)

```html
@defer (on viewport) {
  <app-comments [postId]="postId()" />
} @placeholder {
  <div class="comments-placeholder" style="height: 200px;">
    Comments will load when you scroll here
  </div>
} @loading (minimum 300ms) {
  <app-spinner />
} @error {
  <p>Failed to load comments.</p>
}
```

### Multiple Triggers

```html
<!-- Load on interaction OR after 5 seconds -->
@defer (on interaction; on timer(5s)) {
  <app-analytics-dashboard />
} @placeholder {
  <button>Click to load dashboard</button>
}
```

### Prefetching

```html
<!-- Prefetch when idle, render when visible -->
@defer (on viewport; prefetch on idle) {
  <app-heavy-widget />
}
```

### Impact on Bundle Size

| Without @defer | With @defer |
|----------------|-------------|
| All components in initial bundle | Deferred components in separate chunks |
| Larger initial download | Smaller initial download |
| Longer Time to Interactive | Faster Time to Interactive |

---

## Lazy Loading Routes and Components

### Lazy Loading Routes

```typescript
export const routes: Routes = [
  { path: '', component: HomeComponent },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes),
  },
  {
    path: 'profile',
    loadComponent: () => import('./profile/profile.component').then(m => m.ProfileComponent),
  },
];
```

### Measuring Lazy Loading Impact

```
Before lazy loading:
  main.js  → 450 KB (everything)

After lazy loading:
  main.js  → 180 KB (core app)
  admin.js → 120 KB (loaded on /admin)
  profile.js → 90 KB (loaded on /profile)
  editor.js → 60 KB (loaded on /editor)
```

---

## TrackBy and Track Expressions

### @for with track (Angular 17+)

The `track` expression tells Angular how to identify items in a list, enabling efficient DOM reuse:

```html
<!-- ✅ Track by unique identifier -->
@for (user of users(); track user.id) {
  <app-user-card [user]="user" />
}

<!-- ✅ Track by index (when items have no unique ID) -->
@for (item of items(); track $index) {
  <div>{{ item }}</div>
}
```

### Why Track Matters

```
Without track (or poor track expression):
  Angular destroys and recreates ALL DOM elements on every change

With track by unique ID:
  Angular reuses existing DOM elements, only updating changed ones

Impact: 10x faster rendering for large lists (1000+ items)
```

### Legacy trackBy (with *ngFor)

```html
<!-- Legacy syntax -->
<div *ngFor="let user of users; trackBy: trackByUserId">
  {{ user.name }}
</div>
```

```typescript
trackByUserId(index: number, user: User): string {
  return user.id;
}
```

---

## Pure Pipes

Pure pipes are memoized — Angular only re-executes them when their input value changes (by reference).

```typescript
@Pipe({
  name: 'fileSize',
  standalone: true,
  pure: true,  // Default — only re-runs when input reference changes
})
export class FileSizePipe implements PipeTransform {
  transform(bytes: number): string {
    if (bytes === 0) return '0 B';
    const units = ['B', 'KB', 'MB', 'GB', 'TB'];
    const i = Math.floor(Math.log(bytes) / Math.log(1024));
    return `${(bytes / Math.pow(1024, i)).toFixed(1)} ${units[i]}`;
  }
}
```

### Pipe vs Method in Template

```html
<!-- ✅ Pipe: memoized, only recalculates when input changes -->
<span>{{ fileSize | fileSize }}</span>

<!-- ❌ Method: recalculates on EVERY change detection cycle -->
<span>{{ formatFileSize(fileSize) }}</span>
```

---

## Ahead-of-Time Compilation

AOT compilation converts Angular templates and TypeScript into efficient JavaScript at **build time** rather than at runtime.

| Aspect | AOT (Default) | JIT |
|--------|---------------|-----|
| **When** | Build time | Runtime (in browser) |
| **Bundle size** | Smaller (no compiler) | Larger (includes compiler) |
| **Error detection** | Build errors caught early | Errors at runtime |
| **Security** | Templates are pre-compiled | Templates evaluated at runtime |
| **Startup speed** | Faster | Slower |

AOT is the **default** for `ng build` since Angular 9. No configuration needed.

---

## Bundle Optimization and Tree Shaking

### Analyze Bundle Size

```bash
# Generate a bundle analysis report
ng build --stats-json
npx webpack-bundle-analyzer dist/my-app/stats.json
```

### Optimization Strategies

| Strategy | Impact | How |
|----------|--------|-----|
| **Lazy loading** | Major | Split routes into separate chunks |
| **Tree shaking** | Major | Use `providedIn: 'root'` for tree-shakable services |
| **Remove unused imports** | Medium | Audit `import` statements, use `sideEffects: false` |
| **Optimize third-party libs** | Major | Import specific modules, not entire libraries |
| **Compression** | Major | Enable gzip/Brotli on your server |
| **Differential loading** | Medium | Serve ES2022 to modern browsers, ES5 to legacy |

### Import Optimization

```typescript
// ❌ Imports entire library
import * as _ from 'lodash';

// ✅ Import only what you need
import { debounce } from 'lodash-es';

// ❌ Imports all Material modules
import { MatFormFieldModule, MatInputModule, MatButtonModule } from '@angular/material';

// ✅ Import individually (they are tree-shakable)
import { MatFormField } from '@angular/material/form-field';
import { MatInput } from '@angular/material/input';
import { MatButton } from '@angular/material/button';
```

---

## Image Optimization

### NgOptimizedImage

Angular's built-in image directive optimizes loading, layout stability, and performance:

```typescript
import { NgOptimizedImage } from '@angular/common';

@Component({
  selector: 'app-hero',
  standalone: true,
  imports: [NgOptimizedImage],
  template: `
    <!-- LCP image: priority loading -->
    <img ngSrc="/assets/hero.jpg" width="1200" height="600" priority />

    <!-- Below-the-fold image: lazy loaded by default -->
    <img ngSrc="/assets/feature.jpg" width="800" height="400" />

    <!-- Responsive image with srcset -->
    <img
      ngSrc="/assets/product.jpg"
      width="400"
      height="300"
      sizes="(max-width: 768px) 100vw, 400px"
    />
  `,
})
export class HeroComponent {}
```

### Image Loader (CDN Integration)

```typescript
import { provideImageKitLoader } from '@angular/common';

// In app.config.ts
providers: [
  provideImageKitLoader('https://ik.imagekit.io/your_account'),
],
```

---

## Server-Side Rendering

### @angular/ssr (Angular 17+)

SSR renders Angular pages on the server, sending fully formed HTML to the browser for faster first paint and better SEO.

```bash
# Add SSR to an existing project
ng add @angular/ssr
```

### How SSR Works

```
Browser requests page
    │
    ▼
Server renders Angular app to HTML
    │
    ▼
Server sends fully rendered HTML + CSS
    │
    ▼
Browser displays content immediately (fast FCP/LCP)
    │
    ▼
Browser downloads JavaScript bundles
    │
    ▼
Angular hydrates — attaches event listeners to existing DOM
    │
    ▼
App becomes fully interactive
```

### SSR Benefits

| Benefit | Impact |
|---------|--------|
| **SEO** | Search engines see fully rendered content |
| **First Contentful Paint** | Content visible before JavaScript loads |
| **Social media previews** | Open Graph tags are in the initial HTML |
| **Performance perception** | Users see content sooner |

### Platform Checks

```typescript
import { isPlatformBrowser, isPlatformServer } from '@angular/common';
import { PLATFORM_ID, inject } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class StorageService {
  private platformId = inject(PLATFORM_ID);

  get(key: string): string | null {
    if (isPlatformBrowser(this.platformId)) {
      return localStorage.getItem(key);
    }
    return null;  // No localStorage on server
  }
}
```

---

## Hydration

### Full Hydration (Angular 16+)

Hydration reuses server-rendered DOM instead of destroying and recreating it:

```typescript
// app.config.ts — hydration is enabled by default with @angular/ssr
export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration(),
  ],
};
```

### Incremental Hydration (Angular 18+)

Hydrate only the parts of the page the user interacts with:

```html
@defer (on viewport; hydrate on interaction) {
  <app-comments [postId]="postId()" />
} @placeholder {
  <div>Comments section</div>
}
```

### Hydration Modes

| Mode | When Hydrated | Bundle Impact | Interactivity |
|------|--------------|---------------|---------------|
| **Full** | On page load | Full JS loaded | Immediate |
| **Incremental (on interaction)** | User clicks/focuses | Partial JS loaded | On demand |
| **Incremental (on viewport)** | Element scrolls into view | Partial JS loaded | On scroll |
| **Incremental (on idle)** | Browser is idle | Partial JS loaded | Delayed |

---

## Web Workers

Move heavy computations off the main thread to keep the UI responsive.

### Creating a Web Worker

```bash
ng generate web-worker heavy-calculation
```

```typescript
// heavy-calculation.worker.ts
addEventListener('message', ({ data }) => {
  // CPU-intensive work happens off the main thread
  const result = performHeavyCalculation(data);
  postMessage(result);
});

function performHeavyCalculation(data: number[]): number {
  return data.reduce((sum, val) => sum + Math.sqrt(val), 0);
}
```

### Using in a Component

```typescript
@Component({ /* ... */ })
export class AnalyticsComponent {
  result = signal<number | null>(null);

  processData(data: number[]): void {
    if (typeof Worker !== 'undefined') {
      const worker = new Worker(new URL('./heavy-calculation.worker', import.meta.url));
      worker.onmessage = ({ data: result }) => {
        this.result.set(result);
        worker.terminate();
      };
      worker.postMessage(data);
    }
  }
}
```

---

## Performance Profiling

### Angular DevTools

The Angular DevTools browser extension provides:

- **Component tree inspector** — view the component hierarchy and state
- **Change detection profiler** — identify which components are checked and how long each takes
- **Dependency injection graph** — visualize provider relationships

### Profiling Checklist

| Check | Tool | What to Look For |
|-------|------|-----------------|
| Bundle size | `ng build` output, webpack-bundle-analyzer | Bundles exceeding budget |
| Change detection | Angular DevTools profiler | Components checked unnecessarily |
| Runtime performance | Chrome DevTools Performance tab | Long tasks (> 50ms), layout thrashing |
| Core Web Vitals | Lighthouse, PageSpeed Insights | LCP > 2.5s, INP > 200ms, CLS > 0.1 |
| Memory leaks | Chrome DevTools Memory tab | Growing heap size over time |
| Network | Chrome DevTools Network tab | Large requests, missing caching |

### Performance Optimization Checklist

```
□ All components use OnPush change detection (or signals + zoneless)
□ Lists use track expressions (@for ... track item.id)
□ Routes are lazy loaded (loadComponent / loadChildren)
□ Below-the-fold content uses @defer
□ Images use NgOptimizedImage with width/height
□ LCP image has priority attribute
□ Pure pipes used instead of methods in templates
□ No heavy computation in templates or change detection
□ Third-party libraries are tree-shaken (import specific modules)
□ SSR enabled for SEO-critical pages
□ Bundle size within budget (< 200 KB gzipped for initial load)
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Production best practices |
| 2 | [11-ANTI-PATTERNS](11-ANTI-PATTERNS.md) | Common performance anti-patterns |
| 3 | [01-COMPONENTS-AND-TEMPLATES](01-COMPONENTS-AND-TEMPLATES.md) | @defer and component architecture |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Angular performance optimization documentation |
