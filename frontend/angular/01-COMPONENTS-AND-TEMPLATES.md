# Components & Templates

## Table of Contents

1. [Overview](#overview)
2. [Component Architecture](#component-architecture)
3. [Component Lifecycle Hooks](#component-lifecycle-hooks)
4. [Template Syntax](#template-syntax)
5. [New Control Flow Syntax](#new-control-flow-syntax)
6. [Content Projection](#content-projection)
7. [View Queries](#view-queries)
8. [Signal-Based Inputs and Outputs](#signal-based-inputs-and-outputs)
9. [Component Communication Patterns](#component-communication-patterns)
10. [Smart vs Presentational Components](#smart-vs-presentational-components)
11. [Component Styling](#component-styling)
12. [Dynamic Components](#dynamic-components)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

Components are the fundamental building blocks of Angular applications. Every Angular application is a tree of components — from the root `AppComponent` down to the smallest UI element. This document covers component architecture, lifecycle hooks, template syntax, the new control flow, content projection, signal-based inputs/outputs, and styling strategies.

### Scope

- Component class structure, metadata, and decorators
- Lifecycle hooks and their execution order
- Template syntax: interpolation, binding, events, two-way binding
- New built-in control flow (@if, @for, @switch, @defer) — Angular 17+
- Content projection with `ng-content` and `ngTemplateOutlet`
- ViewChild, ViewChildren, ContentChild, ContentChildren queries
- Signal-based `input()` and `output()` functions (Angular 17.1+)
- Smart vs presentational component patterns
- Component styling with ViewEncapsulation

---

## Component Architecture

### Anatomy of a Component

Every Angular component consists of three parts:

```typescript
import { Component, signal } from '@angular/core';

@Component({
  // --- Metadata ---
  selector: 'app-user-card',       // HTML tag name
  standalone: true,                 // Self-contained (no NgModule needed)
  imports: [DatePipe],              // Dependencies used in template
  templateUrl: './user-card.component.html',  // External template
  styleUrl: './user-card.component.css',      // External styles
  changeDetection: ChangeDetectionStrategy.OnPush,  // Performance optimization
})
export class UserCardComponent {
  // --- Class (logic + state) ---
  name = signal('Ada Lovelace');
  joined = signal(new Date(2023, 0, 15));
}
```

```html
<!-- user-card.component.html (Template) -->
<article class="card">
  <h2>{{ name() }}</h2>
  <p>Joined: {{ joined() | date:'mediumDate' }}</p>
</article>
```

### Inline vs External Templates

| Approach | When to Use |
|----------|-------------|
| `template: '...'` (inline) | Small components with < 10 lines of HTML |
| `templateUrl: '...'` (external) | Any component with significant markup |

### Component Metadata Options

| Option | Purpose | Example |
|--------|---------|---------|
| `selector` | CSS selector for the component | `'app-user-card'`, `'[appTooltip]'` |
| `standalone` | Self-contained component | `true` (default in Angular 17+) |
| `imports` | Dependencies (components, directives, pipes) | `[CommonModule, RouterLink]` |
| `template` / `templateUrl` | Inline or external HTML template | `'<h1>Hello</h1>'` |
| `styles` / `styleUrl` | Inline or external CSS | `['h1 { color: red; }']` |
| `changeDetection` | Change detection strategy | `ChangeDetectionStrategy.OnPush` |
| `encapsulation` | CSS encapsulation mode | `ViewEncapsulation.Emulated` |
| `host` | Host element bindings | `{ class: 'card', '[class.active]': 'isActive()' }` |
| `providers` | Component-level DI providers | `[MyLocalService]` |

---

## Component Lifecycle Hooks

Angular components go through a defined lifecycle. Hooks let you run logic at specific points:

### Lifecycle Execution Order

```
Constructor
    │
    ▼
ngOnChanges()         ← Called when input properties change (before ngOnInit)
    │
    ▼
ngOnInit()            ← Called once after first ngOnChanges
    │
    ▼
ngDoCheck()           ← Called on every change detection run
    │
    ▼
ngAfterContentInit()  ← Called after content projection (ng-content) is initialized
    │
    ▼
ngAfterContentChecked() ← Called after every check of projected content
    │
    ▼
ngAfterViewInit()     ← Called after the component's view (and children) are initialized
    │
    ▼
ngAfterViewChecked()  ← Called after every check of the component's view
    │
    ▼
ngOnDestroy()         ← Called once before the component is destroyed
```

### Most Important Hooks

```typescript
import { Component, OnInit, OnDestroy, OnChanges, SimpleChanges, input } from '@angular/core';

@Component({
  selector: 'app-user-profile',
  standalone: true,
  template: `<h2>{{ userId() }}</h2>`,
})
export class UserProfileComponent implements OnInit, OnChanges, OnDestroy {
  userId = input.required<string>();

  ngOnChanges(changes: SimpleChanges): void {
    // Runs when input properties change
    if (changes['userId']) {
      console.log('User ID changed:', changes['userId'].currentValue);
    }
  }

  ngOnInit(): void {
    // Runs once after inputs are initialized
    // Good for: fetching data, setting up subscriptions
    console.log('Component initialized with user:', this.userId());
  }

  ngOnDestroy(): void {
    // Runs once before component is destroyed
    // Good for: cleanup, unsubscribing, removing event listeners
    console.log('Component destroyed');
  }
}
```

### Modern Alternative: afterNextRender / afterRender

For DOM-dependent initialization, prefer `afterNextRender` (Angular 16+):

```typescript
import { Component, afterNextRender, ElementRef, viewChild } from '@angular/core';

@Component({
  selector: 'app-chart',
  standalone: true,
  template: `<canvas #canvas></canvas>`,
})
export class ChartComponent {
  canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('canvas');

  constructor() {
    afterNextRender(() => {
      // Runs once after the component's first render in the browser
      // Safe to access DOM elements here
      const ctx = this.canvas().nativeElement.getContext('2d');
      this.initChart(ctx);
    });
  }

  private initChart(ctx: CanvasRenderingContext2D | null): void {
    // Chart initialization logic
  }
}
```

---

## Template Syntax

### Interpolation

Display component data in the template using double curly braces:

```html
<h1>{{ title }}</h1>
<p>{{ user.name }}</p>
<p>{{ getFullName() }}</p>
<p>{{ price | currency:'USD' }}</p>  <!-- with pipe -->
<p>{{ count() }}</p>                 <!-- signal read -->
```

### Property Binding

Bind component data to element properties:

```html
<!-- Bind to element property -->
<img [src]="imageUrl" [alt]="imageAlt">

<!-- Bind to component input -->
<app-user [user]="currentUser">

<!-- Bind to attribute (use attr. prefix) -->
<td [attr.colspan]="columnSpan">

<!-- Bind to class -->
<div [class.active]="isActive()">
<div [class]="dynamicClasses">

<!-- Bind to style -->
<div [style.width.px]="boxWidth()">
<div [style.color]="textColor">
```

### Event Binding

Respond to user actions:

```html
<!-- DOM events -->
<button (click)="save()">Save</button>
<input (input)="onInput($event)">
<form (submit)="onSubmit()">
<div (keydown.escape)="close()">

<!-- Component output events -->
<app-search (searchChange)="onSearch($event)">
```

### Two-Way Binding

Combine property binding and event binding:

```html
<!-- Two-way binding with ngModel (requires FormsModule) -->
<input [(ngModel)]="username">

<!-- Two-way binding with signals (Angular 17.2+) -->
<app-slider [(value)]="sliderValue">

<!-- Equivalent expanded form -->
<input [ngModel]="username" (ngModelChange)="username = $event">
```

### Template Reference Variables

Create references to DOM elements or components:

```html
<input #nameInput type="text">
<button (click)="greet(nameInput.value)">Greet</button>

<app-counter #counter></app-counter>
<button (click)="counter.increment()">Increment</button>
```

---

## New Control Flow Syntax

Angular 17 introduced a **new built-in control flow** that replaces `*ngIf`, `*ngFor`, and `*ngSwitch` with a cleaner, more powerful syntax.

### @if — Conditional Rendering

```html
<!-- Basic conditional -->
@if (user()) {
  <app-user-profile [user]="user()" />
} @else {
  <app-login-prompt />
}

<!-- Multiple conditions -->
@if (status() === 'loading') {
  <app-spinner />
} @else if (status() === 'error') {
  <app-error [message]="errorMessage()" />
} @else {
  <app-content [data]="data()" />
}
```

### @for — List Rendering

```html
<!-- Basic loop with required track -->
@for (item of items(); track item.id) {
  <app-item-card [item]="item" />
} @empty {
  <p>No items found.</p>
}
```

`@for` provides implicit variables:

| Variable | Type | Description |
|----------|------|-------------|
| `$index` | `number` | Zero-based index of the current item |
| `$first` | `boolean` | True if the current item is the first |
| `$last` | `boolean` | True if the current item is the last |
| `$even` | `boolean` | True if the index is even |
| `$odd` | `boolean` | True if the index is odd |
| `$count` | `number` | Total number of items in the collection |

```html
@for (user of users(); track user.id; let i = $index, let isLast = $last) {
  <div class="user-row">
    <span>{{ i + 1 }}. {{ user.name }}</span>
    @if (!isLast) {
      <hr />
    }
  </div>
} @empty {
  <p>No users registered.</p>
}
```

### @switch — Multi-Way Conditional

```html
@switch (role()) {
  @case ('admin') {
    <app-admin-dashboard />
  }
  @case ('editor') {
    <app-editor-dashboard />
  }
  @case ('viewer') {
    <app-viewer-dashboard />
  }
  @default {
    <app-guest-view />
  }
}
```

### @defer — Deferred Loading

`@defer` lazily loads a section of the template, reducing initial bundle size:

```html
<!-- Load when the element enters the viewport -->
@defer (on viewport) {
  <app-heavy-chart [data]="chartData()" />
} @placeholder {
  <div class="chart-placeholder">Chart will load here</div>
} @loading (minimum 500ms) {
  <app-spinner />
} @error {
  <p>Failed to load chart component.</p>
}
```

Defer triggers:

| Trigger | When It Loads |
|---------|--------------|
| `on viewport` | Element enters the viewport |
| `on interaction` | User interacts (click, focus, etc.) |
| `on hover` | User hovers over the placeholder |
| `on idle` | Browser is idle |
| `on immediate` | After initial render completes |
| `on timer(5s)` | After a specified delay |
| `when condition` | When a boolean expression becomes true |

---

## Content Projection

Content projection lets a component accept and render content from its parent.

### Single-Slot Projection

```typescript
// card.component.ts
@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card">
      <ng-content />
    </div>
  `,
})
export class CardComponent {}
```

```html
<!-- Usage -->
<app-card>
  <h2>Card Title</h2>
  <p>Card content goes here.</p>
</app-card>
```

### Multi-Slot Projection

```typescript
@Component({
  selector: 'app-dialog',
  standalone: true,
  template: `
    <div class="dialog">
      <header>
        <ng-content select="[dialog-title]" />
      </header>
      <section>
        <ng-content />
      </section>
      <footer>
        <ng-content select="[dialog-actions]" />
      </footer>
    </div>
  `,
})
export class DialogComponent {}
```

```html
<!-- Usage -->
<app-dialog>
  <h2 dialog-title>Confirm Action</h2>
  <p>Are you sure you want to delete this item?</p>
  <div dialog-actions>
    <button (click)="cancel()">Cancel</button>
    <button (click)="confirm()">Confirm</button>
  </div>
</app-dialog>
```

### ngTemplateOutlet

Render templates dynamically:

```typescript
@Component({
  selector: 'app-list',
  standalone: true,
  imports: [NgTemplateOutlet],
  template: `
    @for (item of items(); track item.id) {
      <ng-container
        [ngTemplateOutlet]="itemTemplate() || defaultTemplate"
        [ngTemplateOutletContext]="{ $implicit: item, index: $index }"
      />
    }
    <ng-template #defaultTemplate let-item>
      <p>{{ item.name }}</p>
    </ng-template>
  `,
})
export class ListComponent<T> {
  items = input.required<T[]>();
  itemTemplate = input<TemplateRef<any>>();
}
```

---

## View Queries

### viewChild and viewChildren

Query elements or components in the component's own template:

```typescript
import { Component, viewChild, viewChildren, ElementRef, afterNextRender } from '@angular/core';

@Component({
  selector: 'app-form',
  standalone: true,
  template: `
    <input #nameInput type="text" />
    <input #emailInput type="email" />
    <app-tooltip #tooltip />
  `,
})
export class FormComponent {
  // Signal-based view queries (Angular 17+)
  nameInput = viewChild.required<ElementRef<HTMLInputElement>>('nameInput');
  tooltip = viewChild(TooltipComponent);
  allInputs = viewChildren<ElementRef<HTMLInputElement>>('nameInput, emailInput');

  constructor() {
    afterNextRender(() => {
      this.nameInput().nativeElement.focus();
    });
  }
}
```

### contentChild and contentChildren

Query projected content:

```typescript
@Component({
  selector: 'app-tab-group',
  standalone: true,
  template: `
    <div class="tab-headers">
      @for (tab of tabs(); track tab.label()) {
        <button (click)="selectTab(tab)">{{ tab.label() }}</button>
      }
    </div>
    <ng-content />
  `,
})
export class TabGroupComponent {
  tabs = contentChildren(TabComponent);
}
```

---

## Signal-Based Inputs and Outputs

Angular 17.1+ introduced signal-based `input()` and `output()` functions that replace the `@Input()` and `@Output()` decorators.

### Signal Inputs

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-badge',
  standalone: true,
  template: `
    <span class="badge" [class]="variant()">
      {{ name() }}
    </span>
  `,
})
export class UserBadgeComponent {
  // Required input (must be provided by parent)
  name = input.required<string>();

  // Optional input with default value
  variant = input<'primary' | 'secondary'>('primary');

  // Input with transform
  disabled = input(false, { transform: booleanAttribute });
}
```

```html
<!-- Usage -->
<app-user-badge [name]="currentUser().name" variant="secondary" />
```

### Signal Outputs

```typescript
import { Component, output } from '@angular/core';

@Component({
  selector: 'app-search-box',
  standalone: true,
  template: `
    <input
      type="text"
      [value]="query()"
      (input)="onInput($event)"
      placeholder="Search..."
    />
  `,
})
export class SearchBoxComponent {
  query = signal('');
  searchChange = output<string>();

  onInput(event: Event): void {
    const value = (event.target as HTMLInputElement).value;
    this.query.set(value);
    this.searchChange.emit(value);
  }
}
```

```html
<!-- Usage -->
<app-search-box (searchChange)="onSearch($event)" />
```

### Model Inputs (Two-Way Binding)

```typescript
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-rating',
  standalone: true,
  template: `
    @for (star of stars; track star) {
      <button (click)="value.set(star)" [class.filled]="star <= value()">
        ★
      </button>
    }
  `,
})
export class RatingComponent {
  value = model(0);  // Creates two-way bindable signal
  stars = [1, 2, 3, 4, 5];
}
```

```html
<!-- Two-way binding -->
<app-rating [(value)]="userRating" />
```

---

## Component Communication Patterns

### Parent → Child (Inputs)

```typescript
// Parent
<app-child [data]="parentData()" />

// Child
data = input.required<Data>();
```

### Child → Parent (Outputs)

```typescript
// Child
saved = output<Item>();
this.saved.emit(item);

// Parent
<app-child (saved)="onSaved($event)" />
```

### Shared Service (Siblings / Distant Components)

```typescript
@Injectable({ providedIn: 'root' })
export class SelectionService {
  selectedId = signal<string | null>(null);

  select(id: string): void {
    this.selectedId.set(id);
  }
}
```

### Signals for Reactive Communication

```typescript
// Parent creates a signal, child reads it
export class ParentComponent {
  filter = signal('all');
  items = computed(() => this.allItems().filter(i => matchesFilter(i, this.filter())));
}
```

---

## Smart vs Presentational Components

### Presentational Components (Dumb)

- Receive data through inputs
- Emit events through outputs
- No injected services
- No side effects
- Easy to test and reuse

```typescript
@Component({
  selector: 'app-user-list',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @for (user of users(); track user.id) {
      <div class="user-row" (click)="userSelected.emit(user)">
        {{ user.name }}
      </div>
    }
  `,
})
export class UserListComponent {
  users = input.required<User[]>();
  userSelected = output<User>();
}
```

### Smart Components (Container)

- Inject services
- Manage state and side effects
- Compose presentational components
- Handle routing and navigation

```typescript
@Component({
  selector: 'app-user-page',
  standalone: true,
  imports: [UserListComponent, UserDetailComponent],
  template: `
    <app-user-list
      [users]="users()"
      (userSelected)="onUserSelected($event)"
    />
    @if (selectedUser()) {
      <app-user-detail [user]="selectedUser()!" />
    }
  `,
})
export class UserPageComponent {
  private userService = inject(UserService);

  users = toSignal(this.userService.getUsers(), { initialValue: [] });
  selectedUser = signal<User | null>(null);

  onUserSelected(user: User): void {
    this.selectedUser.set(user);
  }
}
```

---

## Component Styling

### ViewEncapsulation

| Mode | Behavior |
|------|----------|
| `Emulated` (default) | Scopes styles to the component using attribute selectors |
| `ShadowDom` | Uses native Shadow DOM (true encapsulation) |
| `None` | No encapsulation — styles are global |

### :host and :host-context

```css
/* Style the host element itself */
:host {
  display: block;
  padding: 16px;
  border: 1px solid #e0e0e0;
}

/* Conditional host styling */
:host(.active) {
  border-color: #3b82f6;
}

/* Style based on ancestor */
:host-context(.dark-theme) {
  background: #1a1a1a;
  color: white;
}
```

### ::ng-deep (Deprecated — Use with Caution)

```css
/* Pierces encapsulation — apply to child components */
/* ⚠️ Deprecated: use it only as a last resort */
:host ::ng-deep .mat-form-field {
  width: 100%;
}
```

**Preferred alternatives to `::ng-deep`:**

- Use CSS custom properties (variables) for theming
- Use `ViewEncapsulation.None` for shared styles
- Pass style inputs to child components

---

## Dynamic Components

### Using @defer for Lazy Components

The simplest way to load components dynamically in modern Angular:

```html
@defer (when showEditor()) {
  <app-rich-editor [content]="content()" />
} @placeholder {
  <button (click)="showEditor.set(true)">Open Editor</button>
}
```

### Programmatic Dynamic Components

For advanced use cases with `ViewContainerRef`:

```typescript
import { Component, ViewContainerRef, inject } from '@angular/core';

@Component({
  selector: 'app-dynamic-host',
  standalone: true,
  template: `<ng-container #host />`,
})
export class DynamicHostComponent {
  private vcr = inject(ViewContainerRef);

  async loadComponent(componentType: 'chart' | 'table'): Promise<void> {
    this.vcr.clear();

    if (componentType === 'chart') {
      const { ChartComponent } = await import('./chart.component');
      this.vcr.createComponent(ChartComponent);
    } else {
      const { TableComponent } = await import('./table.component');
      this.vcr.createComponent(TableComponent);
    }
  }
}
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-SERVICES-AND-DEPENDENCY-INJECTION](02-SERVICES-AND-DEPENDENCY-INJECTION.md) | Services, DI, providers |
| 2 | [03-ROUTING-AND-NAVIGATION](03-ROUTING-AND-NAVIGATION.md) | Router config, lazy loading, guards |
| 3 | [04-FORMS](04-FORMS.md) | Template-driven and reactive forms |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial components and templates documentation |
