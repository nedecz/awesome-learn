# Responsive Design & Mobile-First

## Table of Contents

1. [Overview](#overview)
2. [Mobile-First Design Philosophy](#mobile-first-design-philosophy)
3. [Media Queries and Breakpoint Strategies](#media-queries-and-breakpoint-strategies)
4. [Fluid Typography and Spacing](#fluid-typography-and-spacing)
5. [Responsive Images](#responsive-images)
6. [Viewport Units and Container Queries](#viewport-units-and-container-queries)
7. [Touch-Friendly Interactions](#touch-friendly-interactions)
8. [Progressive Enhancement vs Graceful Degradation](#progressive-enhancement-vs-graceful-degradation)
9. [Testing Responsive Layouts](#testing-responsive-layouts)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

---

## Overview

Responsive design is the practice of building web interfaces that adapt to any screen size, from mobile phones to ultra-wide monitors. In a world where over 60% of web traffic comes from mobile devices, responsive design is not optional — it is a baseline expectation.

This document covers the principles, techniques, and tools for building responsive web applications using a mobile-first approach.

### Scope

- Mobile-first design philosophy and why it leads to better outcomes
- Media queries, breakpoints, and modern query syntax
- Fluid typography and spacing using `clamp()` and viewport units
- Responsive images with `srcset`, the `<picture>` element, and modern formats
- Viewport units, container queries, and their use cases
- Touch-friendly interactions and input considerations
- Progressive enhancement vs graceful degradation
- Testing responsive layouts across devices and viewports

---

## Mobile-First Design Philosophy

Mobile-first means designing for the smallest screen first, then progressively enhancing the experience for larger screens. This approach results in better performance, simpler code, and more inclusive experiences.

### Why Mobile-First?

| Aspect | Mobile-First | Desktop-First |
|--------|-------------|---------------|
| **Performance** | Start lean, add complexity | Start heavy, try to remove |
| **CSS size** | Smaller base, additive media queries | Larger base, overriding media queries |
| **Content priority** | Forces prioritization of essential content | Encourages content sprawl |
| **Progressive enhancement** | Natural fit — build up from baseline | Requires graceful degradation — strip down |

### Mobile-First CSS Pattern

```css
/* Base styles: mobile (no media query needed) */
.container {
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.card {
  padding: 1rem;
  border-radius: 0.5rem;
}

/* Tablet and up */
@media (width >= 768px) {
  .container {
    flex-direction: row;
    flex-wrap: wrap;
    padding: 2rem;
  }
  .card {
    flex: 1 1 calc(50% - 0.5rem);
  }
}

/* Desktop and up */
@media (width >= 1024px) {
  .container {
    max-width: 1200px;
    margin: 0 auto;
  }
  .card {
    flex: 1 1 calc(33.333% - 0.667rem);
  }
}
```

---

## Media Queries and Breakpoint Strategies

### Modern Media Query Syntax

CSS Media Queries Level 4 introduced a range syntax that is clearer and less error-prone:

```css
/* Old syntax */
@media (min-width: 768px) and (max-width: 1023px) { }

/* New range syntax (preferred) */
@media (768px <= width < 1024px) { }
@media (width >= 1024px) { }
```

### Common Breakpoint System

| Name | Width | Typical Devices |
|------|-------|-----------------|
| **xs** | < 576px | Small phones |
| **sm** | ≥ 576px | Large phones |
| **md** | ≥ 768px | Tablets |
| **lg** | ≥ 1024px | Laptops, small desktops |
| **xl** | ≥ 1280px | Desktops |
| **2xl** | ≥ 1536px | Large desktops |

### Breakpoints as CSS Custom Properties

```css
:root {
  --bp-sm: 576px;
  --bp-md: 768px;
  --bp-lg: 1024px;
  --bp-xl: 1280px;
}

/* Note: CSS custom properties cannot be used directly in media queries.
   Use them in JavaScript or with a preprocessor. In plain CSS, use the
   literal values in media queries. */
@media (width >= 768px) { /* md */ }
@media (width >= 1024px) { /* lg */ }
```

### Content-Based Breakpoints

Rather than using fixed device breakpoints, set breakpoints where your content breaks:

```css
/* When the article text line length exceeds ~75 characters, add columns */
@media (width >= 680px) {
  .article {
    column-count: 2;
    column-gap: 2rem;
  }
}
```

---

## Fluid Typography and Spacing

### The `clamp()` Function

`clamp()` defines a value with a minimum, preferred, and maximum — creating fluid scaling without media queries.

```css
/* Fluid font size: 1rem minimum, scales with viewport, 2.5rem maximum */
h1 {
  font-size: clamp(1.5rem, 4vw + 0.5rem, 3rem);
}

/* Fluid spacing */
.section {
  padding: clamp(1rem, 3vw, 3rem);
}

/* Fluid container width */
.container {
  width: clamp(320px, 90vw, 1200px);
  margin-inline: auto;
}
```

### Fluid Type Scale

```css
:root {
  --text-sm: clamp(0.8rem, 0.17vw + 0.76rem, 0.89rem);
  --text-base: clamp(1rem, 0.34vw + 0.91rem, 1.19rem);
  --text-lg: clamp(1.25rem, 0.61vw + 1.1rem, 1.58rem);
  --text-xl: clamp(1.56rem, 1vw + 1.31rem, 2.11rem);
  --text-2xl: clamp(1.95rem, 1.56vw + 1.56rem, 2.81rem);
  --text-3xl: clamp(2.44rem, 2.38vw + 1.85rem, 3.75rem);
}
```

---

## Responsive Images

### The `srcset` Attribute

```html
<!-- Browser selects the most appropriate image based on viewport width -->
<img
  src="hero-800.jpg"
  srcset="
    hero-400.jpg   400w,
    hero-800.jpg   800w,
    hero-1200.jpg 1200w,
    hero-1600.jpg 1600w
  "
  sizes="(width >= 1024px) 50vw, 100vw"
  alt="Hero banner showing a cityscape"
  loading="lazy"
  decoding="async"
/>
```

### The `<picture>` Element

```html
<!-- Art direction: different images for different viewports -->
<picture>
  <source
    media="(width >= 1024px)"
    srcset="hero-desktop.avif"
    type="image/avif"
  />
  <source
    media="(width >= 1024px)"
    srcset="hero-desktop.webp"
    type="image/webp"
  />
  <source
    media="(width >= 768px)"
    srcset="hero-tablet.webp"
    type="image/webp"
  />
  <img
    src="hero-mobile.jpg"
    alt="Hero banner"
    loading="lazy"
    decoding="async"
  />
</picture>
```

### Modern Image Formats

| Format | Compression | Browser Support | Best For |
|--------|------------|----------------|----------|
| **AVIF** | Best (50% smaller than JPEG) | Chrome, Firefox, Safari 16+ | Photos, complex images |
| **WebP** | Great (25-35% smaller than JPEG) | All modern browsers | General-purpose |
| **JPEG** | Good | Universal | Fallback for older browsers |
| **PNG** | Lossless | Universal | Transparency, icons, screenshots |
| **SVG** | Vector (scalable) | Universal | Icons, logos, illustrations |

---

## Viewport Units and Container Queries

### Viewport Units

| Unit | Description |
|------|-------------|
| `vw` | 1% of viewport width |
| `vh` | 1% of viewport height |
| `dvh` | 1% of dynamic viewport height (accounts for mobile browser chrome) |
| `svh` | 1% of smallest viewport height |
| `lvh` | 1% of largest viewport height |
| `vi` | 1% of viewport inline axis (writing-mode aware) |
| `vb` | 1% of viewport block axis |

```css
/* Use dvh for full-height mobile layouts (avoids iOS Safari address bar issue) */
.hero {
  min-height: 100dvh;
}
```

### Container Queries

Container queries allow components to respond to their container's size rather than the viewport. This enables truly reusable, context-aware components.

```css
/* Define a containment context */
.card-container {
  container-type: inline-size;
  container-name: card;
}

/* Style based on container width, not viewport */
.card {
  display: flex;
  flex-direction: column;
  padding: 1rem;
}

@container card (width >= 400px) {
  .card {
    flex-direction: row;
    gap: 1.5rem;
  }
}

@container card (width >= 600px) {
  .card {
    padding: 2rem;
  }
  .card__title {
    font-size: 1.5rem;
  }
}
```

---

## Touch-Friendly Interactions

### Minimum Touch Target Sizes

WCAG 2.2 recommends a minimum touch target size of 24×24 CSS pixels, with 44×44 pixels being the ideal target size for comfortable interaction.

```css
/* Ensure buttons and links are large enough to tap comfortably */
button,
a,
[role="button"] {
  min-height: 44px;
  min-width: 44px;
  padding: 0.75rem 1rem;
}

/* Add spacing between interactive elements to prevent mis-taps */
.nav-list li + li {
  margin-top: 0.5rem;
}
```

### Hover vs Touch Considerations

```css
/* Only apply hover effects on devices that support hover */
@media (hover: hover) {
  .card:hover {
    box-shadow: 0 4px 12px rgb(0 0 0 / 0.15);
    transform: translateY(-2px);
  }
}

/* Provide alternative feedback for touch devices */
@media (hover: none) {
  .card:active {
    background-color: var(--color-active);
  }
}

/* Detect coarse pointer (touch) vs fine pointer (mouse) */
@media (pointer: coarse) {
  .slider-thumb {
    width: 44px;
    height: 44px;
  }
}
```

---

## Progressive Enhancement vs Graceful Degradation

### Progressive Enhancement (Recommended)

Start with a baseline experience that works everywhere, then layer on enhancements for capable browsers and devices.

```
Layer 1: Semantic HTML         → Works in any browser, accessible
Layer 2: CSS layout & styling  → Visual presentation
Layer 3: CSS animations        → Enhanced visual feedback
Layer 4: JavaScript            → Rich interactivity
Layer 5: Advanced features     → Web APIs, offline support
```

### Feature Detection with CSS

```css
/* Apply grid layout only if supported (fallback is flexbox or block) */
@supports (display: grid) {
  .layout {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  }
}

/* Use container queries where supported */
@supports (container-type: inline-size) {
  .widget-wrapper {
    container-type: inline-size;
  }
}
```

### Feature Detection with JavaScript

```javascript
// Check for API support before using it
if ("IntersectionObserver" in window) {
  // Use Intersection Observer for lazy loading
} else {
  // Fallback: load all images immediately
}
```

---

## Testing Responsive Layouts

### Testing Checklist

| Test | Tool / Method |
|------|--------------|
| **Multiple viewports** | Chrome DevTools Device Mode, Firefox Responsive Design Mode |
| **Real devices** | Test on at least one iOS and one Android device |
| **Orientation** | Verify both portrait and landscape on tablets |
| **Touch targets** | Ensure all interactive elements are ≥ 44×44px |
| **Text readability** | Verify font sizes are readable at every breakpoint |
| **Zoom** | Test at 200% and 400% zoom levels |
| **Reduced motion** | Enable `prefers-reduced-motion` and verify |
| **Container queries** | Test components in different container widths |
| **Print** | Verify `@media print` styles work correctly |

### Automated Testing

```javascript
// Playwright: test at multiple viewport sizes
const viewports = [
  { width: 375, height: 812 },  // Mobile
  { width: 768, height: 1024 }, // Tablet
  { width: 1440, height: 900 }, // Desktop
];

for (const viewport of viewports) {
  test(`renders correctly at ${viewport.width}x${viewport.height}`, async ({ page }) => {
    await page.setViewportSize(viewport);
    await page.goto("/");
    await expect(page.locator(".hero")).toBeVisible();
  });
}
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-ACCESSIBILITY](09-ACCESSIBILITY.md) | WCAG, keyboard navigation, screen readers |
| 2 | [06-PERFORMANCE](06-PERFORMANCE.md) | Image optimization, Core Web Vitals |
| 3 | [01-HTML-CSS](01-HTML-CSS.md) | CSS Grid, Flexbox, modern CSS features |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial responsive design and mobile-first documentation |
