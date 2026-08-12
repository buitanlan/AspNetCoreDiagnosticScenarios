# Table of contents

- [.NET patterns](#net-patterns)
  - [Register and resolve](#register-and-resolve)
    - [`Add{Feature}` extension methods](#addfeature-extension-methods)
    - [Fluent module / vertical slice registration](#fluent-module--vertical-slice-registration)
    - [`TryAdd` / `TryAddEnumerable`](#tryadd--tryaddenumerable)
    - [Generic types as a factory](#generic-types-as-a-factory)
    - [Lazy initialization of services](#lazy-initialization-of-services)
    - [Resolve by delegate (`Func<T>` / named delegates)](#resolve-by-delegate-funct--named-delegates)
    - [Creating instances of types from an `IServiceProvider`](#creating-instances-of-types-from-an-iserviceprovider)
    - [Single implementation, multiple interfaces](#single-implementation-multiple-interfaces)
    - [Keyed services (.NET 8+)](#keyed-services-net-8)
    - [Multiple implementations via `IEnumerable<T>`](#multiple-implementations-via-ienumerablet)
    - [Decorator around an existing registration](#decorator-around-an-existing-registration)
    - [Open-generic handlers (strategy per `T`)](#open-generic-handlers-strategy-per-t)
    - [Composite over `IEnumerable<T>`](#composite-over-ienumerablet)
    - [`IServiceProviderIsService`](#iserviceproviderisservice)
  - [Lifetimes, scopes, and tenancy](#lifetimes-scopes-and-tenancy)
    - [Caching singletons in generic types](#caching-singletons-in-generic-types)
    - [Scoped work from a singleton (`IServiceScopeFactory`)](#scoped-work-from-a-singleton-iservicescopefactory)
    - [`IDbContextFactory<T>` / async factories](#idbcontextfactoryt--async-factories)
    - [`IAsyncDisposable` services](#iasyncdisposable-services)
    - [Multi-tenant: singleton vs scoped](#multi-tenant-singleton-vs-scoped)
  - [Options and configuration](#options-and-configuration)
    - [Resolving services when using `IOptions<T>`](#resolving-services-when-using-ioptionst)
    - [Named options, `IOptionsMonitor`, and `IOptionsSnapshot`](#named-options-ioptionsmonitor-and-ioptionssnapshot)
    - [Validate options at start](#validate-options-at-start)
    - [`IChangeToken` (reload without polling)](#ichangetoken-reload-without-polling)
    - [`IConfigurationProvider`](#iconfigurationprovider)
  - [HTTP clients](#http-clients)
    - [Typed and named `HttpClient`](#typed-and-named-httpclient)
    - [`HttpMessageHandler` pipeline](#httpmessagehandler-pipeline)
    - [Http resilience](#http-resilience)
  - [Hosting and background work](#hosting-and-background-work)
    - [`IHostedService` / `BackgroundService`](#ihostedservice--backgroundservice)
    - [Singleton `Channel<T>` producer/consumer](#singleton-channelt-producerconsumer)
    - [`IHostApplicationLifetime`](#ihostapplicationlifetime)
  - [ASP.NET Core pipeline](#aspnet-core-pipeline)
    - [`IStartupFilter` (library middleware)](#istartupfilter-library-middleware)
    - [Factory-activated `IMiddleware`](#factory-activated-imiddleware)
    - [Endpoint filters](#endpoint-filters)
    - [`IFeatureCollection`](#ifeaturecollection)
    - [`IExceptionHandler` / `IProblemDetailsService`](#iexceptionhandler--iproblemdetailsservice)
    - [`AuthenticationHandler<TOptions>` / `AuthorizationHandler<T>`](#authenticationhandlertoptions--authorizationhandlert)
    - [Rate limiter partitions](#rate-limiter-partitions)
    - [`IHealthCheck`](#ihealthcheck)
  - [Persistence and domain](#persistence-and-domain)
    - [Unit of work / transaction scope](#unit-of-work--transaction-scope)
    - [Strongly-typed IDs](#strongly-typed-ids)
  - [Caching](#caching)
    - [Cache-aside (`IMemoryCache` / `HybridCache`)](#cache-aside-imemorycache--hybridcache)
    - [`FrozenDictionary` for startup lookups](#frozendictionary-for-startup-lookups)
  - [Observability](#observability)
    - [Source-generated logging (`LoggerMessage`)](#source-generated-logging-loggermessage)
    - [`NullLogger` in libraries](#nulllogger-in-libraries)
    - [`ActivitySource` / `IMeterFactory`](#activitysource--imeterfactory)
  - [Performance and library primitives](#performance-and-library-primitives)
    - [`TimeProvider`](#timeprovider)
    - [Object pooling](#object-pooling)
    - [`IPooledObjectPolicy<T>`](#ipooledobjectpolicyt)
    - [`IBufferWriter<T>` / `PipeReader`](#ibufferwritert--pipereader)
    - [`StringValues` / `StringSegment`](#stringvalues--stringsegment)
    - [`JsonSerializerContext` (source-gen JSON)](#jsonserializercontext-source-gen-json)
    - [Library `ConfigureAwait(false)`](#library-configureawaitfalse)
- [Related guides](#related-guides)

# .NET patterns

These are patterns used throughout **Microsoft.Extensions.\*** and **Microsoft.AspNetCore.\***. They are how you *structure* DI, options, and factories—not HTTP-pipeline pitfalls. For those see [AspNetCoreGuidance.md](AspNetCoreGuidance.md), [AsyncGuidance.md](AsyncGuidance.md), and [HttpClientGuidance.md](HttpClientGuidance.md).

## Register and resolve

How the composition root registers types, and how callers resolve them without a service locator.

### `Add{Feature}` extension methods

Every Microsoft.Extensions library is an `IServiceCollection` extension: `AddLogging`, `AddHttpClient`, `AddOptions`. Apps and your own libraries should do the same so registration is one call and defaults use `TryAdd`.

```C#
public static class InventoryServiceCollectionExtensions
{
    public static IServiceCollection AddInventory(this IServiceCollection services, IConfiguration config)
    {
        services.AddOptions<InventoryOptions>()
            .BindConfiguration("Inventory")
            .ValidateDataAnnotations()
            .ValidateOnStart();

        services.TryAddSingleton<IInventory, Inventory>();
        services.AddHttpClient<IStockClient, StockClient>();
        services.AddHostedService<InventoryWarmup>();
        return services;
    }
}

builder.Services.AddInventory(builder.Configuration);
```

Keep side effects in `Add*`. Do not `BuildServiceProvider()` inside the extension.

### Fluent module / vertical slice registration

A giant `Program.cs` that registers every handler, client, and endpoint does not scale. Each **feature** (vertical slice) owns its DI, options, and HTTP surface. The host only discovers modules and calls them.

This is the same idea as [`Add{Feature}`](#addfeature-extension-methods), composed: many small `AddOrders()` / `IModule` instead of one `AddApplication()`.

#### Contract

```C#
public interface IModule
{
    void AddServices(IHostApplicationBuilder builder);
    void MapEndpoints(IEndpointRouteBuilder app);
}
```

Keep **registration** and **mapping** on the same type so a slice is one file (or one folder). Do not split “DI in Infrastructure, endpoints in Api” for the same feature.

#### One slice

```C#
public sealed class OrdersModule : IModule
{
    public void AddServices(IHostApplicationBuilder builder)
    {
        builder.Services.AddOptions<OrdersOptions>()
            .BindConfiguration("Orders")
            .ValidateDataAnnotations()
            .ValidateOnStart();

        builder.Services.TryAddScoped<IOrderStore, OrderStore>();
        builder.Services.AddHttpClient<IPricingClient, PricingClient>();
    }

    public void MapEndpoints(IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/orders").WithTags("Orders");
        g.MapGet("/{id:guid}", GetOrder);
        g.MapPost("/", CreateOrder);
    }

    static async Task<IResult> GetOrder(Guid id, IOrderStore store, CancellationToken ct)
    {
        var order = await store.GetAsync(id, ct);
        return order is null ? Results.NotFound() : Results.Ok(order);
    }

    static async Task<IResult> CreateOrder(CreateOrderRequest body, IOrderStore store, CancellationToken ct)
    {
        var id = await store.CreateAsync(body, ct);
        return Results.Created($"/orders/{id}", new { id });
    }
}
```

#### Host: explicit list (prefer) or scan

```C#
public static class ModuleExtensions
{
    public static IHostApplicationBuilder AddModules(
        this IHostApplicationBuilder builder, params IModule[] modules)
    {
        foreach (var m in modules)
            m.AddServices(builder);
        builder.Services.AddSingleton<IReadOnlyList<IModule>>(modules);
        return builder;
    }

    public static WebApplication MapModules(this WebApplication app)
    {
        foreach (var m in app.Services.GetRequiredService<IReadOnlyList<IModule>>())
            m.MapEndpoints(app);
        return app;
    }
}

var builder = WebApplication.CreateBuilder(args);
builder.AddModules(new OrdersModule(), new BillingModule(), new IdentityModule());

var app = builder.Build();
app.MapModules();
app.Run();
```

Assembly scan is fine for apps that add slices often, but **cache the result** and filter to `IModule`. Do not register every class in the assembly.

```C#
public static IModule[] ScanModules(params Assembly[] assemblies) =>
    assemblies
        .SelectMany(a => a.GetTypes())
        .Where(t => t is { IsAbstract: false, IsInterface: false }
                    && typeof(IModule).IsAssignableFrom(t))
        .Select(t => (IModule)Activator.CreateInstance(t)!)
        .ToArray();

builder.AddModules(ScanModules(typeof(OrdersModule).Assembly));
```

`Activator.CreateInstance` is only for the **module type** (stateless, no ctor deps). Feature services still come from DI. If a module needs `IConfiguration` at construction, take `IHostApplicationBuilder` in `AddServices` instead of injecting into the module ctor.

#### ❌ BAD — reflection-register everything

```C#
foreach (var t in assembly.GetTypes().Where(t => t.Name.EndsWith("Service")))
    services.AddScoped(t); // wrong lifetime, duplicates, internal helpers, slow
```

#### Rules

- Use `TryAdd*` inside modules so two slices can depend on the same `IClock` without double-registering.
- Modules must be **idempotent** and order-independent except when one slice truly depends on another (then call `AddBilling()` from `OrdersModule`, or compose in `Program.cs`).
- Do not `BuildServiceProvider()` inside a module.
- Carter’s `ICarterModule`, FastEndpoints, and Wolverine HTTP all follow this shape. You do not need those libraries to use the pattern.
- Shared kernel (`AddPersistence`, `AddMessaging`) stays a normal [`Add{Feature}`](#addfeature-extension-methods) called once from `Program.cs`; slices do not each add `DbContext`.

### `TryAdd` / `TryAddEnumerable`

Library authors should not overwrite the app’s registration:

```C#
services.TryAddSingleton<IFoo, Foo>();           // no-op if IFoo already registered
services.TryAddEnumerable(ServiceDescriptor.Singleton<IHostedService, MyWorker>());
```

`TryAddEnumerable` compares **implementation type**, so two different `IHostedService` implementations both get registered. `AddSingleton<IHostedService, MyWorker>()` twice would add two workers.

### Generic types as a factory

This pattern is used in **Microsoft.Extensions.\*** and in **Microsoft.AspNetCore.\***. The idea is that you can use a generic type as a factory instead of a function. The type argument is the type you want to instantiate. Consider the example below where we have an `IServiceFactory<TService>` that can resolve the `TService` from the DI container or create an instance if it is not in the container.

```C#
public interface IServiceFactory<TService>
{
    TService Service { get; }
}

public class ServiceFactory<TService> : IServiceFactory<TService>
{
    public ServiceFactory(IServiceProvider service)
    {
        Service = service.GetService<TService>()
            ?? ActivatorUtilities.CreateInstance<TService>(service);
    }

    public TService Service { get; }
}
```

The constructor has access to any service *and* the generic type being requested. These open generic services are registered like this:

```C#
services.AddTransient(typeof(IServiceFactory<>), typeof(ServiceFactory<>));
```

Then inject `IServiceFactory<MyThing>` anywhere. The closed generic is built by the container.

### Lazy initialization of services

Sometimes it is necessary to create services later than a constructor. The usual workaround is to inject an `IServiceProvider` into the constructor (service locator).

```C#
public class Service
{
    private readonly IServiceProvider _serviceProvider;
    public Service(IServiceProvider serviceProvider) => _serviceProvider = serviceProvider;

    public IFoo CreateFoo() => _serviceProvider.GetRequiredService<IFoo>();
    public IBar CreateBar() => _serviceProvider.GetRequiredService<IBar>();
}
```

If the types are known ahead of time, encapsulate that with an open generic lazy:

```C#
public interface ILazy<T>
{
    T Value { get; }
}

public class LazyFactory<T> : ILazy<T>
{
    private readonly Lazy<T> _lazy;

    public LazyFactory(IServiceProvider service)
    {
        _lazy = new Lazy<T>(() => service.GetRequiredService<T>());
    }

    public T Value => _lazy.Value;
}

public class Service
{
    private readonly ILazy<IFoo> _foo;
    private readonly ILazy<IBar> _bar;
    public Service(ILazy<IFoo> foo, ILazy<IBar> bar)
    {
        _foo = foo;
        _bar = bar;
    }

    public IFoo CreateFoo() => _foo.Value;
    public IBar CreateBar() => _bar.Value;
}
```

```C#
// Transient so the injected IServiceProvider matches the caller's lifetime (scoped vs root).
services.AddTransient(typeof(ILazy<>), typeof(LazyFactory<>));
```

`Lazy<T>` can also be registered per closed type:

```C#
services.AddTransient(sp => new Lazy<IFoo>(() => sp.GetRequiredService<IFoo>()));
services.AddTransient(sp => new Lazy<IBar>(() => sp.GetRequiredService<IBar>()));
```

:bulb: **NOTE:** `Lazy<T>` is not thread-safe by default in the sense of *which* provider it captured. Do not resolve a scoped service through a singleton's `Lazy<T>` that closed over the root provider. See [captive dependencies](AspNetCoreGuidance.md#avoid-capturing-scoped-services-in-singletons).

### Resolve by delegate (`Func<T>` / named delegates)

Two related uses of delegates in DI:

1. **Factory registration** — `Add*(sp => …)` builds the instance when it is resolved.
2. **Delegate as the service** — inject `Func<T>` (or a named `delegate`) so the consumer **resolves later**, without taking `IServiceProvider`.

Prefer a typed delegate over `IServiceProvider` (service locator). Prefer injecting `T` when you always need it in the ctor.

#### 1. Factory registration (`IServiceProvider` → instance)

```C#
services.AddScoped<IOrderStore>(sp =>
{
    var options = sp.GetRequiredService<IOptions<StoreOptions>>().Value;
    var inner = ActivatorUtilities.CreateInstance<SqlOrderStore>(sp);
    return options.UseCache
        ? new CachedOrderStore(inner, sp.GetRequiredService<IMemoryCache>())
        : inner;
});
```

The lambda runs **on each resolve** (for scoped/transient) with the **current** provider. That is how forwarding (`IFoo` → same `FooAndBar`) and [decorators](#decorator-around-an-existing-registration) work.

Do not call `BuildServiceProvider()` inside the lambda. Do not capture `HttpContext` from composition time.

#### 2. Inject `Func<T>` — resolve when you need it

Use this when construction is expensive, optional, or must happen **after** some input exists (tenant id, user, first call).

```C#
services.AddScoped<IReportBuilder, ReportBuilder>();
services.AddScoped<Func<IReportBuilder>>(sp =>
    () => sp.GetRequiredService<IReportBuilder>());

public sealed class ExportJob(Func<IReportBuilder> reports)
{
    public async Task RunAsync(CancellationToken cancellationToken)
    {
        if (!ShouldExport())
            return;

        var builder = reports(); // resolved now, not in the ctor
        await builder.WriteAsync(cancellationToken);
    }
}
```

This is the same idea as [`Lazy<T>`](#lazy-initialization-of-services), but **each call** can create a new transient. `Lazy<T>` caches the first value.

Register `Func<T>` as **transient or scoped**, matching `T`. The `sp` closed over must be the request/job provider, not the root.

#### 3. Named delegate — parameters and a stable type

`Func<string, IOrderStore>` is anonymous and collides if you have two factories. A **named delegate** is the service type:

```C#
public delegate IOrderStore OrderStoreResolver(string tenantId);
public delegate DateTimeOffset GetUtcNow();

services.AddSingleton<GetUtcNow>(_ => () => DateTimeOffset.UtcNow); // or TimeProvider

services.AddSingleton<ITenantStore, TenantStore>();
services.AddScoped<OrderStoreResolver>(sp => tenantId =>
{
    var tenants = sp.GetRequiredService<ITenantStore>();
    var info = tenants.Get(tenantId);
    return ActivatorUtilities.CreateInstance<SqlOrderStore>(sp, info.ConnectionString);
});

public sealed class Orders(OrderStoreResolver resolve, TenantContext tenant)
{
    public Task<Order?> GetAsync(OrderId id, CancellationToken cancellationToken) =>
        resolve(tenant.TenantId).GetAsync(id, cancellationToken);
}
```

ASP.NET Core uses this everywhere: `RequestDelegate`, `MapGet` handlers, `IStartupFilter`, options `Configure(name, Action<T>)`.

For a clock, prefer [`TimeProvider`](#timeprovider) over `GetUtcNow` unless you are wrapping an existing callback.

#### 4. Keyed resolve by delegate (.NET 8+)

```C#
services.AddKeyedSingleton<ICache, MemoryCache>("acme");
services.AddKeyedSingleton<ICache, MemoryCache>("contoso");

services.AddScoped<Func<string, ICache>>(sp =>
    key => sp.GetRequiredKeyedService<ICache>(key));
```

Still only for a **fixed** key set. Unbounded tenants: a singleton map, not keyed DI. See [multi-tenant](#multi-tenant-singleton-vs-scoped).

#### ❌ BAD — singleton `Func<T>` over a scoped service

```C#
services.AddScoped<AppDbContext>();
services.AddSingleton<Func<AppDbContext>>(sp =>
    () => sp.GetRequiredService<AppDbContext>()); // root provider, captive
```

The lambda captures the **root** `IServiceProvider`. `DbContext` is then resolved from the root (or throws). Same bug as injecting `IServiceProvider` into a singleton.

#### ✅ GOOD — scoped `Func<T>`, or a scope factory

```C#
services.AddScoped<Func<AppDbContext>>(sp =>
    () => sp.GetRequiredService<AppDbContext>());

// from a singleton / hosted service:
public sealed class Worker(IServiceScopeFactory scopes) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await using var scope = scopes.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    }
}
```

See [scoped work from a singleton](#scoped-work-from-a-singleton-iservicescopefactory).

#### 5. `ObjectFactory` — compiled ctor, extra args

When the type is not registered and you pass extra constructor arguments, cache [`ActivatorUtilities.CreateFactory`](#creating-instances-of-types-from-an-iserviceprovider) (`ObjectFactory` is `Func<IServiceProvider, object?[], object>`).

```C#
private static readonly ObjectFactory Factory =
    ActivatorUtilities.CreateFactory(typeof(SqlOrderStore), [typeof(string)]);

public SqlOrderStore Create(IServiceProvider sp, string connectionString) =>
    (SqlOrderStore)Factory(sp, [connectionString]);
```

#### Rule of thumb

```text
Add*(sp => new T(...))     = how to *create* T (composition)
inject T                   = always needed, ctor is fine
inject Func<T>             = resolve later / optionally; same lifetime as T
named delegate             = Func with a name, or parameters (tenant, key)
inject IServiceProvider    = last resort (unknown types, plugins)
singleton Func<Scoped>     = captive dependency — do not
```

### Creating instances of types from an `IServiceProvider`

Dynamically discovered types (MVC controllers, plugins) are often not registered. [`ActivatorUtilities`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.activatorutilities) constructs them using registered dependencies.

```C#
public class MyDependency
{
    public MyDependency(ILogger<MyDependency> logger) { }
}

public class MyDependencyFactory
{
    private readonly IServiceProvider _serviceProvider;
    public MyDependencyFactory(IServiceProvider serviceProvider) => _serviceProvider = serviceProvider;

    public MyDependency GetInstance() =>
        ActivatorUtilities.CreateInstance<MyDependency>(_serviceProvider);
}
```

A faster variant precomputes the constructor with [`CreateFactory`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.activatorutilities.createfactory):

```C#
public class MyDependencyFactory
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ObjectFactory _factory;

    public MyDependencyFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
        _factory = ActivatorUtilities.CreateFactory(typeof(MyDependency), Type.EmptyTypes);
    }

    public MyDependency GetInstance() =>
        (MyDependency)_factory(_serviceProvider, null);
}
```

Pass extra constructor arguments as the second `CreateFactory` / `CreateInstance` array (the "unregistered" parameters).

**NOTE:** Disposable instances created with this API are **not** tracked or disposed by the container. Dispose them yourself, or create them from a scope you own.

`[ActivatorUtilitiesConstructor]` selects which ctor to use when there are several.

### Single implementation, multiple interfaces

The built-in `IServiceCollection` does not natively register one instance as several interfaces. Forward to a single concrete registration:

```C#
public class FooAndBar : IFoo, IBar { }
```

```C#
services.AddSingleton<FooAndBar>();
services.AddSingleton<IFoo>(sp => sp.GetRequiredService<FooAndBar>());
services.AddSingleton<IBar>(sp => sp.GetRequiredService<FooAndBar>());
```

Resolving `FooAndBar`, `IFoo`, and `IBar` yields the **same** instance.

Registering `AddSingleton<IFoo, FooAndBar>()` **and** `AddSingleton<IBar, FooAndBar>()` creates **two** instances.

### Keyed services (.NET 8+)

When you need **two implementations of the same interface** distinguished by a key (not by a second interface):

```C#
services.AddKeyedSingleton<ICache, MemoryCache>("memory");
services.AddKeyedSingleton<ICache, RedisCache>("redis");

public class Checkout(
    [FromKeyedServices("redis")] ICache cache)
{ }
```

Resolve manually:

```C#
var cache = sp.GetRequiredKeyedService<ICache>("redis");
```

Keyed and non-keyed registrations are separate. `GetServices<ICache>()` does **not** include keyed ones.

### Multiple implementations via `IEnumerable<T>`

The container stacks registrations. Injecting `IEnumerable<T>` yields **all** of them (in registration order).

```C#
services.AddSingleton<INotification, EmailNotification>();
services.AddSingleton<INotification, SmsNotification>();

public class Notifier(IEnumerable<INotification> channels)
{
    public Task NotifyAllAsync(string msg) =>
        Task.WhenAll(channels.Select(c => c.SendAsync(msg)));
}
```

Last registration wins for `GetRequiredService<INotification>()` (a single `T`). Use `IEnumerable<T>` when you mean “all”.

### Decorator around an existing registration

Wrap an existing `T` without the app changing call sites (caching, logging, retry):

```C#
public sealed class CachedRepo(IRepo inner, IMemoryCache cache) : IRepo
{
    public Task<Item> GetAsync(int id, CancellationToken cancellationToken) =>
        cache.GetOrCreateAsync(id, _ => inner.GetAsync(id, cancellationToken))!;
}

// After the real IRepo is registered:
services.Decorate(); // Scrutor, or:

var descriptor = services.Last(d => d.ServiceType == typeof(IRepo));
services.AddSingleton<IRepo>(sp =>
{
    var inner = (IRepo)descriptor.ImplementationFactory!(sp);
    return new CachedRepo(inner, sp.GetRequiredService<IMemoryCache>());
});
```

Keep the inner registration resolvable (factory or concrete) so you do not recurse.

### Open-generic handlers (strategy per `T`)

MediatR, MVC formatters, and EF interceptors all use **one open generic** so each closed `T` gets its own handler without a giant `switch`.

```C#
public interface ICommandHandler<TCommand, TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken cancellationToken);
}

public sealed class CreateOrderHandler : ICommandHandler<CreateOrder, OrderId>
{
    public Task<OrderId> HandleAsync(CreateOrder command, CancellationToken cancellationToken) { /* ... */ }
}

services.AddTransient(typeof(ICommandHandler<,>), typeof(CreateOrderHandler));
// or scan:
// services.Scan(s => s.FromAssemblyOf<CreateOrderHandler>()
//     .AddClasses(c => c.AssignableTo(typeof(ICommandHandler<,>)))
//     .AsImplementedInterfaces());
```

Dispatch:

```C#
public sealed class Bus(IServiceProvider sp)
{
    public Task<TResult> SendAsync<TCommand, TResult>(TCommand command, CancellationToken cancellationToken) =>
        sp.GetRequiredService<ICommandHandler<TCommand, TResult>>()
          .HandleAsync(command, cancellationToken);
}
```

This is the usual “advanced” replacement for `if (cmd is CreateOrder)`.

### Composite over `IEnumerable<T>`

When callers should not know there are many implementations, wrap `IEnumerable<T>` in one composite and register **that** as `T`.

```C#
public sealed class CompositeNotification(IEnumerable<INotification> inner) : INotification
{
    public Task SendAsync(string message, CancellationToken cancellationToken) =>
        Task.WhenAll(inner.Select(n => n.SendAsync(message, cancellationToken)));
}

services.AddSingleton<EmailNotification>();
services.AddSingleton<SmsNotification>();
services.AddSingleton<INotification>(sp => new CompositeNotification(
    new INotification[]
    {
        sp.GetRequiredService<EmailNotification>(),
        sp.GetRequiredService<SmsNotification>(),
    }));
```

Consumers inject `INotification` and fan-out is an implementation detail. Contrast with injecting `IEnumerable<INotification>` when the caller *should* choose.

### `IServiceProviderIsService`

MVC, minimal APIs, and validators ask “is this type in the container?” without resolving it (which would construct a singleton too early):

```C#
public sealed class Binder(IServiceProviderIsService isService)
{
    public bool CanBind(Type type) => isService.IsService(type);
}
```

Register nothing extra—the default provider implements this. Use it in libraries that optionally take a service if the app registered one.

## Lifetimes, scopes, and tenancy

What is process-wide vs per-request. Captive dependencies and tenant isolation live here.

### Caching singletons in generic types

If you need to cache an instance per `T`, a static field on a generic nested type lets the JIT keep one slot per closed type—cheaper than `ConcurrentDictionary<Type, T>` on a hot path.

```C#
public class Factory
{
    public T Create<T>() where T : new() => Cache<T>.Instance;

    private static class Cache<T> where T : new()
    {
        public static readonly T Instance = new();
    }
}
```

Same idea for expensive one-time setup:

```C#
private static class Cache<T> where T : new()
{
    public static readonly T Instance = Create();

    private static T Create()
    {
        // expensive reflection / compilation / options bind
        return new T();
    }
}
```

This is how many `Logger<T>` / options caches work internally. Do not store **scoped** state this way—the static lives for the AppDomain.

### Scoped work from a singleton (`IServiceScopeFactory`)

Singletons must not take scoped services in the ctor. For per-operation scoped work (EF, `IOptionsSnapshot`):

```C#
public sealed class QueueProcessor(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await using var scope = scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.SaveChangesAsync(stoppingToken);
    }
}
```

`CreateAsyncScope` disposes `IAsyncDisposable` services. See [Do not capture request-scoped services](AspNetCoreGuidance.md#do-not-capture-request-scoped-services-on-background-work).

### `IDbContextFactory<T>` / async factories

When you need **many short-lived** `DbContext` instances (parallel queries, background jobs) without an HTTP scope:

```C#
services.AddDbContextFactory<AppDbContext>(o => o.UseSqlServer(cs));

public class Worker(IDbContextFactory<AppDbContext> factory)
{
    public async Task RunAsync(CancellationToken cancellationToken)
    {
        await using var db = await factory.CreateDbContextAsync(cancellationToken);
        await db.Orders.CountAsync(cancellationToken);
    }
}
```

Do not share one `DbContext` across `Task.WhenAll`. See [Avoid `DbContext` as a singleton](AspNetCoreGuidance.md#avoid-dbcontext-as-a-singleton).

For non-EF async construction, use a lazy/`IHostedService` factory instead of `.Result` in `AddSingleton`. See [Avoid `.Result` in DI](AspNetCoreGuidance.md#avoid-result--async-work-inside-di-registration).

### `IAsyncDisposable` services

If a service owns async resources (connection, `PipeWriter`), implement `IAsyncDisposable` (and usually `IDisposable` as a sync fallback).

```C#
public sealed class Bus : IAsyncDisposable
{
    public async ValueTask DisposeAsync() => await _connection.DisposeAsync();
}

services.AddSingleton<Bus>();
```

Resolve it from a scope and `await using` that scope (`CreateAsyncScope`). The root provider disposes singletons on host shutdown.

### Multi-tenant: singleton vs scoped

**Tenant identity is per request (scoped). Shared infrastructure is singleton, keyed by tenant id.** Putting “current tenant” on a singleton field mixes tenants under load.

| Kind | Lifetime | Examples |
|------|----------|----------|
| Current tenant | **Scoped** | `ITenantContext`, `HttpContext` item, EF interceptor |
| Tenant catalog | **Singleton** | `ITenantStore`, `FrozenDictionary<string, TenantInfo>` |
| Per-tenant resource | **Singleton map** or **keyed** | cache, `HttpClient`, connection pool, rate limiter partition |
| Tenant DB context | **Scoped** | `DbContext` + global query filter |

#### 1. Resolve tenant per request (scoped)

Use a **scoped holder** so HTTP and background jobs bind tenant the same way. Middleware (or the worker) sets it; everything else injects `TenantContext`.

```C#
public sealed class TenantContext
{
    public required string TenantId { get; init; }
}

public sealed class TenantContextHolder
{
    public TenantContext? Current { get; set; }
}

public sealed class TenantResolverMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context, TenantContextHolder holder)
    {
        var tenantId =
            context.Request.Headers["X-Tenant"].FirstOrDefault()
            ?? context.Request.Host.Host; // or claim / subdomain

        if (string.IsNullOrEmpty(tenantId))
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            return;
        }

        holder.Current = new TenantContext { TenantId = tenantId };
        await next(context);
    }
}

services.AddScoped<TenantContextHolder>();
services.AddScoped(sp =>
    sp.GetRequiredService<TenantContextHolder>().Current
    ?? throw new InvalidOperationException("Tenant not bound"));
```

Inject `TenantContext` into **scoped** services only. Do **not** inject it into a singleton (captive dependency: the first request’s tenant sticks for the process lifetime).

#### 2. ❌ BAD — singleton holds “current” tenant

```C#
public sealed class BadTenantCache
{
    public string? CurrentTenantId { get; set; } // races across requests
    public Dictionary<string, object> Data { get; } = new();
}

services.AddSingleton<BadTenantCache>();
```

Two concurrent requests overwrite `CurrentTenantId`. You leak data across tenants.

#### 3. ✅ GOOD — singleton keyed by tenant id

```C#
public sealed class TenantCache
{
    private readonly MemoryCache _cache = new(new MemoryCacheOptions { SizeLimit = 10_000 });

    public Task<T> GetOrCreateAsync<T>(string tenantId, string key, Func<Task<T>> factory) =>
        _cache.GetOrCreateAsync($"{tenantId}:{key}", async e =>
        {
            e.Size = 1;
            e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return await factory();
        })!;
}

services.AddSingleton<TenantCache>();

public sealed class Orders(TenantContext tenant, TenantCache cache, IStore store)
{
    public Task<Order> GetAsync(int id, CancellationToken cancellationToken) =>
        cache.GetOrCreateAsync(tenant.TenantId, $"order:{id}",
            () => store.LoadAsync(tenant.TenantId, id, cancellationToken));
}
```

Always prefix cache, log, and metric tags with `tenantId`. `HybridCache` / Redis: same key shape `tenant:{id}:order:{orderId}`.

#### 4. Per-tenant clients (lazy singleton map)

When each tenant has a different base URL or API key, a singleton **factory** caches clients. Do not `new HttpClient()` per request.

```C#
public sealed class TenantHttpFactory(IHttpClientFactory factory, ITenantStore store)
{
    public HttpClient Create(string tenantId)
    {
        var info = store.Get(tenantId);
        var client = factory.CreateClient("tenant");
        client.BaseAddress = info.BaseAddress;
        client.DefaultRequestHeaders.Remove("X-Api-Key");
        client.DefaultRequestHeaders.TryAddWithoutValidation("X-Api-Key", info.ApiKey);
        return client;
    }
}

services.AddSingleton<ITenantStore, TenantStore>();
services.AddSingleton<TenantHttpFactory>();
services.AddHttpClient("tenant");
```

If the tenant set is **fixed and small**, .NET 8 keyed services:

```C#
services.AddKeyedSingleton<ICache, MemoryCache>("acme");
services.AddKeyedSingleton<ICache, MemoryCache>("contoso");

public sealed class Checkout(TenantContext tenant, IServiceProvider sp)
{
    public ICache Cache => sp.GetRequiredKeyedService<ICache>(tenant.TenantId);
}
```

Keyed registrations are not for unbounded tenant ids (you would register thousands of services). Use a dictionary/cache for that.

#### 5. Named options per tenant

```C#
services.Configure<BlobOptions>("acme", configuration.GetSection("Tenants:acme:Blob"));
services.Configure<BlobOptions>("contoso", configuration.GetSection("Tenants:contoso:Blob"));

public sealed class Blobs(TenantContext tenant, IOptionsMonitor<BlobOptions> monitor)
{
    public BlobOptions Options => monitor.Get(tenant.TenantId);
}
```

Dynamic tenants: `ITenantStore` + `IOptionsMonitor` does not scale; load settings from the store instead.

#### 6. EF: scoped `DbContext`, filter by tenant

```C#
public sealed class AppDbContext(DbContextOptions<AppDbContext> options, TenantContext tenant)
    : DbContext(options)
{
    public string TenantId => tenant.TenantId;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>().HasQueryFilter(o => o.TenantId == TenantId);
    }

    public override int SaveChanges()
    {
        foreach (var e in ChangeTracker.Entries<Order>().Where(e => e.State == EntityState.Added))
            e.Entity.TenantId = TenantId;
        return base.SaveChanges();
    }
}
```

`DbContext` stays **scoped**. Never a singleton. For background jobs, create a scope and set `TenantContext` from the **message**, not from `IHttpContextAccessor`.

#### 7. Background work must carry tenant id

`IHttpContextAccessor` is empty in a worker. Put `tenantId` on the message, open a scope, and bind the same `TenantContextHolder` from §1.

```C#
public sealed record WorkItem(string TenantId, int OrderId);

// enqueue from the request:
await channel.Writer.WriteAsync(new WorkItem(tenant.TenantId, orderId));

// worker:
await using var scope = scopeFactory.CreateAsyncScope();
var holder = scope.ServiceProvider.GetRequiredService<TenantContextHolder>();
holder.Current = new TenantContext { TenantId = item.TenantId };
var orders = scope.ServiceProvider.GetRequiredService<Orders>();
await orders.GetAsync(item.OrderId, stoppingToken);
```

Do not capture `HttpContext` or request-scoped `TenantContext` in `Task.Run`. See [AspNetCoreGuidance.md](AspNetCoreGuidance.md).

#### 8. Ambient tenant (`AsyncLocal`) — use sparingly

Some libraries use `AsyncLocal<TenantContext>` so deep code does not thread the id. That is the same class of bug as `AsyncLocal` leaks ([AsyncGuidance](AsyncGuidance.md#asynclocalt)). Prefer explicit `TenantContext` in ctors. If you use ambient:

- Set it in middleware, clear it at the end of the request
- Do not store it on a singleton field
- Background jobs set it inside their own scope, then clear

#### Rule of thumb

```text
singleton  = process-wide, must be tenant-safe (no “current tenant” field)
scoped     = one tenant, one request (or one job scope)
keyed/map  = singleton that looks up by tenantId
message    = tenantId on the payload when work leaves the request
```

## Options and configuration

Bind, name, reload, and validate options. Custom configuration sources.

### Resolving services when using `IOptions<T>`

The options pattern lets callers mutate a POCO used to configure a library. To use other services while configuring:

```C#
public class LibraryOptions
{
    public int Setting { get; set; }
}

public class MyConfigureOptions : IConfigureOptions<LibraryOptions>
{
    private readonly ISomeService _service;
    public MyConfigureOptions(ISomeService service) => _service = service;

    public void Configure(LibraryOptions options) =>
        options.Setting = _service.ComputeSetting();
}

services.AddSingleton<IConfigureOptions<LibraryOptions>, MyConfigureOptions>();
```

Shorter:

```C#
services.AddOptions<LibraryOptions>()
    .Configure<ISomeService>((options, service) =>
    {
        options.Setting = service.ComputeSetting();
    });
```

`Configure` runs once when the options instance is created. For reloadable config, use `IOptionsMonitor<T>` / `IConfigureNamedOptions<T>` / `PostConfigure`. Do not inject `IConfiguration` into app services to read `config["Section:Key"]` — bind a POCO. See [Prefer `IOptions<T>`](AspNetCoreGuidance.md#prefer-ioptionst--ioptionsmonitort-over-iconfiguration).

### Named options, `IOptionsMonitor`, and `IOptionsSnapshot`

| Type | Lifetime | Reloads | Typical use |
|------|----------|---------|-------------|
| `IOptions<T>` | Singleton | No | Immutable settings |
| `IOptionsSnapshot<T>` | Scoped | Once per scope/request | Per-request view of current config |
| `IOptionsMonitor<T>` | Singleton | Yes (`OnChange`) | Long-lived services that must see updates |

```C#
services.Configure<SmtpOptions>("alerts", configuration.GetSection("Smtp:Alerts"));
services.Configure<SmtpOptions>("bulk", configuration.GetSection("Smtp:Bulk"));

public class Mailer(IOptionsMonitor<SmtpOptions> monitor)
{
    public void SendAlert() => Send(monitor.Get("alerts"));
}
```

`IConfigureNamedOptions<T>.Configure(string? name, T options)` is the named equivalent of `IConfigureOptions<T>`.

```C#
services.AddOptions<SmtpOptions>("alerts")
    .BindConfiguration("Smtp:Alerts")
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

### Validate options at start

Fail fast at boot instead of on the first request:

```C#
services.AddOptions<LibraryOptions>()
    .BindConfiguration("Library")
    .Validate(o => o.Setting > 0, "Library:Setting must be > 0")
    .ValidateOnStart();
```

`ValidateDataAnnotations()` uses `[Required]`, `[Range]`, etc. on the POCO. `IValidateOptions<T>` is the extensibility point for complex rules.

### `IChangeToken` (reload without polling)

Config, Razor, DataProtection, and `IOptionsMonitor` all use [`IChangeToken`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.primitives.ichangetoken) from **Microsoft.Extensions.Primitives**. You register a callback; the producer signals once. No timers.

```C#
public sealed class FileWatch(IConfiguration config)
{
    public IChangeToken Watch() => config.GetReloadToken();
}

ChangeToken.OnChange(
    () => configuration.GetReloadToken(),
    () => Console.WriteLine("config reloaded"));
```

`CancellationChangeToken` wraps a `CancellationToken`. `IOptionsChangeTokenSource<T>` is how named options reload. `IMemoryCache` entries can expire on a change token (file, config, eviction).

Libraries: return an `IChangeToken` from `GetReloadToken()`; do not expose `FileSystemWatcher` directly.

### `IConfigurationProvider`

Config is a chain of **providers** (JSON, env, command line, KeyPerFile). Custom sources implement `ConfigurationProvider` + `IConfigurationSource`:

```C#
public sealed class VaultSource : IConfigurationSource
{
    public IConfigurationProvider Build(IConfigurationBuilder builder) => new VaultProvider();
}

public sealed class VaultProvider : ConfigurationProvider
{
    public override void Load() => Data["Vault:Token"] = Fetch();
}

builder.Configuration.Add(new VaultSource());
```

`IChangeToken` on the provider enables reload. This is how `AddJsonFile(..., reloadOnChange: true)` works.

## HTTP clients

Outbound HTTP. For inbound body/HttpContext pitfalls see [AspNetCoreGuidance.md](AspNetCoreGuidance.md). Platform HttpClient notes: [HttpClientGuidance.md](HttpClientGuidance.md).

### Typed and named `HttpClient`

```C#
services.AddHttpClient<DocsClient>(client =>
{
    client.BaseAddress = new Uri("https://example.com");
    client.Timeout = TimeSpan.FromSeconds(10);
});

services.AddHttpClient("github", client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
});
```

Typed: inject `DocsClient` (it takes `HttpClient` in the ctor). Named: `IHttpClientFactory.CreateClient("github")`.

Do not `new HttpClient()` per request. See [Prefer `IHttpClientFactory`](AspNetCoreGuidance.md#prefer-ihttpclientfactory-over-new-httpclient).

The factory sets `SocketsHttpHandler.PooledConnectionLifetime` to 2 minutes. If you `ConfigurePrimaryHttpMessageHandler`, set that property yourself or DNS will go stale. See [HttpClientGuidance.md](HttpClientGuidance.md).

### `HttpMessageHandler` pipeline

Cross-cutting HTTP policy (auth, correlation, logging) is a `DelegatingHandler` chain—the same idea as ASP.NET middleware, outbound.

```C#
public sealed class CorrelationHandler(IHttpContextAccessor accessor) : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        if (accessor.HttpContext?.TraceIdentifier is { } id)
            request.Headers.TryAddWithoutValidation("X-Correlation-ID", id);
        return base.SendAsync(request, cancellationToken);
    }
}

services.AddTransient<CorrelationHandler>();
services.AddHttpClient<OrdersClient>()
    .AddHttpMessageHandler<CorrelationHandler>();
```

Handlers are typically **transient**; the factory pools the inner primary handler. Do not capture `HttpContext` beyond the request (see [AspNetCoreGuidance](AspNetCoreGuidance.md#do-not-capture-httpcontext-on-background-work)).

### Http resilience

Most production lists now use **Microsoft.Extensions.Http.Resilience** (or Polly) instead of hand-rolled retry loops.

```C#
services.AddHttpClient<OrdersClient>()
    .AddStandardResilienceHandler(); // timeout, retry, circuit breaker, hedged retry defaults
```

Tune:

```C#
.AddStandardResilienceHandler(o =>
{
    o.Retry.MaxRetryAttempts = 3;
    o.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
    o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);
});
```

Prefer this on **idempotent** GETs. Do not blindly retry POST without idempotency keys.

## Hosting and background work

Process lifetime, queues, and work that outlives a request.

### `IHostedService` / `BackgroundService`

Long-running work belongs in the host, not `Task.Run` from a controller.

```C#
services.AddHostedService<QueuedHostedService>();
services.AddSingleton<IBackgroundTaskQueue, BackgroundTaskQueue>();

public sealed class QueuedHostedService(IBackgroundTaskQueue queue) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var work = await queue.DequeueAsync(stoppingToken);
            await work(stoppingToken);
        }
    }
}
```

`ExecuteAsync` is not awaited by the host until shutdown—catch exceptions inside the loop. See [Channel + hosted services](AsyncGuidance.md#prefer-channelt-and-hosted-services-for-background-work).

.NET 8+ also has `IHostedLifecycleService` (`StartingAsync` / `StartedAsync` / `StoppingAsync` / `StoppedAsync`) for ordered startup.

### Singleton `Channel<T>` producer/consumer

In-process queue: singleton `Channel<T>`, controllers write, `BackgroundService` reads. Bounded channel = backpressure.

```C#
services.AddSingleton(_ => Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(256)
{
    FullMode = BoundedChannelFullMode.Wait
}));
services.AddHostedService<WorkConsumer>();

public sealed class WorkConsumer(Channel<WorkItem> channel) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var item in channel.Reader.ReadAllAsync(stoppingToken))
            await ProcessAsync(item, stoppingToken);
    }
}
```

Not durable across process restart. See [Channel + hosted services](AsyncGuidance.md#prefer-channelt-and-hosted-services-for-background-work).

### `IHostApplicationLifetime`

Hook process start/stop without a full `IHostedService` when you only need a callback (flush telemetry, stop accepting work):

```C#
public sealed class Drain(IHostApplicationLifetime lifetime, Channel<WorkItem> channel) : IHostedService
{
    public Task StartAsync(CancellationToken cancellationToken)
    {
        lifetime.ApplicationStopping.Register(() => channel.Writer.TryComplete());
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

`ApplicationStarted` / `ApplicationStopping` / `ApplicationStopped` are signaled by the host. Prefer `BackgroundService` + `stoppingToken` for loops; use lifetime when something **outside** the worker must observe shutdown.

## ASP.NET Core pipeline

Library and app hooks in the HTTP pipeline. Pair with [AspNetCoreGuidance.md](AspNetCoreGuidance.md).

### `IStartupFilter` (library middleware)

Libraries that must insert middleware **without** the app calling `UseX()` in a specific order use `IStartupFilter` (how HTTPS redirection / routing extras get wired).

```C#
public sealed class CorrelationStartupFilter : IStartupFilter
{
    public Action<IApplicationBuilder> Configure(Action<IApplicationBuilder> next) =>
        app =>
        {
            app.UseMiddleware<CorrelationMiddleware>();
            next(app);
        };
}

services.TryAddEnumerable(ServiceDescriptor.Singleton<IStartupFilter, CorrelationStartupFilter>());
```

Prefer explicit `app.UseCorrelation()` when the app should control order. Use `IStartupFilter` for defaults that must run around user middleware.

### Factory-activated `IMiddleware`

Convention middleware (`app.Use`) is constructed per request via `ActivatorUtilities` (not DI-tracked). For middleware that needs **scoped** services, implement `IMiddleware` and register it in DI:

```C#
public sealed class TenantMiddleware(ITenantContext tenant) : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        tenant.Bind(context);
        await next(context);
    }
}

services.AddScoped<TenantMiddleware>();
app.UseMiddleware<TenantMiddleware>();
```

`IMiddlewareFactory` resolves it from the request scope and disposes it. This is what Microsoft uses when middleware is not a simple singleton.

### Endpoint filters

Minimal APIs’ equivalent of MVC filters: validation, logging, short-circuit—without a full middleware.

```C#
app.MapPost("/orders", (CreateOrder body) => Results.Created($"/orders/{body.Id}", body))
   .AddEndpointFilter(async (context, next) =>
   {
       var cmd = context.GetArgument<CreateOrder>(0);
       if (cmd.Qty <= 0)
           return TypedResults.ValidationProblem(new Dictionary<string, string[]>
           {
               ["Qty"] = ["Must be positive"]
           });
       return await next(context);
   });
```

Reusable:

```C#
public sealed class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        // validate context.GetArgument<T>(0)
        return await next(context);
    }
}

route.AddEndpointFilter<ValidationFilter<CreateOrder>>();
```

### `IFeatureCollection`

`HttpContext.Features` is a type-keyed bag. Servers and middleware add **features** (`IHttpRequestLifetimeFeature`, `IHttpResponseBodyFeature`, `IHttpResetFeature`, `IHttpActivityFeature`) instead of growing `HttpContext` forever. This is how HTTP/2, HTTP/3, and custom transports plug in.

```C#
var lifetime = context.Features.Get<IHttpRequestLifetimeFeature>();
lifetime?.Abort();

context.Features.Set<IMyFeature>(new MyFeature());
```

Library middleware should `Get<T>()` and no-op if missing, not assume Kestrel. Prefer `IHttpResponseBodyFeature.Writer` / `StartAsync` over wrapping `Response.Body` when you need the real response pipe.

### `IExceptionHandler` / `IProblemDetailsService`

.NET 8+ pipeline for errors (replaces ad-hoc `UseExceptionHandler` lambdas in new apps):

```C#
public sealed class AppExceptionHandler(IProblemDetailsService problems) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken cancellationToken)
    {
        if (exception is not AppException app)
            return false;

        context.Response.StatusCode = app.StatusCode;
        await problems.WriteAsync(new ProblemDetailsContext
        {
            HttpContext = context,
            ProblemDetails = { Title = app.Message, Status = app.StatusCode }
        });
        return true;
    }
}

services.AddExceptionHandler<AppExceptionHandler>();
services.AddProblemDetails();
app.UseExceptionHandler();
```

Returning `false` lets the next handler run. This is the in-box replacement for “middleware that catches Exception and writes JSON”.

### `AuthenticationHandler<TOptions>` / `AuthorizationHandler<T>`

Auth libraries (cookies, JWT, your scheme) are **options + handler**, not random middleware:

```C#
public sealed class ApiKeyHandler(IOptionsMonitor<ApiKeyOptions> options, ILoggerFactory logger, UrlEncoder encoder)
    : AuthenticationHandler<ApiKeyOptions>(options, logger, encoder)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-Api-Key", out var key) || key != Options.Key)
            return Task.FromResult(AuthenticateResult.Fail("missing key"));

        var id = new ClaimsIdentity(Scheme.Name);
        id.AddClaim(new Claim(ClaimTypes.Name, "api"));
        return Task.FromResult(AuthenticateResult.Success(new AuthenticationTicket(new ClaimsPrincipal(id), Scheme.Name)));
    }
}

builder.Services.AddAuthentication("ApiKey")
    .AddScheme<ApiKeyOptions, ApiKeyHandler>("ApiKey", _ => { });
```

Authorization:

```C#
public sealed class MinAgeHandler : AuthorizationHandler<MinAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, MinAgeRequirement requirement)
    {
        if (context.User.HasClaim(c => c.Type == "age" && int.Parse(c.Value) >= requirement.Age))
            context.Succeed(requirement);
        return Task.CompletedTask;
    }
}

services.AddSingleton<IAuthorizationHandler, MinAgeHandler>();
```

This is how **all** `Microsoft.AspNetCore.Authentication.*` packages plug in.

### Rate limiter partitions

`System.Threading.RateLimiting` + `Microsoft.AspNetCore.RateLimiting` partition by key (IP, user, API key)—the same pattern as `IMemoryCache` keys:

```C#
builder.Services.AddRateLimiter(o =>
{
    o.AddPolicy("per-user", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            context.User.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString() ?? "anon",
            _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));
});

app.UseRateLimiter();
app.MapGet("/search", () => "ok").RequireRateLimiting("per-user");
```

Libraries that expose quotas should take a `PartitionedRateLimiter<T>` (or options to hook one) rather than a static `lock` + counter.

### `IHealthCheck`

Every Microsoft hosting template and container orchestrator expects `/health`. Libraries that own a dependency (DB, queue, HTTP) expose an `IHealthCheck`:

```C#
public sealed class RedisHealth(IConnectionMultiplexer redis) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            await redis.GetDatabase().PingAsync();
            return HealthCheckResult.Healthy();
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("redis ping failed", ex);
        }
    }
}

services.AddHealthChecks().AddCheck<RedisHealth>("redis");
app.MapHealthChecks("/health");
```

Keep checks fast and independent of request `HttpContext`.

## Persistence and domain

Transactions, identity types, and the request as a unit of work.

### Unit of work / transaction scope

In ASP.NET Core + EF Core, **the scoped `DbContext` is the unit of work**. `SaveChangesAsync` already wraps tracked changes in a transaction. A second `IUnitOfWork` that only calls `SaveChanges` is ceremony unless you have **multiple** stores or a non-EF store.

#### 1. One `SaveChanges` per request (usual case)

```C#
public sealed class CreateOrderHandler(AppDbContext db)
{
    public async Task<OrderId> HandleAsync(CreateOrder command, CancellationToken cancellationToken)
    {
        var order = new Order { Id = OrderId.New(), CustomerId = command.CustomerId };
        db.Orders.Add(order);
        await db.SaveChangesAsync(cancellationToken); // one transaction
        return order.Id;
    }
}
```

Register `DbContext` **scoped** (never singleton). The request (or job scope) is the UoW boundary. See [multi-tenant](#multi-tenant-singleton-vs-scoped) and [`IDbContextFactory<T>`](#idbcontextfactoryt--async-factories) for workers.

#### 2. Explicit transaction — several `SaveChanges` or mixed commands

```C#
public sealed class PlaceOrder(AppDbContext db, IClock clock)
{
    public async Task HandleAsync(OrderId id, CancellationToken cancellationToken)
    {
        await using var tx = await db.Database.BeginTransactionAsync(cancellationToken);
        try
        {
            var order = await db.Orders.SingleAsync(o => o.Id == id, cancellationToken);
            order.Place(clock.UtcNow);
            await db.SaveChangesAsync(cancellationToken);

            db.Outbox.Add(new OutboxMessage { Type = "OrderPlaced", OrderId = id });
            await db.SaveChangesAsync(cancellationToken);

            await tx.CommitAsync(cancellationToken);
        }
        catch
        {
            await tx.RollbackAsync(cancellationToken);
            throw;
        }
    }
}
```

If you only insert both rows then call `SaveChanges` **once**, you do not need `BeginTransactionAsync`.

#### 3. Retries: transaction **inside** the execution strategy

SQL Server / Azure SQL retry on transient faults. A transaction opened **outside** `CreateExecutionStrategy().ExecuteAsync` will throw.

```C#
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync(cancellationToken);
    // work + SaveChanges
    await tx.CommitAsync(cancellationToken);
});
```

#### 4. Optional `IUnitOfWork` — facade, not a second tracker

Useful when handlers should not take `DbContext`, or you commit **two** contexts together:

```C#
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

public sealed class EfUnitOfWork(AppDbContext db) : IUnitOfWork
{
    public Task<int> SaveChangesAsync(CancellationToken cancellationToken = default) =>
        db.SaveChangesAsync(cancellationToken);
}

services.AddDbContext<AppDbContext>(...);
services.AddScoped<IUnitOfWork, EfUnitOfWork>();
```

Simpler: implement `IUnitOfWork` on the context, or skip the interface. Do **not** start a `TransactionScope` inside `SaveChanges` and another in the handler.

#### 5. `TransactionScope` — ambient, easy to get wrong

Use only when you must enlist **multiple resource managers** (two SQL connections, SQL + DTC). For a single EF context, prefer `BeginTransactionAsync`.

```C#
using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    new TransactionOptions { IsolationLevel = IsolationLevel.ReadCommitted },
    TransactionScopeAsyncFlowOption.Enabled); // required for async

await db.SaveChangesAsync(cancellationToken);
scope.Complete();
```

#### ❌ BAD

```C#
using var scope = new TransactionScope(); // AsyncFlow disabled by default
await db.SaveChangesAsync();              // often runs *outside* the ambient tx
scope.Complete();
```

Also bad: `TransactionScope` across `await` without `Enabled`; nesting scopes with `RequiresNew` by accident; using it with providers that do not enlist cleanly; holding a transaction open while calling HTTP.

#### 6. Do not share a UoW across concurrent work

`DbContext` is not thread-safe. Two `SaveChangesAsync` in `Task.WhenAll` on the same instance corrupt the tracker. Sequential work in one scope, or one context per parallel branch via `IDbContextFactory`. See [AsyncGuidance](AsyncGuidance.md).

#### Rule of thumb

```text
one request / job scope     = one DbContext = one UoW
one SaveChanges             = one transaction (enough for most handlers)
BeginTransactionAsync       = multiple SaveChanges or explicit commit/rollback
execution strategy wraps tx = retries with SQL Server
TransactionScope            = last resort, always AsyncFlowOption.Enabled
tenant / HttpContext        = not captured inside the transaction callback
```

### Strongly-typed IDs

`Guid` / `int` / `string` for every entity id is a type hole: `GetOrder(customerId)` compiles. A **readonly record struct** (or a source-generated wrapper) makes that a compile error and still sits in a register.

#### ❌ BAD — primitive obsession

```C#
Task<Order?> GetAsync(Guid id, CancellationToken cancellationToken);
await orders.GetAsync(customer.Id, ct); // compiles, wrong aggregate
```

#### ✅ GOOD — `readonly record struct`

```C#
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.CreateVersion7());
    public override string ToString() => Value.ToString();
}

public readonly record struct CustomerId(Guid Value);

public interface IOrderStore
{
    Task<Order?> GetAsync(OrderId id, CancellationToken cancellationToken);
}

public sealed class Order
{
    public OrderId Id { get; init; }
    public CustomerId CustomerId { get; init; }
}
```

`record struct` gives value equality and `GetHashCode` for dictionary keys. Prefer **struct** over class so ids do not allocate. `Guid.CreateVersion7()` (.NET 9+) sorts by time; otherwise `Guid.NewGuid()`.

#### ASP.NET Core binding (.NET 7+)

Implement `IParsable<T>` / `ISpanParsable<T>` so route and query bind without a custom model binder:

```C#
public readonly record struct OrderId(Guid Value) : IParsable<OrderId>
{
    public static OrderId Parse(string s, IFormatProvider? provider) =>
        new(Guid.Parse(s, provider));

    public static bool TryParse(string? s, IFormatProvider? provider, out OrderId result)
    {
        if (Guid.TryParse(s, provider, out var g))
        {
            result = new(g);
            return true;
        }

        result = default;
        return false;
    }
}

app.MapGet("/orders/{id}", (OrderId id, IOrderStore store, CancellationToken ct) =>
    store.GetAsync(id, ct));
```

#### EF Core

```C#
modelBuilder.Entity<Order>()
    .Property(o => o.Id)
    .HasConversion(id => id.Value, v => new OrderId(v));

modelBuilder.Entity<Order>()
    .Property(o => o.CustomerId)
    .HasConversion(id => id.Value, v => new CustomerId(v));
```

For many ids, a shared `ValueConverter<TId, Guid>` or a source generator (Vogen, StronglyTypedIds) avoids copy-paste. Store the **primitive** in the column (`uniqueidentifier` / `bigint`), not a JSON blob.

#### JSON

System.Text.Json will serialize a `record struct OrderId(Guid Value)` as `{"value":"..."}` unless you add a converter. For APIs, you usually want a raw guid string:

```C#
public sealed class OrderIdConverter : JsonConverter<OrderId>
{
    public override OrderId Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options) =>
        new(reader.GetGuid());

    public override void Write(Utf8JsonWriter writer, OrderId value, JsonSerializerOptions options) =>
        writer.WriteStringValue(value.Value);
}
```

Register on `JsonSerializerOptions` or `[JsonConverter(typeof(OrderIdConverter))]`. With source-gen JSON, add the converter to the `[JsonSerializable]` context.

#### Rules

- One id type per aggregate (`OrderId` ≠ `CustomerId` ≠ `TenantId`).
- Do not use `string` as the wrapper payload unless the source is truly opaque (external keys). Prefer `Guid` / `long`.
- `default(OrderId)` is `Guid.Empty` — reject it at the boundary (`id == default`).
- Do not put behavior beyond parse/format on the id. It is a value, not an entity.
- Dapper: `SqlMapper.AddTypeHandler` (or a type handler per id) so parameters are `Guid`, not the struct.

## Caching

Per-key caches vs immutable startup maps.

### Cache-aside (`IMemoryCache` / `HybridCache`)

The pattern almost every app uses: look up, on miss compute, store.

```C#
public sealed class ProductService(IMemoryCache cache, IStore store)
{
    public async Task<Product> GetAsync(int id, CancellationToken cancellationToken)
    {
        return (await cache.GetOrCreateAsync($"product:{id}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            entry.Size = 1;
            return await store.LoadAsync(id, cancellationToken);
        }))!;
    }
}

services.AddMemoryCache(o => o.SizeLimit = 1024);
```

`GetOrCreateAsync` can **stampede** (many callers miss at once). .NET 9 [`HybridCache`](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid) coalesces in-flight work and can L2 to Redis:

```C#
services.AddHybridCache();

Product product = await hybrid.GetOrCreateAsync(
    $"product:{id}",
    async cancel => await store.LoadAsync(id, cancel),
    cancellationToken: cancellationToken);
```

Always set size + expiration. See [unbounded caches](AspNetCoreGuidance.md#avoid-unbounded-static-caches).

### `FrozenDictionary` for startup lookups

When a map is built once at startup and then read on every request, [`FrozenDictionary<TKey, TValue>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.frozen.frozendictionary-2) (.NET 8) is faster than `Dictionary` / `ConcurrentDictionary` for reads.

```C#
services.AddSingleton(sp =>
{
    var items = sp.GetRequiredService<IEnumerable<IHandler>>();
    return items.ToFrozenDictionary(h => h.Name, StringComparer.OrdinalIgnoreCase);
});

public sealed class Dispatcher(FrozenDictionary<string, IHandler> handlers)
{
    public IHandler Get(string name) => handlers[name];
}
```

Do not mutate after `ToFrozenDictionary`. For data that changes at runtime, use `ConcurrentDictionary` or `IOptionsMonitor`.

## Observability

Logging, metrics, and tracing without allocating on the hot path.

### Source-generated logging (`LoggerMessage`)

Avoid `ILogger.LogInformation($"id={id}")` (templates + boxing) and `Console.WriteLine` in libraries. Use a message template, or `[LoggerMessage]` (.NET 6+) on hot paths:

```C#
public static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Handled {Path} in {ElapsedMs}ms")]
    public static partial void Handled(ILogger logger, string path, long elapsedMs);
}

Log.Handled(_logger, context.Request.Path, elapsed.ElapsedMilliseconds);
```

High-performance libraries in-box use this. `LoggerMessage.Define` is the older non-source-gen equivalent.

### `NullLogger` in libraries

Framework libraries take `ILogger<T>` but must run in tests and in hosts that did not call `AddLogging`. Use [`NullLogger<T>.Instance`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.abstractions.nulllogger-1) as the default.

```C#
public sealed class Parser
{
    private readonly ILogger _logger;
    public Parser(ILogger<Parser>? logger = null) =>
        _logger = logger ?? NullLogger<Parser>.Instance;
}
```

`NullLoggerFactory.Instance` for APIs that need `ILoggerFactory`. Never `new LoggerFactory()` inside a library.

### `ActivitySource` / `IMeterFactory`

ASP.NET, `HttpClient`, and EF emit **traces** (`System.Diagnostics.Activity`) and **metrics** (`System.Diagnostics.Metrics`). Libraries should do the same so OpenTelemetry picks them up with no extra DI type.

```C#
public static class Telemetry
{
    public static readonly ActivitySource Activity = new("MyCompany.Orders");
    public static readonly Meter Meter = new("MyCompany.Orders");
    public static readonly Counter<int> Orders = Meter.CreateCounter<int>("orders.created");
}

using var activity = Telemetry.Activity.StartActivity("PlaceOrder");
activity?.SetTag("order.id", id);
Telemetry.Orders.Add(1);
```

In ASP.NET Core, prefer `IMeterFactory` so meters are collected and disposed with the host:

```C#
public sealed class OrdersMetrics(IMeterFactory factory)
{
    private readonly Counter<int> _orders =
        factory.Create("MyCompany.Orders").CreateCounter<int>("orders.created");
}
```

Do not invent a parallel “ITelemetry” abstraction; OTel already listens to `ActivitySource` + `Meter` names.

## Performance and library primitives

Types Microsoft libraries use internally. Prefer these over rolling your own buffers, clocks, and JSON metadata.

### `TimeProvider`

Do not call `DateTime.UtcNow` / `Task.Delay` directly if you need tests to fake time (.NET 8+).

```C#
services.AddSingleton(TimeProvider.System);

public class TokenService(TimeProvider time)
{
    public DateTimeOffset ExpiresAt() => time.GetUtcNow().AddHours(1);
    public Task WaitAsync(TimeSpan delay, CancellationToken cancellationToken) =>
        Task.Delay(delay, time, cancellationToken);
}
```

In tests, pass `FakeTimeProvider`.

### Object pooling

Reuse expensive objects (`StringBuilder`, buffers, `JsonSerializer` state) with `Microsoft.Extensions.ObjectPool`:

```C#
services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
services.AddSingleton(sp =>
    sp.GetRequiredService<ObjectPoolProvider>().CreateStringBuilderPool());

public class Renderer(ObjectPool<StringBuilder> pool)
{
    public string Render()
    {
        var sb = pool.Get();
        try
        {
            sb.Append("ok");
            return sb.ToString();
        }
        finally
        {
            pool.Return(sb);
        }
    }
}
```

For byte buffers, prefer `ArrayPool<T>.Shared` or `IMemoryOwner<T>` from `MemoryPool<T>`. **Always return** what you rent (`try`/`finally`). `Rent(n)` may yield a larger array—only use `n` bytes. Do not rent tens of MB as a substitute for streaming. See [ArrayPool leaks](AspNetCoreGuidance.md#avoid-arraypoolt-leaks-or-oversized-rentals).

### `IPooledObjectPolicy<T>`

`ObjectPool<T>` in Microsoft.Extensions uses a **policy** for create/return (the same idea as ArrayPool, but for objects):

```C#
public sealed class StringBuilderPolicy : IPooledObjectPolicy<StringBuilder>
{
    public StringBuilder Create() => new(256);

    public bool Return(StringBuilder obj)
    {
        if (obj.Capacity > 64 * 1024)
            return false; // drop oversized instances
        obj.Clear();
        return true;
    }
}

services.AddSingleton(sp =>
    sp.GetRequiredService<ObjectPoolProvider>().Create(new StringBuilderPolicy()));
```

Kestrel pools `HttpContext`, `CancellationTokenSource` (`TryReset`), and pipes this way. Always **reset** on return.

### `IBufferWriter<T>` / `PipeReader`

Kestrel, `System.Text.Json`, SignalR, and `HttpClient` write to [`IBufferWriter<byte>`](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.ibufferwriter-1) and read with [`PipeReader`](https://learn.microsoft.com/en-us/dotnet/api/system.io.pipelines.pipereader) (`Request.BodyReader` / `Response.BodyWriter`). Avoid `byte[]` + `Stream` copies in high-throughput libraries.

```C#
PipeReader reader = context.Request.BodyReader;
ReadResult result = await reader.ReadAsync(context.RequestAborted);
ReadOnlySequence<byte> buffer = result.Buffer;
// parse buffer ...
reader.AdvanceTo(consumed, examined);
```

Always `AdvanceTo(consumed, examined)`. Do not `Complete()` the request pipe unless you own it. For length-prefix / line parsers, prefer this over `Stream.Read` + `new byte[]` per frame. See [Prefer `PipeReader`](AspNetCoreGuidance.md#prefer-pipereader--pipewriter-for-custom-framing).

### `StringValues` / `StringSegment`

ASP.NET headers, query strings, and form fields are [`StringValues`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.primitives.stringvalues) (0, 1, or many strings without `List<string>`). Parsing uses [`StringSegment`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.primitives.stringsegment) over a slice of the original string.

```C#
StringValues accept = context.Request.Headers.Accept; // may be 0..n
if (StringValues.IsNullOrEmpty(accept)) { /* ... */ }

foreach (var value in accept) { /* each header line */ }
```

Kestrel, `HeaderDictionary`, and `QueryHelpers` are built on this. Prefer `StringValues` in public HTTP APIs instead of `string[]` so you match the stack and avoid extra arrays.

### `JsonSerializerContext` (source-gen JSON)

ASP.NET, gRPC JSON transcoding, and AOT use source-generated metadata instead of reflection:

```C#
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<Order>))]
internal partial class AppJsonContext : JsonSerializerContext { }

services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));

var json = JsonSerializer.Serialize(order, AppJsonContext.Default.Order);
```

Libraries that ship AOT-friendly APIs expose a `JsonSerializerContext` or `IJsonTypeInfoResolver`. Trimming/AOT will warn if you only use reflection `JsonSerializer.Serialize(obj)`.

### Library `ConfigureAwait(false)`

Microsoft.Extensions and BCL libraries use `ConfigureAwait(false)` on **almost every await** so they do not capture a UI / legacy ASP.NET sync context. App code in ASP.NET Core usually should not. See [`ConfigureAwait`](AsyncGuidance.md#configureawait).

```C#
public async Task<string> ReadAllAsync(Stream stream, CancellationToken cancellationToken)
{
    using var reader = new StreamReader(stream, leaveOpen: true);
    return await reader.ReadToEndAsync(cancellationToken).ConfigureAwait(false);
}
```

## Related guides

- [AspNetCoreGuidance.md](AspNetCoreGuidance.md) — `HttpContext`, request bodies, captive scoped services
- [AsyncGuidance.md](AsyncGuidance.md) — `async`/`await`, `ConfigureAwait`, channels, Runtime Async
- [HttpClientGuidance.md](HttpClientGuidance.md) — `HttpClient` lifetime and platform handlers
- [Gotchas.md](Gotchas.md) — `Random.Shared`, `GeneratedRegex`, `StringComparison`, `ThrowIfNull`
