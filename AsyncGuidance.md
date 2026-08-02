# Table of contents
 - [Asynchronous Programming](#asynchronous-programming)
   - [Mental model](#mental-model)
   - [Asynchrony is viral](#asynchrony-is-viral)
   - [Async void](#async-void)
   - [Prefer `Task.FromResult` / `ValueTask` over `Task.Run` for pre-computed data](#prefer-taskfromresult--valuetask-over-taskrun-for-pre-computed-data)
   - [Avoid using `Task.Run` for long-running work that blocks the thread](#avoid-using-taskrun-for-long-running-work-that-blocks-the-thread)
   - [Prefer `Channel<T>` and hosted services for background work](#prefer-channelt-and-hosted-services-for-background-work)
   - [Avoid using `Task.Result` and `Task.Wait`](#avoid-using-taskresult-and-taskwait)
   - [Prefer `await` over `ContinueWith`](#prefer-await-over-continuewith)
   - [Always create `TaskCompletionSource<T>` with `TaskCreationOptions.RunContinuationsAsynchronously`](#always-create-taskcompletionsourcet-with-taskcreationoptionsruncontinuationsasynchronously)
   - [Always dispose `CancellationTokenSource`(s) used for timeouts](#always-dispose-cancellationtokensources-used-for-timeouts)
   - [Always flow `CancellationToken`(s) to APIs that take a `CancellationToken`](#always-flow-cancellationtokens-to-apis-that-take-a-cancellationtoken)
   - [Cancelling uncancellable operations](#cancelling-uncancellable-operations)
   - [Always call `FlushAsync` / prefer `await using` before disposing writers](#always-call-flushasync--prefer-await-using-before-disposing-writers)
   - [Prefer `async`/`await` over directly returning `Task`](#prefer-asyncawait-over-directly-returning-task)
   - [`ValueTask` / `ValueTask<T>` guidelines](#valuetask--valuetaskt-guidelines)
   - [Coordinating multiple tasks](#coordinating-multiple-tasks)
   - [`IAsyncEnumerable<T>` and `await foreach`](#iasyncenumerablet-and-await-foreach)
   - [`Parallel.ForEachAsync`](#parallelforeachasync)
   - [Async synchronization with `SemaphoreSlim`](#async-synchronization-with-semaphoreslim)
   - [`AsyncLocal<T>`](#asynclocalt)
   - [Async runtime](#async-runtime)
   - [`ConfigureAwait`](#configureawait)
 - [Scenarios](#scenarios)
   - [Timer callbacks](#timer-callbacks)
   - [Implicit `async void` delegates](#implicit-async-void-delegates)
   - [`ConcurrentDictionary.GetOrAdd`](#concurrentdictionarygetoradd)
   - [Constructors](#constructors)
   - [`WindowsIdentity.RunImpersonated`](#windowsidentityrunimpersonated)



# Asynchronous Programming

Asynchronous programming is the default model for scalable .NET server apps. Since C# 5 introduced `async`/`await`, and with ASP.NET Core being fully asynchronous end-to-end, most web and cloud code is async by necessity—not by choice.

That ubiquity created a second problem: confusion about *how* to use async correctly. Blocking, fire-and-forget, mishandled cancellation, and poorly chosen concurrency primitives still cause production outages years after `async`/`await` shipped.

This guide focuses on practical patterns for modern .NET (**6+**, with callouts for **.NET 8/9** APIs). Examples use bad/good pairs so you can map guidance onto real codebases. Much of this is general-purpose; ASP.NET Core is the primary lens because that is where these mistakes hurt the most.

## Mental model

A few facts that make the rest of this document click (see [Async runtime](#async-runtime) for the deeper mechanics):

1. **`async` does not mean "run on another thread".** An `async` method runs synchronously until it hits an incomplete `await`. Only then does it yield.
2. **Async is about freeing threads during waits** (I/O, timers, other tasks)—not about making CPU work faster. For CPU-bound work, use `Task.Run` / `Parallel` / channels deliberately.
3. **Scalability comes from not blocking thread-pool threads** while waiting on I/O. Sync-over-async often uses *more* threads than a sync API would.
4. **ASP.NET Core has no `SynchronizationContext`.** Classic ASP.NET / UI deadlock folklore still matters for libraries and desktop apps, but starvation (not deadlock) is the usual ASP.NET Core failure mode.
5. **Cancellation is cooperative.** Tokens only work if every layer observes them.
6. **`ExecutionContext` (including `AsyncLocal`) flows across awaits; `SynchronizationContext` does not**—it is optionally *posted to* after an await, which is what `ConfigureAwait` controls.



## Asynchrony is viral

Once you go async, callers **SHOULD** be async. Partial asynchrony is often worse than staying fully synchronous: you pay the state-machine and allocation costs, then block a thread waiting anyway.

❌ **BAD** Uses `Task.Result` and blocks the current thread. This is [sync over async](#avoid-using-taskresult-and-taskwait).

```C#
public int DoSomething()
{
    var result = CallDependencyAsync().Result;
    return result + 1;
}
```

:white_check_mark: **GOOD** Uses `await` so the thread can be reused while the dependency runs.

```C#
public async Task<int> DoSomethingAsync()
{
    var result = await CallDependencyAsync();
    return result + 1;
}
```

:bulb: **NOTE:** If you truly cannot change a synchronous public surface, keep the *implementation* synchronous end-to-end, or introduce a separate async API (`*Async`) and migrate callers. Do not paper over the gap with `.Result`.

## Async void

`async void` in ASP.NET Core applications is **ALWAYS** bad. Avoid it. It is usually a mistaken fire-and-forget attempt from a controller or middleware. `async void` methods cannot be awaited, and an unhandled exception can tear down the process.

❌ **BAD** `async void` cannot be observed; unhandled exceptions can crash the app.

```C#
public class MyController : Controller
{
    [HttpPost("/start")]
    public IActionResult Post()
    {
        BackgroundOperationAsync();
        return Accepted();
    }

    public async void BackgroundOperationAsync()
    {
        var result = await CallDependencyAsync();
        DoSomething(result);
    }
}
```

❌ **BAD** `Task.Run` from a request without lifetime management loses DI scopes, request abort, and structured logging context. Exceptions become unobserved.

```C#
public class MyController : Controller
{
    [HttpPost("/start")]
    public IActionResult Post()
    {
        _ = Task.Run(BackgroundOperationAsync);
        return Accepted();
    }

    public async Task BackgroundOperationAsync()
    {
        var result = await CallDependencyAsync();
        DoSomething(result);
    }
}
```

:white_check_mark: **GOOD** Queue work to a hosted service (or another out-of-process worker) that owns lifetime, cancellation, and error handling.

```C#
public class MyController : Controller
{
    private readonly IBackgroundTaskQueue _queue;

    public MyController(IBackgroundTaskQueue queue) => _queue = queue;

    [HttpPost("/start")]
    public async Task<IActionResult> Post(CancellationToken cancellationToken)
    {
        await _queue.QueueAsync(BackgroundOperationAsync, cancellationToken);
        return Accepted();
    }

    private static async Task BackgroundOperationAsync(CancellationToken cancellationToken)
    {
        var result = await CallDependencyAsync(cancellationToken);
        DoSomething(result);
    }
}
```

See [Prefer](#prefer-channelt-and-hosted-services-for-background-work) `Channel<T>` [and hosted services for background work](#prefer-channelt-and-hosted-services-for-background-work) for a full sketch.

:bulb: **NOTE:** The only legitimate `async void` usages in modern .NET are UI event handlers (WPF/WinForms/MAUI). Server apps do not have that excuse.

## Prefer `Task.FromResult` / `ValueTask` over `Task.Run` for pre-computed data

For already-computed or trivially computed results, do not call `Task.Run`. That queues a thread-pool work item that immediately completes. Use `Task.FromResult`, `Task.CompletedTask`, or `ValueTask`/`ValueTask<T>` instead.

❌ **BAD** Wastes a thread-pool thread to return a trivial value.

```C#
public class MyLibrary
{
    public Task<int> AddAsync(int a, int b)
    {
        return Task.Run(() => a + b);
    }
}
```

:white_check_mark: **GOOD** `Task.FromResult` wraps the value with no extra thread.

```C#
public class MyLibrary
{
    public Task<int> AddAsync(int a, int b)
    {
        return Task.FromResult(a + b);
    }
}
```

:white_check_mark: **GOOD** `ValueTask<int>` can avoid the `Task` allocation on the hot path.

```C#
public class MyLibrary
{
    public ValueTask<int> AddAsync(int a, int b)
    {
        return new ValueTask<int>(a + b);
    }
}
```

:bulb: **NOTE:** Prefer `Task.CompletedTask` for successful void-like completions, and `Task.FromException` / `Task.FromCanceled` when you need to return a faulted or canceled task synchronously.

:bulb: **NOTE:** `Task.Run` *is* appropriate for offloading **CPU-bound** work off the request thread when you intentionally want parallelism. Do not use it as a blanket "make it async" wrapper around sync I/O.

## Avoid using `Task.Run` for long-running work that blocks the thread

Long-running here means work that occupies a thread for the lifetime of the process (queue pumping, blocking `Receive`, sleep loops). `Task.Run` assumes the work finishes quickly enough that the thread can be reused. Stealing a pool thread forever hurts timer callbacks, continuations, and request handling.

:bulb: **NOTE:** The thread pool grows when threads block, but relying on that growth is a scalability bug waiting to happen.

:bulb: **NOTE:** `Task.Factory.StartNew(..., TaskCreationOptions.LongRunning)` can create a dedicated thread, but it is a *hint* to the scheduler—not a guarantee—and several parameters are easy to get wrong across platforms.

:bulb: **NOTE:** Never combine `LongRunning` with `async` lambdas. The dedicated thread is released at the first `await`, leaving you with ordinary thread-pool continuations and a wasted thread.

❌ **BAD** Steals a thread-pool thread forever to pump a `BlockingCollection<T>`.

```C#
public class QueueProcessor
{
    private readonly BlockingCollection<Message> _messageQueue = new();

    public void StartProcessing()
    {
        Task.Run(ProcessQueue);
    }

    public void Enqueue(Message message) => _messageQueue.Add(message);

    private void ProcessQueue()
    {
        foreach (var item in _messageQueue.GetConsumingEnumerable())
        {
            ProcessItem(item);
        }
    }

    private void ProcessItem(Message message) { }
}
```

:white_check_mark: **GOOD** Use a dedicated background thread for *blocking* legacy pumps.

```C#
public class QueueProcessor
{
    private readonly BlockingCollection<Message> _messageQueue = new();

    public void StartProcessing()
    {
        var thread = new Thread(ProcessQueue)
        {
            // Allows the process to exit while this thread is running
            IsBackground = true,
            Name = "QueueProcessor"
        };
        thread.Start();
    }

    public void Enqueue(Message message) => _messageQueue.Add(message);

    private void ProcessQueue()
    {
        foreach (var item in _messageQueue.GetConsumingEnumerable())
        {
            ProcessItem(item);
        }
    }

    private void ProcessItem(Message message) { }
}
```

:white_check_mark: **GOOD** `TaskCreationOptions.LongRunning` when you want a `Task` surface over a dedicated thread for **synchronous** work.

```C#
public class QueueProcessor
{
    private readonly BlockingCollection<Message> _messageQueue = new();

    public Task StartProcessing() =>
        Task.Factory.StartNew(
            ProcessQueue,
            CancellationToken.None,
            TaskCreationOptions.LongRunning,
            TaskScheduler.Default);

    public void Enqueue(Message message) => _messageQueue.Add(message);

    private void ProcessQueue()
    {
        foreach (var item in _messageQueue.GetConsumingEnumerable())
        {
            ProcessItem(item);
        }
    }

    private void ProcessItem(Message message) { }
}
```

Advantages vs raw `Thread`:

- Combines with `await`, `Task.WhenAll`, etc.
- Unhandled exceptions become faulted `Task`s (`AggregateException`) instead of crashing via an unhandled thread exception.

:bulb: **NOTE:** Prefer the next section (`Channel<T>` + hosted service) for *new* async-native pipelines. Dedicated threads are for blocking/legacy interop.

## Prefer `Channel<T>` and hosted services for background work

Modern ASP.NET Core apps should push background work into `[BackgroundService](https://learn.microsoft.com/en-us/dotnet/core/extensions/timer-service)` / `IHostedService` and communicate with `[System.Threading.Channels](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels)`. Channels are async-friendly, bounded-capacity capable, and do not block pool threads while waiting for items.

:white_check_mark: **GOOD** Bounded channel + hosted consumer.

```C#
public interface IBackgroundTaskQueue
{
    ValueTask QueueAsync(Func<CancellationToken, Task> workItem, CancellationToken cancellationToken = default);
    ValueTask<Func<CancellationToken, Task>> DequeueAsync(CancellationToken cancellationToken);
}

public sealed class BackgroundTaskQueue : IBackgroundTaskQueue
{
    private readonly Channel<Func<CancellationToken, Task>> _queue =
        Channel.CreateBounded<Func<CancellationToken, Task>>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait
        });

    public async ValueTask QueueAsync(
        Func<CancellationToken, Task> workItem,
        CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(workItem);
        await _queue.Writer.WriteAsync(workItem, cancellationToken);
    }

    public async ValueTask<Func<CancellationToken, Task>> DequeueAsync(CancellationToken cancellationToken)
    {
        return await _queue.Reader.ReadAsync(cancellationToken);
    }
}

public sealed class QueuedHostedService : BackgroundService
{
    private readonly IBackgroundTaskQueue _queue;
    private readonly ILogger<QueuedHostedService> _logger;

    public QueuedHostedService(IBackgroundTaskQueue queue, ILogger<QueuedHostedService> logger)
    {
        _queue = queue;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var workItem = await _queue.DequeueAsync(stoppingToken);
                await workItem(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                // Shutting down
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Background work item failed");
            }
        }
    }
}
```

Register both in DI (`AddSingleton<IBackgroundTaskQueue, BackgroundTaskQueue>()` + `AddHostedService<QueuedHostedService>()`). Controllers enqueue; the host owns execution and shutdown.

:bulb: **NOTE:** For multi-instance deployments, in-process queues are not durable. Use a real message bus / queue when work must survive process restarts.

## Avoid using `Task.Result` and `Task.Wait`

There are very few correct uses of `Task.Result` / `Task.Wait` / `task.GetAwaiter().GetResult()`. Prefer to treat them as banned in application code.

### :warning: Sync over `async`

Blocking on an asynchronous operation with `.Result` / `.Wait` is *worse* than calling a truly synchronous API. High-level sequence:

1. An asynchronous operation starts.
2. The calling thread blocks waiting for completion.
3. Completion resumes elsewhere (often another pool thread).

You spent **two threads** to do one unit of work. Under load this causes [thread-pool starvation](https://learn.microsoft.com/en-us/archive/blogs/vancem/diagnosing-net-core-threadpool-starvation-with-perfview-why-my-service-is-not-saturating-all-cores-or-seems-to-stall) and service outages.

### :warning: Deadlocks

`SynchronizationContext` lets application models control where continuations run. Classic ASP.NET, WPF, and Windows Forms can deadlock when you block the main/request thread that the continuation needs. Clever "thread-safe" blocking snippets do not fix the underlying problem—they only dodge one failure mode.

:bulb: **NOTE:** ASP.NET Core has no `SynchronizationContext`, so the classic deadlock is uncommon. **Starvation** from sync-over-async is still very common.

❌ **BAD** All of the following still block a thread (sync over async), even when they avoid deadlocks.

```C#
public string DoOperationBlocking()
{
    // Blocks the entering thread.
    // Exceptions are wrapped in AggregateException.
    return Task.Run(() => DoAsyncOperation()).Result;
}

public string DoOperationBlocking2()
{
    // Blocks the entering thread.
    // GetAwaiter().GetResult() unwraps AggregateException.
    return Task.Run(() => DoAsyncOperation()).GetAwaiter().GetResult();
}

public string DoOperationBlocking3()
{
    // Blocks the entering thread *and* a pool thread inside.
    return Task.Run(() => DoAsyncOperation().Result).Result;
}

public string DoOperationBlocking4()
{
    return Task.Run(() => DoAsyncOperation().GetAwaiter().GetResult()).GetAwaiter().GetResult();
}

public string DoOperationBlocking5()
{
    // Blocks; also deadlocks if a capturing SynchronizationContext is present.
    return DoAsyncOperation().Result;
}

public string DoOperationBlocking6()
{
    return DoAsyncOperation().GetAwaiter().GetResult();
}

public string DoOperationBlocking7()
{
    var task = DoAsyncOperation();
    task.Wait();
    return task.GetAwaiter().GetResult();
}
```

:white_check_mark: **GOOD** Stay asynchronous.

```C#
public async Task<string> DoOperationAsync()
{
    return await DoAsyncOperation();
}
```



## Prefer `await` over `ContinueWith`

`Task` predated `async`/`await` and exposed continuation APIs. Prefer `async`/`await`. `ContinueWith` does not capture `SynchronizationContext` by default and is easy to misuse (`TaskScheduler`, exception handling, nested `Task<Task>`).

❌ **BAD** Uses `ContinueWith` and `.Result` inside the continuation.

```C#
public Task<int> DoSomethingAsync()
{
    return CallDependencyAsync().ContinueWith(task =>
    {
        return task.Result + 1;
    });
}
```

:white_check_mark: **GOOD** Uses `await`.

```C#
public async Task<int> DoSomethingAsync()
{
    var result = await CallDependencyAsync();
    return result + 1;
}
```



## Always create `TaskCompletionSource<T>` with `TaskCreationOptions.RunContinuationsAsynchronously`

`TaskCompletionSource<T>` adapts non-Task APIs into awaitables and builds higher-level combinators. By default, continuations run *inline* on the thread that calls `TrySet`* / `Set*`. Library code then unexpectedly runs arbitrary user code on its thread—deadlocks, reentrancy, corrupted state, starvation.

Always pass `TaskCreationOptions.RunContinuationsAsynchronously` so continuations are scheduled on the thread pool.

❌ **BAD** Missing `RunContinuationsAsynchronously`.

```C#
public Task<int> DoSomethingAsync()
{
    var tcs = new TaskCompletionSource<int>();

    var operation = new LegacyAsyncOperation();
    operation.Completed += result =>
    {
        // Awaiters may resume on this thread!
        tcs.SetResult(result);
    };

    return tcs.Task;
}
```

:white_check_mark: **GOOD** Continuations are forced asynchronous.

```C#
public Task<int> DoSomethingAsync()
{
    var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);

    var operation = new LegacyAsyncOperation();
    operation.Completed += result =>
    {
        // Awaiters resume on a thread-pool thread
        tcs.SetResult(result);
    };

    return tcs.Task;
}
```

:bulb: **NOTE:** Do not confuse `[TaskCreationOptions.RunContinuationsAsynchronously](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcreationoptions)` with `[TaskContinuationOptions.RunContinuationsAsynchronously](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcontinuationoptions)`.

:bulb: **NOTE:** Prefer `TrySetResult` / `TrySetException` / `TrySetCanceled` unless you intentionally want to throw if the task is already completed.

## Always dispose `CancellationTokenSource`(s) used for timeouts

`CancellationTokenSource` instances created with a timeout (constructor overload or `CancelAfter`) register timers. Failing to dispose them leaves pressure on the timer queue.

❌ **BAD** Timer remains queued for up to 10 seconds after each call.

```C#
public async Task<Stream> HttpClientAsyncWithCancellationBad()
{
    var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

    using var client = _httpClientFactory.CreateClient();
    var response = await client.GetAsync("http://backend/api/1", cts.Token);
    return await response.Content.ReadAsStreamAsync(cts.Token);
}
```

:white_check_mark: **GOOD** Dispose the `CancellationTokenSource` (and prefer linking with the request token).

```C#
public async Task<Stream> HttpClientAsyncWithCancellationGood(CancellationToken cancellationToken)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    cts.CancelAfter(TimeSpan.FromSeconds(10));

    using var client = _httpClientFactory.CreateClient();
    var response = await client.GetAsync("http://backend/api/1", cts.Token);
    return await response.Content.ReadAsStreamAsync(cts.Token);
}
```

:bulb: **NOTE (.NET 8+):** Prefer `await cts.CancelAsync()` over `cts.Cancel()` when cancellation callbacks may be asynchronous or you want to avoid blocking the caller on synchronous callback execution.

## Always flow `CancellationToken`(s) to APIs that take a `CancellationToken`

Cancellation in .NET is cooperative. Every layer must observe the token. Dropping it at any hop makes the operation effectively uncancellable from above.

❌ **BAD** Forgets to pass the token into `ReadAsync`.

```C#
public async Task<string> DoAsyncThing(CancellationToken cancellationToken = default)
{
    byte[] buffer = new byte[1024];
    // cancellationToken is ignored
    int read = await _stream.ReadAsync(buffer.AsMemory(0, buffer.Length));
    return Encoding.UTF8.GetString(buffer, 0, read);
}
```

:white_check_mark: **GOOD** Flows the token and uses the modern `Memory<byte>` overload.

```C#
public async Task<string> DoAsyncThing(CancellationToken cancellationToken = default)
{
    byte[] buffer = new byte[1024];
    int read = await _stream.ReadAsync(buffer.AsMemory(0, buffer.Length), cancellationToken);
    return Encoding.UTF8.GetString(buffer, 0, read);
}
```

:bulb: **NOTE:** In ASP.NET Core, prefer accepting `CancellationToken` on controller actions / minimal APIs so request abort flows naturally. Do not capture `HttpContext.RequestAborted` into long-running work that outlives the request unless that is intentional.

## Cancelling uncancellable operations

Sometimes you need to abandon wait on an API that does not accept a `CancellationToken`. The modern answer is almost always `[Task.WaitAsync](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.waitasync)` (.NET 6+). Older `WhenAny` + `Delay` patterns are easy to leak.

### Prefer `Task.WaitAsync` (.NET 6+)

:white_check_mark: **GOOD** Timeout and/or cancellation without building combinators by hand.

```C#
public async Task<T> WithCancellation<T>(Task<T> task, CancellationToken cancellationToken)
{
    return await task.WaitAsync(cancellationToken);
}

public async Task<T> TimeoutAfter<T>(Task<T> task, TimeSpan timeout)
{
    return await task.WaitAsync(timeout);
}

public async Task<T> WaitAsync<T>(Task<T> task, TimeSpan timeout, CancellationToken cancellationToken)
{
    return await task.WaitAsync(timeout, cancellationToken);
}
```

:bulb: **NOTE:** `WaitAsync` cancels *waiting*, not the underlying operation. The original task may still run to completion in the background. If the API supports a token, pass it so work actually stops.

### Using `CancellationToken` (legacy pattern)

❌ **BAD** `Task.Delay(-1, token)` can leak the internal registration if the primary task wins forever in some designs; prefer `WaitAsync` or dispose registrations explicitly.

```C#
public static async Task<T> WithCancellation<T>(this Task<T> task, CancellationToken cancellationToken)
{
    var delayTask = Task.Delay(Timeout.Infinite, cancellationToken);

    var resultTask = await Task.WhenAny(task, delayTask);
    if (resultTask == delayTask)
    {
        throw new OperationCanceledException(cancellationToken);
    }

    return await task;
}
```

:white_check_mark: **GOOD** Dispose the `CancellationTokenRegistration` when either side completes (pre-.NET 6).

```C#
public static async Task<T> WithCancellation<T>(this Task<T> task, CancellationToken cancellationToken)
{
    var tcs = new TaskCompletionSource(TaskCreationOptions.RunContinuationsAsynchronously);

    using (cancellationToken.Register(
        static state => ((TaskCompletionSource)state!).TrySetResult(),
        tcs))
    {
        var resultTask = await Task.WhenAny(task, tcs.Task);
        if (resultTask == tcs.Task)
        {
            throw new OperationCanceledException(cancellationToken);
        }

        return await task;
    }
}
```



### Using a timeout (legacy pattern)

❌ **BAD** Leaves the delay timer alive after success; timer queue pressure under load.

```C#
public static async Task<T> TimeoutAfter<T>(this Task<T> task, TimeSpan timeout)
{
    var delayTask = Task.Delay(timeout);

    var resultTask = await Task.WhenAny(task, delayTask);
    if (resultTask == delayTask)
    {
        throw new TimeoutException();
    }

    return await task;
}
```

:white_check_mark: **GOOD** Cancel the delay when the operation wins (pre-.NET 6).

```C#
public static async Task<T> TimeoutAfter<T>(this Task<T> task, TimeSpan timeout)
{
    using var cts = new CancellationTokenSource();
    var delayTask = Task.Delay(timeout, cts.Token);

    var resultTask = await Task.WhenAny(task, delayTask);
    if (resultTask == delayTask)
    {
        throw new TimeoutException();
    }

    await cts.CancelAsync(); // .NET 8+; use cts.Cancel() on earlier TFMs
    return await task;
}
```



## Always call `FlushAsync` / prefer `await using` before disposing writers

Even when writes use async overloads, `Stream` / `StreamWriter` may buffer. Synchronous `Dispose` can flush on the calling thread and starve the pool. Prefer `IAsyncDisposable` via `await using`, or call `FlushAsync` before dispose.

:bulb: **NOTE:** This matters when the underlying sink performs real I/O (HTTP response bodies, files, network streams).

❌ **BAD** Implicit synchronous flush on dispose blocks the request.

```C#
app.Run(async context =>
{
    using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        await streamWriter.WriteAsync("Hello World");
    }
});
```

:white_check_mark: **GOOD** `await using` disposes asynchronously.

```C#
app.Run(async context =>
{
    await using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        await streamWriter.WriteAsync("Hello World");
    }
});
```

:white_check_mark: **GOOD** Explicit `FlushAsync` before dispose.

```C#
app.Run(async context =>
{
    using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        await streamWriter.WriteAsync("Hello World");
        await streamWriter.FlushAsync();
    }
});
```



## Prefer `async`/`await` over directly returning `Task`

Using `async`/`await` instead of returning an inner `Task` directly has behavioral benefits:

- Synchronous and asynchronous exceptions are normalized onto the returned `Task`.
- Easier to evolve (add `using` / logging / `finally`).
- Better diagnostics (async stacks, hang analysis).
- Async locals do not leak out of `async` methods the way they can from non-async methods.

❌ **BAD** Directly returns the inner task.

```C#
public Task<int> DoSomethingAsync()
{
    return CallDependencyAsync();
}
```

:white_check_mark: **GOOD** Uses `async`/`await`.

```C#
public async Task<int> DoSomethingAsync()
{
    return await CallDependencyAsync();
}
```

:bulb: **NOTE:** Directly returning the `Task` is slightly cheaper (no state machine). That micro-optimization is fine in mature, leaf library helpers where behavior differences are understood and covered by tests. Default to `async`/`await` in application code.

:bulb: **NOTE:** One important behavioral difference: if `CallDependencyAsync()` *throws synchronously* before returning a task, the eliding version throws to the caller synchronously; the `async` version places the exception on the returned `Task`.

## `ValueTask` / `ValueTask<T>` guidelines

`ValueTask`/`ValueTask<T>` reduce allocations when results are often already available (caches, completed I/O). They are not a drop-in "faster `Task`".

Rules of thumb:

1. **Prefer** `Task`**/**`Task<T>` **unless profiling shows allocation pressure** on a completed-hot path.
2. **Await a** `ValueTask` **once.** Do not await twice, store it for later concurrent awaiters, or block on it.
3. If you need to await more than once or hand it to multiple consumers, call `.AsTask()` or ensure you have a `Task`.
4. Do not use `.Result` / `.GetAwaiter().GetResult()` on `ValueTask` in application code.
5. Implementing APIs that *return* `ValueTask` is advanced—often you want `IValueTaskSource` pooling. Consuming them with `await` is easy and fine.

❌ **BAD** Awaits a `ValueTask` twice.

```C#
public async Task<int> BrokenAsync(IAsyncCache cache)
{
    ValueTask<int> vt = cache.GetAsync("id");
    var a = await vt;
    var b = await vt; // undefined / dangerous
    return a + b;
}
```

:white_check_mark: **GOOD** Convert to `Task` when fan-out or multiple awaits are required.

```C#
public async Task<int> OkAsync(IAsyncCache cache)
{
    Task<int> task = cache.GetAsync("id").AsTask();
    var a = await task;
    var b = await task;
    return a + b;
}
```



## Coordinating multiple tasks



### `Task.WhenAll`

Use when you need every operation's result (or exception). Prefer starting tasks, then awaiting `WhenAll`, rather than awaiting each call sequentially when they are independent.

❌ **BAD** Sequential awaits that could be concurrent.

```C#
public async Task<(User user, Order[] orders)> LoadAsync(int userId)
{
    var user = await _users.GetAsync(userId);
    var orders = await _orders.GetForUserAsync(userId);
    return (user, orders);
}
```

:white_check_mark: **GOOD** Overlap independent I/O.

```C#
public async Task<(User user, Order[] orders)> LoadAsync(int userId)
{
    var userTask = _users.GetAsync(userId);
    var ordersTask = _orders.GetForUserAsync(userId);
    await Task.WhenAll(userTask, ordersTask);
    return (await userTask, await ordersTask);
}
```

:bulb: **NOTE:** `WhenAll` faults if *any* task faults. After `WhenAll` throws, you can still inspect individual tasks for partial success/failure.

### `Task.WhenAny`

Use when the first completion wins (timeouts historically, speculative execution, first healthy replica). Prefer `WaitAsync` for simple timeout/cancel cases.

### `Task.WhenEach` (.NET 9+)

`[Task.WhenEach](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.wheneach)` yields tasks as they complete—cleaner than a manual `WhenAny` loop.

:white_check_mark: **GOOD** Process completions in finish order.

```C#
public async Task ProcessAsync(IEnumerable<Task<Data>> tasks, CancellationToken cancellationToken)
{
    await foreach (var task in Task.WhenEach(tasks).WithCancellation(cancellationToken))
    {
        var data = await task; // propagates fault/cancel for that task
        Handle(data);
    }
}
```



## `IAsyncEnumerable<T>` and `await foreach`

Use async streams when producing a sequence where each element may require I/O (paging APIs, change feeds, large result sets). Prefer streaming over buffering entire results into a `List<T>`.

:white_check_mark: **GOOD** Stream pages to the consumer.

```C#
public async IAsyncEnumerable<Item> GetItemsAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    string? cursor = null;
    do
    {
        var page = await _client.GetPageAsync(cursor, cancellationToken);
        foreach (var item in page.Items)
        {
            yield return item;
        }

        cursor = page.NextCursor;
    }
    while (cursor is not null);
}

// Consumer
await foreach (var item in GetItemsAsync(cancellationToken))
{
    await ProcessAsync(item, cancellationToken);
}
```

:bulb: **NOTE:** With ASP.NET Core, you can return `IAsyncEnumerable<T>` from endpoints; be mindful of JSON serialization buffering settings and client cancellation.

## `Parallel.ForEachAsync`

[.NET 6+](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel.foreachasync) is the modern way to process a collection with bounded async parallelism. Prefer it over `Task.WhenAll` on huge unbounded fan-out.

❌ **BAD** Unbounded concurrency can exhaust sockets, DB connections, and memory.

```C#
public async Task ProcessAllAsync(IEnumerable<int> ids)
{
    await Task.WhenAll(ids.Select(id => ProcessAsync(id)));
}
```

:white_check_mark: **GOOD** Bounded parallel async work.

```C#
public async Task ProcessAllAsync(IEnumerable<int> ids, CancellationToken cancellationToken)
{
    await Parallel.ForEachAsync(
        ids,
        new ParallelOptions
        {
            MaxDegreeOfParallelism = 8,
            CancellationToken = cancellationToken
        },
        async (id, ct) => await ProcessAsync(id, ct));
}
```



## Async synchronization with `SemaphoreSlim`

`lock` / `Monitor` / `Mutex` are not async-friendly—you cannot `await` inside a `lock`. Use `SemaphoreSlim` (or higher-level pipelines) to gate async work.

❌ **BAD** Holds a lock across an await (compile error / dangerous patterns with `Monitor`).

```C#
lock (_gate)
{
    await LoadAsync(); // does not compile — and must not be forced
}
```

:white_check_mark: **GOOD** Async-friendly mutual exclusion.

```C#
private readonly SemaphoreSlim _gate = new(1, 1);

public async Task CriticalSectionAsync(CancellationToken cancellationToken)
{
    await _gate.WaitAsync(cancellationToken);
    try
    {
        await LoadAsync(cancellationToken);
    }
    finally
    {
        _gate.Release();
    }
}
```

:bulb: **NOTE:** For throttling concurrent async operations (e.g., max 10 outbound calls), initialize `SemaphoreSlim` with that count rather than `1`.

## `AsyncLocal<T>`

Async locals store ambient state across asynchronous flows. They are powerful and easy to misuse. Values hitch a ride on the [execution context](https://learn.microsoft.com/en-us/dotnet/api/system.threading.executioncontext), which flows *everywhere* unless you use advanced `Unsafe`* APIs.

### Creating an `AsyncLocal<T>`

Prefer explicit parameters or DI. If you must use an async local, values should be:

1. Not disposable
2. Immutable / read-only / thread-safe



#### 1. ❌ **BAD** A disposable object stored in an async local

```C#
using (var thing = new DisposableThing())
{
    // Make the disposable object available ambiently
    DisposableThing.Current = thing;

    Dispatch();

    // We're about to dispose the object so make sure nobody else captures this instance
    DisposableThing.Current = null;
}

void Dispatch()
{
    // Task.Run will capture the current execution context (which means async locals are captured in the callback)
    _ = Task.Run(async () =>
    {
        // Delay for a second then log
        await Task.Delay(1000);

        Log();
    });
}

void Log()
{
    try
    {
        // Get the current value and make sure it's not null before reading the value
        var thing = DisposableThing.Current;
        if (thing is not null)
        {
            Console.WriteLine($"Logging ambient value {thing.Value}");
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex);
    }
}

Console.ReadLine();

class DisposableThing : IDisposable
{
    private static readonly AsyncLocal<DisposableThing?> _current = new();

    private bool _disposed;

    public static DisposableThing? Current
    {
        get => _current.Value;
        set => _current.Value = value;
    }

    public int Value
    {
        get
        {
            if (_disposed) throw new ObjectDisposedException(GetType().FullName);
            return 1;
        }
    }

    public void Dispose()
    {
        _disposed = true;
    }
}
```

This always ends in `ObjectDisposedException`. Setting `DisposableThing.Current = null` does not clear previously captured execution contexts—execution context is copy-on-write. Future reads see `null`; earlier captures still hold the disposed instance.

```C#
DisposableThing.Current = new DisposableThing();
Console.WriteLine("After setting thing " + ExecutionContext.Capture()!.GetHashCode());

DisposableThing.Current = null;
Console.WriteLine("After setting Current to null " + ExecutionContext.Capture()!.GetHashCode());
```

The hash code changes each time you set a new value—new contexts, not mutation of old ones.

⚠️ Mutating through `StrongBox<T>` can clear ambient references across copied contexts:

```C#
class DisposableThing : IDisposable
{
    private static readonly AsyncLocal<StrongBox<DisposableThing?>> _current = new();

    private bool _disposed;

    public static DisposableThing? Current
    {
        get => _current.Value?.Value;
        set
        {
            var box = _current.Value;
            if (box is not null)
            {
                // Mutate the value in any execution context that was copied
                box.Value = null;
            }

            if (value is not null)
            {
                _current.Value = new StrongBox<DisposableThing?>(value);
            }
        }
    }

    public int Value
    {
        get
        {
            if (_disposed) throw new ObjectDisposedException(GetType().FullName);
            return 1;
        }
    }

    public void Dispose()
    {
        _disposed = true;
    }
}
```

⚠️ Even then, code may have already copied the reference into a local before clear/dispose—races remain. **Do not store disposable objects in async locals.**

#### 2. ❌ **BAD** A non-thread-safe object stored in an async local

```C#
AmbientValues.Current = new Dictionary<int, string>();

Parallel.For(0, 10, i =>
{
    AmbientValues.Current[i] = "processing";
    LogCurrentValues();
    AmbientValues.Current[i] = "done";
});

void LogCurrentValues()
{
    foreach (var pair in AmbientValues.Current)
    {
        Console.WriteLine(pair);
    }
}

class AmbientValues
{
    private static readonly AsyncLocal<Dictionary<int, string>> _current = new();

    public static Dictionary<int, string> Current
    {
        get => _current.Value!;
        set => _current.Value = value;
    }
}
```

Arbitrary code on arbitrary threads may touch ambient state. Assume concurrency.

:white_check_mark: **GOOD** Store a thread-safe type.

```C#
class AmbientValues
{
    private static readonly AsyncLocal<ConcurrentDictionary<int, string>> _current = new();

    public static ConcurrentDictionary<int, string> Current
    {
        get => _current.Value!;
        set => _current.Value = value;
    }
}
```



### Don't leak your `AsyncLocal<T>`

Async locals flow across `await` and are captured by APIs that call `ExecutionContext.Capture`. Lifetime mismatches cause large memory leaks.

#### Common APIs that capture the execution context

- `Timer`
- `CancellationToken.Register`
- `FileSystemWatcher`
- `SocketAsyncEventArgs`
- `Task.Run`
- `ThreadPool.QueueUserWorkItem`

❌ **BAD** Singleton cache captures per-request async locals for an hour via `CancellationToken.Register`.

```C#
using System.Collections.Concurrent;

// Singleton cache
var cache = new NumberCache(TimeSpan.FromHours(1));

var executionContext = ExecutionContext.Capture();

// Simulate 10000 concurrent requests
Parallel.For(0, 10000, i =>
{
    // Restore the initial ExecutionContext per "request"
    ExecutionContext.Restore(executionContext!);

    ChunkyObject.Current = new ChunkyObject();

    cache.Add(i);
});

Console.WriteLine("Before GC: " + BytesAsString(GC.GetGCMemoryInfo().HeapSizeBytes));
Console.ReadLine();

GC.Collect();
GC.WaitForPendingFinalizers();

Console.WriteLine("After GC: " + BytesAsString(GC.GetGCMemoryInfo().HeapSizeBytes));
Console.ReadLine();

static string BytesAsString(long bytes)
{
    string[] suffix = { "B", "KB", "MB", "GB", "TB" };
    int i;
    double doubleBytes = 0;

    for (i = 0; bytes / 1024 > 0; i++, bytes /= 1024)
    {
        doubleBytes = bytes / 1024.0;
    }

    return string.Format("{0:0.00} {1}", doubleBytes, suffix[i]);
}

public class NumberCache
{
    private readonly ConcurrentDictionary<int, CancellationTokenSource> _cache = new();
    private readonly TimeSpan _timeSpan;

    public NumberCache(TimeSpan timeSpan) => _timeSpan = timeSpan;

    public void Add(int key)
    {
        var cts = _cache.GetOrAdd(key, _ => new CancellationTokenSource());
        // Delete entry on expiration
        cts.Token.Register((_, _) => _cache.TryRemove(key, out _), null);

        // Start count down
        cts.CancelAfter(_timeSpan);
    }
}

class ChunkyObject
{
    private static readonly AsyncLocal<ChunkyObject?> _current = new();

    // Stores lots of data (but it should be gen0)
    private readonly string _data = new string('A', 1024 * 32);

    public static ChunkyObject? Current
    {
        get => _current.Value;
        set => _current.Value = value;
    }

    public string Data => _data;
}
```

Instead of caching a number + CTS, this implicitly retains **all async locals** for an hour.

Typical numbers:

```
Before GC: 654.65 MB
After GC: 659.68 MB
```

Heap shape: `CancellationTokenSource -> ExecutionContext -> AsyncLocalValueMap -> ChunkyObject -> string`.



:white_check_mark: **GOOD** Use `CancellationToken.UnsafeRegister` to avoid capturing execution context:

```C#
public class NumberCache
{
    private readonly ConcurrentDictionary<int, CancellationTokenSource> _cache = new();
    private readonly TimeSpan _timeSpan;

    public NumberCache(TimeSpan timeSpan) => _timeSpan = timeSpan;

    public void Add(int key)
    {
        var cts = _cache.GetOrAdd(key, _ => new CancellationTokenSource());
        cts.Token.UnsafeRegister((_, _) => _cache.TryRemove(key, out _), null);
        cts.CancelAfter(_timeSpan);
    }
}
```

```
Before GC: 10.32 MB
After GC: 5.10 MB
```



:bulb: **NOTE:** You rarely control whether an API captures execution context. Minimize ambient state and null it out aggressively when crossing long-lived boundaries (see `StrongBox` technique above).

```C#
using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

// Singleton cache
var cache = new NumberCache(TimeSpan.FromHours(1));

var executionContext = ExecutionContext.Capture();

// Simulate 10000 concurrent requests
Parallel.For(0, 10000, i =>
{
    // Restore the initial ExecutionContext per "request"
    ExecutionContext.Restore(executionContext!);

    ChunkyObject.Current = new ChunkyObject();

    cache.Add(i);

    // Null out the chunky object so the GC can release the memory
    ChunkyObject.Current = default;
});

class ChunkyObject
{
    private static readonly AsyncLocal<StrongBox<ChunkyObject?>> _current = new();

    // Stores lots of data (but it should be gen0)
    private readonly string _data = new string('A', 1024 * 32);

    public static ChunkyObject? Current
    {
        get => _current.Value?.Value;
        set
        {
            var box = _current.Value;
            if (box is not null)
            {
                // Mutate the value in any execution context that was copied
                box.Value = null;
            }

            if (value is not null)
            {
                _current.Value = new StrongBox<ChunkyObject?>(value);
            }
        }
    }

    public string Data => _data;
}
```

Memory drops substantially:

```
Before GC: 7.91 MB
After GC: 5.66 MB
```

You may still retain `StrongBox<ChunkyObject>` shells with null references—technically a small leak, but orders of magnitude cheaper.



### Avoid setting `AsyncLocal<T>` values outside of async methods

Async methods restore ambient state on exit in a way ordinary methods do not.

❌ **BAD** Mutations leak out of synchronous helpers:

```C#
var local = new AsyncLocal<int>();
MethodA();
Console.WriteLine(local.Value);

void MethodA()
{
    local.Value = 1;
    MethodB();
    Console.WriteLine(local.Value);
}

void MethodB()
{
    local.Value = 2;
    Console.WriteLine(local.Value);
}
```

Prints `2, 2, 2`.

:white_check_mark: **GOOD** Set async locals inside `async` methods:

```C#
var local = new AsyncLocal<int>();
await MethodA();
Console.WriteLine(local.Value);

async Task MethodA()
{
    local.Value = 1;
    await MethodB();
    Console.WriteLine(local.Value);
}

async Task MethodB()
{
    local.Value = 2;
    Console.WriteLine(local.Value);
}
```

Prints `2, 1, 0`—each async method restores the prior context on the way out.

## Async runtime

`async`/`await` is language sugar over a compiler-generated **state machine** and the runtime's awaiter / scheduling infrastructure. Understanding that machinery makes `ConfigureAwait`, deadlocks, starvation, and `AsyncLocal` behavior predictable instead of magical.

Primary references: [Async/Await FAQ](https://devblogs.microsoft.com/dotnet/asyncawait-faq/), [ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/), [ExecutionContext vs SynchronizationContext](https://devblogs.microsoft.com/dotnet/executioncontext-vs-synchronizationcontext/).

### What the compiler generates

An `async` method is rewritten into a state machine (`IAsyncStateMachine`) that:

1. Runs synchronously from the start of the method (or from the last resume point).
2. At each `await`, asks the awaitable whether it is already complete (`IsCompleted`).
3. If complete (**fast path**): continues on the *same* thread with no yield, no context capture for scheduling, and often no allocation beyond what already existed.
4. If incomplete (**slow path**): captures state, hooks a continuation via `AwaitUnsafeOnCompleted` / `OnCompleted`, and returns an incomplete `Task`/`ValueTask` to the caller.
5. When the awaited operation finishes, the continuation resumes the state machine at the next state.

```C#
public async Task<int> AddOneAsync()
{
    // Runs on the caller thread until the first incomplete await
    int value = await GetValueAsync();
    // May resume on a different thread (pool / UI / custom scheduler)
    return value + 1;
}
```

Implications:

- **`async` ≠ background thread.** CPU work before the first incomplete `await` still runs on the caller.
- **Already-completed awaits do not force a thread hop.** Hot caches and completed `Task`/`ValueTask` paths stay synchronous through the `await`.
- **The returned `Task` represents the whole method**, not a particular thread.

### Awaitables and awaiters

`await expr` requires `expr` to be *awaitable*: it exposes `GetAwaiter()`, and the awaiter provides `IsCompleted`, `GetResult()`, and `OnCompleted` / `UnsafeOnCompleted`.

Common awaitables:

| Awaitable | Typical use |
|-----------|-------------|
| `Task` / `Task<T>` | General async APIs |
| `ValueTask` / `ValueTask<T>` | Allocation-sensitive completed-hot paths |
| `ConfiguredTaskAwaitable` | Result of `ConfigureAwait(...)` |
| custom types | Channels, timers, framework primitives |

`GetResult()` either returns the result or throws the stored exception (for `Task`, without wrapping in `AggregateException`—unlike `.Result` / `.Wait()`).

### Three contexts people mix up

These are related but not the same thing:

| Concept | Role | Flows across `await`? | Controlled by |
|---------|------|------------------------|---------------|
| [`ExecutionContext`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.executioncontext) | Ambient state (`AsyncLocal<T>`, security, etc.) | **Yes** — restored for the continuation | Capture is automatic; `SuppressFlow` / `Unsafe*` APIs are escape hatches |
| [`SynchronizationContext`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.synchronizationcontext) | App-model scheduler ("run this on the UI / request context") | **No** — not flowed; optionally *posted to* after await | `ConfigureAwait` |
| [`TaskScheduler`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler) | TPL scheduler for delegate-backed tasks | Used when no `SynchronizationContext` is present and `ConfigureAwait(true)` captures the current scheduler | `Task.Factory.StartNew`, custom schedulers, `ConfigureAwait` |

Mental model:

- **`ExecutionContext`** answers: *what ambient data is visible?*
- **`SynchronizationContext` / `TaskScheduler`** answer: *where should my code resume?*

ASP.NET Core installs **no** custom `SynchronizationContext`. Continuations therefore typically resume on the thread pool (subject to inline-continuation rules below). Classic ASP.NET, WPF, WinForms, and many UI stacks *do* install one—that is where `ConfigureAwait` correctness arguments originate.

### Where continuations run

When an awaited `Task` completes, the runtime roughly chooses among:

1. **Run the continuation inline** on the completing thread (common and cheap), or
2. **Queue** it to the captured `SynchronizationContext` / `TaskScheduler`, or
3. **Queue** it to the thread pool.

`TaskCreationOptions.RunContinuationsAsynchronously` / `TaskContinuationOptions.RunContinuationsAsynchronously` / `ConfigureAwait(ConfigureAwaitOptions.ForceYielding)` force asynchrony so the completer does not suddenly execute arbitrary user code on its stack.

That is why library code completing a `TaskCompletionSource` without `RunContinuationsAsynchronously` is dangerous: *your* thread may run *their* continuation, re-entering your locks or blocking your I/O thread.

### Thread pool and starvation

The .NET thread pool runs:

- `Task` continuations (when not inlined onto another thread)
- `Task.Run` / `ThreadPool.QueueUserWorkItem` work
- Timer callbacks
- Many async I/O completions

Blocking pool threads (sync-over-async, sync I/O, long CPU loops without yielding) reduces throughput. The pool injects more threads when it detects starvation, but injection is slow relative to request spikes—latency climbs first, then thread count.

ASP.NET Core scalability depends on **not blocking** pool threads during I/O waits. Yielding via incomplete `await` returns the thread to the pool; blocking with `.Result` / `.Wait()` keeps the thread occupied and often requires a *second* thread to finish the work.

### Completed await (fast path) vs incomplete await (slow path)

```C#
Task<int> cached = Task.FromResult(42);
int a = await cached;          // fast path: still on this thread
int b = await SlowIoAsync();   // slow path: yield, resume later
```

This matters for `ConfigureAwait(false)`: if the awaited task is **already complete**, there is nothing to schedule, so `ConfigureAwait(false)` does **not** hop off the current context. Code after that `await` still runs on the caller context. That is why "only `ConfigureAwait(false)` on the first await" is insufficient in libraries—see below.

### `async void` in the runtime

`async Task` methods report failure on the returned `Task`. `async void` methods are designed for UI event handlers: the runtime posts exceptions to the current `SynchronizationContext` when present, otherwise they can terminate the process. Server apps should treat `async void` as unsupported.

### Practical debugging tips

- **Parallel Stacks / Tasks** windows (Visual Studio / Rider) show incomplete awaits and blocked threads.
- A thread stuck in `Wait` / `GetResult` / `Monitor` while others wait for the pool is a classic starvation/deadlock signature.
- `AsyncLocal` / activity IDs surviving longer than a request often mean an API captured `ExecutionContext` (timers, `CancellationToken.Register`, etc.).

## `ConfigureAwait`

[`ConfigureAwait`](https://devblogs.microsoft.com/dotnet/configureawait-faq/) selects the awaitable used by `await`. For `Task`, it decides whether the continuation should marshal back to the captured `SynchronizationContext` / `TaskScheduler`.

```C#
// Default: capture context/scheduler when the await yields
await SomethingAsync();

// Equivalent explicit form
await SomethingAsync().ConfigureAwait(continueOnCapturedContext: true);

// Do not marshal back; resume on pool / completing thread
await SomethingAsync().ConfigureAwait(false);
```

`ConfigureAwait` does **not**:

- create parallelism by itself
- suppress `ExecutionContext` / `AsyncLocal` flow
- flow automatically into called methods (each `await` chooses independently)
- fix sync-over-async (blocking is still blocking)

### Default behavior (`ConfigureAwait(true)`)

When an await **yields** (incomplete task):

1. Capture `SynchronizationContext.Current`, or if null, `TaskScheduler.Current` when it is not the default pool scheduler.
2. On completion, `Post` / schedule the continuation back to that captured target when one was captured.
3. If nothing was captured, resume without marshaling (typically thread pool / inline on completer).

That default is why UI code can `await` and safely touch controls afterward—the continuation is posted back to the UI thread.

### `ConfigureAwait(false)`

Tells the awaiter: **do not** capture/marshal to `SynchronizationContext` / `TaskScheduler`. After an incomplete await, the continuation may run inline on the completer or on the thread pool.

Use it when the code after `await` does not need the app model context (no UI controls, no legacy ASP.NET request-thread affinity, etc.).

### Application code (ASP.NET Core, workers, most modern apps)

ASP.NET Core does **not** install a `SynchronizationContext`. In app code, `ConfigureAwait(false)` usually does **not** change correctness and rarely wins measurable performance. Prefer plain `await` for readability.

```C#
public async Task<User> GetUserAsync(int id, CancellationToken cancellationToken)
{
    // Fine in ASP.NET Core application code — no sync context to capture
    var user = await _db.Users.FindAsync([id], cancellationToken);
    user.LastSeenUtc = DateTime.UtcNow;
    await _db.SaveChangesAsync(cancellationToken);
    return user;
}
```

:bulb: **NOTE:** "No `SynchronizationContext`" does not mean "no threading rules." `HttpContext` is still **not** thread-safe for concurrent access, and request services still follow DI scopes. `ConfigureAwait` does not address those.

### Library / reusable code

General-purpose libraries may run under UI contexts, classic ASP.NET, tests with custom schedulers, etc. Use `ConfigureAwait(false)` unless you intentionally need the caller's context (uncommon in libraries).

```C#
public async Task<string> ReadAllAsync(Stream stream, CancellationToken cancellationToken)
{
    using var reader = new StreamReader(stream, leaveOpen: true);
    return await reader.ReadToEndAsync(cancellationToken).ConfigureAwait(false);
}
```

Why libraries care even if *your* app is ASP.NET Core:

1. The library author does not control the host app model.
2. `ConfigureAwait(false)` reduces deadlock risk if a consumer blocks on the returned `Task` (they should not—but they will).
3. Avoids unnecessary marshaling overhead in hosts that *do* have a sync context.

### Why "only on the first await" is wrong

❌ **BAD** Assumes the first `ConfigureAwait(false)` permanently leaves the context.

```C#
public async Task WorkAsync()
{
    await A().ConfigureAwait(false);
    // If A() was already completed, we never left the UI/request context!
    await B(); // may still capture and marshal
    await C();
}
```

If `A()` completes synchronously, execution continues on the original context and later awaits without `ConfigureAwait(false)` capture again. In library code, apply it to **each** `await` (or consciously opt in where context is required).

### Deadlock pattern `ConfigureAwait` does not excuse

❌ **BAD** Classic UI / legacy ASP.NET deadlock. `ConfigureAwait(false)` *inside* the callee can avoid it, but the real fix is: **do not block**.

```C#
// UI or classic ASP.NET thread
public void Button_Click(object sender, EventArgs e)
{
    // Blocks the context thread while the continuation needs that same context
    var result = GetDataAsync().Result;
}

public async Task<string> GetDataAsync()
{
    var data = await _http.GetStringAsync("https://example.com"); // captures context
    return data.ToUpperInvariant();
}
```

:white_check_mark: **GOOD (app code)** Keep it async end-to-end.

```C#
public async void Button_Click(object sender, EventArgs e) // UI event handler: async void is the exception
{
    var result = await GetDataAsync();
    textBox.Text = result;
}
```

:white_check_mark: **GOOD (library code)** Defend with `ConfigureAwait(false)` so consumers who block are less likely to deadlock—and still document that blocking is unsupported.

```C#
public async Task<string> GetDataAsync()
{
    var data = await _http.GetStringAsync("https://example.com").ConfigureAwait(false);
    return data.ToUpperInvariant();
}
```

In ASP.NET Core app code this deadlock is uncommon (no sync context), but **thread-pool starvation** from the same `.Result` pattern remains a production killer.

### `ConfigureAwaitOptions` (.NET 8+)

.NET 8 adds [`ConfigureAwait(ConfigureAwaitOptions)`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.configureawaitoptions):

| Flag | Purpose |
|------|---------|
| `ContinueOnCapturedContext` | Same as `ConfigureAwait(true)` |
| `None` | Same as `ConfigureAwait(false)` |
| `ForceYielding` | Always yield—even if the task is already complete; avoids stack-dives and can improve fairness |
| `SuppressThrowing` | Await completion without throwing; inspect `IsFaulted` / `IsCanceled` yourself. Only valid for non-generic `Task` |

```C#
// .NET 8+: wait for completion but handle errors manually
await task.ConfigureAwait(ConfigureAwaitOptions.SuppressThrowing);
if (task.IsFaulted)
{
    _logger.LogError(task.Exception!.InnerException, "background flush failed");
}
```

```C#
// Force an actual yield so the caller can make progress before this continues
await Task.CompletedTask.ConfigureAwait(ConfigureAwaitOptions.ForceYielding);
```

Flags can be combined where it makes sense, e.g. `ConfigureAwaitOptions.None | ConfigureAwaitOptions.ForceYielding`.

### Interaction with `ExecutionContext` / `AsyncLocal`

`ConfigureAwait(false)` does **not** stop `AsyncLocal<T>` from flowing. Ambient data still restores for the continuation. To avoid capturing ambient state into long-lived callbacks, use APIs like `CancellationToken.UnsafeRegister`, or clear ambient values at the right boundaries (see [`AsyncLocal<T>`](#asynclocalt)).

### Practical rules

1. **ASP.NET Core / worker app code:** plain `await` by default.
2. **Libraries and shared infrastructure:** `ConfigureAwait(false)` on awaits unless you truly need the caller context.
3. **UI app code:** plain `await` when the continuation touches UI; use `ConfigureAwait(false)` for internal non-UI helper paths if you want.
4. **Never** use `ConfigureAwait(false)` as permission to keep `.Result` / `.Wait()`—remove the blocking.
5. **`ConfigureAwait` is per-`await`**, not per-method or per-AppDomain. Completed awaits do not "leave" the context.
6. Prefer `TaskCreationOptions.RunContinuationsAsynchronously` when *your* code completes tasks that strangers await.
7. Use `.NET 8+` `ConfigureAwaitOptions.SuppressThrowing` / `ForceYielding` for the specific cases they were built for—not as a new default everywhere.

# Scenarios

General rules help, but real systems invent the bad patterns first. These scenarios map common production mistakes to fixes.

## Timer callbacks

❌ **BAD** `async void` timer callback—exceptions can crash the process.

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;

    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public async void Heartbeat(object? state)
    {
        await _client.GetAsync("http://mybackend/api/ping");
    }
}
```

❌ **BAD** Blocks the timer/thread-pool callback ([sync over async](#warning-sync-over-async)).

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;

    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public void Heartbeat(object? state)
    {
        _client.GetAsync("http://mybackend/api/ping").GetAwaiter().GetResult();
    }
}
```

:warning: **ACCEPTABLE BUT FRAGILE** Discarding a `Task` from a `Timer` callback avoids `async void`, but still lacks structured lifetime, overlap protection, and DI scope management.

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;

    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public void Heartbeat(object? state)
    {
        _ = DoAsyncPing();
    }

    private async Task DoAsyncPing()
    {
        try
        {
            await _client.GetAsync("http://mybackend/api/ping");
        }
        catch (Exception ex)
        {
            // Log — do not let this become unobserved forever without diagnostics
            Console.Error.WriteLine(ex);
        }
    }
}
```

:white_check_mark: **GOOD** `[PeriodicTimer](https://learn.microsoft.com/en-us/dotnet/api/system.threading.periodictimer)` (.NET 6+) — async-native loop, no `async void`.

```C#
public sealed class Pinger : IAsyncDisposable
{
    private readonly PeriodicTimer _timer;
    private readonly HttpClient _client;
    private readonly Task _loop;

    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
        _loop = RunAsync();
    }

    private async Task RunAsync()
    {
        try
        {
            while (await _timer.WaitForNextTickAsync())
            {
                try
                {
                    await _client.GetAsync("http://mybackend/api/ping");
                }
                catch (Exception ex)
                {
                    Console.Error.WriteLine(ex);
                }
            }
        }
        catch (OperationCanceledException)
        {
            // timer disposed
        }
    }

    public async ValueTask DisposeAsync()
    {
        _timer.Dispose();
        await _loop;
    }
}
```

:white_check_mark: **BETTER (ASP.NET Core)** Use a `[BackgroundService](https://learn.microsoft.com/en-us/dotnet/core/extensions/timer-service)` so the host controls startup/shutdown and cancellation.

```C#
public sealed class PingerService : BackgroundService
{
    private readonly HttpClient _client;
    private readonly ILogger<PingerService> _logger;

    public PingerService(HttpClient client, ILogger<PingerService> logger)
    {
        _client = client;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(1));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await _client.GetAsync("http://mybackend/api/ping", stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                throw;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "ping failed");
            }
        }
    }
}
```



## Implicit `async void` delegates

APIs that only accept `Action` force callers into blocking or `async void` lambdas.

❌ **BAD** Sync-only callback surface.

```C#
public class BackgroundQueue
{
    public static void FireAndForget(Action action) { }
}
```

❌ **BAD** Caller creates an implicit `async void` lambda (legal C#, dangerous runtime).

```C#
public class Program
{
    public static void Main(string[] args)
    {
        var httpClient = new HttpClient();
        BackgroundQueue.FireAndForget(async () =>
        {
            await httpClient.GetAsync("http://pinger/api/1");
        });

        Console.ReadLine();
    }
}
```

:white_check_mark: **GOOD** Offer a `Func<Task>` (or `Func<CancellationToken, Task>`) overload.

```C#
public class BackgroundQueue
{
    public static void FireAndForget(Action action) { }
    public static void FireAndForget(Func<Task> action) { }
    public static void FireAndForget(Func<CancellationToken, Task> action) { }
}
```



## `ConcurrentDictionary.GetOrAdd`

Caching async results in `ConcurrentDictionary` is common. `GetOrAdd`'s factory is synchronous, which tempts `.Result` and starvation.

❌ **BAD** Blocks the request thread on a cache miss.

```C#
public class PersonController : Controller
{
    private readonly AppDbContext _db;

    // This cache needs expiration
    private static readonly ConcurrentDictionary<int, Person> Cache = new();

    public PersonController(AppDbContext db) => _db = db;

    public IActionResult Get(int id)
    {
        var person = Cache.GetOrAdd(id, key => _db.People.FindAsync(key).Result);
        return Ok(person);
    }
}
```

:white_check_mark: **GOOD** Cache the `Task` (or `ValueTask` via `.AsTask()`), not the raw value, so awaiters share the in-flight work.

:warning: `GetOrAdd` may run the factory more than once under concurrency. Duplicate work is possible.

```C#
public class PersonController : Controller
{
    private readonly AppDbContext _db;

    // This cache needs expiration
    private static readonly ConcurrentDictionary<int, Task<Person>> Cache = new();

    public PersonController(AppDbContext db) => _db = db;

    public async Task<IActionResult> Get(int id)
    {
        var person = await Cache.GetOrAdd(id, key => _db.People.FindAsync(key).AsTask());
        return Ok(person);
    }
}
```

:white_check_mark: **GOOD** Async lazy pattern so the factory delegate runs once per key.

```C#
public class PersonController : Controller
{
    private readonly AppDbContext _db;

    // This cache needs expiration / eviction of faulted tasks
    private static readonly ConcurrentDictionary<int, AsyncLazy<Person>> Cache = new();

    public PersonController(AppDbContext db) => _db = db;

    public async Task<IActionResult> Get(int id)
    {
        var person = await Cache.GetOrAdd(
            id,
            key => new AsyncLazy<Person>(() => _db.People.FindAsync(key).AsTask())).Value;
        return Ok(person);
    }

    private sealed class AsyncLazy<T> : Lazy<Task<T>>
    {
        public AsyncLazy(Func<Task<T>> valueFactory) : base(valueFactory) { }
    }
}
```

:bulb: **NOTE:** Evict faulted/canceled tasks from the dictionary, or every subsequent caller observes the same failure forever. For production caching, prefer `IMemoryCache` / `HybridCache` (.NET 9+) / a dedicated library with expiration and stampede protection.

## Constructors

Constructors cannot be `async`. If initialization requires awaitable work, use an async factory, lazy initialization, or initialize in `IHostedService.StartAsync`.

```C#
public interface IRemoteConnectionFactory
{
    Task<IRemoteConnection> ConnectAsync(CancellationToken cancellationToken = default);
}

public interface IRemoteConnection : IAsyncDisposable
{
    Task PublishAsync(string channel, string message, CancellationToken cancellationToken = default);
}
```

❌ **BAD** Blocks in the constructor.

```C#
public class Service : IService
{
    private readonly IRemoteConnection _connection;

    public Service(IRemoteConnectionFactory connectionFactory)
    {
        _connection = connectionFactory.ConnectAsync().Result;
    }
}
```

:white_check_mark: **GOOD** Async factory.

```C#
public class Service : IService
{
    private readonly IRemoteConnection _connection;

    private Service(IRemoteConnection connection) => _connection = connection;

    public static async Task<Service> CreateAsync(
        IRemoteConnectionFactory connectionFactory,
        CancellationToken cancellationToken = default)
    {
        var connection = await connectionFactory.ConnectAsync(cancellationToken);
        return new Service(connection);
    }
}
```

:white_check_mark: **GOOD** For DI-heavy apps, register a factory / `IAsyncDisposable` hosted initializer rather than doing sync-over-async inside ctor injection.

## `WindowsIdentity.RunImpersonated`

This API runs work as an impersonated Windows identity. An [async overload](https://learn.microsoft.com/en-us/dotnet/api/system.security.principal.windowsidentity.runimpersonatedasync) exists since .NET 5.

❌ **BAD** Starts async work inside impersonation, then awaits outside—work may run without the impersonated context.

```C#
public async Task<IEnumerable<Product>> GetDataImpersonatedAsync(SafeAccessTokenHandle safeAccessTokenHandle)
{
    Task<IEnumerable<Product>>? products = null;
    WindowsIdentity.RunImpersonated(
        safeAccessTokenHandle,
        () =>
        {
            products = _db.QueryAsync("SELECT Name from Products");
        });
    return await products!;
}
```

❌ **BAD** Sync over async inside impersonation.

```C#
public IEnumerable<Product> GetDataImpersonated(SafeAccessTokenHandle safeAccessTokenHandle)
{
    return WindowsIdentity.RunImpersonated(
        safeAccessTokenHandle,
        () => _db.QueryAsync("SELECT Name from Products").Result);
}
```

:white_check_mark: **GOOD** (pre-.NET 5) Pass an async callback to `RunImpersonated` and await the returned task.

```C#
public Task<IEnumerable<Product>> GetDataImpersonatedAsync(SafeAccessTokenHandle safeAccessTokenHandle)
{
    return WindowsIdentity.RunImpersonated(
        safeAccessTokenHandle,
        () => _db.QueryAsync("SELECT Name from Products"));
}
```

:white_check_mark: **GOOD** Prefer `RunImpersonatedAsync` on .NET 5+.

```C#
public async Task<IEnumerable<Product>> GetDataImpersonatedAsync(SafeAccessTokenHandle safeAccessTokenHandle)
{
    return await WindowsIdentity.RunImpersonatedAsync(
        safeAccessTokenHandle,
        () => _db.QueryAsync("SELECT Name from Products"));
}
```

