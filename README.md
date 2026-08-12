# ASP.NET Core Diagnostic Scenarios
 
The goal of this repository is to show problematic application patterns for ASP.NET Core applications and a walk-through on how to solve those issues.
It shall serve as a collection of knowledge from real-life application issues our customers have encountered.

## Common pitfalls writing scalable services in ASP.NET Core

Guides for writing scalable ASP.NET Core services. Some of the guidance is general-purpose; the examples use web services because that is where these mistakes hurt most.

- [ASP.NET Core pipeline](AspNetCoreGuidance.md) — `HttpContext`, bodies, headers, request DI
- [Asynchronous programming](AsyncGuidance.md) — `async`/`await`, starvation, `ConfigureAwait`, Runtime Async
- [.NET patterns](DotnetPattern.md) — DI, options, factories, modules, tenancy, transactions
- [HttpClient](HttpClientGuidance.md) — lifetime, handlers, platform implementations
- [.NET API gotchas](Gotchas.md) — `Random.Shared`, `GeneratedRegex`, `StringComparison`, `ThrowIfNull`

*NOTE:* The examples shown here are based on experiences with customer applications and issues found on GitHub and Stack Overflow.

### All Thanks to Our Contributors:
<a href="https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=davidfowl/AspNetCoreDiagnosticScenarios" />
</a>
