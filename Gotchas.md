# Table of contents

- [.NET API gotchas](#net-api-gotchas)
  - [Prefer `Random.Shared` over `new Random()`](#prefer-randomshared-over-new-random)
  - [Prefer `[GeneratedRegex]` over repeatedly constructing `Regex`](#prefer-generatedregex-over-repeatedly-constructing-regex)
  - [Prefer `StringBuilder` when concatenating in a loop](#prefer-stringbuilder-when-concatenating-in-a-loop)
  - [Prefer `StringComparison.Ordinal` for non-linguistic comparisons](#prefer-stringcomparisonordinal-for-non-linguistic-comparisons)
  - [Prefer `Contains(..., StringComparison.Ordinal)` over culture-sensitive compares](#prefer-contains-stringcomparisonordinal-over-culture-sensitive-compares)
  - [Prefer `ArgumentNullException.ThrowIfNull`](#prefer-argumentnullexceptionthrowifnull)
  - [Prefer pattern matching over verbose type/null checks](#prefer-pattern-matching-over-verbose-typenull-checks)
  - [Prefer `using` declarations over nested `using` blocks](#prefer-using-declarations-over-nested-using-blocks)
- [Related guides](#related-guides)

# .NET API gotchas

Small BCL choices that show up as allocations, culture bugs, or noisy code. HTTP and DI pitfalls live in [AspNetCoreGuidance.md](AspNetCoreGuidance.md) and [DotnetPattern.md](DotnetPattern.md).

## Prefer `Random.Shared` over `new Random()`

`new Random()` per call uses a time-based seed on older runtimes (colliding values under concurrency) and allocates. [`Random.Shared`](https://learn.microsoft.com/en-us/dotnet/api/system.random.shared) is a process-wide, thread-safe instance (.NET 6+).

❌ **BAD**

```C#
var n = new Random().Next(0, 100);
```

:white_check_mark: **GOOD**

```C#
var n = Random.Shared.Next(0, 100);
```

Need isolated, seedable sequences in tests? Inject a `Random` / wrap `RandomNumberGenerator`—do not use `Random.Shared` for cryptographic tokens (`RandomNumberGenerator.GetBytes`).

## Prefer `[GeneratedRegex]` over repeatedly constructing `Regex`

`new Regex(pattern)` parses and compiles on every call. A `static readonly Regex` is better; `[GeneratedRegex]` (.NET 7+) compiles at build time (AOT-friendly, no parse on first use).

❌ **BAD**

```C#
if (new Regex(@"^\d+$").IsMatch(s)) { /* ... */ }
```

:white_check_mark: **GOOD**

```C#
public static partial class Patterns
{
    [GeneratedRegex(@"^\d+$", RegexOptions.CultureInvariant | RegexOptions.NonBacktracking)]
    public static partial Regex Digits();
}

if (Patterns.Digits().IsMatch(s)) { /* ... */ }
```

`RegexOptions.NonBacktracking` (.NET 7+) avoids catastrophic backtracking on untrusted input. Prefer `IsMatch(ReadOnlySpan<char>)` when you already have a span.

## Prefer `StringBuilder` when concatenating in a loop

`s += chunk` in a loop allocates a new string each iteration (`O(n²)`). `string.Join` / interpolation is fine for a **fixed** number of pieces.

❌ **BAD**

```C#
var s = "";
foreach (var line in lines)
    s += line + "\n";
```

:white_check_mark: **GOOD**

```C#
var sb = new StringBuilder();
foreach (var line in lines)
    sb.AppendLine(line);
return sb.ToString();
```

Hot path / many small builders: `ObjectPool<StringBuilder>` or `ArrayPool<char>`. See [object pooling](DotnetPattern.md#object-pooling).

## Prefer `StringComparison.Ordinal` for non-linguistic comparisons

Default `==` / `StartsWith(string)` on some overloads use **culture**. Path, header, media-type, and id compares must not depend on the current culture (`i` vs `İ` in Turkish).

❌ **BAD**

```C#
if (contentType.StartsWith("application/json")) { /* culture-sensitive */ }
if (a.ToLower() == b.ToLower()) { /* extra allocations */ }
```

:white_check_mark: **GOOD**

```C#
if (contentType.StartsWith("application/json", StringComparison.OrdinalIgnoreCase)) { /* ... */ }
if (string.Equals(a, b, StringComparison.OrdinalIgnoreCase)) { /* ... */ }
```

Use `CurrentCulture` / `CurrentCultureIgnoreCase` only for text you **display** or sort for humans. Invariant is for data formats (ISO dates), not for ASCII protocol tokens—`Ordinal` is cheaper and stable.

## Prefer `Contains(..., StringComparison.Ordinal)` over culture-sensitive compares

`string.Contains(string)` used culture on .NET Framework; the `(string, StringComparison)` overload is explicit everywhere.

❌ **BAD**

```C#
if (path.Contains("..")) { /* intent unclear; culture on older runtimes */ }
```

:white_check_mark: **GOOD**

```C#
if (path.Contains("..", StringComparison.Ordinal)) { /* ... */ }
if (path.AsSpan().Contains("/../", StringComparison.Ordinal)) { /* ... */ }
```

Same for `IndexOf`, `Replace`, `Split` overloads that take `StringComparison`. CA1307 / CA1310 flag the culture-default overloads.

## Prefer `ArgumentNullException.ThrowIfNull`

Manual `if (x is null) throw new ArgumentNullException(nameof(x))` is easy to get the name wrong and is not trimmed as well.

❌ **BAD**

```C#
public OrderService(IOrderStore store)
{
    if (store == null) throw new ArgumentNullException("store");
    _store = store;
}
```

:white_check_mark: **GOOD**

```C#
public OrderService(IOrderStore store)
{
    ArgumentNullException.ThrowIfNull(store);
    _store = store;
}
```

Also: `ArgumentOutOfRangeException.ThrowIfNegative`, `ThrowIfZero`, `ObjectDisposedException.ThrowIf`. Primary constructors + `required` / nullable analysis often make the check unnecessary on DI-injected services (the container does not pass null).

## Prefer pattern matching over verbose type/null checks

❌ **BAD**

```C#
if (ex != null && ex is InvalidOperationException)
{
    var ioe = (InvalidOperationException)ex;
    return ioe.Message;
}
if (obj != null && obj.GetType() == typeof(string))
    return (string)obj;
```

:white_check_mark: **GOOD**

```C#
if (ex is InvalidOperationException ioe)
    return ioe.Message;

return obj switch
{
    string s => s,
    null => "",
    _ => obj.ToString() ?? ""
};
```

`is not null`, `is { Length: > 0 }`, and `property is { } value` replace most `!= null` plus cast pairs. Empty `catch (Exception)` plus `is` is still a smell—filter with `catch (InvalidOperationException ioe)`.

## Prefer `using` declarations over nested `using` blocks

❌ **BAD**

```C#
using (var file = File.Create(path))
{
    using (var buffer = new MemoryStream())
    {
        await stream.CopyToAsync(buffer, cancellationToken);
        buffer.Position = 0;
        await buffer.CopyToAsync(file, cancellationToken);
    }
}
```

:white_check_mark: **GOOD** Dispose at the end of the block (method, loop iteration, or explicit `{ }` scope).

```C#
await using var file = File.Create(path);
await stream.CopyToAsync(file, cancellationToken);
```

Keep a nested `using` only when the inner object must dispose **before** later code in the same method (e.g. flush a writer before reading the file back). Prefer `await using` for `IAsyncDisposable` (`HttpResponseMessage`, EF `DbContext`, scopes).

# Related guides

- [AspNetCoreGuidance.md](AspNetCoreGuidance.md) — `HttpContext`, bodies, DI capture, `IHttpClientFactory`
- [AsyncGuidance.md](AsyncGuidance.md) — `async`/`await`, `ConfigureAwait`, Runtime Async
- [DotnetPattern.md](DotnetPattern.md) — DI, options, factories, pooling
- [HttpClientGuidance.md](HttpClientGuidance.md) — `HttpClient` lifetime and handlers
