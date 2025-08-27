---
layout: post
title: "ValueTask: Zero-Allocation Async Operations"
date: 2024-08-27
---

`ValueTask<T>` is a struct-based alternative to `Task<T>` that eliminates allocations when operations complete synchronously or can be pooled.

**Available since:** .NET Core 2.0 / .NET Standard 2.1 / C# 7.0

## Basic Concept

```csharp
// Traditional Task - always allocates
public async Task<string> GetFromCacheAsync(string key)
{
    return _cache.TryGetValue(key, out string value) ? value : await LoadFromDatabaseAsync(key);
}

// ValueTask - zero allocation when cache hits
public async ValueTask<string> GetFromCacheAsync(string key)
{
    return _cache.TryGetValue(key, out string value) ? value : await LoadFromDatabaseAsync(key);
}
```

## Synchronous Completion

```csharp
public ValueTask<int> ReadAsync(Memory<byte> buffer)
{
    if (_bytesAvailable > 0)
    {
        // Synchronous path - no allocation
        int bytesRead = Math.Min(buffer.Length, _bytesAvailable);
        _buffer.AsSpan(0, bytesRead).CopyTo(buffer.Span);
        _bytesAvailable -= bytesRead;
        return new ValueTask<int>(bytesRead);
    }
    
    // Asynchronous path - allocation occurs here
    return ReadAsyncCore(buffer);
}

private async ValueTask<int> ReadAsyncCore(Memory<byte> buffer)
{
    await WaitForDataAsync();
    return ReadAsync(buffer).Result;
}
```

## IValueTaskSource for Pooling

```csharp
public sealed class PooledValueTask<T> : IValueTaskSource<T>
{
    private static readonly ConcurrentQueue<PooledValueTask<T>> s_pool = new();
    private ManualResetValueTaskSourceCore<T> _core;
    
    public static PooledValueTask<T> Rent()
    {
        return s_pool.TryDequeue(out var task) ? task : new PooledValueTask<T>();
    }
    
    public ValueTask<T> Task => new ValueTask<T>(this, _core.Version);
    
    public void SetResult(T result)
    {
        _core.SetResult(result);
    }
    
    public void SetException(Exception error)
    {
        _core.SetException(error);
    }
    
    public T GetResult(short token) => _core.GetResult(token);
    public ValueTaskSourceStatus GetStatus(short token) => _core.GetStatus(token);
    public void OnCompleted(Action<object> continuation, object state, short token, ValueTaskSourceOnCompletedFlags flags)
        => _core.OnCompleted(continuation, state, token, flags);
        
    private void Return()
    {
        _core.Reset();
        s_pool.Enqueue(this);
    }
}
```

## ConfigureAwait(false) Still Matters

```csharp
// Library code - avoid deadlocks
public async ValueTask<Data> ProcessDataAsync()
{
    var raw = await LoadDataAsync().ConfigureAwait(false);
    var processed = await TransformAsync(raw).ConfigureAwait(false);
    return processed;
}
```

## Converting Between Task and ValueTask

```csharp
// ValueTask to Task (when needed for Task.WhenAll, etc.)
ValueTask<string> valueTask = GetDataAsync();
Task<string> task = valueTask.AsTask();

// Task to ValueTask (implicit)
Task<string> task = LoadFromDatabaseAsync();
ValueTask<string> valueTask = task; // implicit conversion
```

## Hot Path Optimization

```csharp
private readonly Dictionary<string, string> _cache = new();

public ValueTask<string> GetValueAsync(string key)
{
    // Fast path - synchronous, zero allocation
    if (_cache.TryGetValue(key, out string cached))
        return new ValueTask<string>(cached);
    
    // Slow path - asynchronous, allocation
    return LoadAndCacheAsync(key);
}

private async ValueTask<string> LoadAndCacheAsync(string key)
{
    string value = await _repository.LoadAsync(key);
    _cache[key] = value;
    return value;
}
```

## State Machine Considerations

```csharp
// DON'T: Multiple awaits on same ValueTask
var valueTask = GetDataAsync();
string result1 = await valueTask; // OK
string result2 = await valueTask; // WRONG - undefined behavior

// DO: Use .Preserve() if you need multiple awaits
var valueTask = GetDataAsync();
Task<string> task = valueTask.Preserve();
string result1 = await task; // OK
string result2 = await task; // OK
```

## Performance Pattern

```csharp
public ValueTask<bool> TryProcessAsync()
{
    // Check if we can complete synchronously
    if (CanProcessSynchronously())
    {
        ProcessSynchronously();
        return new ValueTask<bool>(true);
    }
    
    // Fall back to async processing
    return ProcessAsyncCore();
}
```

`ValueTask<T>` reduces pressure on the garbage collector in high-throughput scenarios where many operations complete synchronously, particularly effective in caching layers and I/O abstractions.