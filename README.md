# Exclusive E-Commerce Platform

> Public technical showcase for a full-stack e-commerce application built with Angular 17 and ASP.NET Core 9. This version is intentionally code-rich and designed for open review of architecture, implementation choices, and production-oriented engineering patterns.

![Angular](https://img.shields.io/badge/Angular-17-DD0031?style=flat-square&logo=angular)
![TypeScript](https://img.shields.io/badge/TypeScript-5.4-3178C6?style=flat-square&logo=typescript)
![RxJS](https://img.shields.io/badge/RxJS-7.8-B7178C?style=flat-square&logo=reactivex)
![.NET](https://img.shields.io/badge/.NET-9-512BD4?style=flat-square&logo=dotnet)
![EF Core](https://img.shields.io/badge/EF_Core-9-6DB33F?style=flat-square)
![SQL Server](https://img.shields.io/badge/SQL_Server-CC2927?style=flat-square&logo=microsoftsqlserver)
![SignalR](https://img.shields.io/badge/SignalR-Real--Time-F2C94C?style=flat-square)
![Stripe](https://img.shields.io/badge/Stripe-Payments-635BFF?style=flat-square&logo=stripe)

## Live Demos

- Storefront demo: https://exclusive-e-store.netlify.app
- Admin dashboard demo: https://e-commerce-admindashboard.netlify.app/login
- SwaggerUI: https://premium-commerce.runasp.net/swagger/index.html
- Admin demo login:
  Email: `admin@ecommerce.com`
  Password: `Password123!`

## Table of Contents

- [Overview](#overview)
- [Live Demos](#live-demos)
- [What the System Covers](#what-the-system-covers)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Frontend Design](#frontend-design)
- [Companion Admin Dashboard](#companion-admin-dashboard)
- [Backend Design](#backend-design)
- [Real Code Highlights](#real-code-highlights)
- [Data Integrity and Reliability Patterns](#data-integrity-and-reliability-patterns)
- [Project Structure](#project-structure)
- [Running Locally](#running-locally)
- [Configuration](#configuration)
- [Review Guide](#review-guide)

## Overview

This repository contains a full e-commerce implementation with a customer-facing Angular storefront and a .NET 9 Web API. In the broader workspace, the same backend also powers a companion Angular admin dashboard used for operations, inventory control, order management, and notifications.

Taken together, the system is a multi-frontend commerce platform:

- customer storefront in this repository
- companion admin dashboard in a sibling workspace/repository
- one shared backend API for both applications

The storefront project handles core commerce workflows end to end:

- product browsing and filtering
- category-based discovery
- wishlist and cart management
- authentication with refresh-token handling
- profile and order history
- product reviews
- Stripe checkout and payment confirmation
- real-time operational notifications via SignalR
- Arabic/English language switching with RTL/LTR support

The focus of this README is not marketing. It is meant to show how the system is composed, why certain patterns were chosen, and which parts of the code are worth reviewing first.

## What the System Covers

### Client

- Angular 17 standalone application
- functional HTTP interceptor pipeline
- signal-based local UI state
- `BehaviorSubject`-backed domain state
- route guards and token refresh
- Stripe Elements checkout flow
- runtime translation loading
- direction-aware AR/EN layout

### API

- ASP.NET Core 9 REST API
- EF Core 9 with SQL Server
- repository + unit of work abstractions
- centralized exception middleware
- optimistic concurrency support via `RowVersion`
- transactional order creation
- IMemoryCache-based server-side caching
- SignalR hub for notifications

### Companion Admin Dashboard

- Angular 21 standalone administrative frontend
- operational dashboards and recent-order visibility
- paginated order management with status changes
- product and category management on top of the same API
- JWT-based admin authentication with role checks
- real-time notification stream over SignalR
- persistent dark-mode preferences for long-running operational sessions

## Tech Stack

| Layer | Technologies |
| --- | --- |
| Frontend | Angular 17, TypeScript 5.4, RxJS 7.8, PrimeNG, Bootstrap 5, ngx-translate, ngx-stripe |
| Backend | ASP.NET Core 9, Entity Framework Core 9, SQL Server, SignalR, Serilog |
| Integrations | Stripe, Google OAuth, Cloudinary, SMTP email |
| Patterns | Functional interceptors, clean layering, repository/unit of work, optimistic concurrency, transaction boundaries |

## Architecture

```text
+-----------------------------------+    +-----------------------------------+
| Angular Storefront                |    | Angular Admin Dashboard           |
|                                   |    |                                   |
| Customer discovery and checkout   |    | Operations, inventory, orders     |
| cache -> loading -> auth -> error |    | loading -> auth -> error          |
| signals + BehaviorSubjects        |    | signals + SignalR notifications   |
+-------------------+---------------+    +-------------------+---------------+
                    |                                    |
                    +------------ HTTPS / JWT / JSON ----+
                                         |
                                         v
+--------------------------------------------------------------------+
| Shared ASP.NET Core 9 API                                          |
|                                                                    |
| Controllers -> services -> repositories -> EF Core                 |
| Global exception middleware                                        |
| Payment orchestration                                              |
| Admin endpoints and analytics                                      |
| Caching                                                            |
| SignalR notifications                                              |
+------------------------------+-------------------------------------+
                               |
                               v
                    SQL Server + External Services
              Stripe / Google OAuth / Cloudinary / Email
```

## Frontend Design

The frontend does not use a monolithic global store. State is intentionally split by responsibility:

- view-local state uses Angular `signal()`
- cross-feature state uses service-level `BehaviorSubject`
- request lifecycle concerns live in interceptors
- route access and bootstrap safety are handled in guards

That keeps the codebase predictable without forcing every workflow through a single store abstraction.


## Companion Admin Dashboard

The admin dashboard is a separate Angular frontend in the workspace that consumes the same backend APIs as the storefront. Functionally, it is the operational side of the platform: where products are managed, orders are monitored, customer-facing events are surfaced, and administrative actions are executed against the same domain model.

This companion app currently lives in the sibling workspace/repository `../admin-dashboard`, and its architecture mirrors the production concerns of the storefront while optimizing for internal operations:

- Angular 21 standalone application
- interceptor-driven HTTP pipeline
- role-aware auth guard that requires the `Admin` claim
- SignalR-backed notification feed
- signal-based dashboard and list state
- admin services for dashboard stats, users, orders, products, and categories
- dark-mode persistence for prolonged internal usage



## Backend Design

The API is structured into `Domain`, `Application`, `Infrastructure`, and `Api` layers.

- `Domain` contains entities, abstractions, and shared primitives such as `BaseEntity`.
- `Application` contains DTOs, contracts, validators, and configuration models.
- `Infrastructure` handles EF Core, repositories, services, mapping, caching, authentication, and external integrations.
- `Api` composes the application, exposes controllers, hosts middleware, and exposes the SignalR hub.



## Real Code Highlights

All snippets below are taken directly from this repository.

### 1. Interceptor Composition on the Client

The interceptor order is deliberate. Cache runs first so cached responses can short-circuit before the loading indicator appears. Auth then attaches tokens, and the global error layer stays last.

File: `client/src/app/app.config.ts`

```ts
provideHttpClient(
  withInterceptors([
    cacheInterceptor,
    loadingInterceptor,
    UserInterceptor,
    globalErrorInterceptor
  ]),
  withFetch(),
),
```

### 2. Client-Side LRU Cache with TTL

The cache service stores HTTP responses in memory, supports TTL-based expiration, and evicts least-recently-used entries when the configured capacity is hit.

File: `client/src/app/core/services/cache.service.ts`

```ts
export class CacheService {
  private cache = new Map<string, CacheEntry<any>>();
  private readonly maxEntries = 100;
  private readonly accessOrder: string[] = [];

  stats = signal<CacheStats>({ hits: 0, misses: 0, size: 0 });

  get<T>(key: string): T | null {
    const entry = this.cache.get(key);

    if (!entry) {
      this.updateStats('miss');
      return null;
    }

    if (Date.now() > entry.timestamp + entry.ttl) {
      this.delete(key);
      this.updateStats('miss');
      return null;
    }

    this.updateAccessOrder(key);
    this.updateStats('hit');

    return entry.data as T;
  }

  set<T>(key: string, data: T, ttlMs: number): void {
    if (this.cache.size >= this.maxEntries && !this.cache.has(key)) {
      this.evictLRU();
    }

    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl: ttlMs
    });

    this.updateAccessOrder(key);
    this.updateStats('size');
  }
}
```

### 3. Route-Aware HTTP Caching and Invalidation

Instead of caching every GET, the interceptor applies TTLs by endpoint class and skips mutable or sensitive resources such as auth, cart, orders, payments, and profile.

File: `client/src/app/core/interceptors/cache.interceptor.ts`

```ts
const CACHE_CONFIGS: CacheConfig[] = [
  { pattern: /\/categories(\?|$)/, ttl: 10 * 60 * 1000 },
  { pattern: /\/products\/flash-sales/, ttl: 1 * 60 * 1000 },
  { pattern: /\/products\/best-selling/, ttl: 5 * 60 * 1000 },
  { pattern: /\/products\/new-arrivals/, ttl: 5 * 60 * 1000 },
  { pattern: /\/products\/\d+$/, ttl: 5 * 60 * 1000 },
  { pattern: /\/products(\?|$)/, ttl: 2 * 60 * 1000 },
  { pattern: /\/wishlist/, ttl: 30 * 1000 },
];

const NO_CACHE_PATTERNS = [
  /\/auth\//,
  /\/cart/,
  /\/orders/,
  /\/payments/,
  /\/profile/,
  /\/notifications/,
];
```

```ts
export const cacheInterceptor: HttpInterceptorFn = (req, next) => {
  const cacheService = inject(CacheService);

  if (req.method !== 'GET') {
    if (['POST', 'PUT', 'DELETE', 'PATCH'].includes(req.method)) {
      invalidateRelatedCaches(req.url, cacheService);
    }
    return next(req);
  }

  const ttl = getCacheTTL(req.url);
  if (ttl === 0) return next(req);

  const cacheKey = getCacheKey(req.urlWithParams);
  const cachedResponse = cacheService.get<HttpResponse<any>>(cacheKey);
  if (cachedResponse) {
    return of(cachedResponse.clone());
  }

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse && event.status === 200) {
        cacheService.set(cacheKey, event.clone(), ttl);
      }
    })
  );
};
```

### 4. Refresh-Token Request Queueing

One of the most useful auth details in this codebase is the queued refresh pattern. If several requests fail with `401` at once, only one refresh call is allowed to execute. The rest wait for the refreshed token and retry.

File: `client/src/app/services/auth/interceptors/auth.interceptor.ts`

```ts
let isRefreshing = false;
let refreshSubject$ = new BehaviorSubject<string | null>(null);

if (error.status === 401 && user.refreshToken && !req.url.includes('/refresh-token')) {
  if (isRefreshing) {
    return refreshSubject$.pipe(
      filter(token => token !== null),
      take(1),
      switchMap(token => {
        const retryReq = req.clone({
          headers: req.headers.set('Authorization', `Bearer ${token}`)
        });
        return next(retryReq);
      })
    );
  }

  isRefreshing = true;
  refreshSubject$.next(null);

  return authService.refreshToken(false).pipe(
    switchMap(() => {
      const newUser = authService.user.value;
      isRefreshing = false;

      if (newUser && newUser.token) {
        refreshSubject$.next(newUser.token);
        const retryReq = req.clone({
          headers: req.headers.set('Authorization', `Bearer ${newUser.token}`)
        });
        return next(retryReq);
      }
      return throwError(() => error);
    })
  );
}
```

### 5. Guarding Against Auth Bootstrap Race Conditions

The guard waits for `autoLogin()` to finish before deciding whether the user is allowed through. That avoids false redirects during page reloads.

File: `client/src/app/services/auth/auth.guard.ts`

```ts
return auth.authReady$.pipe(
  filter(ready => ready),
  take(1),
  switchMap(() => auth.user.pipe(
    take(1),
    switchMap(user => {
      if (user && user.token) {
        return of(true);
      }

      if (user && user.refreshToken) {
        return auth.refreshToken(false).pipe(
          map(() => true as boolean | UrlTree),
          catchError(() => of(router.createUrlTree(['auth/login'])))
        );
      }

      return of(router.createUrlTree(['auth/login']));
    })
  ))
);
```

### 6. Payment-First Checkout

The checkout UI does not create the order before the payment is confirmed. It first creates a Stripe checkout session, stores the `clientSecret`, and only then moves the user to the payment step.

File: `client/src/app/pages/checkout-wizard/checkout-wizard.component.ts`

```ts
this.paymentsService.createCheckoutSession(checkoutRequest).pipe(
  tap((response) => {
    this.clientSecret.set(response.clientSecret);
    this.paymentIntentId.set(response.paymentIntentId);
    this.currentStep.set(2);
  }),
  finalize(() => {
    this.isLoading.set(false);
  })
).subscribe({
  error: (err) => {
    this.messageService.add({
      severity: 'error',
      summary: 'Checkout Error',
      detail: err.error?.message || 'Failed to create checkout session. Please try again.',
    });
  }
});
```

On the backend, the API creates the `PaymentIntent` and serializes essential checkout metadata into Stripe metadata so the order can be materialized after payment success.

File: `api/EcommerceApi.Infrastructure/Services/StripePaymentService.cs`

```csharp
var checkoutData = new
{
    UserId = userId,
    Items = request.Items.Select(i => new { i.ProductId, i.Quantity, i.UnitPrice }).ToList(),
    Shipping = request.Shipping,
    CustomerEmail = request.CustomerEmail,
    CustomerName = request.CustomerName
};

var checkoutJson = JsonSerializer.Serialize(checkoutData);

var options = new PaymentIntentCreateOptions
{
    Amount = (long)(totalAmount * 100),
    Currency = "usd",
    Description = $"Checkout for {request.CustomerName}",
    ReceiptEmail = request.CustomerEmail,
    AutomaticPaymentMethods = new PaymentIntentAutomaticPaymentMethodsOptions
    {
        Enabled = true,
    },
    Metadata = new Dictionary<string, string>
    {
        ["checkout_data"] = checkoutJson,
        ["user_id"] = userId.ToString(),
        ["created_at"] = DateTime.UtcNow.ToString("O")
    }
};
```

### 7. Idempotent Order Materialization After Payment

The webhook path uses `PaymentIntentId` as the idempotency key, so the same payment cannot produce duplicate orders if Stripe retries delivery.

File: `api/EcommerceApi.Infrastructure/Services/OrderService.cs`

```csharp
var existingOrder = await unitOfWork.Orders.Query()
    .Include(o => o.OrderItems)
    .ThenInclude(oi => oi.Product)
    .FirstOrDefaultAsync(o => o.StripePaymentIntentId == paymentIntentId, cancellationToken);

if (existingOrder != null)
{
    return existingOrder.Adapt<OrderDto>();
}
```

### 8. Transaction Boundaries in Order Creation

Order creation is treated as a business transaction, not a loose sequence of repository calls.

File: `api/EcommerceApi.Infrastructure/Services/OrderService.cs`

```csharp
await unitOfWork.BeginTransactionAsync(cancellationToken);

try
{
    await unitOfWork.Orders.AddAsync(order, cancellationToken);
    await unitOfWork.SaveChangesAsync(cancellationToken);

    foreach (var orderItem in orderItems)
    {
        orderItem.OrderId = order.Id;
        await unitOfWork.OrderItems.AddAsync(orderItem, cancellationToken);
    }

    await unitOfWork.SaveChangesAsync(cancellationToken);
    await unitOfWork.CommitAsync(cancellationToken);
}
catch
{
    await unitOfWork.RollbackAsync(cancellationToken);
    throw;
}
```

### 9. Server-Side Cache Stampede Prevention

The API cache layer prevents multiple concurrent misses from regenerating the same value repeatedly.

File: `api/EcommerceApi.Infrastructure/Services/MemoryCacheService.cs`

```csharp
private static readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();

public async Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiration = null)
{
    if (memoryCache.TryGetValue(key, out T? cachedValue))
    {
        logger.LogInformation("Cache HIT for {CacheKey}", key);
        return cachedValue!;
    }

    var lockObj = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));

    await lockObj.WaitAsync();
    try
    {
        if (memoryCache.TryGetValue(key, out cachedValue))
        {
            logger.LogInformation("Cache HIT (after lock) for {CacheKey}", key);
            return cachedValue!;
        }

        var value = await factory();
        await SetAsync(key, value, expiration);
        return value;
    }
    finally
    {
        lockObj.Release();
    }
}
```

### 10. Concurrency Protection on Product Updates

Products carry a `RowVersion` token, and EF Core is configured to treat it as a concurrency token.

File: `api/EcommerceApi.Infrastructure/Data/Configurations/ProductConfiguration.cs`

```csharp
builder.Property(p => p.RowVersion)
    .IsRowVersion();
```

The API then maps concurrency failures to a meaningful HTTP response.

File: `api/EcommerceApi.Api/Middleware/ExceptionHandlerMiddleware.cs`

```csharp
DbUpdateConcurrencyException => new
{
    StatusCode = (int)HttpStatusCode.Conflict,
    Response = ApiResponse<object>.ErrorResponse(
        "The record was modified by another request. Please refresh and try again.")
},
```

### 11. Real-Time Notification Broadcasts

Low-stock and new-order events are persisted, then broadcast to connected clients through SignalR.

File: `api/EcommerceApi.Api/Services/NotificationService.cs`

```csharp
_context.Notifications.Add(notification);
await _context.SaveChangesAsync();

await _hubContext.Clients.All.SendAsync("ReceiveNotification", notification);
```

### 12. Runtime Language Switching and Direction Control

The app supports English and Arabic using runtime translation loading and document-level direction changes.

File: `client/src/app/shared/components/header/header.component.ts`

```ts
onLangChange(): void {
  if (this.selectedLang) {
    this.translate.use(this.selectedLang.code);
    document.documentElement.dir =
      this.selectedLang.code === 'ar' ? 'rtl' : 'ltr';

    localStorage.setItem('selectedLang', this.selectedLang.code);

    if (this.selectedLang.code === 'ar') {
      document.body.classList.add('ar');
    } else {
      document.body.classList.remove('ar');
    }
  }
}
```

File: `client/src/app/translation.config.ts`

```ts
export function HttpLoaderFactory(http: HttpClient) {
  return new TranslateHttpLoader(http, './assets/i18n/', '.json');
}
```

### 13. Companion Admin Dashboard HTTP Pipeline

The admin app uses its own lean interceptor stack tuned for operational workflows. Unlike the storefront, it does not need public catalog caching, but it does need auth, loading feedback, and centralized error handling for privileged actions.

File: `../admin-dashboard/src/app/app.config.ts`

```ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes, withViewTransitions()),
    provideHttpClient(
      withInterceptors([
        loadingInterceptor,
        authInterceptor,
        globalErrorInterceptor
      ])
    )
  ]
};
```

### 14. Role-Based Admin Route Protection

The admin guard is explicit: a valid token is not enough. The decoded role claim must equal `Admin`, which keeps the operational frontend aligned with privileged API routes.

File: `../admin-dashboard/src/app/core/guards/auth.guard.ts`

```ts
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.getToken()) {
    const role = authService.getUserRole();
    if (role === 'Admin') {
      return true;
    }

    router.navigate(['/login']);
    return false;
  }

  router.navigate(['/login']);
  return false;
};
```

### 15. Admin SignalR Notifications on Top of the Same Backend Hub

The admin dashboard connects to the same `/notificationHub` exposed by the backend API. Notifications are stored in a signal, loaded initially from the REST API, and updated live from SignalR broadcasts.

File: `../admin-dashboard/src/app/core/services/signalr.service.ts`

```ts
notifications = signal<Notification[]>([]);
connectionStatus = signal<'disconnected' | 'connecting' | 'connected'>('disconnected');

startConnection() {
  this.connectionStatus.set('connecting');
  const hubUrl = environment.apiUrl.replace('/api', '') + '/notificationHub';

  this.hubConnection = new signalR.HubConnectionBuilder()
    .withUrl(hubUrl)
    .withAutomaticReconnect()
    .build();

  this.hubConnection
    .start()
    .then(() => {
      this.connectionStatus.set('connected');
      this.addVerifyListener();
      this.addNotificationListener();
      this.loadInitialNotifications();
    })
    .catch(() => {
      this.connectionStatus.set('disconnected');
      setTimeout(() => this.startConnection(), 5000);
    });
}
```

```ts
private addNotificationListener() {
  this.hubConnection?.on(
    'ReceiveNotification',
    (title: any, message: any, type: 'success' | 'error' | 'warning' | 'info') => {
      const safeTitle =
        typeof title === 'string' ? title : (title?.title || title?.Title || JSON.stringify(title));
      const safeMessage =
        typeof message === 'string' ? message : (message?.message || message?.Message || JSON.stringify(message));

      this.toastService.show(`${safeTitle}: ${safeMessage}`, type || 'info');
      setTimeout(() => this.loadInitialNotifications(), 500);
    }
  );
}
```

### 16. Shared Backend, Different UI Surface: Admin Analytics and Orders

The admin frontend consumes dedicated `/admin` endpoints on the same API for metrics, recent orders, user management, and order updates.

File: `../admin-dashboard/src/app/core/services/admin.service.ts`

```ts
export class AdminService {
  private http = inject(HttpClient);
  private apiUrl = `${environment.apiUrl}/admin`;

  getDashboardStats(): Observable<ApiResponse<DashboardStats>> {
    return this.http.get<ApiResponse<DashboardStats>>(`${this.apiUrl}/dashboard/stats`);
  }

  getRecentOrders(count = 5): Observable<ApiResponse<RecentOrder[]>> {
    return this.http.get<ApiResponse<RecentOrder[]>>(`${this.apiUrl}/dashboard/recent-orders`, {
      params: { count: count.toString() }
    });
  }

  getAllOrders(pageNumber = 1, pageSize = 10, status?: string): Observable<ApiResponse<PagedResult<AdminOrder>>> {
    let params = new HttpParams()
      .set('pageNumber', pageNumber.toString())
      .set('pageSize', pageSize.toString());

    if (status) {
      params = params.set('status', status);
    }

    return this.http.get<ApiResponse<PagedResult<AdminOrder>>>(`${this.apiUrl}/orders`, { params });
  }
}
```

File: `../admin-dashboard/src/app/features/dashboard/dashboard.component.ts`

```ts
stats = signal<DashboardStats | null>(null);
recentOrders = signal<RecentOrder[]>([]);
topProducts = signal<TopProduct[]>([]);

loadDashboardData(): void {
  this.adminService.getDashboardStats().subscribe({
    next: (res) => {
      if (res.success) this.stats.set(res.data);
    }
  });

  this.adminService.getRecentOrders(5).subscribe({
    next: (res) => {
      if (res.success) this.recentOrders.set(res.data);
    }
  });
}
```

File: `../admin-dashboard/src/app/features/orders/orders.component.ts`

```ts
updateStatus(orderId: number, status: string): void {
  this.adminService.updateOrderStatus(orderId, status).subscribe({
    next: () => {
      this.orders.update(o => ({
        ...o,
        items: o.items.map(item => item.id === orderId ? { ...item, status } : item)
      }));
    }
  });
}
```

### 17. Admin UX for Long Sessions: Theme Persistence

The admin settings screen persists dark mode in `localStorage` and updates the root element class immediately. It is a small implementation detail, but it matters in internal tools where operators often spend long uninterrupted sessions in the app.

File: `../admin-dashboard/src/app/features/settings/settings.component.ts`

```ts
darkMode = signal(localStorage.getItem('theme') === 'dark');

toggleDarkMode(): void {
  this.darkMode.update(v => !v);
  if (this.darkMode()) {
    document.documentElement.classList.add('dark');
    localStorage.setItem('theme', 'dark');
  } else {
    document.documentElement.classList.remove('dark');
    localStorage.setItem('theme', 'light');
  }
}
```

## Data Integrity and Reliability Patterns

The codebase demonstrates a few reliability patterns that matter in real commerce systems:

- transaction boundaries for multi-entity order writes
- optimistic concurrency support through `RowVersion`
- duplicate-order prevention through payment-intent idempotency
- consistent API error mapping through middleware
- client-side request deduplication behavior through refresh queueing
- server-side stampede prevention in cached read paths
- mutation-aware cache invalidation on the client
- role-based access separation between customer and admin frontends
- one SignalR notification backend serving both internal operations and customer-driven events

## Project Structure

```text
client/
  src/
    app/
      core/
      services/
      pages/
      shared/

api/
  EcommerceApi.Api/
  EcommerceApi.Application/
  EcommerceApi.Infrastructure/
  EcommerceApi.Domain/
