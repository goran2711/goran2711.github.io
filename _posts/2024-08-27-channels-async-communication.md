---
layout: post
title: "Channels: Thread-Safe Async Communication"
date: 2024-08-27
---

Channels provide high-performance, thread-safe communication between producers and consumers with backpressure support and configurable buffering strategies.

**Available since:** .NET Core 2.1 / .NET Standard 2.1

## Basic Producer-Consumer

```csharp
var channel = Channel.CreateUnbounded<int>();
var writer = channel.Writer;
var reader = channel.Reader;

// Producer
await writer.WriteAsync(42);
writer.Complete();

// Consumer
while (await reader.WaitToReadAsync())
{
    while (reader.TryRead(out int item))
    {
        Console.WriteLine(item);
    }
}
```

## Bounded Channel with Backpressure

```csharp
var options = new BoundedChannelOptions(capacity: 100)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = true,
    SingleWriter = false
};

var channel = Channel.CreateBounded<WorkItem>(options);

// Producer blocks when channel is full
async Task ProduceWork()
{
    for (int i = 0; i < 1000; i++)
    {
        await channel.Writer.WriteAsync(new WorkItem(i));
    }
    channel.Writer.Complete();
}
```

## Pipeline Processing

```csharp
public static async Task ProcessPipeline<T>(
    ChannelReader<T> input,
    ChannelWriter<T> output,
    Func<T, Task<T>> processor)
{
    await foreach (var item in input.ReadAllAsync())
    {
        var processed = await processor(item);
        await output.WriteAsync(processed);
    }
    output.Complete();
}

// Usage
var stage1 = Channel.CreateBounded<string>(50);
var stage2 = Channel.CreateBounded<string>(50);

_ = ProcessPipeline(stage1.Reader, stage2.Writer, item => 
    Task.FromResult(item.ToUpper()));
```

## Fan-Out Pattern

```csharp
public static async Task FanOut<T>(
    ChannelReader<T> source,
    params ChannelWriter<T>[] destinations)
{
    await foreach (var item in source.ReadAllAsync())
    {
        var tasks = destinations.Select(dest => dest.WriteAsync(item).AsTask());
        await Task.WhenAll(tasks);
    }
    
    foreach (var dest in destinations)
        dest.Complete();
}
```

## Producer-Consumer with Cancellation

```csharp
public async Task ProcessItems(CancellationToken cancellationToken)
{
    var channel = Channel.CreateBounded<WorkItem>(100);
    
    // Producer task
    var producer = Task.Run(async () =>
    {
        try
        {
            for (int i = 0; !cancellationToken.IsCancellationRequested; i++)
            {
                await channel.Writer.WriteAsync(new WorkItem(i), cancellationToken);
                await Task.Delay(100, cancellationToken);
            }
        }
        finally
        {
            channel.Writer.Complete();
        }
    }, cancellationToken);
    
    // Consumer
    await foreach (var item in channel.Reader.ReadAllAsync(cancellationToken))
    {
        await ProcessItem(item);
    }
}
```

## AsyncEnumerable Bridge

```csharp
public static async IAsyncEnumerable<T> ToAsyncEnumerable<T>(
    this ChannelReader<T> reader,
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    while (await reader.WaitToReadAsync(cancellationToken))
    {
        while (reader.TryRead(out T item))
        {
            yield return item;
        }
    }
}

// Usage
await foreach (var item in channel.Reader.ToAsyncEnumerable())
{
    ProcessItem(item);
}
```

## Drop-Oldest Strategy

```csharp
var options = new BoundedChannelOptions(10)
{
    FullMode = BoundedChannelFullMode.DropOldest,
    AllowSynchronousContinuations = false
};

var channel = Channel.CreateBounded<LogEntry>(options);

// Always succeeds, drops oldest entries when full
channel.Writer.TryWrite(new LogEntry("New message"));
```

Channels excel at decoupling producers from consumers, handling backpressure, and building async processing pipelines with clean cancellation support.