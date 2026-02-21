# HTML & CSS Foundations

## Table of Contents

1. [Overview](#overview)
2. [Semantic HTML5 Elements](#semantic-html5-elements)
3. [CSS Box Model](#css-box-model)
4. [Flexbox](#flexbox)
5. [CSS Grid](#css-grid)
6. [CSS Custom Properties](#css-custom-properties)
7. [CSS Specificity and the Cascade](#css-specificity-and-the-cascade)
8. [CSS Methodologies](#css-methodologies)
9. [CSS-in-JS Approaches](#css-in-js-approaches)
10. [CSS Preprocessors](#css-preprocessors)
11. [Modern CSS Features](#modern-css-features)
12. [Responsive Design Principles](#responsive-design-principles)
13. [Web Fonts and Typography](#web-fonts-and-typography)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

HTML and CSS are the foundational technologies of the web. HTML provides the structural skeleton of every web page, while CSS controls its visual presentation. Mastering these fundamentals is essential before moving to any JavaScript framework — every framework ultimately renders HTML and applies CSS.

### Target Audience

- **New developers** learning web development from scratch
- **Backend developers** building frontend skills
- **Frontend developers** reviewing modern HTML/CSS capabilities

### Scope

- Semantic HTML5 elements and why they matter for accessibility and SEO
- The CSS Box Model, Flexbox, and Grid layout systems
- CSS Custom Properties, specificity, and the cascade
- CSS methodologies, preprocessors, and CSS-in-JS
- Modern CSS features including container queries, `:has()`, nesting, and layers
- Responsive design foundations and web typography

---

## Semantic HTML5 Elements

Semantic HTML uses elements that convey meaning about the content they contain. Instead of using `<div>` for everything, semantic elements tell browsers, screen readers, and search engines what role each section plays.

### Key Semantic Elements

| Element | Purpose | Replaces |
|---------|---------|----------|
| `<header>` | Introductory content, navigation links | `<div class="header">` |
| `<nav>` | Navigation links | `<div class="nav">` |
| `<main>` | Primary content of the page (one per page) | `<div class="main">` |
| `<article>` | Self-contained content (blog post, news story) | `<div class="article">` |
| `<section>` | Thematic grouping of content with a heading | `<div class="section">` |
| `<aside>` | Tangentially related content (sidebar, callout) | `<div class="sidebar">` |
| `<footer>` | Footer content, copyright, links | `<div class="footer">` |
| `<figure>` / `<figcaption>` | Image or diagram with a caption | `<div class="image-wrapper">` |
| `<time>` | Date or time value | `<span class="date">` |
| `<mark>` | Highlighted or referenced text | `<span class="highlight">` |

### Example: Semantic Page Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Article Title</title>
</head>
<body>
  <header>
    <nav aria-label="Main navigation">
      <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/about">About</a></li>
      </ul>
    </nav>
  </header>

  <main>
    <article>
      <h1>Article Title</h1>
      <time datetime="2026-01-15">January 15, 2026</time>
      <section>
        <h2>Introduction</h2>
        <p>Content goes here...</p>
      </section>
    </article>

    <aside>
      <h2>Related Articles</h2>
      <ul>
        <li><a href="/related-1">Related Post 1</a></li>
      </ul>
    </aside>
  </main>

  <footer>
    <p>&copy; 2026 Company Name</p>
  </footer>
</body>
</html>
```

### Why Semantic HTML Matters

- **Accessibility:** Screen readers use semantic elements to navigate pages — `<nav>` becomes a landmark, `<main>` lets users skip to content
- **SEO:** Search engines weight content in `<article>` and `<main>` higher than generic `<div>` containers
- **Maintainability:** Semantic tags communicate intent to other developers reading the code

---

## CSS Box Model

Every HTML element is a rectangular box. The CSS Box Model defines how the size of that box is calculated.

```
┌───────────────────────────────────────────┐
│                 Margin                    │
│   ┌───────────────────────────────────┐   │
│   │             Border                │   │
│   │   ┌───────────────────────────┐   │   │
│   │   │          Padding          │   │   │
│   │   │   ┌───────────────────┐   │   │   │
│   │   │   │     Content       │   │   │   │
│   │   │   │  (width × height) │   │   │   │
│   │   │   └───────────────────┘   │   │   │
│   │   └───────────────────────────┘   │   │
│   └───────────────────────────────────┘   │
└───────────────────────────────────────────┘
```

### Box Sizing

```css
/* Default: width applies to content only */
.content-box {
  box-sizing: content-box; /* default */
  width: 200px;
  padding: 20px;
  border: 2px solid;
  /* Total width: 200 + 20 + 20 + 2 + 2 = 244px */
}

/* Recommended: width includes padding and border */
.border-box {
  box-sizing: border-box;
  width: 200px;
  padding: 20px;
  border: 2px solid;
  /* Total width: 200px (content shrinks to 156px) */
}

/* Apply border-box globally — standard modern practice */
*,
*::before,
*::after {
  box-sizing: border-box;
}
```

---

## Flexbox

Flexbox is a one-dimensional layout system for arranging items in a row or column with powerful alignment and distribution controls.

### Core Concepts

```css
.container {
  display: flex;
  flex-direction: row;        /* row | row-reverse | column | column-reverse */
  justify-content: center;    /* Main axis alignment */
  align-items: center;        /* Cross axis alignment */
  gap: 1rem;                  /* Space between items */
  flex-wrap: wrap;             /* Allow items to wrap to next line */
}

.item {
  flex: 1;                    /* Shorthand: flex-grow flex-shrink flex-basis */
  /* flex: 1 means flex-grow: 1, flex-shrink: 1, flex-basis: 0% */
}
```

### Common Flexbox Patterns

```css
/* Center anything horizontally and vertically */
.center {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Navigation bar: logo left, links right */
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

/* Equal-width columns */
.columns {
  display: flex;
  gap: 1rem;
}
.columns > * {
  flex: 1;
}

/* Sticky footer: content fills available space */
.page {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}
.page > main {
  flex: 1;
}
```

---

## CSS Grid

CSS Grid is a two-dimensional layout system for creating complex layouts with rows and columns simultaneously.

### Core Concepts

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr); /* 3 equal columns */
  grid-template-rows: auto 1fr auto;     /* Header, content, footer */
  gap: 1rem;                              /* Row and column gap */
}

/* Place items explicitly */
.header { grid-column: 1 / -1; }          /* Span all columns */
.sidebar { grid-row: 2 / 3; }
.content { grid-column: 2 / -1; }
```

### Grid vs Flexbox

| Feature | Flexbox | Grid |
|---------|---------|------|
| **Dimensions** | One-dimensional (row OR column) | Two-dimensional (rows AND columns) |
| **Best for** | Components, navbars, card rows | Page layouts, complex grids |
| **Item sizing** | Content-driven (items determine size) | Container-driven (grid defines size) |
| **Use together** | Grid for page layout, Flexbox for component internals | ✅ Complementary |

### Responsive Grid Without Media Queries

```css
/* Auto-fill columns that are at least 250px wide */
.auto-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1rem;
}
```

---

## CSS Custom Properties

CSS Custom Properties (CSS variables) enable reusable, dynamic values that cascade and can be changed at runtime with JavaScript.

```css
:root {
  --color-primary: #3b82f6;
  --color-text: #1f2937;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 2rem;
  --font-body: system-ui, -apple-system, sans-serif;
  --radius: 0.375rem;
}

.button {
  background-color: var(--color-primary);
  color: white;
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--radius);
  font-family: var(--font-body);
}

/* Override in a scope — theming */
.dark-theme {
  --color-primary: #60a5fa;
  --color-text: #f9fafb;
}
```

---

## CSS Specificity and the Cascade

### Specificity Hierarchy (Lowest to Highest)

| Selector Type | Example | Specificity |
|--------------|---------|-------------|
| Universal | `*` | `0-0-0` |
| Element / pseudo-element | `div`, `::before` | `0-0-1` |
| Class / attribute / pseudo-class | `.card`, `[type="text"]`, `:hover` | `0-1-0` |
| ID | `#header` | `1-0-0` |
| Inline style | `style="..."` | `1-0-0-0` |
| `!important` | `color: red !important` | Overrides all specificity |

### Rules

1. **More specific selectors win** — `#header .nav a` beats `.nav a`
2. **Later rules win** when specificity is equal — source order matters
3. **Avoid `!important`** — it breaks the cascade and makes debugging painful
4. **Avoid ID selectors in CSS** — they create high specificity that's hard to override

---

## CSS Methodologies

CSS methodologies provide naming conventions and organizational strategies to keep stylesheets maintainable at scale.

### BEM (Block Element Modifier)

```css
/* Block: standalone component */
.card { }

/* Element: part of the block (double underscore) */
.card__title { }
.card__body { }
.card__image { }

/* Modifier: variant of a block or element (double hyphen) */
.card--featured { }
.card__title--large { }
```

### Comparison

| Methodology | Naming Convention | Philosophy |
|------------|-------------------|-----------|
| **BEM** | `.block__element--modifier` | Flat specificity, explicit relationships |
| **OOCSS** | Separate structure from skin | Reusable, composable utility objects |
| **SMACSS** | Categorize rules (base, layout, module, state) | Organized architecture |
| **Atomic CSS** | `.mt-4`, `.flex`, `.text-center` | Single-purpose utility classes (Tailwind) |

---

## CSS-in-JS Approaches

CSS-in-JS writes styles in JavaScript, co-locating them with components. This approach is popular in React ecosystems.

| Library | Approach | Runtime CSS? |
|---------|----------|-------------|
| **styled-components** | Tagged template literals | Yes |
| **Emotion** | Object styles or template literals | Yes |
| **Vanilla Extract** | TypeScript-authored, zero-runtime | No (build-time) |
| **CSS Modules** | Scoped class names via build tool | No (build-time) |
| **Tailwind CSS** | Utility classes in HTML | No (build-time) |

**Trend:** The industry is moving toward **zero-runtime solutions** (Vanilla Extract, CSS Modules, Tailwind) to avoid the performance overhead of runtime CSS injection.

---

## CSS Preprocessors

Preprocessors extend CSS with features like variables, nesting, mixins, and functions. They compile to standard CSS.

| Preprocessor | File Extension | Key Features |
|-------------|---------------|-------------|
| **Sass (SCSS)** | `.scss` | Variables, nesting, mixins, partials, functions, `@extend` |
| **Less** | `.less` | Variables, nesting, mixins, functions |
| **PostCSS** | `.css` (with plugins) | Plugin-based transforms (autoprefixer, nesting, custom media) |

**Note:** With modern CSS features (custom properties, nesting, `:has()`), the need for preprocessors is diminishing. PostCSS remains valuable for autoprefixing and polyfilling new CSS features.

---

## Modern CSS Features

CSS has evolved significantly. These features reduce the need for JavaScript and preprocessors.

### CSS Nesting (Native)

```css
.card {
  background: white;
  border-radius: 0.5rem;

  & .title {
    font-size: 1.25rem;
    font-weight: 600;
  }

  &:hover {
    box-shadow: 0 4px 12px rgb(0 0 0 / 0.1);
  }

  @media (width >= 768px) {
    padding: 2rem;
  }
}
```

### Container Queries

```css
.card-container {
  container-type: inline-size;
  container-name: card;
}

@container card (width >= 400px) {
  .card {
    display: grid;
    grid-template-columns: 200px 1fr;
  }
}
```

### The `:has()` Selector (Parent Selector)

```css
/* Style a card differently when it contains an image */
.card:has(img) {
  grid-template-rows: 200px 1fr;
}

/* Style a form group when its input is invalid */
.form-group:has(:invalid) {
  border-color: red;
}
```

### CSS Cascade Layers

```css
@layer base, components, utilities;

@layer base {
  a { color: blue; }
}

@layer components {
  .nav a { color: white; }
}

@layer utilities {
  .text-red { color: red; }
}
/* Layers establish specificity precedence regardless of selector specificity */
```

---

## Responsive Design Principles

Responsive design ensures web pages look and function well across all screen sizes.

### Core Techniques

```css
/* Mobile-first: start with mobile styles, add complexity for larger screens */
.container {
  padding: 1rem;
}

@media (width >= 768px) {
  .container {
    padding: 2rem;
    max-width: 720px;
    margin: 0 auto;
  }
}

@media (width >= 1024px) {
  .container {
    max-width: 960px;
  }
}
```

See [04-RESPONSIVE-DESIGN.md](04-RESPONSIVE-DESIGN.md) for a complete deep dive.

---

## Web Fonts and Typography

### System Font Stack

```css
body {
  font-family: system-ui, -apple-system, BlinkMacSystemFont,
    "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
}
```

### Loading Web Fonts Efficiently

```css
@font-face {
  font-family: "Inter";
  src: url("/fonts/inter-var.woff2") format("woff2");
  font-weight: 100 900;
  font-display: swap; /* Show fallback font immediately, swap when loaded */
}
```

### Typography Scale

```css
:root {
  --text-xs: 0.75rem;    /* 12px */
  --text-sm: 0.875rem;   /* 14px */
  --text-base: 1rem;     /* 16px */
  --text-lg: 1.125rem;   /* 18px */
  --text-xl: 1.25rem;    /* 20px */
  --text-2xl: 1.5rem;    /* 24px */
  --text-3xl: 1.875rem;  /* 30px */
  --text-4xl: 2.25rem;   /* 36px */
}
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-JAVASCRIPT-FUNDAMENTALS](02-JAVASCRIPT-FUNDAMENTALS.md) | ES6+, closures, async/await, DOM APIs |
| 2 | [04-RESPONSIVE-DESIGN](04-RESPONSIVE-DESIGN.md) | Mobile-first, media queries, fluid typography |
| 3 | [09-ACCESSIBILITY](09-ACCESSIBILITY.md) | Semantic HTML for accessibility, ARIA roles |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial HTML & CSS foundations documentation |
