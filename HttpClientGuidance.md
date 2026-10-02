# Table of contents

- [Using `HttpClient`](#using-httpclient)
- [Prefer `IHttpClientFactory`](#prefer-ihttpclientfactory)
- [`Handler lifetime and DNS`](#handler-lifetime-and-dns)
- [Dispose the response](#dispose-the-response)
- [`BaseAddress` and the relative URI](#baseaddress-and-the-relative-uri)
- [`Timeout` is not the caller's token](#timeout-is-not-the-callers-token)
- [Cookies are shared with the handler](#cookies-are-shared-with-the-handler)
- [`SocketsHttpHandler` defaults](#socketshttphandler-defaults)
- [Platform handlers](#platform-handlers)
- [A note about `WebClient`](#a-note-about-webclient)
- [Related guides](#related-guides)

# Using `HttpClient`

[`HttpClient`](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpclient) is the outbound HTTP API. It is a thin wrapper over an `HttpMessageHandler`. Lifetime of the **handler** is what matters: create-per-request exhausts sockets; a never-recycled handler never refreshes DNS.

## Prefer `IHttpClientFactory`

❌ **BAD** New client per call (and `using` disposes the handler — worse).

```C#
public async Task<string> GetAsync()
{
    using var client = new HttpClient();
    return await client.GetStringAsync("https://example.com");
}
```

:white_check_mark: **GOOD** Typed client (handler pooled by the factory).

```C#
builder.Services.AddHttpClient<DocsClient>(client =>
{
    client.BaseAddress = new Uri("https://example.com");
    client.Timeout = TimeSpan.FromSeconds(10);
});

public sealed class DocsClient(HttpClient client)
{
    public Task<string> GetAsync(CancellationToken cancellationToken) =>
        client.GetStringAsync("doc.json", cancellationToken);
}
```

Do not dispose the `HttpClient` injected into a typed client. That instance is the one later methods on the same object still use, and `Dispose` makes those calls throw `ObjectDisposedException`. `using` around `IHttpClientFactory.CreateClient()` is safe: it does not dispose the pooled handler. `using` around `new HttpClient()` does, and that is the socket leak above. See [Prefer `IHttpClientFactory`](AspNetCoreGuidance.md#prefer-ihttpclientfactory-over-new-httpclient).

## Handler lifetime and DNS

`IHttpClientFactory` caches one `HttpMessageHandler` per client name and replaces it when [`HandlerLifetime`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.httpclientfactoryoptions.handlerlifetime) elapses. The default is **2 minutes**. That rotation is what picks up DNS changes. The factory does **not** set `SocketsHttpHandler.PooledConnectionLifetime`.

`HttpClient` instances from the factory are meant to be short-lived. Disposing one does not dispose the handler. A typed client captured by a singleton keeps the first handler, so DNS stays stale. Inject `IHttpClientFactory` and call `CreateClient` per operation.

A process-wide `static HttpClient` never rotates. Set `PooledConnectionLifetime` on its handler. Two minutes matches the factory default. Pick the interval from how often DNS for that host changes. `SendAsync` on that shared instance is safe from many threads. `BaseAddress`, `Timeout`, and `DefaultRequestHeaders` are not: set them when the client is created. Per-call headers belong on `HttpRequestMessage`.

When you supply the handler yourself and the client must live for the process, recycle connections there and disable factory rotation. Leaving rotation on while also setting `PooledConnectionLifetime` closes pools on two clocks. Disabling rotation without `PooledConnectionLifetime` freezes DNS.

```C#
builder.Services.AddHttpClient("github")
    .UseSocketsHttpHandler((handler, _) =>
        handler.PooledConnectionLifetime = TimeSpan.FromMinutes(2))
    .SetHandlerLifetime(Timeout.InfiniteTimeSpan);
```

Long-lived client **without** the factory:

```C#
static readonly HttpClient Client = new(new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2)
}, disposeHandler: true);
```

## Dispose the response

The handler will not reuse a connection while the response body is still open. `GetStringAsync` and `GetFromJsonAsync` close it. `GetAsync` / `SendAsync` do not.

❌ **BAD** The connection stays checked out until the finalizer runs.

```C#
HttpResponseMessage response = await client.GetAsync(url, cancellationToken);
string body = await response.Content.ReadAsStringAsync(cancellationToken);
```

:white_check_mark: **GOOD**

```C#
using HttpResponseMessage response = await client.GetAsync(url, cancellationToken);
string body = await response.Content.ReadAsStringAsync(cancellationToken);
```

`GetAsync` also buffers the whole body before it returns. For a large download, ask for headers only and then read the stream:

```C#
using HttpResponseMessage response = await client.GetAsync(
    url, HttpCompletionOption.ResponseHeadersRead, cancellationToken);
await using Stream body = await response.Content.ReadAsStreamAsync(cancellationToken);
```

:hammer: **Hands-on** Call `GetAsync` in a loop without `using` and watch established connections (`netstat`) stick around. Add `using` and they return to the pool.

## `BaseAddress` and the relative URI

`Uri` combination drops the base path in two common cases: the base has no trailing slash, or the relative URI starts with `/`.

❌ **BAD** Both of these request `https://example.com/v1/orders`. The `api` segment is gone.

```C#
client.BaseAddress = new Uri("https://example.com/api");
await client.GetAsync("v1/orders", cancellationToken);

client.BaseAddress = new Uri("https://example.com/api/");
await client.GetAsync("/v1/orders", cancellationToken);
```

:white_check_mark: **GOOD** Base ends with `/`. Relative URI does not start with `/`.

```C#
client.BaseAddress = new Uri("https://example.com/api/");
await client.GetAsync("v1/orders", cancellationToken); // https://example.com/api/v1/orders
```

## `Timeout` is not the caller's token

Default `HttpClient.Timeout` is **100 seconds**. When it fires, `GetAsync` throws `TaskCanceledException` and the caller's `CancellationToken` is still not canceled. `ConnectTimeout` on `SocketsHttpHandler` defaults to infinite, so this 100 second timer is also what stops a TCP connect that never completes.

How to tell that timeout apart from a real cancel is in [AsyncGuidance](AsyncGuidance.md#httpclienttimeout-is-a-separate-timer).

## Cookies are shared with the handler

The factory pools the handler, and the handler owns one `CookieContainer`. Every caller of that named client shares those cookies. When `HandlerLifetime` elapses the handler is thrown away and the cookies go with it.

If the app needs a cookie jar, do not use `IHttpClientFactory` for that client. A server calling an API with a bearer token usually wants cookies off:

```C#
builder.Services.AddHttpClient("github")
    .UseSocketsHttpHandler((handler, _) => handler.UseCookies = false);
```

## `SocketsHttpHandler` defaults

These are the handler's own defaults. The factory does not change them, except by throwing the whole handler away on `HandlerLifetime`.

| Setting | Default | What people expect |
|---------|---------|-------------------|
| `PooledConnectionLifetime` | infinite | DNS refresh. It never happens until you set this, or until the factory rotates the handler |
| `PooledConnectionIdleTimeout` | 1 minute | Closes an idle connection. A busy connection is unaffected. This is not DNS refresh |
| `ConnectTimeout` | infinite | Bounded in practice by `HttpClient.Timeout` (100 seconds) |
| `MaxConnectionsPerServer` | `int.MaxValue` | Unlimited. .NET Framework `ServicePointManager.DefaultConnectionLimit` was 2, and it does not apply to `SocketsHttpHandler` |
| `AutomaticDecompression` | none | A gzip body stays compressed in `ReadAsStringAsync` |
| `EnableMultipleHttp2Connections` | false | One HTTP/2 connection, many streams. Turn it on only after a server stream cap shows up in traces |

```C#
builder.Services.AddHttpClient("github")
    .UseSocketsHttpHandler((handler, _) =>
    {
        handler.AutomaticDecompression = DecompressionMethods.All;
        handler.MaxConnectionsPerServer = 32;
        handler.ConnectTimeout = TimeSpan.FromSeconds(5);
    });
```

`DecompressionMethods.All` covers gzip, deflate, and Brotli. Setting `ServicePointManager` in the same process does not change this handler.

## Platform handlers

The innermost handler makes the request:

| Handler | Where |
|---------|--------|
| `SocketsHttpHandler` | Default inner handler on .NET Core 2.1+ and .NET 5+ |
| `HttpClientHandler` | Public wrapper. On current .NET it delegates to `SocketsHttpHandler` |
| `WinHttpHandler` | Opt-in Windows handler (`System.Net.Http.WinHttpHandler`). Not the default |

This document is for **server** apps. Prefer `SocketsHttpHandler` settings (`PooledConnectionLifetime`, `ConnectTimeout`, `EnableMultipleHttp2Connections`) over `ServicePointManager` (.NET Framework).

## A note about `WebClient`

`WebClient` is obsolete. Do not use it for new code. It is synchronous by default and has no handler pooling.

❌ **BAD**

```C#
public string DoSomething()
{
    using var client = new WebClient();
    return client.DownloadString("https://example.com");
}
```

:white_check_mark: **GOOD** `IHttpClientFactory` / typed `HttpClient` and `GetStringAsync`.

# Related guides

- [AspNetCoreGuidance.md](AspNetCoreGuidance.md) — factory vs `new HttpClient()`, request abort
- [DotnetPattern.md](DotnetPattern.md) — typed/named clients, `DelegatingHandler`, resilience
- [AsyncGuidance.md](AsyncGuidance.md) — cancellation, sync-over-async
