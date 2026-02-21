# Web Accessibility (a11y)

## Table of Contents

1. [Overview](#overview)
2. [WCAG Guidelines Overview](#wcag-guidelines-overview)
3. [ARIA Roles States and Properties](#aria-roles-states-and-properties)
4. [Semantic HTML for Accessibility](#semantic-html-for-accessibility)
5. [Keyboard Navigation and Focus Management](#keyboard-navigation-and-focus-management)
6. [Screen Reader Compatibility](#screen-reader-compatibility)
7. [Color Contrast and Visual Accessibility](#color-contrast-and-visual-accessibility)
8. [Form Accessibility](#form-accessibility)
9. [Dynamic Content and ARIA Live Regions](#dynamic-content-and-aria-live-regions)
10. [Accessibility Testing Tools and Workflows](#accessibility-testing-tools-and-workflows)
11. [Legal Requirements and Compliance](#legal-requirements-and-compliance)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Web accessibility (often abbreviated as **a11y** — "a" + 11 letters + "y") ensures that websites and applications are usable by everyone, including people with disabilities. This includes users who are blind, deaf, motor-impaired, or have cognitive disabilities.

Accessibility is not a feature — it is a fundamental quality of well-built software. Building accessible applications benefits everyone: screen reader users, keyboard-only users, people with temporary injuries, mobile users in bright sunlight, and users on slow connections.

### Scope

- WCAG 2.1/2.2 guidelines and conformance levels
- ARIA roles, states, and properties
- Semantic HTML as the foundation of accessibility
- Keyboard navigation and focus management
- Screen reader compatibility patterns
- Color contrast and visual accessibility
- Form accessibility
- Dynamic content and ARIA live regions
- Testing tools and workflows
- Legal requirements and compliance

---

## WCAG Guidelines Overview

The **Web Content Accessibility Guidelines (WCAG)** are the international standard for web accessibility, published by the W3C's Web Accessibility Initiative (WAI).

### WCAG Principles (POUR)

| Principle | Meaning | Examples |
|-----------|---------|---------|
| **Perceivable** | Users can perceive the content | Alt text for images, captions for video, sufficient color contrast |
| **Operable** | Users can interact with the interface | Keyboard navigation, enough time to read, no seizure-inducing flashes |
| **Understandable** | Users can comprehend content and UI | Clear language, consistent navigation, input assistance |
| **Robust** | Content works with current and future assistive technologies | Valid HTML, ARIA when needed, standard UI patterns |

### Conformance Levels

| Level | Description | Requirement |
|-------|-------------|-------------|
| **A** | Minimum accessibility | Baseline — must meet for basic accessibility |
| **AA** | Standard accessibility | **Target for most organizations** — covers contrast, resize, keyboard |
| **AAA** | Enhanced accessibility | Highest level — not always achievable for all content |

### Key WCAG 2.2 Criteria (AA Level)

| Criterion | Number | Requirement |
|-----------|--------|-------------|
| Non-text Content | 1.1.1 | All non-text content has a text alternative |
| Color Contrast | 1.4.3 | Text has at least 4.5:1 contrast ratio (3:1 for large text) |
| Resize Text | 1.4.4 | Text can be resized to 200% without loss of content |
| Keyboard | 2.1.1 | All functionality available via keyboard |
| Focus Visible | 2.4.7 | Keyboard focus indicator is visible |
| Focus Not Obscured | 2.4.11 | Focus indicator is not hidden by other content (new in 2.2) |
| Target Size | 2.5.8 | Interactive targets are at least 24×24 CSS pixels (new in 2.2) |
| Error Identification | 3.3.1 | Input errors are identified and described to the user |
| Name, Role, Value | 4.1.2 | UI components have accessible name, role, and state |

---

## ARIA Roles States and Properties

ARIA (Accessible Rich Internet Applications) provides attributes that communicate the purpose, state, and behavior of custom UI elements to assistive technologies.

### The First Rule of ARIA

> **Do not use ARIA if you can use a native HTML element or attribute that already has the semantics and behavior you need.**

Native HTML is always preferable because it comes with built-in keyboard handling and screen reader support.

```html
<!-- ❌ Avoid: custom button with ARIA -->
<div role="button" tabindex="0" aria-pressed="false" onclick="toggle()">
  Toggle
</div>

<!-- ✅ Prefer: native button -->
<button type="button" aria-pressed="false" onclick="toggle()">
  Toggle
</button>
```

### Common ARIA Roles

| Role | Purpose | Native Alternative |
|------|---------|-------------------|
| `role="navigation"` | Navigation landmark | `<nav>` |
| `role="main"` | Main content | `<main>` |
| `role="banner"` | Site header | `<header>` (page-level) |
| `role="alert"` | Important, time-sensitive message | None — ARIA required |
| `role="dialog"` | Modal or dialog box | `<dialog>` |
| `role="tablist"` / `role="tab"` / `role="tabpanel"` | Tab interface | None — ARIA required |
| `role="progressbar"` | Progress indicator | `<progress>` |

### Common ARIA States and Properties

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `aria-label` | Provides an accessible name | `<button aria-label="Close dialog">×</button>` |
| `aria-labelledby` | References another element as the label | `<div aria-labelledby="heading-id">` |
| `aria-describedby` | References additional descriptive text | `<input aria-describedby="help-text">` |
| `aria-expanded` | Indicates if a collapsible section is open | `<button aria-expanded="true">Menu</button>` |
| `aria-hidden` | Hides an element from assistive technology | `<span aria-hidden="true">🔍</span>` |
| `aria-current` | Indicates the current item in a set | `<a aria-current="page" href="/about">About</a>` |
| `aria-required` | Indicates a required form field | `<input aria-required="true">` |
| `aria-invalid` | Indicates an invalid form field | `<input aria-invalid="true">` |

---

## Semantic HTML for Accessibility

Semantic HTML provides the foundation for accessibility. Screen readers and other assistive technologies rely on HTML semantics to understand page structure and navigate content.

### Landmark Elements

```html
<body>
  <header>         <!-- banner landmark -->
    <nav>          <!-- navigation landmark -->
      ...
    </nav>
  </header>

  <main>           <!-- main landmark (one per page) -->
    <article>      <!-- article landmark -->
      <h1>Title</h1>
      <section>    <!-- region landmark (when labelled) -->
        <h2>Section Heading</h2>
        ...
      </section>
    </article>

    <aside>        <!-- complementary landmark -->
      ...
    </aside>
  </main>

  <footer>         <!-- contentinfo landmark -->
    ...
  </footer>
</body>
```

### Heading Hierarchy

```html
<!-- ✅ Correct: logical heading hierarchy -->
<h1>Page Title</h1>
  <h2>Section One</h2>
    <h3>Subsection</h3>
  <h2>Section Two</h2>
    <h3>Subsection</h3>

<!-- ❌ Incorrect: skipped heading level -->
<h1>Page Title</h1>
  <h4>Jumped to h4</h4>  <!-- Screen readers flag this as a structural error -->
```

---

## Keyboard Navigation and Focus Management

All interactive elements must be operable with a keyboard alone. Many users cannot use a mouse — they navigate with Tab, Enter, Space, and arrow keys.

### Keyboard Interaction Patterns

| Key | Expected Behavior |
|-----|------------------|
| `Tab` | Move focus to the next interactive element |
| `Shift + Tab` | Move focus to the previous interactive element |
| `Enter` | Activate buttons, links, and form submission |
| `Space` | Activate buttons, toggle checkboxes |
| `Escape` | Close modals, dropdowns, tooltips |
| `Arrow keys` | Navigate within composite widgets (tabs, menus, radio groups) |

### Focus Management for Modals

```javascript
function openModal(modalElement) {
  modalElement.showModal(); // <dialog> method

  // Trap focus inside the modal
  const focusableElements = modalElement.querySelectorAll(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const firstFocusable = focusableElements[0];
  const lastFocusable = focusableElements[focusableElements.length - 1];

  firstFocusable.focus();

  modalElement.addEventListener("keydown", (e) => {
    if (e.key === "Tab") {
      if (e.shiftKey && document.activeElement === firstFocusable) {
        e.preventDefault();
        lastFocusable.focus();
      } else if (!e.shiftKey && document.activeElement === lastFocusable) {
        e.preventDefault();
        firstFocusable.focus();
      }
    }
  });
}
```

### Visible Focus Indicators

```css
/* Never remove focus outlines without providing an alternative */
/* ❌ Bad */
*:focus { outline: none; }

/* ✅ Good: custom focus indicator */
:focus-visible {
  outline: 2px solid var(--color-focus);
  outline-offset: 2px;
  border-radius: 2px;
}

/* Only remove default outline when providing a custom one */
:focus:not(:focus-visible) {
  outline: none;
}
```

---

## Screen Reader Compatibility

### How Screen Readers Navigate

Screen readers provide multiple navigation modes:

1. **Browse mode** — read through content linearly
2. **Heading navigation** — jump between headings (H1–H6)
3. **Landmark navigation** — jump between regions (nav, main, aside)
4. **Form mode** — navigate between form controls
5. **Table mode** — navigate cells by row and column

### Visually Hidden Content

Sometimes information needs to be available to screen readers but hidden visually.

```css
/* Screen-reader-only utility class */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

```html
<!-- Provide context for screen readers -->
<button>
  <svg aria-hidden="true"><!-- icon --></svg>
  <span class="sr-only">Close dialog</span>
</button>

<!-- Skip navigation link -->
<a href="#main-content" class="sr-only">Skip to main content</a>
```

---

## Color Contrast and Visual Accessibility

### Contrast Requirements (WCAG AA)

| Text Type | Minimum Contrast Ratio |
|-----------|----------------------|
| Normal text (< 18px / < 14px bold) | 4.5:1 |
| Large text (≥ 18px / ≥ 14px bold) | 3:1 |
| UI components and graphical objects | 3:1 |

### Never Rely on Color Alone

```html
<!-- ❌ Bad: color is the only indicator of error -->
<input style="border-color: red" />

<!-- ✅ Good: color + icon + text -->
<div class="field-error">
  <input aria-invalid="true" aria-describedby="email-error" />
  <p id="email-error" role="alert">
    ⚠️ Please enter a valid email address
  </p>
</div>
```

### Respecting User Preferences

```css
/* Respect reduced motion preference */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

/* Support high contrast mode */
@media (forced-colors: active) {
  .button {
    border: 2px solid ButtonText;
  }
}

/* Support dark mode */
@media (prefers-color-scheme: dark) {
  :root {
    --bg: #1a1a1a;
    --text: #f0f0f0;
  }
}
```

---

## Form Accessibility

### Labeling Form Controls

```html
<!-- ✅ Explicit label association -->
<label for="email">Email address</label>
<input type="email" id="email" name="email" autocomplete="email" required />

<!-- ✅ Group related fields with fieldset/legend -->
<fieldset>
  <legend>Shipping address</legend>
  <label for="street">Street</label>
  <input type="text" id="street" name="street" autocomplete="street-address" />
  <label for="city">City</label>
  <input type="text" id="city" name="city" autocomplete="address-level2" />
</fieldset>
```

### Error Handling in Forms

```html
<form novalidate>
  <div class="form-group">
    <label for="password">Password</label>
    <input
      type="password"
      id="password"
      aria-describedby="password-requirements password-error"
      aria-invalid="true"
      required
    />
    <p id="password-requirements" class="hint">
      Must be at least 8 characters with one uppercase letter and one number
    </p>
    <p id="password-error" role="alert" class="error">
      Password does not meet requirements
    </p>
  </div>
</form>
```

---

## Dynamic Content and ARIA Live Regions

ARIA live regions announce dynamic content changes to screen readers without requiring focus to move.

### Live Region Types

| Attribute | Behavior |
|-----------|----------|
| `aria-live="polite"` | Announces changes when the user is idle (most common) |
| `aria-live="assertive"` | Interrupts the user to announce immediately (use sparingly) |
| `role="alert"` | Equivalent to `aria-live="assertive"` + `aria-atomic="true"` |
| `role="status"` | Equivalent to `aria-live="polite"` + `aria-atomic="true"` |
| `role="log"` | Append-only log of changes (chat, activity feed) |

### Examples

```html
<!-- Toast notification — polite announcement -->
<div role="status" aria-live="polite">
  <!-- Dynamically inject content here -->
  Settings saved successfully
</div>

<!-- Critical error — assertive announcement -->
<div role="alert">
  Your session has expired. Please log in again.
</div>

<!-- Search results count — polite update -->
<div aria-live="polite" aria-atomic="true">
  Showing 42 results for "accessibility"
</div>
```

---

## Accessibility Testing Tools and Workflows

### Automated Testing

| Tool | Type | What It Catches |
|------|------|----------------|
| **axe DevTools** | Browser extension | WCAG violations, best practices |
| **Lighthouse** | Chrome DevTools / CLI | Accessibility audit as part of page audit |
| **axe-core** | Library (jest-axe, @axe-core/playwright) | Automated testing in CI |
| **eslint-plugin-jsx-a11y** | ESLint plugin | JSX accessibility issues at lint time |
| **Pa11y** | CLI / CI tool | Automated WCAG testing |

### Manual Testing Checklist

| Test | How |
|------|-----|
| **Keyboard-only navigation** | Unplug mouse, navigate with Tab/Enter/Space/Esc |
| **Screen reader** | Test with VoiceOver (macOS), NVDA (Windows), or Orca (Linux) |
| **Zoom to 200%** | Verify no content is lost or overlapping |
| **Color contrast** | Check with tools or browser DevTools |
| **Reduced motion** | Enable `prefers-reduced-motion` and verify |
| **No images** | Verify alt text conveys meaning |

### Testing Workflow

```
1. Lint time      → eslint-plugin-jsx-a11y catches code-level issues
2. Unit tests     → jest-axe tests components for WCAG violations
3. E2E tests      → @axe-core/playwright tests full pages
4. Code review    → Reviewer checks semantic HTML and ARIA usage
5. Manual testing → Keyboard and screen reader testing before release
```

---

## Legal Requirements and Compliance

| Law / Standard | Region | Applies To |
|---------------|--------|-----------|
| **ADA** (Americans with Disabilities Act) | USA | Public-facing websites and apps |
| **Section 508** | USA | Federal government websites |
| **EN 301 549** | EU | Public sector digital services |
| **European Accessibility Act (EAA)** | EU | Private sector products and services (2025+) |
| **AODA** | Canada (Ontario) | Public and private sector organizations |
| **Equality Act 2010** | UK | All service providers |

**Key takeaway:** Accessibility is increasingly a legal requirement, not just a best practice. Courts have ruled that websites are "places of public accommodation" under disability rights laws.

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-HTML-CSS](01-HTML-CSS.md) | Semantic HTML foundations |
| 2 | [07-TESTING](07-TESTING.md) | Accessibility testing automation |
| 3 | [10-BEST-PRACTICES](10-BEST-PRACTICES.md) | Inclusive design in production |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial web accessibility documentation |
