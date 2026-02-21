# Frontend Best Practices

## Table of Contents

1. [Overview](#overview)
2. [Project Structure and Organization](#project-structure-and-organization)
3. [Component Design Principles](#component-design-principles)
4. [Code Style and Linting](#code-style-and-linting)
5. [Error Boundaries and Error Handling](#error-boundaries-and-error-handling)
6. [Security](#security)
7. [SEO Fundamentals](#seo-fundamentals)
8. [Internationalization and Localization](#internationalization-and-localization)
9. [Progressive Web Apps](#progressive-web-apps)
10. [Design Systems and Component Libraries](#design-systems-and-component-libraries)
11. [Documentation and Storybook](#documentation-and-storybook)
12. [Checklist](#checklist)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

This document consolidates production-ready best practices for frontend development. These patterns have been proven in large-scale applications and are applicable regardless of which framework you use.

### Scope

- Project structure and file organization conventions
- Component design principles: single responsibility, composition, separation of concerns
- Code style enforcement with ESLint and Prettier
- Error handling strategies for frontend applications
- Security: XSS prevention, CSP, HTTPS
- SEO fundamentals for frontend developers
- Internationalization (i18n) and localization (l10n)
- Progressive Web Apps (PWAs)
- Design systems and component libraries
- Documentation with Storybook
- Production readiness checklist

---

## Project Structure and Organization

### Feature-Based Structure (Recommended)

```
src/
├── features/              # Feature modules (domain-oriented)
│   ├── auth/
│   │   ├── components/    # Feature-specific components
│   │   ├── hooks/         # Feature-specific hooks/composables
│   │   ├── services/      # API calls, business logic
│   │   ├── types/         # TypeScript types
│   │   └── index.ts       # Public API (barrel export)
│   ├── dashboard/
│   └── settings/
├── shared/                # Shared across features
│   ├── components/        # Reusable UI components (Button, Modal, etc.)
│   ├── hooks/             # Shared hooks/composables
│   ├── utils/             # Pure utility functions
│   ├── types/             # Shared TypeScript types
│   └── constants/         # App-wide constants
├── layouts/               # Page layouts (MainLayout, AuthLayout)
├── pages/                 # Route-level components (one per route)
├── styles/                # Global styles, theme, CSS variables
├── App.tsx                # Root component
└── main.tsx               # Entry point
```

### Principles

| Principle | Guideline |
|-----------|-----------|
| **Co-location** | Keep related files together (component + test + styles + types) |
| **Feature isolation** | Features should not import from other features directly |
| **Public API** | Each feature exports only what other features need via `index.ts` |
| **Flat within features** | Avoid deeply nested folders — 2–3 levels maximum |

---

## Component Design Principles

### Single Responsibility

Each component should do one thing well. If a component has too many responsibilities, split it.

```typescript
// ❌ Bad: one component doing everything
function UserPage() {
  // Fetches data, handles form, renders list, manages pagination...
  // 300+ lines of intertwined logic
}

// ✅ Good: composed from focused components
function UserPage() {
  return (
    <PageLayout>
      <UserSearchForm onSearch={handleSearch} />
      <UserList users={users} onSelect={handleSelect} />
      <Pagination page={page} total={total} onPageChange={setPage} />
    </PageLayout>
  );
}
```

### Composition Over Configuration

```typescript
// ❌ Bad: one component with many boolean props
<Card
  showHeader={true}
  showFooter={true}
  showImage={true}
  showActions={true}
  variant="elevated"
/>

// ✅ Good: composable children
<Card variant="elevated">
  <Card.Header>Title</Card.Header>
  <Card.Image src="photo.jpg" alt="Description" />
  <Card.Body>Content</Card.Body>
  <Card.Footer>
    <Button>Save</Button>
  </Card.Footer>
</Card>
```

### Presentational vs Container Components

| Type | Responsibility | Example |
|------|---------------|---------|
| **Presentational** | How things look — receives data via props, no side effects | `<UserCard user={user} />` |
| **Container** | How things work — fetches data, manages state, handles logic | `<UserCardContainer userId={id} />` |

---

## Code Style and Linting

### ESLint + Prettier

Use ESLint for code quality rules and Prettier for formatting. They complement each other.

```json
// .eslintrc.json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended",
    "prettier"
  ],
  "rules": {
    "no-console": "warn",
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/no-explicit-any": "warn"
  }
}
```

```json
// .prettierrc
{
  "semi": true,
  "trailingComma": "all",
  "singleQuote": false,
  "printWidth": 100,
  "tabWidth": 2
}
```

### Pre-commit Hooks

```json
// package.json (using lint-staged + husky)
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{css,scss}": ["prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

---

## Error Boundaries and Error Handling

### Error Boundaries (React)

```typescript
import { Component, type ErrorInfo, type ReactNode } from "react";

interface Props {
  fallback: ReactNode;
  children: ReactNode;
}

class ErrorBoundary extends Component<Props, { hasError: boolean }> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Log to error tracking service
    reportError({ error, errorInfo });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<ErrorPage />}>
  <App />
</ErrorBoundary>
```

### API Error Handling

```typescript
// Centralized error handling for API calls
async function apiCall<T>(url: string, options?: RequestInit): Promise<T> {
  try {
    const response = await fetch(url, {
      headers: { "Content-Type": "application/json" },
      ...options,
    });

    if (!response.ok) {
      const error = await response.json().catch(() => ({}));
      throw new ApiError(response.status, error.message ?? "Request failed");
    }

    return response.json();
  } catch (error) {
    if (error instanceof ApiError) throw error;
    throw new ApiError(0, "Network error — please check your connection");
  }
}

class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message);
    this.name = "ApiError";
  }
}
```

---

## Security

### XSS Prevention

| Attack Vector | Prevention |
|--------------|-----------|
| **Injecting scripts via user input** | Frameworks auto-escape by default (React JSX, Angular templates) |
| **`dangerouslySetInnerHTML` / `innerHTML`** | Sanitize HTML with DOMPurify before rendering |
| **URL injection** (`javascript:` protocol) | Validate URLs before rendering in `href` |
| **Third-party scripts** | Use Subresource Integrity (SRI) hashes |

```typescript
// ❌ Dangerous: rendering unsanitized user content
element.innerHTML = userInput;

// ✅ Safe: sanitize first
import DOMPurify from "dompurify";
element.innerHTML = DOMPurify.sanitize(userInput);
```

### Content Security Policy (CSP)

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' https://images.example.com;
  connect-src 'self' https://api.example.com;
  font-src 'self';
">
```

### Other Security Best Practices

- **Always use HTTPS** — enforce with HSTS headers
- **Set `HttpOnly`, `Secure`, `SameSite` flags on cookies** — prevent XSS and CSRF
- **Validate input on both client and server** — never trust client-side validation alone
- **Keep dependencies updated** — use `npm audit` and Dependabot/Renovate
- **Never store secrets in frontend code** — it is visible to anyone

---

## SEO Fundamentals

### Key SEO Practices for Frontend Developers

| Practice | Implementation |
|----------|---------------|
| **Semantic HTML** | Use `<h1>`–`<h6>`, `<nav>`, `<main>`, `<article>` correctly |
| **Meta tags** | `<title>`, `<meta name="description">`, Open Graph tags |
| **Canonical URLs** | `<link rel="canonical" href="...">` to prevent duplicate content |
| **Structured data** | JSON-LD schema for rich search results |
| **Server-side rendering** | SSR/SSG ensures search engines see full content |
| **Performance** | Core Web Vitals are a ranking signal |
| **Mobile-friendly** | Responsive design, viewport meta tag |
| **Sitemap** | `sitemap.xml` for search engine discovery |

### Meta Tags Template

```html
<head>
  <title>Page Title — Site Name</title>
  <meta name="description" content="Concise description of the page (150-160 chars)" />
  <link rel="canonical" href="https://example.com/page" />

  <!-- Open Graph (social sharing) -->
  <meta property="og:title" content="Page Title" />
  <meta property="og:description" content="Description for social cards" />
  <meta property="og:image" content="https://example.com/og-image.jpg" />
  <meta property="og:url" content="https://example.com/page" />
  <meta property="og:type" content="article" />
</head>
```

---

## Internationalization and Localization

### i18n Principles

| Principle | Guideline |
|-----------|-----------|
| **Externalize strings** | Never hardcode user-facing text in components |
| **Use ICU message format** | Handles plurals, gender, and complex patterns |
| **Support RTL layouts** | Use CSS logical properties (`margin-inline-start` instead of `margin-left`) |
| **Format dates/numbers** | Use `Intl.DateTimeFormat` and `Intl.NumberFormat` |
| **Design for text expansion** | German text is ~30% longer than English |

### Libraries

| Library | Framework | Features |
|---------|-----------|----------|
| **i18next** | Any (react-i18next, vue-i18next) | Mature, plugins, namespaces |
| **@angular/localize** | Angular | Built-in, compile-time i18n |
| **vue-i18n** | Vue | Composition API, message format |
| **FormatJS (react-intl)** | React | ICU message format, formatting |

```typescript
// i18next example
import { useTranslation } from "react-i18next";

function WelcomeBanner() {
  const { t } = useTranslation();
  return <h1>{t("welcome.title", { name: user.name })}</h1>;
}

// en.json: { "welcome": { "title": "Welcome, {{name}}!" } }
// es.json: { "welcome": { "title": "¡Bienvenido, {{name}}!" } }
```

---

## Progressive Web Apps

PWAs combine the reach of the web with the experience of native apps: offline support, push notifications, and install-to-home-screen.

### PWA Requirements

| Requirement | How |
|-------------|-----|
| **HTTPS** | Secure origin required for service workers |
| **Web App Manifest** | `manifest.json` with name, icons, display mode |
| **Service Worker** | Enables offline support, caching, background sync |
| **Responsive** | Works on all screen sizes |
| **Installable** | Users can add to home screen |

### Web App Manifest

```json
{
  "name": "My Application",
  "short_name": "MyApp",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "icons": [
    { "src": "/icons/192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

---

## Design Systems and Component Libraries

A design system provides a shared language between designers and developers: consistent components, tokens, and patterns.

### Building a Design System

| Layer | Contents |
|-------|----------|
| **Design Tokens** | Colors, spacing, typography, shadows, breakpoints |
| **Base Components** | Button, Input, Card, Modal, Badge, Tooltip |
| **Composite Components** | DataTable, Form, Navigation, Sidebar |
| **Patterns** | Layout patterns, form patterns, error patterns |
| **Documentation** | Usage guidelines, do's and don'ts, code examples |

### Design Tokens as CSS Variables

```css
:root {
  /* Colors */
  --color-primary-50: #eff6ff;
  --color-primary-500: #3b82f6;
  --color-primary-900: #1e3a5f;

  /* Spacing */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-4: 1rem;
  --space-8: 2rem;

  /* Typography */
  --font-sans: system-ui, -apple-system, sans-serif;
  --font-mono: "JetBrains Mono", monospace;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px rgb(0 0 0 / 0.1);
}
```

---

## Documentation and Storybook

Storybook is a tool for building and documenting UI components in isolation.

### Benefits

- **Visual component catalog** — browse all components and their variants
- **Interactive playground** — test components with different props
- **Documentation** — auto-generated docs from component props/types
- **Visual testing** — integrate with Chromatic for screenshot comparison
- **Accessibility testing** — built-in a11y addon

### Example Story

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { Button } from "./Button";

const meta: Meta<typeof Button> = {
  component: Button,
  tags: ["autodocs"],
  argTypes: {
    variant: { control: "select", options: ["primary", "secondary", "danger"] },
    size: { control: "select", options: ["sm", "md", "lg"] },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: {
    variant: "primary",
    children: "Click me",
  },
};

export const Disabled: Story = {
  args: {
    variant: "primary",
    children: "Disabled",
    disabled: true,
  },
};
```

---

## Checklist

### Pre-Launch Checklist

| Category | Check | Status |
|----------|-------|--------|
| **Performance** | Core Web Vitals pass (LCP ≤ 2.5s, INP ≤ 200ms, CLS ≤ 0.1) | ☐ |
| **Performance** | JavaScript bundle ≤ 200 KB compressed | ☐ |
| **Performance** | Images use modern formats (AVIF/WebP) with proper sizing | ☐ |
| **Accessibility** | WCAG 2.2 AA compliance verified | ☐ |
| **Accessibility** | Keyboard navigation works for all interactive elements | ☐ |
| **Accessibility** | Screen reader tested (VoiceOver / NVDA) | ☐ |
| **Security** | CSP headers configured | ☐ |
| **Security** | No secrets in frontend code | ☐ |
| **Security** | Dependencies audited (`npm audit`) | ☐ |
| **SEO** | Meta tags, Open Graph, canonical URLs | ☐ |
| **SEO** | Semantic HTML with proper heading hierarchy | ☐ |
| **Testing** | Unit test coverage ≥ 70% | ☐ |
| **Testing** | E2E tests cover critical user journeys | ☐ |
| **Error handling** | Error boundaries/handlers catch and report errors | ☐ |
| **Error handling** | API failures show user-friendly messages | ☐ |
| **i18n** | No hardcoded user-facing strings | ☐ |
| **Build** | Production build produces cache-busted filenames | ☐ |
| **Build** | Source maps uploaded to error tracking service | ☐ |

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [11-ANTI-PATTERNS](11-ANTI-PATTERNS.md) | Common mistakes to avoid |
| 2 | [06-PERFORMANCE](06-PERFORMANCE.md) | Performance optimization deep dive |
| 3 | [09-ACCESSIBILITY](09-ACCESSIBILITY.md) | Accessibility deep dive |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial frontend best practices documentation |
