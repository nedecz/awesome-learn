# Routing & Navigation

## Table of Contents

1. [Overview](#overview)
2. [Router Setup and Configuration](#router-setup-and-configuration)
3. [Route Parameters and Query Parameters](#route-parameters-and-query-parameters)
4. [Child Routes and Nested Routing](#child-routes-and-nested-routing)
5. [Lazy Loading](#lazy-loading)
6. [Route Guards](#route-guards)
7. [Functional Guards](#functional-guards)
8. [Router Events and Navigation Lifecycle](#router-events-and-navigation-lifecycle)
9. [Preloading Strategies](#preloading-strategies)
10. [Route Resolvers](#route-resolvers)
11. [Named Outlets and Auxiliary Routes](#named-outlets-and-auxiliary-routes)
12. [URL Serialization and Custom URL Handling](#url-serialization-and-custom-url-handling)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

Angular's router maps URL paths to components, enabling navigation in single-page applications without full page reloads. This document covers route configuration, lazy loading with standalone components, functional guards and resolvers, preloading strategies, and advanced routing patterns.

### Scope

- Setting up the router with `provideRouter()` and standalone components
- Route parameters (`:id`), query parameters, and fragments
- Child routes and nested `<router-outlet>`
- Lazy loading with `loadComponent` and `loadChildren`
- Route guards: `canActivate`, `canDeactivate`, `canMatch`, `resolve`
- Functional guards (modern approach since Angular 15)
- Router events and the navigation lifecycle
- Preloading strategies for faster navigation
- Route resolvers for data pre-fetching
- Named outlets and auxiliary routes

---

## Router Setup and Configuration

### Providing the Router (Standalone)

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),  // Bind route params to component inputs
    ),
  ],
};
```

### Defining Routes

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  { path: 'users/:id', component: UserDetailComponent },
  { path: '**', component: NotFoundComponent },  // Wildcard — must be last
];
```

### Router Outlet

The `<router-outlet>` directive marks where routed components are rendered:

```typescript
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, NavComponent],
  template: `
    <app-nav />
    <main>
      <router-outlet />
    </main>
  `,
})
export class AppComponent {}
```

### Navigation

```typescript
// Template-based navigation
<a routerLink="/users" routerLinkActive="active">Users</a>
<a [routerLink]="['/users', userId()]">User Detail</a>

// Programmatic navigation
private router = inject(Router);

navigateToUser(id: string): void {
  this.router.navigate(['/users', id]);
}

navigateWithQuery(): void {
  this.router.navigate(['/search'], {
    queryParams: { q: 'angular', page: 1 },
  });
}
```

---

## Route Parameters and Query Parameters

### Route Parameters

```typescript
// Route definition
{ path: 'users/:id', component: UserDetailComponent }

// Component — with input binding (recommended)
@Component({ /* ... */ })
export class UserDetailComponent {
  id = input.required<string>();  // Automatically bound from :id
}

// Component — with ActivatedRoute (legacy)
@Component({ /* ... */ })
export class UserDetailComponent implements OnInit {
  private route = inject(ActivatedRoute);

  ngOnInit(): void {
    // Snapshot (non-reactive — value at navigation time)
    const id = this.route.snapshot.paramMap.get('id');

    // Observable (reactive — updates on param change)
    this.route.paramMap.subscribe(params => {
      const id = params.get('id');
    });
  }
}
```

### Query Parameters

```typescript
// Navigation with query params
<a [routerLink]="['/search']" [queryParams]="{ q: 'angular', page: 1 }">Search</a>

// Reading query params (with input binding)
@Component({ /* ... */ })
export class SearchComponent {
  q = input<string>('');
  page = input<number>(1);
}

// Reading query params (with ActivatedRoute)
private route = inject(ActivatedRoute);
query$ = this.route.queryParamMap.pipe(
  map(params => params.get('q') ?? '')
);
```

### Route Data and Title

```typescript
// Static route data
{
  path: 'dashboard',
  component: DashboardComponent,
  title: 'Dashboard',
  data: { role: 'admin' },
}

// Dynamic title
{
  path: 'users/:id',
  component: UserDetailComponent,
  title: (route: ActivatedRouteSnapshot) => {
    return `User ${route.paramMap.get('id')}`;
  },
}
```

---

## Child Routes and Nested Routing

### Defining Child Routes

```typescript
export const routes: Routes = [
  {
    path: 'settings',
    component: SettingsLayoutComponent,
    children: [
      { path: '', redirectTo: 'profile', pathMatch: 'full' },
      { path: 'profile', component: ProfileSettingsComponent },
      { path: 'security', component: SecuritySettingsComponent },
      { path: 'notifications', component: NotificationSettingsComponent },
    ],
  },
];
```

### Layout Component with Nested Outlet

```typescript
@Component({
  selector: 'app-settings-layout',
  standalone: true,
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <div class="settings-layout">
      <nav class="sidebar">
        <a routerLink="profile" routerLinkActive="active">Profile</a>
        <a routerLink="security" routerLinkActive="active">Security</a>
        <a routerLink="notifications" routerLinkActive="active">Notifications</a>
      </nav>
      <section class="content">
        <router-outlet />
      </section>
    </div>
  `,
})
export class SettingsLayoutComponent {}
```

---

## Lazy Loading

Lazy loading defers the download of route code until the user navigates to that route, reducing the initial bundle size.

### Lazy Loading a Standalone Component

```typescript
{
  path: 'admin',
  loadComponent: () =>
    import('./admin/admin.component').then(m => m.AdminComponent),
}
```

### Lazy Loading a Set of Child Routes

```typescript
{
  path: 'admin',
  loadChildren: () =>
    import('./admin/admin.routes').then(m => m.adminRoutes),
}
```

```typescript
// admin/admin.routes.ts
export const adminRoutes: Routes = [
  { path: '', component: AdminDashboardComponent },
  { path: 'users', component: AdminUsersComponent },
  { path: 'settings', component: AdminSettingsComponent },
];
```

### Lazy Loading with Route-Level Providers

```typescript
{
  path: 'editor',
  loadChildren: () => import('./editor/editor.routes').then(m => m.editorRoutes),
  providers: [EditorStateService],  // Scoped to this lazy-loaded section
}
```

### Lazy Loading Comparison

| Approach | Loads | Use When |
|----------|-------|----------|
| `loadComponent` | Single component | Simple route with one component |
| `loadChildren` | Set of routes | Feature with multiple sub-routes |
| `@defer` (in template) | Template section | Deferred content within a component |

---

## Route Guards

Guards control access to routes. Angular supports several guard types:

| Guard | When It Runs | Purpose |
|-------|-------------|---------|
| `canActivate` | Before entering a route | Check if user can access the route |
| `canActivateChild` | Before entering a child route | Check access to all child routes |
| `canDeactivate` | Before leaving a route | Confirm unsaved changes |
| `canMatch` | Before matching the route | Conditionally match routes |
| `resolve` | Before route activation | Pre-fetch data |

### Class-Based Guard (Legacy)

```typescript
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  private authService = inject(AuthService);
  private router = inject(Router);

  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): boolean | UrlTree {
    if (this.authService.isAuthenticated()) {
      return true;
    }
    return this.router.createUrlTree(['/login'], {
      queryParams: { returnUrl: state.url },
    });
  }
}

// Usage
{ path: 'admin', component: AdminComponent, canActivate: [AuthGuard] }
```

---

## Functional Guards

Functional guards are the **modern, preferred approach since Angular 15**. They are simpler, tree-shakable, and require less boilerplate.

### canActivate (Functional)

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url },
  });
};

// Usage
{ path: 'admin', component: AdminComponent, canActivate: [authGuard] }
```

### canDeactivate (Functional)

```typescript
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm('You have unsaved changes. Do you really want to leave?');
  }
  return true;
};
```

### canMatch (Functional)

`canMatch` determines whether a route should be matched at all, enabling feature-flag-based routing:

```typescript
export const adminMatchGuard: CanMatchFn = () => {
  const authService = inject(AuthService);
  return authService.hasRole('admin');
};

// Two routes for the same path — canMatch selects the right one
{
  path: 'dashboard',
  loadComponent: () => import('./admin-dashboard.component'),
  canMatch: [adminMatchGuard],
},
{
  path: 'dashboard',
  loadComponent: () => import('./user-dashboard.component'),
},
```

### Composing Guards

```typescript
// Combine multiple guards
{ 
  path: 'admin/settings',
  component: AdminSettingsComponent,
  canActivate: [authGuard, roleGuard('admin')],
}

// Role guard factory
export function roleGuard(requiredRole: string): CanActivateFn {
  return () => {
    const authService = inject(AuthService);
    return authService.hasRole(requiredRole);
  };
}
```

---

## Router Events and Navigation Lifecycle

The router emits events throughout the navigation lifecycle:

```
NavigationStart
    │
    ▼
RoutesRecognized
    │
    ▼
GuardsCheckStart
    │
    ▼
GuardsCheckEnd
    │
    ▼
ResolveStart
    │
    ▼
ResolveEnd
    │
    ▼
NavigationEnd (or NavigationCancel / NavigationError)
```

### Listening to Router Events

```typescript
@Component({ /* ... */ })
export class AppComponent {
  private router = inject(Router);

  isLoading = signal(false);

  constructor() {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        this.isLoading.set(true);
      }
      if (
        event instanceof NavigationEnd ||
        event instanceof NavigationCancel ||
        event instanceof NavigationError
      ) {
        this.isLoading.set(false);
      }
    });
  }
}
```

### Using withNavigationErrorHandler

```typescript
provideRouter(
  routes,
  withNavigationErrorHandler((error: NavigationError) => {
    const router = inject(Router);
    console.error('Navigation error:', error);
    router.navigate(['/error']);
  }),
),
```

---

## Preloading Strategies

Preloading strategies load lazy-loaded routes in the background after the initial navigation completes.

| Strategy | Behavior |
|----------|----------|
| `NoPreloading` (default) | Load only when navigated to |
| `PreloadAllModules` | Load all lazy routes after initial load |
| Custom strategy | Load specific routes based on criteria |

### PreloadAllModules

```typescript
provideRouter(
  routes,
  withPreloading(PreloadAllModules),
),
```

### Custom Preloading Strategy

```typescript
@Injectable({ providedIn: 'root' })
export class SelectivePreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Only preload routes marked with data.preload = true
    if (route.data?.['preload']) {
      return load();
    }
    return of(null);
  }
}

// Route config
{ path: 'dashboard', loadChildren: () => import('./dashboard.routes'), data: { preload: true } },
{ path: 'admin', loadChildren: () => import('./admin.routes') },  // Not preloaded
```

---

## Route Resolvers

Resolvers pre-fetch data before a route activates, ensuring the component has data on initialization.

### Functional Resolver (Preferred)

```typescript
// user.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';

export const userResolver: ResolveFn<User> = (route) => {
  const userService = inject(UserService);
  const id = route.paramMap.get('id')!;
  return userService.getById(id);
};

// Route config
{
  path: 'users/:id',
  component: UserDetailComponent,
  resolve: { user: userResolver },
}

// Component — access resolved data
@Component({ /* ... */ })
export class UserDetailComponent {
  user = input.required<User>();  // With withComponentInputBinding()
}
```

### Resolver with Error Handling

```typescript
export const userResolver: ResolveFn<User | null> = (route) => {
  const userService = inject(UserService);
  const router = inject(Router);
  const id = route.paramMap.get('id')!;

  return userService.getById(id).pipe(
    catchError(() => {
      router.navigate(['/not-found']);
      return of(null);
    }),
  );
};
```

---

## Named Outlets and Auxiliary Routes

Named outlets allow rendering multiple routes simultaneously in different parts of the layout.

### Defining Named Outlets

```html
<!-- app.component.html -->
<router-outlet />                        <!-- Primary outlet -->
<router-outlet name="sidebar" />         <!-- Named outlet -->
<router-outlet name="dialog" />          <!-- Named outlet -->
```

### Route Configuration

```typescript
export const routes: Routes = [
  { path: 'dashboard', component: DashboardComponent },
  { path: 'help', component: HelpPanelComponent, outlet: 'sidebar' },
  { path: 'confirm', component: ConfirmDialogComponent, outlet: 'dialog' },
];
```

### Navigation to Named Outlets

```html
<!-- Template -->
<a [routerLink]="[{ outlets: { sidebar: ['help'] } }]">Open Help</a>
<a [routerLink]="[{ outlets: { sidebar: null } }]">Close Help</a>
```

```typescript
// Programmatic
this.router.navigate([{ outlets: { sidebar: ['help'] } }]);
```

> **Note:** Named outlets add URL complexity. For most use cases, prefer conditional rendering with `@if` or dialog services instead.

---

## URL Serialization and Custom URL Handling

### URL Structure

```
https://example.com/users/42?page=2&sort=name#section1
                    ──────── ───────────────── ────────
                     path     query params      fragment
```

### Custom URL Serializer

Override default URL parsing for special characters or formats:

```typescript
import { UrlSerializer, UrlTree, DefaultUrlSerializer } from '@angular/router';

export class CustomUrlSerializer implements UrlSerializer {
  private default = new DefaultUrlSerializer();

  parse(url: string): UrlTree {
    // Custom parsing logic (e.g., decode special characters)
    url = url.replace(/\+/g, '%20');
    return this.default.parse(url);
  }

  serialize(tree: UrlTree): string {
    return this.default.serialize(tree);
  }
}

// Register
bootstrapApplication(AppComponent, {
  providers: [
    { provide: UrlSerializer, useClass: CustomUrlSerializer },
  ],
});
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-FORMS](04-FORMS.md) | Template-driven and reactive forms |
| 2 | [02-SERVICES-AND-DEPENDENCY-INJECTION](02-SERVICES-AND-DEPENDENCY-INJECTION.md) | Services and DI for route guards and resolvers |
| 3 | [09-PERFORMANCE](09-PERFORMANCE.md) | Lazy loading and preloading for performance |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial routing and navigation documentation |
