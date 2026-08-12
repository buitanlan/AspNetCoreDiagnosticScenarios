# Table of contents

- [ASP.NET Core Guidance](#aspnet-core-guidance)
  - [Mental model](#mental-model)
  - [Avoid synchronous Read/Write on `HttpRequest.Body` and `HttpResponse.Body`](#avoid-synchronous-readwrite-on-httprequestbody-and-httpresponsebody)
  - [Prefer `HttpRequest.ReadFormAsync()` over `HttpRequest.Form`](#prefer-httprequestreadformasync-over-httprequestform)
  - [Avoid reading large request or response bodies into memory](#avoid-reading-large-request-or-response-bodies-into-memory)
  - [Prefer streaming serializers (`System.Text.Json`) over buffering then Newtonsoft](#prefer-streaming-serializers-systemtextjson-over-buffering-then-newtonsoft)
  - [Limit request size (DoS)](#limit-request-size-dos)
  - [Do not store `IHttpContextAccessor.HttpContext` in a field](#do-not-store-ihttpcontextaccessorhttpcontext-in-a-field)
  - [Do not access `HttpContext` from multiple threads in parallel](#do-not-access-httpcontext-from-multiple-threads-in-parallel)
  - [Do not use `HttpContext` after the request is complete](#do-not-use-httpcontext-after-the-request-is-complete)
  - [Do not capture `HttpContext` on background work](#do-not-capture-httpcontext-on-background-work)
  - [Do not capture request-scoped services on background work](#do-not-capture-request-scoped-services-on-background-work)
  - [Avoid adding headers after the response has started](#avoid-adding-headers-after-the-response-has-started)
  - [Flow `CancellationToken` / `RequestAborted`](#flow-cancellationtoken--requestaborted)
  - [Do not enable synchronous IO on Kestrel](#do-not-enable-synchronous-io-on-kestrel)
  - [Middleware: call `next` correctly](#middleware-call-next-correctly)
  - [Prefer `IHttpClientFactory` over `new HttpClient()`](#prefer-ihttpclientfactory-over-new-httpclient)
  - [Avoid `.Result` / async work inside DI registration](#avoid-result--async-work-inside-di-registration)
  - [Avoid `BuildServiceProvider()` inside `ConfigureServices`](#avoid-buildserviceprovider-inside-configureservices)
  - [Avoid capturing scoped services in singletons](#avoid-capturing-scoped-services-in-singletons)
  - [Avoid `DbContext` as a singleton](#avoid-dbcontext-as-a-singleton)
  - [Avoid `EnableBuffering` unless you must reread the body](#avoid-enablebuffering-unless-you-must-reread-the-body)
  - [Avoid unbounded static caches](#avoid-unbounded-static-caches)
  - [Prefer `ILogger<T>` over `Console.WriteLine`](#prefer-ilogert-over-consolewriteline)
  - [Prefer structured logging over interpolated strings](#prefer-structured-logging-over-interpolated-strings)
  - [Prefer typed results (`Results<T>`) over untyped `IResult`](#prefer-typed-results-resultst-over-untyped-iresult)
  - [Prefer `ProblemDetails` over ad-hoc error JSON](#prefer-problemdetails-over-ad-hoc-error-json)
  - [Prefer `IOptions<T>` / `IOptionsMonitor<T>` over `IConfiguration`](#prefer-ioptionst--ioptionsmonitort-over-iconfiguration)
  - [Prefer `TimeProvider` over `DateTime.UtcNow`](#prefer-timeprovider-over-datetimeutcnow)
  - [Do not resolve scoped services from the root provider](#do-not-resolve-scoped-services-from-the-root-provider)
  - [Do not resolve disposable transients from the root provider](#do-not-resolve-disposable-transients-from-the-root-provider)
  - [Avoid capturing `IServiceProvider` in singletons](#avoid-capturing-iserviceprovider-in-singletons)
  - [Avoid `ArrayPool<T>` leaks or oversized rentals](#avoid-arraypoolt-leaks-or-oversized-rentals)
  - [Avoid inspecting `HttpResponse` after `next` returns](#avoid-inspecting-httpresponse-after-next-returns)
  - [Prefer `PipeReader` / `PipeWriter` for custom framing](#prefer-pipereader--pipewriter-for-custom-framing)
- [Related guides](#related-guides)

# ASP.NET Core Guidance

ASP.NET Core is fully asynchronous end-to-end. Kestrel, the request pipeline, model binding, and `HttpContext` are built around that. Most production outages in this area are not "slow SQL"—they are **blocked thread-pool threads**, **request-scoped objects used after the request ends**, or **unbounded buffering** (OOM / LOH / DoS).

This guide is the HTTP-pipeline companion to [AsyncGuidance.md](AsyncGuidance.md). Examples use bad/good pairs. Each section has a **Hands-on** you can paste into a minimal API or controller. Applies to MVC, Razor Pages, and minimal APIs—the `HttpContext` rules are the same.

## Mental model

1. **One `HttpContext` per request, recycled when the pipeline `Task` completes.** After that, the instance may be pooled/reused. Holding it is a use-after-free.
2. **`HttpContext` is not thread-safe.** Concurrent `WhenAll` that touches `Request`/`Response`/`User` is undefined behavior.
3. **Kestrel does not support synchronous body IO by default** (`AllowSynchronousIO` is false). Sync `Stream.Read`/`Write` is sync-over-async and can starve the pool. See [sync over async](AsyncGuidance.md#avoid-using-taskresult-and-taskwait).
4. **The first write to the response sends headers.** After `HasStarted`, status, headers, and cookies are frozen.
5. **Request DI scope dies with the request.** `DbContext` and other scoped services must not outlive it.
6. **`IHttpContextAccessor.HttpContext` is ambient and request-thread oriented.** It is `null` (or wrong) on thread-pool work after the request finishes.

### Avoid / prefer at a glance

| Avoid | Prefer |
|-------|--------|
| Sync `Request.Body` / `Response.Body` / `Request.Form` | Async overloads, `[FromBody]`, `ReadFormAsync` |
| `ReadToEnd` + Newtonsoft into a `string` | `JsonSerializer.DeserializeAsync` / model binding |
| Unbounded body size | Kestrel / `[RequestSizeLimit]` / `FormOptions` |
| `IHttpContextAccessor.HttpContext` stored in a field | Read at use time, or pass `User` / values in |
| `HttpContext` from `WhenAll` / `Task.Run` | Copy scalars on the request thread; hosted queue for background |
| Scoped `DbContext` captured after the request | New `IServiceScope` / `BackgroundService` |
| Headers/status after `HasStarted` | Set before write, or `OnStarting` |
| `AllowSynchronousIO = true` | Fix callers; STJ / `[FromBody]` |
| `new HttpClient()` per request (or per-using dispose of a shared handler) | `IHttpClientFactory` / typed clients |
| `.Result` in `AddSingleton` / ctors | Async factory, `IHostedService.StartAsync`, lazy connect |
| `BuildServiceProvider()` in `ConfigureServices` | `IOptions`, `IHostedService`, or resolve in `StartAsync` |
| Singleton that takes a scoped service in the ctor | `IServiceScopeFactory`, or make the consumer scoped |
| `AddDbContext` as singleton / one context for `WhenAll` | Scoped context, or `IDbContextFactory<T>` |
| `Request.EnableBuffering()` on every request | Only when you must reread; then rewind + size limit |
| Static `ConcurrentDictionary` with no eviction | `IMemoryCache` / `HybridCache` with size + expiration |
| `Task.Run` from an action | [Channel + `BackgroundService`](AsyncGuidance.md#prefer-channelt-and-hosted-services-for-background-work) |
| `new CancellationTokenSource()` in an action | `CancellationToken` / `RequestAborted` |
| `Console.WriteLine` in app code | `ILogger<T>` |
| `$"id={id}"` in logs | Message templates / `[LoggerMessage]` |
| Untyped `Results.Ok` when OpenAPI matters | `TypedResults` / `Results<Ok<T>, NotFound>` |
| Ad-hoc `{ error = "..." }` JSON | `ProblemDetails` / `IProblemDetailsService` |
| Inject `IConfiguration` into services | `IOptions<T>` / `IOptionsMonitor<T>` |
| `DateTime.UtcNow` in testable code | `TimeProvider` |
| `GetRequiredService<Scoped>()` on the root | `CreateAsyncScope()` |
| `IDisposable` transient resolved from root | Resolve from a request/job scope |
| Singleton holds `IServiceProvider` | `IServiceScopeFactory` |
| Call `next` twice, or skip it with no response | Call `next` **exactly once**, or short-circuit fully |
| Read `Response` after `await next()` | `OnStarting`, or wrap the body **before** `next` |
| `ArrayPool.Rent` without `Return` | `try`/`finally` return; don't over-rent |
| `Stream` + `byte[]` loops for framing | `PipeReader` / `Request.BodyReader` |
| Custom `SocketsHttpHandler` with no lifetime | `PooledConnectionLifetime` (factory default 2 min) |

## Avoid synchronous Read/Write on `HttpRequest.Body` and `HttpResponse.Body`

Prefer async overloads (`ReadAsync`, `WriteAsync`, `CopyToAsync`, `ReadToEndAsync`). Sync reads on Kestrel throw or block a pool thread while the client trickles bytes.

❌ **BAD** Sync `ReadToEnd` + Newtonsoft. Slow clients + large bodies = starvation and LOH pressure.

```C#
[HttpPost("/pokemon")]
public ActionResult<PokemonData> Post()
{
    var json = new StreamReader(Request.Body).ReadToEnd();
    return JsonConvert.DeserializeObject<PokemonData>(json);
}
```

:white_check_mark: **GOOD** Let MVC / minimal APIs bind `[FromBody]` (framework buffers/streams correctly).

```C#
[HttpPost("/pokemon")]
public ActionResult<PokemonData> Post([FromBody] PokemonData data) => data;
```

:white_check_mark: **GOOD** Manual read: async + `System.Text.Json` streaming (no extra `string`).

```C#
[HttpPost("/pokemon")]
public async Task<ActionResult<PokemonData>> Post(CancellationToken cancellationToken)
{
    var data = await JsonSerializer.DeserializeAsync<PokemonData>(Request.Body, cancellationToken: cancellationToken);
    return data is null ? BadRequest() : Ok(data);
}
```

:bulb: **NOTE:** `ReadToEndAsync` into a `string` still allocates the whole payload. Prefer deserialize-from-stream. See [large bodies](#avoid-reading-large-request-or-response-bodies-into-memory).

:hammer: **Hands-on** POST a large JSON file with `curl --limit-rate 10k`. The sync action holds a thread for the whole upload; the async/`[FromBody]` action does not. With `AllowSynchronousIO` false (default), sync read often throws `InvalidOperationException: Synchronous operations are disallowed`.

## Prefer `HttpRequest.ReadFormAsync()` over `HttpRequest.Form`

`Request.Form` can block (sync-over-async) if the form has not been read yet. Use `ReadFormAsync`. Reading `Request.Form` is only safe after that (cached).

❌ **BAD**

```C#
[HttpPost("/form-body")]
public IActionResult Post()
{
    var form = Request.Form; // may block
    Process(form["id"], form["name"]);
    return Accepted();
}
```

:white_check_mark: **GOOD**

```C#
[HttpPost("/form-body")]
public async Task<IActionResult> Post(CancellationToken cancellationToken)
{
    var form = await Request.ReadFormAsync(cancellationToken);
    Process(form["id"], form["name"]);
    return Accepted();
}
```

:white_check_mark: **GOOD** Model bind (`[FromForm]`, `IFormFile`) and let the framework read the form.

:hammer: **Hands-on** Slow-upload a `multipart/form-data` POST against `Request.Form` vs `ReadFormAsync` with a tiny thread pool (see [AsyncGuidance starvation lab](AsyncGuidance.md#thread-pool-and-starvation)). The sync path queues up; the async path does not.

## Avoid reading large request or response bodies into memory

Allocations ≥ ~85KB land on the large object heap ([LOH](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/large-object-heap)). LOH allocations are expensive (zeroed memory) and collected with Gen2 (full GC). Buffering a 20MB JSON body per request is a DoS: memory spike, full GCs, Kestrel accepts fewer connections.

Prefer:

- `[FromBody]` / `JsonSerializer.DeserializeAsync` (stream)
- `Request.Body.CopyToAsync(destination)` to a file or pipe
- `IAsyncEnumerable<T>` / `JsonSerializer.SerializeAsync` for large responses
- `ArrayPool<byte>` / `PipeReader` (`Request.BodyReader`) for manual framing

❌ **BAD** Entire body as `string` / `byte[]`.

```C#
var json = await new StreamReader(Request.Body).ReadToEndAsync();
var bytes = await Request.Body.ReadAllBytes(); // same problem
```

:white_check_mark: **GOOD** Stream to disk or deserialize incrementally.

```C#
[HttpPost("/upload")]
[RequestSizeLimit(100_000_000)]
public async Task<IActionResult> Upload(CancellationToken cancellationToken)
{
    var path = Path.GetTempFileName();
    await using (var file = File.Create(path))
    {
        await Request.Body.CopyToAsync(file, cancellationToken);
    }
    return Accepted();
}
```

:hammer: **Hands-on** `dotnet-counters monitor System.Runtime --name <app>` while POSTing 10× 30MB bodies to a `ReadToEndAsync` endpoint vs `CopyToAsync` to a file. Watch `loh-size` / Gen2 collections.

## Prefer streaming serializers (`System.Text.Json`) over buffering then Newtonsoft

The old workaround for JSON.NET (sync-only `Deserialize`) was: `CopyToAsync` into a `MemoryStream`, then sync deserialize. That still puts the whole payload on the heap.

Today the default is `System.Text.Json`, which has `DeserializeAsync` / `SerializeAsync`. MVC uses it unless you add Newtonsoft.

❌ **BAD** Buffer everything so a sync serializer can run.

```C#
await using var buffer = new MemoryStream();
await Request.Body.CopyToAsync(buffer);
buffer.Position = 0;
var data = JsonConvert.DeserializeObject<PokemonData>(Encoding.UTF8.GetString(buffer.ToArray()));
```

:white_check_mark: **GOOD** Stream.

```C#
var data = await JsonSerializer.DeserializeAsync<PokemonData>(Request.Body, cancellationToken: cancellationToken);
await JsonSerializer.SerializeAsync(Response.Body, data, cancellationToken: cancellationToken);
```

If you **must** use Newtonsoft, still read the stream asynchronously (`JsonTextReader` + `JToken.ReadFromAsync`) rather than `ReadToEnd` + `DeserializeObject`. See `Scenarios/Controllers/BigJsonInputController.cs`.

:bulb: **NOTE:** Returning `IAsyncEnumerable<T>` from an endpoint streams JSON arrays; watch serializer buffering settings and client cancel.

:hammer: **Hands-on** Compare allocations (`GC.GetAllocatedBytesForCurrentThread` or `dotnet-trace`) for `ReadToEndAsync` + `JsonConvert` vs `JsonSerializer.DeserializeAsync` on a 5MB payload.

## Limit request size (DoS)

Streaming does not make unbounded bodies safe. Kestrel and form limits exist so one client cannot fill the disk or LOH.

Defaults (check your version): Kestrel `MaxRequestBodySize` is 30MB; form options have their own caps (`MultipartBodyLengthLimit`, `ValueCountLimit`).

```C#
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10 MB
});

builder.Services.Configure<FormOptions>(options =>
{
    options.MultipartBodyLengthLimit = 10 * 1024 * 1024;
});
```

Per endpoint:

```C#
[HttpPost("/upload")]
[RequestSizeLimit(1_000_000)]
[RequestFormLimits(MultipartBodyLengthLimit = 1_000_000)]
public async Task<IActionResult> Upload(IFormFile file) { /* ... */ }
```

❌ **BAD** `[DisableRequestSizeLimit]` on a public endpoint with no auth, quota, or streaming-to-blob strategy.

:hammer: **Hands-on** POST a body larger than the limit. You should get `413 Payload Too Large`, not an OOM.

## Do not store `IHttpContextAccessor.HttpContext` in a field

`IHttpContextAccessor.HttpContext` returns the **current** request's context when called on a request flow. Caching the `HttpContext` reference in a constructor captures whatever was current then (often `null` in a singleton), or a context that will be recycled.

❌ **BAD**

```C#
public class AdminGate
{
    private readonly HttpContext _context;

    public AdminGate(IHttpContextAccessor accessor) => _context = accessor.HttpContext!;

    public void EnsureAdmin()
    {
        if (!_context.User.IsInRole("admin"))
            throw new UnauthorizedAccessException();
    }
}
```

:white_check_mark: **GOOD** Store the accessor (or, better, pass `HttpContext`/`ClaimsPrincipal` as a method argument). Read `HttpContext` at use time; null-check.

```C#
public class AdminGate
{
    private readonly IHttpContextAccessor _accessor;

    public AdminGate(IHttpContextAccessor accessor) => _accessor = accessor;

    public void EnsureAdmin()
    {
        var user = _accessor.HttpContext?.User
            ?? throw new InvalidOperationException("No active request");
        if (!user.IsInRole("admin"))
            throw new UnauthorizedAccessException();
    }
}
```

Prefer **not** using `IHttpContextAccessor` in application services at all. Pass `HttpContext.User` from the endpoint. The accessor is easy to call from a singleton on a background thread and get `null` or another request's user.

:hammer: **Hands-on** Register `AdminGate` as **singleton**, capture `HttpContext` in the ctor, then call `EnsureAdmin` from two concurrent requests. You will see null, wrong user, or a recycled context. Capturing the accessor and reading it per call (on the request thread) is consistent.

## Do not access `HttpContext` from multiple threads in parallel

`HttpContext` is **not** thread-safe. `Task.WhenAll` that logs `HttpContext.Request.Path` (or writes the response) from several tasks is a race: hangs, crashes, mixed headers, corrupted features.

❌ **BAD** Parallel tasks all read `HttpContext`.

```C#
[HttpGet("/search")]
public async Task<SearchResults> Get(string query)
{
    var t1 = SearchAsync(SearchEngine.Google, query);
    var t2 = SearchAsync(SearchEngine.Bing, query);
    await Task.WhenAll(t1, t2);
    return SearchResults.Combine(await t1, await t2);
}

private async Task<SearchResults> SearchAsync(SearchEngine engine, string query)
{
    _logger.LogInformation("start {path}", HttpContext.Request.Path);
    var result = await _searchService.SearchAsync(engine, query);
    _logger.LogInformation("end {path}", HttpContext.Request.Path);
    return result;
}
```

:white_check_mark: **GOOD** Copy the values you need on the request thread, then fan out.

```C#
[HttpGet("/search")]
public async Task<SearchResults> Get(string query)
{
    var path = HttpContext.Request.Path.Value;
    var t1 = SearchAsync(SearchEngine.Google, query, path);
    var t2 = SearchAsync(SearchEngine.Bing, query, path);
    await Task.WhenAll(t1, t2);
    return SearchResults.Combine(await t1, await t2);
}

private async Task<SearchResults> SearchAsync(SearchEngine engine, string query, string? path)
{
    _logger.LogInformation("start {path}", path);
    return await _searchService.SearchAsync(engine, query);
}
```

Same rule: do not `WriteAsync` the response from two tasks at once. See [Do not share non-thread-safe state](AsyncGuidance.md#do-not-share-non-thread-safe-state-across-concurrent-awaits).

:hammer: **Hands-on** Fan-out 50 parallel `HttpContext.Request.Headers` reads under load. You may not always crash—that is why this bug ships. Copy `path` first and the race disappears.

## Do not use `HttpContext` after the request is complete

When the pipeline's `Task` completes, ASP.NET Core recycles the `HttpContext`. Writing or reading it afterward is use-after-free (often a process crash).

❌ **BAD** `async void` action: the framework finishes the request at the first incomplete point it does not await, then your continuation writes.

```C#
[HttpGet("/async")]
public async void Get()
{
    await Task.Delay(1000);
    await Response.WriteAsync("Hello World"); // request already completed
}
```

:white_check_mark: **GOOD** Return `Task` so the request stays open until the action finishes.

```C#
[HttpGet("/async")]
public async Task Get()
{
    await Task.Delay(1000);
    await Response.WriteAsync("Hello World");
}
```

`async void` is **always** wrong in ASP.NET Core. See [Async void](AsyncGuidance.md#async-void).

:hammer: **Hands-on** Hit the `async void` endpoint. Process crash or `ObjectDisposedException` / `NullReferenceException` on `Response`. Change to `async Task` and it returns `Hello World`.

## Do not capture `HttpContext` on background work

Closures on `Task.Run` / `QueueUserWorkItem` that touch `HttpContext`, `Request`, `Response`, or `Controller` properties run **after** the request may have finished. The context is recycled; you log another request's path or crash.

❌ **BAD**

```C#
[HttpGet("/fire-and-forget")]
public IActionResult FireAndForget()
{
    _ = Task.Run(async () =>
    {
        await Task.Delay(1000);
        var path = HttpContext.Request.Path; // recycled / wrong request
        Log(path);
    });
    return Accepted();
}
```

:white_check_mark: **GOOD** Copy scalars during the request. Better: do not `Task.Run` from an action at all—use a [hosted queue](AsyncGuidance.md#prefer-channelt-and-hosted-services-for-background-work).

```C#
[HttpGet("/fire-and-forget")]
public IActionResult FireAndForget()
{
    var path = HttpContext.Request.Path.Value;
    var traceId = HttpContext.TraceIdentifier;
    _ = Task.Run(async () =>
    {
        await Task.Delay(1000);
        Log(path, traceId);
    });
    return Accepted();
}
```

:hammer: **Hands-on** Delay 2s in `Task.Run` and log `HttpContext.TraceIdentifier`. Fire two requests quickly. You will often see the second request's id in the first job—or a crash.

## Do not capture request-scoped services on background work

`DbContext`, `HttpClient` typed as scoped, and anything registered `AddScoped` is created per request and **disposed when the request ends**. Using it in `Task.Run` → `ObjectDisposedException` (or silent data loss).

❌ **BAD**

```C#
[HttpGet("/fire-and-forget")]
public IActionResult FireAndForget([FromServices] PokemonDbContext context)
{
    _ = Task.Run(async () =>
    {
        await Task.Delay(1000);
        context.Pokemon.Add(new Pokemon());
        await context.SaveChangesAsync(); // disposed
    });
    return Accepted();
}
```

:white_check_mark: **GOOD** (if work must continue after the response) create a **new** scope. Use `IServiceScopeFactory` (singleton). Prefer `CreateAsyncScope` when services may be `IAsyncDisposable`.

```C#
[HttpGet("/fire-and-forget")]
public IActionResult FireAndForget([FromServices] IServiceScopeFactory scopeFactory)
{
    _ = Task.Run(async () =>
    {
        try
        {
            await using var scope = scopeFactory.CreateAsyncScope();
            var context = scope.ServiceProvider.GetRequiredService<PokemonDbContext>();
            context.Pokemon.Add(new Pokemon());
            await context.SaveChangesAsync();
        }
        catch (Exception ex)
        {
            // log — otherwise unobserved
        }
    });
    return Accepted();
}
```

:white_check_mark: **BETTER** Enqueue to `BackgroundService` / message bus. `Task.Run` from an action still loses shutdown, retries, and backpressure. See [Fire-and-forget](AsyncGuidance.md#fire-and-forget-and-unobserved-exceptions).

:hammer: **Hands-on** Capture `DbContext` in `Task.Run` + `Delay(1000)`. You get `ObjectDisposedException`. `CreateAsyncScope` inside the job succeeds.

## Avoid adding headers after the response has started

ASP.NET Core does **not** buffer the whole response. The first body write flushes headers. After that, `Headers`, `StatusCode`, `Cookies`, and `ContentType` cannot change (`InvalidOperationException` or ignored).

❌ **BAD** Middleware writes, then `next()`, then sets a header (and the inner middleware may have written too).

```C#
app.Use(async (context, next) =>
{
    await context.Response.WriteAsync("Hello ");
    await next();
    context.Response.Headers["test"] = "value"; // often too late
});
```

:white_check_mark: **GOOD** Set headers **before** any write, or use `OnStarting` (runs just before headers are sent).

```C#
app.Use(async (context, next) =>
{
    context.Response.OnStarting(() =>
    {
        context.Response.Headers["someheader"] = "somevalue";
        return Task.CompletedTask;
    });
    await next();
});
```

Check `context.Response.HasStarted` before mutating status/headers. Do not use `HasStarted` as a substitute for `OnStarting` when you need a header on every response that *does* write.

:hammer: **Hands-on** `WriteAsync` then set `StatusCode = 500`. Throws or no-ops. Register `OnStarting` before `next()` and the header appears on the wire (`curl -v`).

## Flow `CancellationToken` / `RequestAborted`

Actions and minimal APIs should take `CancellationToken`. The host binds it to `HttpContext.RequestAborted`. Passing it to EF, `HttpClient`, and `ReadAsync` stops work when the client disconnects.

❌ **BAD** Ignore abort; keep running a 30s query after the client is gone.

```C#
app.MapGet("/report", async (AppDbContext db) =>
    await db.Rows.ToListAsync()); // no token
```

:white_check_mark: **GOOD**

```C#
app.MapGet("/report", async (AppDbContext db, CancellationToken cancellationToken) =>
    await db.Rows.ToListAsync(cancellationToken));
```

Prefer **`HttpContext.RequestAborted`** (or the action's `CancellationToken`, which is the same token) over creating an unrelated `CancellationTokenSource`. A new CTS does not cancel when the client disconnects, so you keep running work for a gone caller.

❌ **BAD** Fresh timeout token that ignores abort.

```C#
app.MapGet("/report", async (AppDbContext db) =>
{
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
    return await db.Rows.ToListAsync(cts.Token);
});
```

:white_check_mark: **GOOD** Link abort with a timeout when you need both.

```C#
app.MapGet("/report", async (AppDbContext db, CancellationToken cancellationToken) =>
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    cts.CancelAfter(TimeSpan.FromSeconds(30));
    return await db.Rows.ToListAsync(cts.Token);
});
```

Do **not** pass `RequestAborted` into work that must outlive the request (that's the hosted-service case). Use `CancellationToken.None` or the host's `IHostApplicationLifetime.ApplicationStopping` there.

See [Always flow CancellationToken](AsyncGuidance.md#always-flow-cancellationtokens-to-apis-that-take-a-cancellationtoken).

:hammer: **Hands-on** `await Task.Delay(TimeSpan.FromSeconds(30), cancellationToken)` then cancel the client (`curl` Ctrl+C). With the token, the action ends immediately; without it, the delay runs to 30s (watch with logging).

## Do not enable synchronous IO on Kestrel

```C#
// ❌ BAD — hides sync-over-async instead of fixing callers
builder.WebHost.ConfigureKestrel(o => o.AllowSynchronousIO = true);
```

This was a common "fix" when JSON.NET or old libraries called `Stream.Read`. It puts starvation back on the table. Prefer async serializers and `[FromBody]`.

:hammer: **Hands-on** Sync `ReadToEnd` with default Kestrel: `InvalidOperationException`. Setting `AllowSynchronousIO = true` makes it "work" and will stall under `ab -c 50` with slow bodies.

## Middleware: call `next` correctly

The delegate is `(HttpContext context, RequestDelegate next)` — **context first**. Older samples swapped the names and still compiled if unused.

Call `next` **exactly once**, or not at all if you fully handled the response.

```C#
app.Use(async (context, next) =>
{
    // before — headers/status still mutable
    await next(context);
    // after — often HasStarted; do not mutate the response (see below)
});
```

In `app.Use(async (HttpContext context, RequestDelegate next) => ...)`, call `await next(context)`. The `Func<Task>` overload uses `await next()` with no argument. Do not mix them.

❌ **BAD** Twice — downstream middleware, the endpoint, and `SaveChanges` run twice.

```C#
await next(context);
await next(context);
```

❌ **BAD** Never call `next` and never write — the client hangs until the request times out.

```C#
app.Use(async (context, next) =>
{
    if (context.Request.Path.StartsWithSegments("/admin"))
        return; // no status, no body, no next
});
```

:white_check_mark: **GOOD** Short-circuit only with a completed response.

```C#
app.Use(async (context, next) =>
{
    if (!context.Request.Path.StartsWithSegments("/admin"))
    {
        await next(context);
        return;
    }

    context.Response.StatusCode = StatusCodes.Status401Unauthorized;
    await context.Response.WriteAsync("unauthorized");
});
```

- Do not swallow exceptions without rethrowing or writing a completed error response.
- `return next(context)` without `await` is fine only if you have no code after it.

:hammer: **Hands-on** Log `HasStarted` before and after `await next(context)` on a controller that returns `Ok("hi")`. After `next`, `HasStarted` is true. Call `next` twice and watch the action log two hits.

## Prefer `IHttpClientFactory` over `new HttpClient()`

`new HttpClient()` per request exhausts sockets (`TIME_WAIT`). `using var client = new HttpClient()` disposes the handler and makes it worse. A process-wide `static HttpClient` avoids that but **never refreshes DNS**.

❌ **BAD**

```C#
[HttpGet("/httpclient-1")]
public async Task<string> Get()
{
    var client = new HttpClient(); // sockets leak
    return await client.GetStringAsync("https://example.com");
}
```

❌ **BAD**

```C#
using var client = new HttpClient(); // disposes the handler; socket exhaustion under load
```

:white_check_mark: **GOOD** Typed client or factory (see `Scenarios/Controllers/HttpClientController.cs`).

```C#
builder.Services.AddHttpClient<PokemonService>(client =>
{
    client.BaseAddress = new Uri("https://example.com");
    client.Timeout = TimeSpan.FromSeconds(10);
});

public class PokemonService(HttpClient client)
{
    public Task<Stream> GetAsync(CancellationToken cancellationToken) =>
        client.GetStreamAsync("pokedex.json", cancellationToken);
}
```

Factory-created clients should **not** be disposed by you in a way that disposes the handler (do not wrap `CreateClient()` in `using`—prefer typed clients).

`IHttpClientFactory` sets [`SocketsHttpHandler.PooledConnectionLifetime`](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.socketshttphandler.pooledconnectionlifetime) to **2 minutes** so DNS changes are picked up. A process-wide `static HttpClient` never does. If you **replace** the primary handler, you must set the lifetime yourself—the factory default is gone.

```C#
builder.Services.AddHttpClient("github")
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(2),
        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(1)
    });
```

A long-lived `HttpClient` **not** created by the factory:

```C#
new SocketsHttpHandler { PooledConnectionLifetime = TimeSpan.FromMinutes(2) }
```

More in [HttpClientGuidance.md](HttpClientGuidance.md).

:hammer: **Hands-on** `ab -n 10000 -c 50` against `new HttpClient()` vs `IHttpClientFactory`. Watch ephemeral ports (`netstat`) climb on the bad path.

## Avoid `.Result` / async work inside DI registration

`AddSingleton(sp => ConnectAsync().Result)` is sync-over-async **during resolve**, holds the container lock, and can **deadlock** if the continuation tries to resolve another service. This repo’s `Startup.cs` shows exactly that.

❌ **BAD**

```C#
services.AddSingleton(sp =>
    sp.GetRequiredService<RemoteConnectionFactory>().ConnectAsync().Result);
```

:white_check_mark: **GOOD** Register a factory / lazy connection; connect on first use or in `IHostedService.StartAsync`.

```C#
services.AddSingleton<RemoteConnectionFactory>();
services.AddSingleton<LazyRemoteConnection>(); // connects on first await, not in the ctor
```

```C#
public sealed class Connector : IHostedService
{
    public async Task StartAsync(CancellationToken cancellationToken) =>
        await _factory.ConnectAsync(cancellationToken);
    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

See [Constructors](AsyncGuidance.md#constructors) and `Scenarios/Controllers/AsyncFactoryController.cs`.

:hammer: **Hands-on** Resolve a singleton whose factory `await`s then calls `GetRequiredService` again (the `LoggingRemoteConnection` pattern in `Startup.cs`). The process hangs on first request.

## Avoid `BuildServiceProvider()` inside `ConfigureServices`

That builds a **second** container. Singletons are created twice; `IDisposable` registrations leak; the app’s real provider is not the one you just built.

❌ **BAD**

```C#
public void ConfigureServices(IServiceCollection services)
{
    services.AddFoo();
    var sp = services.BuildServiceProvider(); // second container
    var foo = sp.GetRequiredService<Foo>();
}
```

:white_check_mark: **GOOD** Resolve in `IHostedService.StartAsync`, `IStartupFilter`, or `app.Services` after `Build()`. Use `IOptions<T>` / `IOptionsMonitor<T>` for config.

## Avoid capturing scoped services in singletons

A singleton ctor that takes `DbContext` / `HttpClient` typed as scoped / `IHttpContextAccessor` usage that assumes a request **lives for the process lifetime**. The scoped instance is whatever was current at first resolve—or the container throws.

❌ **BAD**

```C#
services.AddDbContext<AppDbContext>(...); // scoped
services.AddSingleton<OrderProcessor>(); // ctor(AppDbContext db) — captive dependency
```

:white_check_mark: **GOOD** Make `OrderProcessor` scoped, or inject `IServiceScopeFactory` / `IDbContextFactory<AppDbContext>` and create a scope per operation.

:hammer: **Hands-on** `builder.Host.UseDefaultServiceProvider(o => o.ValidateScopes = true)` (default in Development). Resolving the singleton throws `InvalidOperationException: Cannot consume scoped service`.

Related: [root provider](#do-not-resolve-scoped-services-from-the-root-provider), [disposable transients](#do-not-resolve-disposable-transients-from-the-root-provider), [captured `IServiceProvider`](#avoid-capturing-iserviceprovider-in-singletons).

## Avoid `DbContext` as a singleton

EF Core `DbContext` is not thread-safe and is meant to be short-lived. One instance for the app + concurrent requests = `A second operation was started on this context instance` and corrupted tracking.

❌ **BAD** `services.AddSingleton<AppDbContext>()` or `new AppDbContext()` stored in a static field.

:white_check_mark: **GOOD** `AddDbContext` (scoped). For parallel queries or background work, `AddDbContextFactory<T>()` and create a context per operation.

See [Do not share non-thread-safe state](AsyncGuidance.md#do-not-share-non-thread-safe-state-across-concurrent-awaits).

## Avoid `EnableBuffering` unless you must reread the body

`Request.EnableBuffering()` lets middleware read the body and then rewind for model binding. It **buffers the whole body into memory** (disk after a threshold on some versions). Doing it globally reintroduces the large-body DoS.

❌ **BAD**

```C#
app.Use(async (context, next) =>
{
    context.Request.EnableBuffering();
    await next();
});
```

:white_check_mark: **GOOD** Enable only for the endpoints that need it, then rewind, and keep size limits.

```C#
context.Request.EnableBuffering();
await context.Request.Body.CopyToAsync(Stream.Null, context.RequestAborted);
context.Request.Body.Position = 0;
await next();
```

Prefer `IHttpRequestBodyDetectionFeature` / endpoint filters / `[FromBody]` instead of a global buffer.

:hammer: **Hands-on** Global `EnableBuffering` + 30MB POST: memory tracks the body even if the action never reads it.

## Avoid unbounded static caches

A `static ConcurrentDictionary` with no size/expiration is a memory leak under unique keys (`Scenarios/Controllers/MemoryLeakController.cs`).

❌ **BAD**

```C#
private static readonly ConcurrentDictionary<string, string> Cache = new();

[HttpGet("/leak")]
public string Leak(string id) =>
    Cache.GetOrAdd(id, _ => new string('c', 5 * 1024 * 1024));
```

:white_check_mark: **GOOD** `IMemoryCache` / `HybridCache` with `SizeLimit`, expiration, and compaction. Evict faulted `Task`s if you cache async work ([ConcurrentDictionary.GetOrAdd](AsyncGuidance.md#concurrentdictionarygetoradd)).

:hammer: **Hands-on** Hit `/leak?id=` with a new guid each time. Process RSS climbs until OOM. `IMemoryCache` with `SizeLimit` stays flat.

## Prefer `ILogger<T>` over `Console.WriteLine`

`Console.WriteLine` bypasses log levels, scopes, correlation ids, and sinks (App Insights, OTel, files). In ASP.NET Core, inject `ILogger<T>`.

❌ **BAD**

```C#
app.MapPost("/orders", (Order order) =>
{
    Console.WriteLine($"created {order.Id}");
    return Results.Created($"/orders/{order.Id}", order);
});
```

:white_check_mark: **GOOD**

```C#
app.MapPost("/orders", (Order order, ILogger<Program> logger) =>
{
    logger.LogInformation("Created order {OrderId}", order.Id);
    return TypedResults.Created($"/orders/{order.Id}", order);
});
```

Libraries that must run without `AddLogging` take `ILogger<T>?` and fall back to `NullLogger<T>.Instance`. See [NullLogger](DotnetPattern.md#nulllogger-in-libraries).

## Prefer structured logging over interpolated strings

Interpolated strings are one blob. Sinks cannot query `OrderId`; you also allocate the string even when the level is disabled (unless you guard with `IsEnabled`).

❌ **BAD**

```C#
_logger.LogInformation($"Handled {path} in {elapsedMs}ms");
```

:white_check_mark: **GOOD** Message template (named holes, not `$`).

```C#
_logger.LogInformation("Handled {Path} in {ElapsedMs}ms", path, elapsedMs);
```

Hot path: `[LoggerMessage]` so the template is not parsed per call. See [source-generated logging](DotnetPattern.md#source-generated-logging-loggermessage).

## Prefer typed results (`Results<T>`) over untyped `IResult`

`Results.Ok(order)` is `IResult`. OpenAPI and the compiler cannot see the body type or the 404 branch. `TypedResults` / `Results<Ok<T>, NotFound>` make the contract explicit.

❌ **BAD**

```C#
app.MapGet("/orders/{id}", async (OrderId id, IOrderStore store, CancellationToken ct) =>
{
    var order = await store.GetAsync(id, ct);
    return order is null ? Results.NotFound() : Results.Ok(order);
});
```

:white_check_mark: **GOOD**

```C#
app.MapGet("/orders/{id}", async Task<Results<Ok<Order>, NotFound>> (
    OrderId id, IOrderStore store, CancellationToken ct) =>
{
    var order = await store.GetAsync(id, ct);
    return order is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(order);
});
```

MVC: `ActionResult<Order>` is the same idea. Untyped `IResult` is fine for one-off endpoints where you do not generate a schema.

## Prefer `ProblemDetails` over ad-hoc error JSON

Clients and ASP.NET itself speak [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) (`type`, `title`, `status`, `detail`). `{ error = "oops" }` is a private dialect.

❌ **BAD**

```C#
app.MapGet("/orders/{id}", () => Results.Json(new { error = "missing" }, statusCode: 404));
```

:white_check_mark: **GOOD**

```C#
builder.Services.AddProblemDetails();

app.MapGet("/orders/{id}", () =>
    TypedResults.Problem(statusCode: StatusCodes.Status404NotFound, title: "Order not found"));
```

Unhandled exceptions: `AddExceptionHandler<T>` + `IProblemDetailsService`. See [`IExceptionHandler`](DotnetPattern.md#iexceptionhandler--iproblemdetailsservice).

## Prefer `IOptions<T>` / `IOptionsMonitor<T>` over `IConfiguration`

`IConfiguration["Section:Key"]` is untyped, not validated, and scatters magic strings. Bind a POCO once.

❌ **BAD**

```C#
public sealed class Mailer(IConfiguration config)
{
    public void Send() => _ = config["Smtp:Host"];
}
```

:white_check_mark: **GOOD**

```C#
builder.Services.AddOptions<SmtpOptions>()
    .BindConfiguration("Smtp")
    .ValidateDataAnnotations()
    .ValidateOnStart();

public sealed class Mailer(IOptions<SmtpOptions> options)
{
    public void Send() => _ = options.Value.Host;
}
```

When settings **reload** (file, Key Vault, feature flags), inject **`IOptionsMonitor<T>`** (singleton-safe, `OnChange`) instead of `IOptions<T>` (frozen at first resolve). `IOptionsSnapshot<T>` is the per-request view. See [named options](DotnetPattern.md#named-options-ioptionsmonitor-and-ioptionssnapshot).

## Prefer `TimeProvider` over `DateTime.UtcNow`

`DateTime.UtcNow` / `DateTimeOffset.UtcNow` are not fakeable. Inject [`TimeProvider`](https://learn.microsoft.com/en-us/dotnet/api/system.timeprovider) (`TimeProvider.System` in production, `FakeTimeProvider` in tests).

```C#
builder.Services.AddSingleton(TimeProvider.System);

public sealed class TokenService(TimeProvider time)
{
    public DateTimeOffset Expires() => time.GetUtcNow().AddHours(1);
}
```

See [`TimeProvider`](DotnetPattern.md#timeprovider).

## Do not resolve scoped services from the root provider

`app.Services` / `IHost.Services` is the **root**. `GetRequiredService<AppDbContext>()` there makes the context a de-facto singleton: one instance, never disposed per request, concurrent use, memory growth.

❌ **BAD**

```C#
var db = app.Services.GetRequiredService<AppDbContext>(); // root
```

❌ **BAD** Singleton / hosted service:

```C#
public sealed class Worker(IServiceProvider sp) : BackgroundService
{
    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var db = sp.GetRequiredService<AppDbContext>(); // same root
        return Task.CompletedTask;
    }
}
```

:white_check_mark: **GOOD**

```C#
await using var scope = app.Services.CreateAsyncScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
```

`ValidateScopes = true` (Development default) throws. Production often has it off—the leak is silent. See [`IServiceScopeFactory`](DotnetPattern.md#scoped-work-from-a-singleton-iservicescopefactory).

## Do not resolve disposable transients from the root provider

The root **tracks** every `IDisposable` / `IAsyncDisposable` it creates, including **transients**, and disposes them only when the host shuts down. Resolving a disposable transient from `app.Services` in a loop retains every instance for the process lifetime.

❌ **BAD**

```C#
services.AddTransient<IDbConnection, SqlConnection>(); // IDisposable
var conn = app.Services.GetRequiredService<IDbConnection>(); // rooted until shutdown
```

:white_check_mark: **GOOD** Resolve disposable transients from a **scope** (the request scope, or `CreateAsyncScope` in a worker). Prefer **scoped** for connections/`HttpClient` wrappers you own; prefer `IHttpClientFactory` for HTTP.

Registering a disposable as transient is fine **if** callers always resolve it from a scope. The bug is resolving it from the root, not the `AddTransient` call itself.

## Avoid capturing `IServiceProvider` in singletons

A singleton that stores `IServiceProvider` and calls `GetRequiredService<T>()` without `CreateScope()` is the same captive-scope bug: scoped and disposable transients come from the root.

❌ **BAD**

```C#
public sealed class CacheWarmer(IServiceProvider sp) // singleton
{
    public Task WarmAsync() => sp.GetRequiredService<AppDbContext>().Orders.CountAsync();
}
```

:white_check_mark: **GOOD** Inject `IServiceScopeFactory` (or `IDbContextFactory<T>`) and create a scope per operation.

```C#
public sealed class CacheWarmer(IServiceScopeFactory scopes)
{
    public async Task WarmAsync(CancellationToken cancellationToken)
    {
        await using var scope = scopes.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        _ = await db.Orders.CountAsync(cancellationToken);
    }
}
```

`Func<T>` / `Lazy<T>` that closed over the root provider has the same leak. See [resolve by delegate](DotnetPattern.md#resolve-by-delegate-funct--named-delegates).

## Avoid `ArrayPool<T>` leaks or oversized rentals

`ArrayPool<T>.Shared.Rent(n)` may return an array **larger than `n`**. Not returning it leaks pooled buffers (the pool allocates replacements). Returning twice or using the array after `Return` is a data race.

❌ **BAD**

```C#
var buffer = ArrayPool<byte>.Shared.Rent(1024);
await stream.ReadAsync(buffer); // may be 1024..N; no finally
```

:white_check_mark: **GOOD**

```C#
var buffer = ArrayPool<byte>.Shared.Rent(1024);
try
{
    var read = await stream.ReadAsync(buffer.AsMemory(0, 1024), cancellationToken);
    Process(buffer.AsSpan(0, read));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer); // clearArray: true if it held secrets
}
```

Do not `Rent(20_000_000)` to avoid streaming—that is still an LOH allocation. Prefer `PipeReader` / `CopyToAsync`. Drop oversized instances when returning to an `ObjectPool` ([`IPooledObjectPolicy`](DotnetPattern.md#ipooledobjectpolicyt)).

:hammer: **Hands-on** Rent in a loop without `Return`, watch `dotnet-counters` `gen-0-size` / process RSS. Add `finally Return` and it stays flat.

## Avoid inspecting `HttpResponse` after `next` returns

Once the endpoint (or inner middleware) writes, headers, status, and body are **locked** and often **already on the wire**. Terminal middleware that `await next(context)` then reads `Response.Body` or sets `StatusCode` is too late.

❌ **BAD**

```C#
app.Use(async (context, next) =>
{
    await next(context);
    var status = context.Response.StatusCode;           // maybe ok
    context.Response.Headers["X-Done"] = "1";           // often throws / no-op
    var body = await new StreamReader(context.Response.Body).ReadToEndAsync(); // not rewindable
});
```

:white_check_mark: **GOOD** Mutate **before** `next`, or `OnStarting` (headers only). To capture a body, replace `Response.Body` with a limited buffer **before** `next`, then copy out—never as a global middleware without a size cap (see [EnableBuffering](#avoid-enablebuffering-unless-you-must-reread-the-body)).

```C#
app.Use(async (context, next) =>
{
    context.Response.OnStarting(() =>
    {
        context.Response.Headers["X-Trace"] = context.TraceIdentifier;
        return Task.CompletedTask;
    });
    await next(context);
});
```

See [Avoid adding headers after the response has started](#avoid-adding-headers-after-the-response-has-started).

## Prefer `PipeReader` / `PipeWriter` for custom framing

Custom protocols (length-prefix, gRPC-like frames, line parsers) should use `System.IO.Pipelines`, not `Stream.Read` into a new `byte[]` each time. Kestrel already exposes `Request.BodyReader` / `Response.BodyWriter`.

❌ **BAD**

```C#
var header = new byte[4];
_ = await Request.Body.ReadAsync(header, cancellationToken);
var len = BinaryPrimitives.ReadInt32BigEndian(header);
var payload = new byte[len];
_ = await Request.Body.ReadAsync(payload, cancellationToken);
```

`ReadAsync` can return partial buffers; this also copies every frame.

:white_check_mark: **GOOD**

```C#
var reader = context.Request.BodyReader;
while (!context.RequestAborted.IsCancellationRequested)
{
    var result = await reader.ReadAsync(context.RequestAborted);
    var buffer = result.Buffer;
    if (TryParseFrame(ref buffer, out var frame))
        reader.AdvanceTo(buffer.Start, buffer.End);
    else
        reader.AdvanceTo(buffer.Start, buffer.End); // examined, need more

    if (result.IsCompleted) break;
}
```

Always `AdvanceTo(consumed, examined)`. Do not `Complete()` the request pipe unless you own it. See [`IBufferWriter` / `PipeReader`](DotnetPattern.md#ibufferwritert--pipereader).

# Related guides

- [AsyncGuidance.md](AsyncGuidance.md) — `async`/`await`, starvation, fire-and-forget, `ConfigureAwait`, Runtime Async
- [DotnetPattern.md](DotnetPattern.md) — DI, options, factories, modules, tenancy
- [HttpClientGuidance.md](HttpClientGuidance.md) — `HttpClient` lifetime and handlers
- [Gotchas.md](Gotchas.md) — BCL prefers (`Random.Shared`, `GeneratedRegex`, `StringComparison`, `ThrowIfNull`, …)

Out of scope here: authentication/authorization design, gRPC, SignalR, YARP, Blazor circuit threading, and hosting/deploy (those are product docs, not this diagnostic list).
