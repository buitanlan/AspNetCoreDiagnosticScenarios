# Common pitfalls writing scalable services in ASP.NET Core

Guides for writing scalable ASP.NET Core services. Some of the guidance is general-purpose; the examples use web services because that is where these mistakes hurt most.

| Guide | What it covers |
|-------|----------------|
| [AspNetCoreGuidance.md](AspNetCoreGuidance.md) | HTTP pipeline: `HttpContext`, bodies, headers, request DI |
| [AsyncGuidance.md](AsyncGuidance.md) | `async`/`await`, starvation, `ConfigureAwait`, Runtime Async |
| [DotnetPattern.md](DotnetPattern.md) | DI, options, factories, modules, tenancy, transactions |
| [HttpClientGuidance.md](HttpClientGuidance.md) | `HttpClient` lifetime, handlers, platform implementations |
| [Gotchas.md](Gotchas.md) | BCL: `Random.Shared`, `GeneratedRegex`, `StringComparison`, `ThrowIfNull` |

The examples are based on customer applications and issues found on GitHub and Stack Overflow.
