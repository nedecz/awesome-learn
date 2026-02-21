# Frontend Testing

## Table of Contents

1. [Overview](#overview)
2. [Testing Pyramid for Frontend](#testing-pyramid-for-frontend)
3. [Unit Testing](#unit-testing)
4. [Component Testing](#component-testing)
5. [Integration Testing](#integration-testing)
6. [End-to-End Testing](#end-to-end-testing)
7. [Visual Regression Testing](#visual-regression-testing)
8. [Accessibility Testing](#accessibility-testing)
9. [Mocking APIs with MSW](#mocking-apis-with-msw)
10. [Test-Driven Development in Frontend](#test-driven-development-in-frontend)
11. [Code Coverage Strategies](#code-coverage-strategies)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Testing is essential for building reliable, maintainable frontend applications. A well-designed test suite catches regressions early, serves as living documentation, and gives teams confidence to refactor and ship faster.

This document covers testing strategies, tools, and patterns specific to frontend applications — from unit testing pure functions to end-to-end testing user flows.

### Scope

- Testing pyramid and how it applies to frontend
- Unit testing with Jest and Vitest
- Component testing with Testing Library
- Integration testing approaches
- End-to-end testing with Cypress and Playwright
- Visual regression testing
- Accessibility testing automation
- API mocking with MSW
- TDD workflow for frontend
- Code coverage strategies and pitfalls

---

## Testing Pyramid for Frontend

```
                    ┌──────────┐
                    │   E2E    │  Few — slow, expensive, high confidence
                    │  Tests   │  Test critical user journeys
                    ├──────────┤
                 ┌──┤Component │  Many — fast, medium confidence
                 │  │  Tests   │  Test components in isolation
                 │  ├──────────┤
              ┌──┤  │  Unit    │  Most — fastest, focused confidence
              │  │  │  Tests   │  Test pure functions and logic
              └──┴──┴──────────┘
```

| Level | What It Tests | Speed | Tools |
|-------|--------------|-------|-------|
| **Unit** | Pure functions, utilities, state logic | ⚡ Fastest (ms) | Vitest, Jest |
| **Component** | Single component rendering and interaction | 🔄 Fast (ms–s) | Testing Library + Vitest/Jest |
| **Integration** | Multiple components working together | 🔄 Medium (s) | Testing Library, MSW |
| **E2E** | Full user flows through the real application | 🐢 Slow (s–min) | Playwright, Cypress |
| **Visual** | Screenshot comparison for UI regressions | 🐢 Slow (s–min) | Chromatic, Percy, Playwright |

---

## Unit Testing

Unit tests verify individual functions and modules in isolation. They are fast, deterministic, and easy to write.

### Jest vs Vitest

| Feature | Jest | Vitest |
|---------|------|--------|
| **Speed** | Good | Faster (native ESM, Vite-powered) |
| **Configuration** | Manual (CommonJS default) | Zero-config for Vite projects |
| **API** | `describe`, `it`, `expect` | Same API (Jest-compatible) |
| **ESM support** | Experimental | Native |
| **Watch mode** | ✅ | ✅ (faster, HMR-powered) |
| **Recommendation** | Legacy projects | New projects |

### Unit Test Examples

```typescript
// utils/format.ts
export function formatCurrency(amount: number, currency = "USD"): string {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency,
  }).format(amount);
}

export function truncate(text: string, maxLength: number): string {
  if (text.length <= maxLength) return text;
  return text.slice(0, maxLength - 3) + "...";
}
```

```typescript
// utils/format.test.ts
import { describe, it, expect } from "vitest";
import { formatCurrency, truncate } from "./format";

describe("formatCurrency", () => {
  it("formats USD by default", () => {
    expect(formatCurrency(1234.56)).toBe("$1,234.56");
  });

  it("formats zero correctly", () => {
    expect(formatCurrency(0)).toBe("$0.00");
  });

  it("supports other currencies", () => {
    expect(formatCurrency(1000, "EUR")).toBe("€1,000.00");
  });
});

describe("truncate", () => {
  it("returns text unchanged when shorter than max length", () => {
    expect(truncate("Hello", 10)).toBe("Hello");
  });

  it("truncates and adds ellipsis when text exceeds max length", () => {
    expect(truncate("Hello, World!", 10)).toBe("Hello, ...");
  });

  it("handles exact length", () => {
    expect(truncate("Hello", 5)).toBe("Hello");
  });
});
```

---

## Component Testing

Component tests render a component and verify its behavior from the user's perspective — what they see and interact with, not implementation details.

### Testing Library Philosophy

> "The more your tests resemble the way your software is used, the more confidence they can give you."

- Query by text, role, label — not by CSS class or test ID
- Test behavior, not implementation
- Avoid testing internal state directly

### React Component Test

```typescript
// UserCard.tsx
interface UserCardProps {
  user: { name: string; email: string; role: string };
  onEdit: (email: string) => void;
}

function UserCard({ user, onEdit }: UserCardProps) {
  return (
    <div role="article" aria-label={`User card for ${user.name}`}>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
      <span className="badge">{user.role}</span>
      <button onClick={() => onEdit(user.email)}>Edit</button>
    </div>
  );
}
```

```typescript
// UserCard.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, it, expect, vi } from "vitest";
import { UserCard } from "./UserCard";

const mockUser = { name: "Alice", email: "alice@example.com", role: "Admin" };

describe("UserCard", () => {
  it("renders user information", () => {
    render(<UserCard user={mockUser} onEdit={vi.fn()} />);

    expect(screen.getByText("Alice")).toBeInTheDocument();
    expect(screen.getByText("alice@example.com")).toBeInTheDocument();
    expect(screen.getByText("Admin")).toBeInTheDocument();
  });

  it("calls onEdit with email when Edit button is clicked", async () => {
    const handleEdit = vi.fn();
    render(<UserCard user={mockUser} onEdit={handleEdit} />);

    await userEvent.click(screen.getByRole("button", { name: /edit/i }));

    expect(handleEdit).toHaveBeenCalledWith("alice@example.com");
  });

  it("has accessible label", () => {
    render(<UserCard user={mockUser} onEdit={vi.fn()} />);

    expect(screen.getByRole("article", { name: /alice/i })).toBeInTheDocument();
  });
});
```

### Testing Library Query Priority

| Priority | Query | When to Use |
|----------|-------|-------------|
| 1 | `getByRole` | Buttons, links, headings, textboxes — most accessible |
| 2 | `getByLabelText` | Form inputs with associated labels |
| 3 | `getByPlaceholderText` | Inputs without visible labels |
| 4 | `getByText` | Non-interactive text content |
| 5 | `getByDisplayValue` | Current value of form elements |
| 6 | `getByAltText` | Images |
| 7 | `getByTestId` | Last resort — when no semantic query works |

---

## Integration Testing

Integration tests verify that multiple components work together correctly — data flows between parent and child, API data renders correctly, and user workflows complete end-to-end within a section of the app.

```typescript
// Integration test: SearchPage (input + results + pagination)
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { SearchPage } from "./SearchPage";
import { server } from "../mocks/server"; // MSW server
import { http, HttpResponse } from "msw";

describe("SearchPage integration", () => {
  it("searches and displays results", async () => {
    render(<SearchPage />);

    const searchInput = screen.getByRole("searchbox");
    await userEvent.type(searchInput, "react");
    await userEvent.click(screen.getByRole("button", { name: /search/i }));

    await waitFor(() => {
      expect(screen.getByText("React Documentation")).toBeInTheDocument();
    });

    expect(screen.getByText("Showing 1-10 of 42 results")).toBeInTheDocument();
  });

  it("shows error message when API fails", async () => {
    server.use(
      http.get("/api/search", () => {
        return HttpResponse.json({ error: "Service unavailable" }, { status: 503 });
      })
    );

    render(<SearchPage />);

    await userEvent.type(screen.getByRole("searchbox"), "test");
    await userEvent.click(screen.getByRole("button", { name: /search/i }));

    await waitFor(() => {
      expect(screen.getByRole("alert")).toHaveTextContent(/something went wrong/i);
    });
  });
});
```

---

## End-to-End Testing

E2E tests run against the real application in a real browser, testing complete user journeys.

### Playwright vs Cypress

| Feature | Playwright | Cypress |
|---------|-----------|---------|
| **Browsers** | Chromium, Firefox, WebKit | Chromium, Firefox, WebKit |
| **Language** | JavaScript, TypeScript, Python, Java, .NET | JavaScript, TypeScript |
| **Multi-tab** | ✅ Native support | ❌ Limited |
| **API testing** | ✅ Built-in request context | ✅ `cy.request()` |
| **Visual comparison** | ✅ Built-in (`toHaveScreenshot()`) | Via plugins |
| **Parallel execution** | ✅ Built-in | Via Cypress Cloud |
| **Recommendation** | New projects, complex scenarios | Existing Cypress projects |

### Playwright Example

```typescript
import { test, expect } from "@playwright/test";

test.describe("User Login Flow", () => {
  test("logs in with valid credentials", async ({ page }) => {
    await page.goto("/login");

    await page.getByLabel("Email").fill("alice@example.com");
    await page.getByLabel("Password").fill("password123");
    await page.getByRole("button", { name: /sign in/i }).click();

    await expect(page).toHaveURL("/dashboard");
    await expect(page.getByText("Welcome, Alice")).toBeVisible();
  });

  test("shows error for invalid credentials", async ({ page }) => {
    await page.goto("/login");

    await page.getByLabel("Email").fill("alice@example.com");
    await page.getByLabel("Password").fill("wrong-password");
    await page.getByRole("button", { name: /sign in/i }).click();

    await expect(page.getByRole("alert")).toContainText("Invalid credentials");
    await expect(page).toHaveURL("/login");
  });
});
```

---

## Visual Regression Testing

Visual regression testing compares screenshots of your UI to detect unintended visual changes.

### Tools

| Tool | Approach | CI Integration |
|------|----------|---------------|
| **Chromatic** | Cloud-hosted, Storybook integration | GitHub, GitLab, Bitbucket |
| **Percy** | Cloud-hosted, framework-agnostic | GitHub, GitLab |
| **Playwright** | Built-in `toHaveScreenshot()` | Any CI |
| **BackstopJS** | Self-hosted, config-driven | Any CI |

### Playwright Screenshot Testing

```typescript
import { test, expect } from "@playwright/test";

test("homepage matches visual snapshot", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveScreenshot("homepage.png", {
    maxDiffPixelRatio: 0.01, // Allow 1% pixel difference
  });
});

test("card component matches snapshot", async ({ page }) => {
  await page.goto("/components/card");
  const card = page.locator(".card").first();
  await expect(card).toHaveScreenshot("card.png");
});
```

---

## Accessibility Testing

Automated accessibility testing catches many common issues but cannot replace manual testing with assistive technologies.

### Automated Tools

```typescript
// Using axe with Testing Library
import { axe, toHaveNoViolations } from "jest-axe";

expect.extend(toHaveNoViolations);

it("has no accessibility violations", async () => {
  const { container } = render(<NavigationMenu />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

```typescript
// Playwright accessibility testing
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test("page has no accessibility violations", async ({ page }) => {
  await page.goto("/");

  const results = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa", "wcag21aa"])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

See [09-ACCESSIBILITY.md](09-ACCESSIBILITY.md) for comprehensive accessibility guidance.

---

## Mocking APIs with MSW

Mock Service Worker (MSW) intercepts network requests at the service worker level, providing realistic API mocking that works with any framework and fetching library.

### Setup

```typescript
// mocks/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: "1", name: "Alice", email: "alice@example.com" },
      { id: "2", name: "Bob", email: "bob@example.com" },
    ]);
  }),

  http.post("/api/users", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json(
      { id: "3", ...body },
      { status: 201 }
    );
  }),
];

// mocks/server.ts (for tests)
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

```typescript
// test setup (vitest.setup.ts)
import { beforeAll, afterAll, afterEach } from "vitest";
import { server } from "./mocks/server";

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

## Test-Driven Development in Frontend

### TDD Cycle

```
1. RED    → Write a failing test that describes the desired behavior
2. GREEN  → Write the minimum code to make the test pass
3. REFACTOR → Clean up the code while keeping tests green
```

### Frontend TDD Example

```typescript
// Step 1: RED — Write the test first
describe("PasswordStrength", () => {
  it("shows 'Weak' for passwords under 8 characters", () => {
    render(<PasswordStrength password="abc" />);
    expect(screen.getByText("Weak")).toBeInTheDocument();
  });

  it("shows 'Medium' for passwords with letters and numbers", () => {
    render(<PasswordStrength password="abc12345" />);
    expect(screen.getByText("Medium")).toBeInTheDocument();
  });

  it("shows 'Strong' for passwords with mixed case, numbers, and symbols", () => {
    render(<PasswordStrength password="Abc123!@#" />);
    expect(screen.getByText("Strong")).toBeInTheDocument();
  });
});

// Step 2: GREEN — Implement the component
function PasswordStrength({ password }: { password: string }) {
  const strength = getStrength(password);
  return <span>{strength}</span>;
}

function getStrength(password: string): string {
  if (password.length < 8) return "Weak";
  const hasUpper = /[A-Z]/.test(password);
  const hasLower = /[a-z]/.test(password);
  const hasNumber = /[0-9]/.test(password);
  const hasSymbol = /[^A-Za-z0-9]/.test(password);
  if (hasUpper && hasLower && hasNumber && hasSymbol) return "Strong";
  return "Medium";
}
```

---

## Code Coverage Strategies

### Coverage Metrics

| Metric | What It Measures |
|--------|-----------------|
| **Statement** | Percentage of statements executed |
| **Branch** | Percentage of conditional branches taken |
| **Function** | Percentage of functions called |
| **Line** | Percentage of lines executed |

### Recommended Thresholds

| Metric | Minimum | Target |
|--------|---------|--------|
| Statement | 70% | 80% |
| Branch | 60% | 75% |
| Function | 70% | 80% |
| Line | 70% | 80% |

### Coverage Pitfalls

- **High coverage ≠ quality tests** — 100% coverage with no assertions is useless
- **Don't chase 100%** — diminishing returns above ~80%; focus on critical paths
- **Exclude generated code** — configuration files, type declarations, barrel exports
- **Branch coverage matters most** — it catches untested conditional logic

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      reporter: ["text", "html", "lcov"],
      thresholds: {
        statements: 70,
        branches: 60,
        functions: 70,
        lines: 70,
      },
      exclude: ["**/*.d.ts", "**/*.config.*", "**/mocks/**"],
    },
  },
});
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-ACCESSIBILITY](09-ACCESSIBILITY.md) | Accessibility testing in depth |
| 2 | [08-BUILD-TOOLS](08-BUILD-TOOLS.md) | CI/CD integration for tests |
| 3 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Error handling and quality patterns |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial frontend testing documentation |
