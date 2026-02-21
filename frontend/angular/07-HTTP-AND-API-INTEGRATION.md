# HTTP & API Integration

## Table of Contents

1. [Overview](#overview)
2. [HttpClient Setup](#httpclient-setup)
3. [Making HTTP Requests](#making-http-requests)
4. [Typed Responses](#typed-responses)
5. [HTTP Interceptors](#http-interceptors)
6. [Error Handling and Retry Strategies](#error-handling-and-retry-strategies)
7. [Request and Response Transformation](#request-and-response-transformation)
8. [Authentication Interceptors](#authentication-interceptors)
9. [Caching Strategies](#caching-strategies)
10. [File Upload and Download](#file-upload-and-download)
11. [Testing HTTP Requests](#testing-http-requests)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Angular's `HttpClient` provides a powerful, type-safe API for making HTTP requests. It returns Observables, integrates with interceptors for cross-cutting concerns, and is fully testable. This document covers setup, request patterns, functional interceptors, error handling, authentication, caching, and testing.

### Scope

- Setting up `HttpClient` with `provideHttpClient()`
- GET, POST, PUT, PATCH, DELETE requests
- Typed responses and response options
- Functional interceptors (Angular 15+)
- Error handling and retry with exponential backoff
- Request/response transformation
- JWT authentication interceptors
- Caching strategies
- File upload/download with progress tracking
- Testing with `HttpTestingController`

---

## HttpClient Setup

### Standalone Setup (Modern)

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, loggingInterceptor]),
      withFetch(),  // Use fetch API instead of XMLHttpRequest
    ),
  ],
};
```

### HttpClient Features

| Feature | Function | Purpose |
|---------|----------|---------|
| `withInterceptors()` | Register functional interceptors | Cross-cutting concerns |
| `withFetch()` | Use Fetch API backend | Modern HTTP, streaming support |
| `withXsrfConfiguration()` | Configure XSRF protection | CSRF token handling |
| `withJsonpSupport()` | Enable JSONP | Cross-domain requests (legacy) |
| `withRequestsMadeViaParent()` | Inherit parent interceptors | Lazy-loaded module integration |

---

## Making HTTP Requests

### GET Request

```typescript
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = '/api/users';

  getAll(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }

  getById(id: string): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }

  // With query parameters
  search(query: string, page: number = 1): Observable<PaginatedResponse<User>> {
    const params = new HttpParams()
      .set('q', query)
      .set('page', page.toString())
      .set('limit', '20');

    return this.http.get<PaginatedResponse<User>>(this.apiUrl, { params });
  }
}
```

### POST Request

```typescript
create(user: CreateUserDto): Observable<User> {
  return this.http.post<User>(this.apiUrl, user);
}
```

### PUT and PATCH Requests

```typescript
// Full update
update(id: string, user: User): Observable<User> {
  return this.http.put<User>(`${this.apiUrl}/${id}`, user);
}

// Partial update
patch(id: string, changes: Partial<User>): Observable<User> {
  return this.http.patch<User>(`${this.apiUrl}/${id}`, changes);
}
```

### DELETE Request

```typescript
delete(id: string): Observable<void> {
  return this.http.delete<void>(`${this.apiUrl}/${id}`);
}
```

### Custom Headers

```typescript
getProtected(): Observable<Data> {
  const headers = new HttpHeaders()
    .set('X-Custom-Header', 'value')
    .set('Accept', 'application/json');

  return this.http.get<Data>('/api/data', { headers });
}
```

---

## Typed Responses

### Response Body Only (Default)

```typescript
// Returns Observable<User[]>
this.http.get<User[]>('/api/users');
```

### Full Response (Headers, Status, etc.)

```typescript
this.http.get<User[]>('/api/users', { observe: 'response' }).pipe(
  map(response => {
    console.log('Status:', response.status);
    console.log('Total-Count:', response.headers.get('X-Total-Count'));
    return response.body ?? [];
  }),
);
```

### Text Response

```typescript
this.http.get('/api/health', { responseType: 'text' });
// Returns Observable<string>
```

### Blob Response (Binary Data)

```typescript
this.http.get('/api/reports/pdf', { responseType: 'blob' });
// Returns Observable<Blob>
```

---

## HTTP Interceptors

Interceptors process every HTTP request and/or response. Since Angular 15, **functional interceptors** are the preferred approach.

### Functional Interceptor

```typescript
import { HttpInterceptorFn } from '@angular/common/http';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const started = performance.now();

  return next(req).pipe(
    tap({
      next: () => {
        const elapsed = Math.round(performance.now() - started);
        console.log(`${req.method} ${req.url} completed in ${elapsed}ms`);
      },
      error: (error) => {
        const elapsed = Math.round(performance.now() - started);
        console.error(`${req.method} ${req.url} failed after ${elapsed}ms`, error);
      },
    }),
  );
};
```

### Registering Interceptors

```typescript
provideHttpClient(
  withInterceptors([
    authInterceptor,      // Runs first
    loggingInterceptor,   // Runs second
    errorInterceptor,     // Runs third
  ]),
),
```

### Request Modification

```typescript
export const contentTypeInterceptor: HttpInterceptorFn = (req, next) => {
  // Clone and modify the request (requests are immutable)
  if (!req.headers.has('Content-Type') && req.body) {
    const modified = req.clone({
      headers: req.headers.set('Content-Type', 'application/json'),
    });
    return next(modified);
  }
  return next(req);
};
```

---

## Error Handling and Retry Strategies

### Centralized Error Interceptor

```typescript
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let message = 'An unexpected error occurred';

      if (error.status === 0) {
        message = 'Network error — please check your connection';
      } else if (error.status === 401) {
        message = 'Session expired — please log in again';
        inject(Router).navigate(['/login']);
      } else if (error.status === 403) {
        message = 'You do not have permission to perform this action';
      } else if (error.status === 404) {
        message = 'The requested resource was not found';
      } else if (error.status >= 500) {
        message = 'Server error — please try again later';
      }

      inject(NotificationService).add(message, 'error');
      return throwError(() => error);
    }),
  );
};
```

### Retry with Exponential Backoff

```typescript
export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  // Only retry GET requests (safe to repeat)
  if (req.method !== 'GET') {
    return next(req);
  }

  return next(req).pipe(
    retry({
      count: 3,
      delay: (error, retryCount) => {
        // Don't retry client errors (4xx)
        if (error.status >= 400 && error.status < 500) {
          return throwError(() => error);
        }
        // Exponential backoff: 1s, 2s, 4s
        const delayMs = Math.pow(2, retryCount - 1) * 1000;
        return timer(delayMs);
      },
    }),
  );
};
```

---

## Request and Response Transformation

### API URL Prefix Interceptor

```typescript
export const apiPrefixInterceptor: HttpInterceptorFn = (req, next) => {
  const config = inject(APP_CONFIG);

  // Only modify relative URLs
  if (!req.url.startsWith('http')) {
    const apiReq = req.clone({
      url: `${config.apiUrl}${req.url}`,
    });
    return next(apiReq);
  }

  return next(req);
};
```

### Date Deserialization

```typescript
export const dateInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    map(event => {
      if (event instanceof HttpResponse && event.body) {
        return event.clone({ body: convertDates(event.body) });
      }
      return event;
    }),
  );
};

function convertDates(obj: any): any {
  if (obj === null || obj === undefined) return obj;
  if (typeof obj === 'string' && /^\d{4}-\d{2}-\d{2}T/.test(obj)) {
    return new Date(obj);
  }
  if (Array.isArray(obj)) return obj.map(convertDates);
  if (typeof obj === 'object') {
    return Object.fromEntries(
      Object.entries(obj).map(([key, value]) => [key, convertDates(value)])
    );
  }
  return obj;
}
```

---

## Authentication Interceptors

### JWT Token Interceptor

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getAccessToken();

  // Skip auth for public endpoints
  if (req.url.includes('/auth/login') || req.url.includes('/auth/register')) {
    return next(req);
  }

  if (token) {
    const authReq = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`),
    });
    return next(authReq);
  }

  return next(req);
};
```

### Token Refresh Interceptor

```typescript
export const tokenRefreshInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401 && !req.url.includes('/auth/refresh')) {
        return authService.refreshToken().pipe(
          switchMap(newToken => {
            const retryReq = req.clone({
              headers: req.headers.set('Authorization', `Bearer ${newToken}`),
            });
            return next(retryReq);
          }),
          catchError(() => {
            authService.logout();
            return throwError(() => error);
          }),
        );
      }
      return throwError(() => error);
    }),
  );
};
```

---

## Caching Strategies

### Simple In-Memory Cache Interceptor

```typescript
const cache = new Map<string, { response: HttpResponse<any>; expiry: number }>();

export const cacheInterceptor: HttpInterceptorFn = (req, next) => {
  // Only cache GET requests
  if (req.method !== 'GET') {
    return next(req);
  }

  const cached = cache.get(req.urlWithParams);
  if (cached && cached.expiry > Date.now()) {
    return of(cached.response.clone());
  }

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        cache.set(req.urlWithParams, {
          response: event.clone(),
          expiry: Date.now() + 60_000,  // Cache for 1 minute
        });
      }
    }),
  );
};
```

### Signal-Based Cache Service

```typescript
@Injectable({ providedIn: 'root' })
export class CachedApiService {
  private http = inject(HttpClient);

  private cache = new Map<string, WritableSignal<any>>();

  get<T>(url: string, ttlMs: number = 60_000): Signal<T | null> {
    if (!this.cache.has(url)) {
      const sig = signal<T | null>(null);
      this.cache.set(url, sig);
      this.fetchAndCache<T>(url, sig, ttlMs);
    }
    return this.cache.get(url)!.asReadonly();
  }

  private fetchAndCache<T>(url: string, sig: WritableSignal<T | null>, ttlMs: number): void {
    this.http.get<T>(url).subscribe(data => {
      sig.set(data);
      setTimeout(() => this.cache.delete(url), ttlMs);
    });
  }
}
```

---

## File Upload and Download

### File Upload with Progress

```typescript
@Injectable({ providedIn: 'root' })
export class FileUploadService {
  private http = inject(HttpClient);

  upload(file: File): Observable<UploadProgress> {
    const formData = new FormData();
    formData.append('file', file, file.name);

    return this.http.post('/api/upload', formData, {
      reportProgress: true,
      observe: 'events',
    }).pipe(
      map(event => {
        switch (event.type) {
          case HttpEventType.UploadProgress:
            const progress = event.total
              ? Math.round((event.loaded / event.total) * 100)
              : 0;
            return { status: 'uploading' as const, progress };
          case HttpEventType.Response:
            return { status: 'done' as const, progress: 100, body: event.body };
          default:
            return { status: 'pending' as const, progress: 0 };
        }
      }),
    );
  }
}
```

### File Download

```typescript
download(fileId: string, fileName: string): void {
  this.http.get(`/api/files/${fileId}`, { responseType: 'blob' }).subscribe(blob => {
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = fileName;
    link.click();
    URL.revokeObjectURL(url);
  });
}
```

---

## Testing HTTP Requests

### Setup

```typescript
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });
    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();  // Ensure no unexpected requests
  });
```

### Testing GET

```typescript
  it('should fetch all users', () => {
    const mockUsers: User[] = [
      { id: '1', name: 'Alice', email: 'alice@example.com' },
      { id: '2', name: 'Bob', email: 'bob@example.com' },
    ];

    service.getAll().subscribe(users => {
      expect(users.length).toBe(2);
      expect(users[0].name).toBe('Alice');
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
```

### Testing POST

```typescript
  it('should create a user', () => {
    const newUser: CreateUserDto = { name: 'Carol', email: 'carol@example.com' };
    const createdUser: User = { id: '3', ...newUser };

    service.create(newUser).subscribe(user => {
      expect(user.id).toBe('3');
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(newUser);
    req.flush(createdUser);
  });
```

### Testing Error Handling

```typescript
  it('should handle 404 error', () => {
    service.getById('999').subscribe({
      next: () => fail('Should have failed'),
      error: (error) => {
        expect(error.status).toBe(404);
      },
    });

    const req = httpMock.expectOne('/api/users/999');
    req.flush('Not Found', { status: 404, statusText: 'Not Found' });
  });
```

### Testing Interceptors

```typescript
  it('should add auth header', () => {
    spyOn(authService, 'getAccessToken').and.returnValue('test-token');

    service.getAll().subscribe();

    const req = httpMock.expectOne('/api/users');
    expect(req.request.headers.get('Authorization')).toBe('Bearer test-token');
    req.flush([]);
  });
});
```

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [08-TESTING](08-TESTING.md) | Comprehensive testing strategies |
| 2 | [06-STATE-MANAGEMENT](06-STATE-MANAGEMENT.md) | Managing server state |
| 3 | [05-RXJS-AND-OBSERVABLES](05-RXJS-AND-OBSERVABLES.md) | RxJS operators for HTTP streams |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial HTTP and API integration documentation |
