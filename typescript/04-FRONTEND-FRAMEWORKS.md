# Frontend Frameworks with TypeScript

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [React with TypeScript](#react-with-typescript)
   - [Component Typing](#component-typing)
   - [Props and Children](#props-and-children)
   - [State and Hooks](#state-and-hooks)
   - [Context and Providers](#context-and-providers)
   - [Event Handling](#event-handling)
3. [Angular with TypeScript](#angular-with-typescript)
   - [Components and Templates](#components-and-templates)
   - [Services and Dependency Injection](#services-and-dependency-injection)
   - [Modules and Routing](#modules-and-routing)
   - [Signals](#signals)
4. [Vue with TypeScript](#vue-with-typescript)
   - [Composition API and defineComponent](#composition-api-and-definecomponent)
   - [Script Setup](#script-setup)
   - [Pinia Stores](#pinia-stores)
5. [Shared Patterns Across Frameworks](#shared-patterns-across-frameworks)
   - [Type-Safe API Clients](#type-safe-api-clients)
   - [Routing with Type Safety](#routing-with-type-safety)
   - [State Management Typing](#state-management-typing)
6. [Type-Safe Forms and Validation](#type-safe-forms-and-validation)
   - [React Hook Form with Zod](#react-hook-form-with-zod)
   - [Angular Reactive Forms](#angular-reactive-forms)
   - [Vue Form Validation](#vue-form-validation)
7. [Component Library Typing Patterns](#component-library-typing-patterns)
   - [Polymorphic Components](#polymorphic-components)
   - [Generic Components](#generic-components)
   - [Slot and Render Prop Typing](#slot-and-render-prop-typing)
8. [Migration Strategies](#migration-strategies)
   - [Incremental Adoption](#incremental-adoption)
   - [Configuring Mixed Codebases](#configuring-mixed-codebases)
   - [Common Migration Pitfalls](#common-migration-pitfalls)
9. [Next Steps](#next-steps)
10. [Version History](#version-history)

---

## Overview

Modern frontend frameworks all provide first-class TypeScript support, but each takes a different approach. React relies on community-driven type definitions and JSX generics. Angular is built entirely on TypeScript with decorators at its core. Vue offers deep TypeScript integration through the Composition API and `<script setup>` syntax. This guide covers the essential typing patterns for all three frameworks, the shared patterns that apply across any frontend codebase, and practical strategies for migrating existing JavaScript projects.

### Target Audience

- **Frontend developers adopting TypeScript** — Developers with React, Angular, or Vue experience who want to add type safety to their components and application logic
- **Full-stack TypeScript engineers** — Developers who already use TypeScript on the backend and want consistent typing across the entire stack
- **Team leads evaluating frameworks** — Engineers comparing how each framework handles TypeScript integration, component APIs, and developer experience

### Scope

- React component typing including props, state, hooks, context, and event handling
- Angular components, services, modules, and the new Signals API
- Vue Composition API, `<script setup>`, and Pinia store typing
- Shared patterns for API clients, routing, and state management across frameworks
- Type-safe form handling and validation strategies
- Reusable component library patterns including polymorphic and generic components
- Step-by-step migration from JavaScript to TypeScript in existing projects

---

## React with TypeScript

### Component Typing

React components are functions that accept props and return JSX. TypeScript ensures that every component receives the correct data and renders predictable output.

```tsx
// Function component with an inline type
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
}

// Extracted props interface — preferred for components with multiple props
interface UserCardProps {
  user: {
    id: string;
    name: string;
    email: string;
    avatarUrl?: string;
  };
  onSelect: (userId: string) => void;
  variant?: "compact" | "detailed";
}

function UserCard({ user, onSelect, variant = "compact" }: UserCardProps) {
  return (
    <div className={`user-card user-card--${variant}`} onClick={() => onSelect(user.id)}>
      {user.avatarUrl && <img src={user.avatarUrl} alt={user.name} />}
      <h2>{user.name}</h2>
      {variant === "detailed" && <p>{user.email}</p>}
    </div>
  );
}
```

### Props and Children

The `children` prop and the `PropsWithChildren` utility type control what a component can render inside itself.

```tsx
import { type PropsWithChildren, type ReactNode } from "react";

// Explicit children typing — gives you full control over what children can be
interface PanelProps {
  title: string;
  children: ReactNode;
}

function Panel({ title, children }: PanelProps) {
  return (
    <section>
      <h2>{title}</h2>
      <div className="panel-body">{children}</div>
    </section>
  );
}

// PropsWithChildren utility — adds children automatically
type AlertProps = PropsWithChildren<{
  severity: "info" | "warning" | "error";
}>;

function Alert({ severity, children }: AlertProps) {
  return <div role="alert" className={`alert alert--${severity}`}>{children}</div>;
}

// Render prop pattern — children as a function
interface DataLoaderProps<T> {
  url: string;
  children: (data: T, isLoading: boolean) => ReactNode;
}

function DataLoader<T>({ url, children }: DataLoaderProps<T>) {
  const { data, isLoading } = useFetch<T>(url);
  return <>{children(data as T, isLoading)}</>;
}
```

### State and Hooks

TypeScript infers state types from initial values. For complex state or nullable values, provide an explicit type argument.

```tsx
import { useState, useReducer, useMemo, useCallback } from "react";

// Simple state — TypeScript infers `string` from the initial value
function SearchInput() {
  const [query, setQuery] = useState("");
  // query: string, setQuery: Dispatch<SetStateAction<string>>
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}

// Nullable state — requires explicit type
interface User {
  id: string;
  name: string;
  email: string;
}

function UserProfile() {
  const [user, setUser] = useState<User | null>(null);
  // user: User | null — TypeScript enforces null checks before accessing properties

  if (!user) return <p>Loading...</p>;

  // After the guard, TypeScript narrows the type to `User`
  return <p>{user.name} ({user.email})</p>;
}

// useReducer — typed actions ensure every dispatch is valid
interface CounterState {
  count: number;
  step: number;
}

type CounterAction =
  | { type: "increment" }
  | { type: "decrement" }
  | { type: "setStep"; payload: number }
  | { type: "reset" };

function counterReducer(state: CounterState, action: CounterAction): CounterState {
  switch (action.type) {
    case "increment":
      return { ...state, count: state.count + state.step };
    case "decrement":
      return { ...state, count: state.count - state.step };
    case "setStep":
      return { ...state, step: action.payload };
    case "reset":
      return { count: 0, step: 1 };
  }
}

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0, step: 1 });

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+{state.step}</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-{state.step}</button>
      <button onClick={() => dispatch({ type: "setStep", payload: 5 })}>Step = 5</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}

// Custom hook with explicit return type
function useDebounce<T>(value: T, delayMs: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useMemo(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delayMs);
    return () => clearTimeout(timer);
  }, [value, delayMs]);

  return debouncedValue;
}
```

### Context and Providers

React Context provides a way to pass data through the component tree without prop drilling. Typing the context value ensures consumers always get the right shape.

```tsx
import { createContext, useContext, useState, type ReactNode } from "react";

// Define the context shape
interface AuthContext {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
}

// Create context with undefined default — forces usage inside a provider
const AuthContext = createContext<AuthContext | undefined>(undefined);

// Custom hook that throws if used outside the provider
function useAuth(): AuthContext {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}

// Provider component
function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = async (email: string, password: string): Promise<void> => {
    const response = await fetch("/api/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password }),
    });
    const data: User = await response.json();
    setUser(data);
  };

  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout, isAuthenticated: !!user }}>
      {children}
    </AuthContext.Provider>
  );
}

// Consumer — TypeScript knows the exact shape of the context
function UserMenu() {
  const { user, logout, isAuthenticated } = useAuth();

  if (!isAuthenticated) return <button>Sign In</button>;

  return (
    <div>
      <span>Welcome, {user?.name}</span>
      <button onClick={logout}>Sign Out</button>
    </div>
  );
}
```

### Event Handling

React provides specific event types for every DOM event. Using these types catches mismatched handlers at compile time.

```tsx
import { type ChangeEvent, type FormEvent, type MouseEvent, type KeyboardEvent } from "react";

function LoginForm() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  // ChangeEvent is parameterized by the element type
  const handleEmailChange = (e: ChangeEvent<HTMLInputElement>) => {
    setEmail(e.target.value);
  };

  // FormEvent for form submission
  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    console.log("Submitting:", { email, password });
  };

  // MouseEvent with element specificity
  const handleButtonClick = (e: MouseEvent<HTMLButtonElement>) => {
    console.log("Clicked at:", e.clientX, e.clientY);
  };

  // KeyboardEvent for keyboard interactions
  const handleKeyDown = (e: KeyboardEvent<HTMLInputElement>) => {
    if (e.key === "Enter") {
      e.currentTarget.form?.requestSubmit();
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" value={email} onChange={handleEmailChange} onKeyDown={handleKeyDown} />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <button type="submit" onClick={handleButtonClick}>Log In</button>
    </form>
  );
}
```

---

## Angular with TypeScript

Angular is built on TypeScript from the ground up. Decorators, dependency injection, and template type checking are core to every Angular application.

### Components and Templates

Angular components use decorators to define metadata. TypeScript enforces the types of inputs, outputs, and template bindings.

```typescript
import { Component, Input, Output, EventEmitter } from "@angular/core";

interface Product {
  id: string;
  name: string;
  price: number;
  category: "electronics" | "clothing" | "books";
  inStock: boolean;
}

@Component({
  selector: "app-product-card",
  standalone: true,
  template: `
    <div class="product-card" [class.out-of-stock]="!product.inStock">
      <h3>{{ product.name }}</h3>
      <p class="price">{{ product.price | currency }}</p>
      <span class="badge">{{ product.category }}</span>
      <button (click)="onAddToCart()" [disabled]="!product.inStock">
        {{ product.inStock ? 'Add to Cart' : 'Out of Stock' }}
      </button>
    </div>
  `,
})
export class ProductCardComponent {
  @Input({ required: true }) product!: Product;
  @Output() addToCart = new EventEmitter<string>();

  onAddToCart(): void {
    this.addToCart.emit(this.product.id);
  }
}
```

### Services and Dependency Injection

Angular's DI system is fully typed. Services are injectable classes that provide shared logic across components.

```typescript
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable, BehaviorSubject, map, tap } from "rxjs";

interface ApiResponse<T> {
  data: T;
  total: number;
  page: number;
}

interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
}

@Injectable({ providedIn: "root" })
export class ProductService {
  private readonly apiUrl = "/api/products";
  private productsCache$ = new BehaviorSubject<Product[]>([]);

  constructor(private http: HttpClient) {}

  getProducts(page = 1): Observable<Product[]> {
    return this.http
      .get<ApiResponse<Product[]>>(this.apiUrl, { params: { page: page.toString() } })
      .pipe(
        map((response) => response.data),
        tap((products) => this.productsCache$.next(products))
      );
  }

  getProductById(id: string): Observable<Product> {
    return this.http.get<ApiResponse<Product>>(`${this.apiUrl}/${id}`).pipe(
      map((response) => response.data)
    );
  }

  createProduct(product: Omit<Product, "id">): Observable<Product> {
    return this.http.post<ApiResponse<Product>>(this.apiUrl, product).pipe(
      map((response) => response.data)
    );
  }

  // Expose the cache as a readonly observable
  get products$(): Observable<Product[]> {
    return this.productsCache$.asObservable();
  }
}
```

### Modules and Routing

Angular's router supports typed route parameters and guards that enforce navigation rules at compile time.

```typescript
import { Routes } from "@angular/router";
import { inject } from "@angular/core";
import { CanActivateFn, Router } from "@angular/router";

// Typed route guard
const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(["/login"]);
};

// Route configuration with typed data and resolve
interface ProductRouteData {
  title: string;
  requiresAuth: boolean;
}

const routes: Routes = [
  {
    path: "",
    loadComponent: () =>
      import("./home/home.component").then((m) => m.HomeComponent),
    data: { title: "Home", requiresAuth: false } satisfies ProductRouteData,
  },
  {
    path: "products",
    canActivate: [authGuard],
    loadComponent: () =>
      import("./products/product-list.component").then((m) => m.ProductListComponent),
    data: { title: "Products", requiresAuth: true } satisfies ProductRouteData,
  },
  {
    path: "products/:id",
    canActivate: [authGuard],
    loadComponent: () =>
      import("./products/product-detail.component").then((m) => m.ProductDetailComponent),
    data: { title: "Product Detail", requiresAuth: true } satisfies ProductRouteData,
  },
];

export default routes;
```

### Signals

Angular Signals provide a reactive primitive with full type inference. They replace many uses of RxJS for synchronous state management.

```typescript
import { Component, computed, signal, effect } from "@angular/core";

interface TodoItem {
  id: string;
  text: string;
  completed: boolean;
}

@Component({
  selector: "app-todo-list",
  standalone: true,
  template: `
    <h2>Todos ({{ remainingCount() }} remaining)</h2>
    <input #newTodo (keyup.enter)="addTodo(newTodo.value); newTodo.value = ''" />
    <ul>
      @for (todo of filteredTodos(); track todo.id) {
        <li [class.completed]="todo.completed">
          <input type="checkbox" [checked]="todo.completed" (change)="toggleTodo(todo.id)" />
          {{ todo.text }}
        </li>
      }
    </ul>
    <button (click)="filter.set('all')">All</button>
    <button (click)="filter.set('active')">Active</button>
    <button (click)="filter.set('completed')">Completed</button>
  `,
})
export class TodoListComponent {
  // Writable signals — TypeScript infers the type from the initial value
  todos = signal<TodoItem[]>([]);
  filter = signal<"all" | "active" | "completed">("all");

  // Computed signals — derived state that updates automatically
  remainingCount = computed(() => this.todos().filter((t) => !t.completed).length);

  filteredTodos = computed(() => {
    const currentFilter = this.filter();
    const allTodos = this.todos();

    switch (currentFilter) {
      case "active":
        return allTodos.filter((t) => !t.completed);
      case "completed":
        return allTodos.filter((t) => t.completed);
      default:
        return allTodos;
    }
  });

  constructor() {
    // Effects run whenever their signal dependencies change
    effect(() => {
      console.log(`Todos updated: ${this.todos().length} total, ${this.remainingCount()} remaining`);
    });
  }

  addTodo(text: string): void {
    if (!text.trim()) return;
    this.todos.update((current) => [
      ...current,
      { id: crypto.randomUUID(), text: text.trim(), completed: false },
    ]);
  }

  toggleTodo(id: string): void {
    this.todos.update((current) =>
      current.map((todo) => (todo.id === id ? { ...todo, completed: !todo.completed } : todo))
    );
  }
}
```

---

## Vue with TypeScript

Vue 3 provides excellent TypeScript support through the Composition API and `<script setup>` syntax. The type system integrates deeply with Vue's reactivity model.

### Composition API and defineComponent

The `defineComponent` function provides type inference for the Options API. It is the bridge between Vue's runtime and TypeScript's type checker.

```typescript
import { defineComponent, ref, computed, type PropType } from "vue";

interface Article {
  id: string;
  title: string;
  content: string;
  author: string;
  publishedAt: Date;
}

export default defineComponent({
  name: "ArticleCard",
  props: {
    article: {
      type: Object as PropType<Article>,
      required: true,
    },
    showFullContent: {
      type: Boolean,
      default: false,
    },
  },
  emits: {
    select: (articleId: string) => typeof articleId === "string",
    bookmark: (articleId: string, bookmarked: boolean) => true,
  },
  setup(props, { emit }) {
    // props.article is typed as Article
    const excerpt = computed(() => {
      if (props.showFullContent) return props.article.content;
      return props.article.content.slice(0, 200) + "...";
    });

    const isBookmarked = ref(false);

    function handleSelect(): void {
      emit("select", props.article.id);
    }

    function toggleBookmark(): void {
      isBookmarked.value = !isBookmarked.value;
      emit("bookmark", props.article.id, isBookmarked.value);
    }

    return { excerpt, isBookmarked, handleSelect, toggleBookmark };
  },
});
```

### Script Setup

The `<script setup>` syntax is the recommended approach for Vue 3 with TypeScript. It provides the cleanest integration with full type inference.

```typescript
// ArticleList.vue — <script setup lang="ts">
import { ref, computed } from "vue";

interface Article {
  id: string;
  title: string;
  content: string;
  tags: string[];
}

// defineProps with type-only declaration
const props = defineProps<{
  articles: Article[];
  maxVisible?: number;
}>();

// defineEmits with typed events
const emit = defineEmits<{
  select: [articleId: string];
  delete: [articleId: string];
  filterChange: [tag: string];
}>();

// withDefaults for optional props
const { maxVisible = 10 } = props;

// Reactive state
const searchQuery = ref("");
const selectedTag = ref<string | null>(null);

// Computed properties — fully typed from the props and state above
const filteredArticles = computed(() => {
  let result = props.articles;

  if (searchQuery.value) {
    const query = searchQuery.value.toLowerCase();
    result = result.filter(
      (a) => a.title.toLowerCase().includes(query) || a.content.toLowerCase().includes(query)
    );
  }

  if (selectedTag.value) {
    result = result.filter((a) => a.tags.includes(selectedTag.value!));
  }

  return result.slice(0, maxVisible);
});

const allTags = computed(() => {
  const tags = new Set(props.articles.flatMap((a) => a.tags));
  return Array.from(tags).sort();
});

function selectArticle(id: string): void {
  emit("select", id);
}

function filterByTag(tag: string): void {
  selectedTag.value = tag;
  emit("filterChange", tag);
}
```

### Pinia Stores

Pinia is Vue's official state management library. Its TypeScript integration provides full type inference for state, getters, and actions.

```typescript
import { defineStore } from "pinia";
import { ref, computed } from "vue";

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
}

// Setup store syntax — uses Composition API conventions
export const useCartStore = defineStore("cart", () => {
  // State
  const items = ref<CartItem[]>([]);
  const couponCode = ref<string | null>(null);
  const discountPercent = ref(0);

  // Getters
  const totalItems = computed(() => items.value.reduce((sum, item) => sum + item.quantity, 0));

  const subtotal = computed(() =>
    items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  );

  const discount = computed(() => subtotal.value * (discountPercent.value / 100));

  const total = computed(() => subtotal.value - discount.value);

  // Actions
  function addItem(product: Omit<CartItem, "quantity">): void {
    const existing = items.value.find((item) => item.productId === product.productId);
    if (existing) {
      existing.quantity += 1;
    } else {
      items.value.push({ ...product, quantity: 1 });
    }
  }

  function removeItem(productId: string): void {
    items.value = items.value.filter((item) => item.productId !== productId);
  }

  function updateQuantity(productId: string, quantity: number): void {
    const item = items.value.find((i) => i.productId === productId);
    if (item && quantity > 0) {
      item.quantity = quantity;
    } else if (item && quantity <= 0) {
      removeItem(productId);
    }
  }

  async function applyCoupon(code: string): Promise<boolean> {
    const response = await fetch(`/api/coupons/${code}`);
    if (!response.ok) return false;

    const data: { discount: number } = await response.json();
    couponCode.value = code;
    discountPercent.value = data.discount;
    return true;
  }

  function clearCart(): void {
    items.value = [];
    couponCode.value = null;
    discountPercent.value = 0;
  }

  return {
    items,
    couponCode,
    discountPercent,
    totalItems,
    subtotal,
    discount,
    total,
    addItem,
    removeItem,
    updateQuantity,
    applyCoupon,
    clearCart,
  };
});
```

---

## Shared Patterns Across Frameworks

Regardless of the framework, certain typing patterns appear in every frontend codebase. Extracting these into framework-agnostic modules keeps your code portable and consistent.

### Type-Safe API Clients

A typed API client ensures that every request and response conforms to your data model. This pattern works the same in React, Angular, and Vue.

```typescript
// Shared API client — framework-agnostic
interface ApiResponse<T> {
  data: T;
  meta: {
    total: number;
    page: number;
    pageSize: number;
  };
}

interface ApiError {
  code: string;
  message: string;
  details?: Record<string, string[]>;
}

type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

class ApiClient {
  constructor(private baseUrl: string, private getToken: () => string | null) {}

  private async request<T>(method: HttpMethod, path: string, body?: unknown): Promise<T> {
    const headers: Record<string, string> = {
      "Content-Type": "application/json",
    };

    const token = this.getToken();
    if (token) {
      headers["Authorization"] = `Bearer ${token}`;
    }

    const response = await fetch(`${this.baseUrl}${path}`, {
      method,
      headers,
      body: body ? JSON.stringify(body) : undefined,
    });

    if (!response.ok) {
      const error: ApiError = await response.json();
      throw new ApiRequestError(response.status, error);
    }

    return response.json() as Promise<T>;
  }

  get<T>(path: string): Promise<T> {
    return this.request<T>("GET", path);
  }

  post<T>(path: string, body: unknown): Promise<T> {
    return this.request<T>("POST", path, body);
  }

  put<T>(path: string, body: unknown): Promise<T> {
    return this.request<T>("PUT", path, body);
  }

  delete<T>(path: string): Promise<T> {
    return this.request<T>("DELETE", path);
  }
}

class ApiRequestError extends Error {
  constructor(public status: number, public error: ApiError) {
    super(error.message);
    this.name = "ApiRequestError";
  }
}

// Typed resource endpoints
interface User {
  id: string;
  name: string;
  email: string;
}

interface CreateUserInput {
  name: string;
  email: string;
  password: string;
}

class UserApi {
  constructor(private client: ApiClient) {}

  list(page = 1): Promise<ApiResponse<User[]>> {
    return this.client.get(`/users?page=${page}`);
  }

  getById(id: string): Promise<ApiResponse<User>> {
    return this.client.get(`/users/${id}`);
  }

  create(input: CreateUserInput): Promise<ApiResponse<User>> {
    return this.client.post("/users", input);
  }
}
```

### Routing with Type Safety

Type-safe route definitions prevent broken links and ensure route parameters match their expected types.

```typescript
// Framework-agnostic route definition pattern
interface RouteDefinition<Params extends Record<string, string> = Record<string, never>> {
  path: string;
  build: (params: Params) => string;
}

function defineRoute<Params extends Record<string, string> = Record<string, never>>(
  path: string
): RouteDefinition<Params> {
  return {
    path,
    build(params: Params): string {
      let result = path;
      for (const [key, value] of Object.entries(params)) {
        result = result.replace(`:${key}`, encodeURIComponent(value));
      }
      return result;
    },
  };
}

// Route map — single source of truth for all routes
const routes = {
  home: defineRoute("/"),
  products: defineRoute("/products"),
  productDetail: defineRoute<{ id: string }>("/products/:id"),
  userProfile: defineRoute<{ userId: string }>("/users/:userId"),
  settings: defineRoute("/settings"),
} as const;

// Usage — TypeScript enforces that the correct params are provided
routes.productDetail.build({ id: "abc-123" }); // "/products/abc-123"
routes.userProfile.build({ userId: "user-456" }); // "/users/user-456"
// routes.productDetail.build({}); // Compile error — missing 'id'
```

### State Management Typing

Whether you use Redux, NgRx, Pinia, or Zustand, discriminated unions for actions and strict state interfaces are universal patterns.

```typescript
// Typed state management — works with any state library
interface AppState {
  products: ProductState;
  cart: CartState;
  auth: AuthState;
}

interface ProductState {
  items: Product[];
  loading: boolean;
  error: string | null;
  selectedId: string | null;
}

// Discriminated union for actions
type ProductAction =
  | { type: "products/fetchStart" }
  | { type: "products/fetchSuccess"; payload: Product[] }
  | { type: "products/fetchError"; payload: string }
  | { type: "products/select"; payload: string }
  | { type: "products/clear" };

// Type-safe reducer
function productReducer(state: ProductState, action: ProductAction): ProductState {
  switch (action.type) {
    case "products/fetchStart":
      return { ...state, loading: true, error: null };
    case "products/fetchSuccess":
      return { ...state, loading: false, items: action.payload };
    case "products/fetchError":
      return { ...state, loading: false, error: action.payload };
    case "products/select":
      return { ...state, selectedId: action.payload };
    case "products/clear":
      return { ...state, items: [], selectedId: null };
  }
}

// Typed selectors
function selectProducts(state: AppState): Product[] {
  return state.products.items;
}

function selectSelectedProduct(state: AppState): Product | undefined {
  return state.products.items.find((p) => p.id === state.products.selectedId);
}
```

---

## Type-Safe Forms and Validation

Forms are one of the most error-prone areas in frontend development. TypeScript combined with schema validation libraries creates a robust barrier against invalid data.

### React Hook Form with Zod

React Hook Form's `useForm` generic parameter ties form fields directly to a Zod schema, keeping validation and types in sync.

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

// Define the schema once — it serves as both the validator and the type source
const registrationSchema = z
  .object({
    username: z.string().min(3, "Username must be at least 3 characters"),
    email: z.string().email("Invalid email address"),
    password: z.string().min(8, "Password must be at least 8 characters"),
    confirmPassword: z.string(),
    role: z.enum(["user", "admin", "editor"]),
    agreeToTerms: z.literal(true, {
      errorMap: () => ({ message: "You must agree to the terms" }),
    }),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords don't match",
    path: ["confirmPassword"],
  });

// Derive the TypeScript type from the schema
type RegistrationForm = z.infer<typeof registrationSchema>;

function RegistrationPage() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<RegistrationForm>({
    resolver: zodResolver(registrationSchema),
    defaultValues: {
      role: "user",
    },
  });

  const onSubmit = async (data: RegistrationForm): Promise<void> => {
    // data is fully typed and validated at this point
    await fetch("/api/register", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("username")} placeholder="Username" />
      {errors.username && <span>{errors.username.message}</span>}

      <input {...register("email")} placeholder="Email" type="email" />
      {errors.email && <span>{errors.email.message}</span>}

      <input {...register("password")} placeholder="Password" type="password" />
      {errors.password && <span>{errors.password.message}</span>}

      <input {...register("confirmPassword")} placeholder="Confirm password" type="password" />
      {errors.confirmPassword && <span>{errors.confirmPassword.message}</span>}

      <select {...register("role")}>
        <option value="user">User</option>
        <option value="admin">Admin</option>
        <option value="editor">Editor</option>
      </select>

      <label>
        <input type="checkbox" {...register("agreeToTerms")} />
        I agree to the terms
      </label>
      {errors.agreeToTerms && <span>{errors.agreeToTerms.message}</span>}

      <button type="submit" disabled={isSubmitting}>Register</button>
    </form>
  );
}
```

### Angular Reactive Forms

Angular's `FormBuilder` can be combined with strict typing to create forms where every control is type-checked.

```typescript
import { Component } from "@angular/core";
import { FormBuilder, Validators, ReactiveFormsModule } from "@angular/forms";

interface ContactForm {
  name: string;
  email: string;
  subject: string;
  message: string;
  priority: "low" | "medium" | "high";
}

@Component({
  selector: "app-contact-form",
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="name" placeholder="Name" />
      <input formControlName="email" placeholder="Email" type="email" />
      <select formControlName="priority">
        <option value="low">Low</option>
        <option value="medium">Medium</option>
        <option value="high">High</option>
      </select>
      <input formControlName="subject" placeholder="Subject" />
      <textarea formControlName="message" placeholder="Message"></textarea>
      <button type="submit" [disabled]="form.invalid">Send</button>
    </form>
  `,
})
export class ContactFormComponent {
  // NonNullable form builder ensures controls are never null
  form = this.fb.nonNullable.group({
    name: ["", [Validators.required, Validators.minLength(2)]],
    email: ["", [Validators.required, Validators.email]],
    subject: ["", Validators.required],
    message: ["", [Validators.required, Validators.minLength(10)]],
    priority: ["medium" as ContactForm["priority"], Validators.required],
  });

  constructor(private fb: FormBuilder) {}

  onSubmit(): void {
    if (this.form.valid) {
      // getRawValue returns the strongly-typed form value
      const value = this.form.getRawValue();
      // value.name: string, value.priority: "low" | "medium" | "high"
      console.log("Submitting:", value);
    }
  }
}
```

### Vue Form Validation

Vue's reactivity system pairs naturally with schema validation to create type-safe forms.

```typescript
// ContactForm.vue — <script setup lang="ts">
import { ref, computed } from "vue";
import { z } from "zod";

const schema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Invalid email"),
  message: z.string().min(10, "Message must be at least 10 characters"),
  category: z.enum(["general", "support", "billing"]),
});

type FormData = z.infer<typeof schema>;
type FormErrors = Partial<Record<keyof FormData, string>>;

const formData = ref<FormData>({
  name: "",
  email: "",
  message: "",
  category: "general",
});

const errors = ref<FormErrors>({});
const isSubmitting = ref(false);

const isValid = computed(() => {
  const result = schema.safeParse(formData.value);
  return result.success;
});

function validate(): boolean {
  const result = schema.safeParse(formData.value);
  if (result.success) {
    errors.value = {};
    return true;
  }

  const fieldErrors: FormErrors = {};
  for (const issue of result.error.issues) {
    const field = issue.path[0] as keyof FormData;
    fieldErrors[field] = issue.message;
  }
  errors.value = fieldErrors;
  return false;
}

async function handleSubmit(): Promise<void> {
  if (!validate()) return;
  isSubmitting.value = true;

  await fetch("/api/contact", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(formData.value),
  });

  isSubmitting.value = false;
}
```

---

## Component Library Typing Patterns

Building reusable component libraries requires advanced TypeScript patterns that keep APIs flexible without sacrificing type safety.

### Polymorphic Components

A polymorphic component renders as different HTML elements or other components while preserving the correct props for whatever element it becomes.

```tsx
import { type ElementType, type ComponentPropsWithoutRef } from "react";

// The "as" prop determines which element or component to render
type PolymorphicProps<E extends ElementType> = {
  as?: E;
} & ComponentPropsWithoutRef<E>;

function Box<E extends ElementType = "div">({
  as,
  ...props
}: PolymorphicProps<E>) {
  const Component = as || "div";
  return <Component {...props} />;
}

// Usage — TypeScript adjusts the available props based on the `as` value
function Example() {
  return (
    <>
      {/* Renders as <div>, accepts all div props */}
      <Box className="container">Content</Box>

      {/* Renders as <a>, accepts href and other anchor props */}
      <Box as="a" href="https://example.com">Link</Box>

      {/* Renders as <button>, accepts button-specific props */}
      <Box as="button" type="submit" onClick={() => {}}>Submit</Box>
    </>
  );
}
```

### Generic Components

Generic components accept a type parameter that flows through their props, making them reusable across different data shapes.

```tsx
import { useState, type ReactNode } from "react";

// Generic list component — works with any item type
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => ReactNode;
  keyExtractor: (item: T) => string;
  onItemClick?: (item: T) => void;
  emptyMessage?: string;
}

function List<T>({ items, renderItem, keyExtractor, onItemClick, emptyMessage }: ListProps<T>) {
  if (items.length === 0) {
    return <p>{emptyMessage ?? "No items to display"}</p>;
  }

  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)} onClick={() => onItemClick?.(item)}>
          {renderItem(item, index)}
        </li>
      ))}
    </ul>
  );
}

// Generic select component
interface SelectProps<T> {
  options: T[];
  value: T | null;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
  getValue: (option: T) => string;
  placeholder?: string;
}

function Select<T>({ options, value, onChange, getLabel, getValue, placeholder }: SelectProps<T>) {
  const selectedValue = value ? getValue(value) : "";

  return (
    <select
      value={selectedValue}
      onChange={(e) => {
        const selected = options.find((opt) => getValue(opt) === e.target.value);
        if (selected) onChange(selected);
      }}
    >
      {placeholder && <option value="">{placeholder}</option>}
      {options.map((opt) => (
        <option key={getValue(opt)} value={getValue(opt)}>
          {getLabel(opt)}
        </option>
      ))}
    </select>
  );
}

// Usage — TypeScript infers T from the `items` / `options` prop
interface Country {
  code: string;
  name: string;
  population: number;
}

function CountryPicker() {
  const [selected, setSelected] = useState<Country | null>(null);
  const countries: Country[] = [
    { code: "US", name: "United States", population: 331_000_000 },
    { code: "GB", name: "United Kingdom", population: 67_000_000 },
  ];

  return (
    <Select
      options={countries}
      value={selected}
      onChange={setSelected}
      getLabel={(c) => c.name}
      getValue={(c) => c.code}
      placeholder="Choose a country"
    />
  );
}
```

### Slot and Render Prop Typing

Advanced composition patterns allow consumers to customize rendering while maintaining type safety throughout.

```tsx
import { type ReactNode } from "react";

// Table component with typed column definitions
interface Column<T> {
  key: string;
  header: string;
  render: (item: T) => ReactNode;
  sortable?: boolean;
}

interface TableProps<T> {
  data: T[];
  columns: Column<T>[];
  keyExtractor: (item: T) => string;
  onRowClick?: (item: T) => void;
}

function Table<T>({ data, columns, keyExtractor, onRowClick }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={col.key}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((item) => (
          <tr key={keyExtractor(item)} onClick={() => onRowClick?.(item)}>
            {columns.map((col) => (
              <td key={col.key}>{col.render(item)}</td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Usage — columns are typed against the data shape
interface Employee {
  id: string;
  name: string;
  department: string;
  salary: number;
}

const employeeColumns: Column<Employee>[] = [
  { key: "name", header: "Name", render: (e) => e.name, sortable: true },
  { key: "dept", header: "Department", render: (e) => e.department },
  {
    key: "salary",
    header: "Salary",
    render: (e) => `$${e.salary.toLocaleString()}`,
    sortable: true,
  },
];

function EmployeeTable({ employees }: { employees: Employee[] }) {
  return (
    <Table
      data={employees}
      columns={employeeColumns}
      keyExtractor={(e) => e.id}
      onRowClick={(e) => console.log("Selected:", e.name)}
    />
  );
}
```

---

## Migration Strategies

Migrating an existing JavaScript codebase to TypeScript is a gradual process. The goal is to increase type coverage incrementally without breaking existing functionality.

### Incremental Adoption

Start by renaming files from `.js` to `.ts` (or `.jsx` to `.tsx`) and enabling `allowJs` so JavaScript and TypeScript files coexist.

```typescript
// tsconfig.json — configured for incremental migration
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": false,
    "allowJs": true,
    "checkJs": false,
    "outDir": "dist",
    "skipLibCheck": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true
  },
  "include": ["src"]
}
```

The recommended migration order focuses on high-value, low-risk files first:

1. **Shared types and interfaces** — create `.ts` files for data models used across the application
2. **Utility functions** — pure functions are the easiest to type and provide immediate safety
3. **API layer** — typing API calls prevents data shape mismatches between client and server
4. **State management** — typed stores catch action and selector errors
5. **Components** — start with leaf components and work upward toward page-level containers

### Configuring Mixed Codebases

During migration, gradually tighten the compiler settings as more files are converted.

```typescript
// Phase 1: Allow everything, add types where easy
{
  "compilerOptions": {
    "strict": false,
    "allowJs": true,
    "checkJs": false,
    "noImplicitAny": false
  }
}

// Phase 2: Start enforcing stricter checks
{
  "compilerOptions": {
    "strict": false,
    "allowJs": true,
    "checkJs": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}

// Phase 3: Full strict mode — all files are TypeScript
{
  "compilerOptions": {
    "strict": true,
    "allowJs": false,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### Common Migration Pitfalls

Knowing the common mistakes saves significant time during migration.

```typescript
// Pitfall 1: Overusing `any` to suppress errors
// Bad — defeats the purpose of TypeScript
function processData(data: any): any {
  return data.items.map((item: any) => item.name);
}

// Better — use `unknown` and narrow with type guards
function processData(data: unknown): string[] {
  if (!isDataResponse(data)) {
    throw new Error("Invalid data format");
  }
  return data.items.map((item) => item.name);
}

interface DataResponse {
  items: Array<{ name: string }>;
}

function isDataResponse(value: unknown): value is DataResponse {
  return (
    typeof value === "object" &&
    value !== null &&
    "items" in value &&
    Array.isArray((value as DataResponse).items)
  );
}

// Pitfall 2: Ignoring third-party type definitions
// Many JavaScript libraries have community types on DefinitelyTyped
// npm install --save-dev @types/lodash @types/react-router-dom

// Pitfall 3: Typing component props as a single large interface
// Bad — one massive interface that covers every variant
interface BadButtonProps {
  label: string;
  icon?: string;
  href?: string;
  onClick?: () => void;
  disabled?: boolean;
  type?: "button" | "submit";
  // What if href and onClick are both provided?
}

// Better — use discriminated unions to model exclusive variants
type ButtonProps =
  | { variant: "button"; onClick: () => void; disabled?: boolean; type?: "button" | "submit" }
  | { variant: "link"; href: string; external?: boolean };

type LabelledButtonProps = ButtonProps & { label: string; icon?: string };

// Pitfall 4: Not using the `satisfies` operator for configuration objects
// Without satisfies — TypeScript widens the type
const config = {
  apiUrl: "https://api.example.com",
  retries: 3,
  timeout: 5000,
};
// config.apiUrl is typed as `string` — loses the literal type

// With satisfies — validates the shape while preserving literal types
interface AppConfig {
  apiUrl: string;
  retries: number;
  timeout: number;
}

const config = {
  apiUrl: "https://api.example.com",
  retries: 3,
  timeout: 5000,
} satisfies AppConfig;
// config.apiUrl is typed as "https://api.example.com" — literal preserved
```

---

## Next Steps

Continue your TypeScript learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | TypeScript Fundamentals | Core language features, basic types, and project setup |
| [01-TYPE-SYSTEM.md](01-TYPE-SYSTEM.md) | Type System | Generics, utility types, conditional types, and type inference |
| [02-PATTERNS.md](02-PATTERNS.md) | Design Patterns | Common design patterns implemented in TypeScript |
| [03-NODE-AND-BACKEND.md](03-NODE-AND-BACKEND.md) | Node.js and Backend | Server-side TypeScript with Express, NestJS, Fastify, and databases |
| [05-TESTING.md](05-TESTING.md) | Testing | Unit testing, integration testing, and mocking in TypeScript |
| [06-TOOLING.md](06-TOOLING.md) | Tooling | Build tools, linters, and developer experience |
| [07-BEST-PRACTICES.md](07-BEST-PRACTICES.md) | Best Practices | Coding standards and recommended approaches |
| [08-ANTI-PATTERNS.md](08-ANTI-PATTERNS.md) | Anti-Patterns | Common mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Guided progression through all TypeScript topics |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Frontend Frameworks with TypeScript documentation |
