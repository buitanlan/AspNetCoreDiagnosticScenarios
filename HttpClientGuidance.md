# Table of contents

- [Using `HttpClient`](#using-httpclient)
- [Prefer `IHttpClientFactory`](#prefer-ihttpclientfactory)
- [`PooledConnectionLifetime` and DNS](#pooledconnectionlifetime-and-dns)
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

Do not `using` around `IHttpClientFactory.CreateClient()`. See [Prefer `IHttpClientFactory`](AspNetCoreGuidance.md#prefer-ihttpclientfactory-over-new-httpclient).

## `PooledConnectionLifetime` and DNS

`IHttpClientFactory` sets [`SocketsHttpHandler.PooledConnectionLifetime`](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.socketshttphandler.pooledconnectionlifetime) to **2 minutes**. Connections older than that are dropped so DNS changes apply.

A process-wide `static readonly HttpClient` does **not** recycle connections. That is the stale-DNS bug.

If you replace the primary handler, set the lifetime yourself—the factory default is no longer applied:

```C#
builder.Services.AddHttpClient("github")
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(2)
    });
```

Long-lived client **without** the factory:

```C#
static readonly HttpClient Client = new(new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2)
}, disposeHandler: true);
```

## Platform handlers

The innermost handler makes the request:

| Handler | Where |
|---------|--------|
| `SocketsHttpHandler` | .NET Core 2.1+ / .NET 5+ (default on the server) |
| `HttpClientHandler` | older stacks; wraps platform code |
| `WinHttpHandler` | Windows-specific, both Framework and modern .NET |

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
