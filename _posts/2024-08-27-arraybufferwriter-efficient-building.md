---
layout: post
title: "ArrayBufferWriter: Efficient Buffer Building"
date: 2024-08-27
---

`ArrayBufferWriter<T>` is a resizable buffer that grows efficiently while providing `IBufferWriter<T>` interface for high-performance serialization scenarios.

**Available since:** .NET Core 2.1 / .NET Standard 2.1

## Basic Usage

```csharp
var writer = new ArrayBufferWriter<byte>();

// Get writable memory
Memory<byte> memory = writer.GetMemory(100);
int bytesWritten = EncodeData(memory.Span);

// CRITICAL: Must call Advance with actual bytes written
writer.Advance(bytesWritten);

// Get final result
ReadOnlyMemory<byte> result = writer.WrittenMemory;
```

## When to Use Advance()

`Advance()` is only needed when writing directly to `GetSpan()` or `GetMemory()` results. The `.Write()` extension methods handle advancing automatically.

```csharp
var buffer = new ArrayBufferWriter<char>();

// Method 1: Direct span writing - REQUIRES Advance()
Span<char> span = buffer.GetSpan(100);
"Hello".AsSpan().CopyTo(span);
buffer.Advance(5); // MUST call Advance() with actual bytes written

// Method 2: Using Write extension methods - NO Advance() needed
buffer.Write("World".AsSpan()); // Extension method - automatically advances by 5

Console.WriteLine(buffer.WrittenCount); // Outputs: 10
```

## Advance() Rules and Pitfalls

```csharp
var buffer = new ArrayBufferWriter<byte>();

// CORRECT: Write then advance by actual amount
Span<byte> span1 = buffer.GetSpan(100);
int bytesWritten = EncodeData(span1);
buffer.Advance(bytesWritten);

// CORRECT: Write extension methods handle advancing
byte[] data = [1, 2, 3, 4];
buffer.Write(data.AsSpan()); // Extension method - no Advance() call needed

// WRONG: Advancing before writing
Span<byte> span2 = buffer.GetSpan(100);
buffer.Advance(50); // DON'T DO THIS
WriteData(span2); // Data might not fit in remaining space

// WRONG: Advancing by requested amount instead of actual
Span<byte> span3 = buffer.GetSpan(100);
int actual = WriteData(span3); // Returns 25
buffer.Advance(100); // WRONG - should be buffer.Advance(actual)

// WRONG: Double advancing with extension methods
buffer.Write([5, 6, 7].AsSpan()); // Extension method advances automatically
// buffer.Advance(3); // WRONG - Write() extension already advanced
```

## WrittenSpan vs WrittenMemory

Choose based on your usage pattern and performance needs:

```csharp
var buffer = new ArrayBufferWriter<byte>();
WriteData(buffer);

// WrittenSpan - for immediate, synchronous use
ReadOnlySpan<byte> span = buffer.WrittenSpan;
ProcessDataSync(span); // Fast, stack-friendly

// WrittenMemory - for async operations or storage
ReadOnlyMemory<byte> memory = buffer.WrittenMemory;
await ProcessDataAsync(memory); // Can cross async boundaries

// WrittenMemory also provides span access when needed
ReadOnlySpan<byte> spanFromMemory = memory.Span;
```

## JSON Serialization

```csharp
public static byte[] SerializeToJson<T>(T value)
{
    var buffer = new ArrayBufferWriter<byte>();
    using var writer = new Utf8JsonWriter(buffer);
    
    JsonSerializer.Serialize(writer, value);
    return buffer.WrittenSpan.ToArray();
}
```

## Building Protocol Messages

```csharp
public static ReadOnlyMemory<byte> CreateMessage(int id, ReadOnlySpan<byte> payload)
{
    var buffer = new ArrayBufferWriter<byte>();
    
    // Write header
    Span<byte> header = buffer.GetSpan(8);
    BinaryPrimitives.WriteInt32LittleEndian(header, id);
    BinaryPrimitives.WriteInt32LittleEndian(header[4..], payload.Length);
    buffer.Advance(8);
    
    // Write payload
    payload.CopyTo(buffer.GetSpan(payload.Length));
    buffer.Advance(payload.Length);
    
    return buffer.WrittenMemory;
}
```

## Custom Buffer Writer

Understanding `IBufferWriter<T>` helps demystify how `ArrayBufferWriter<T>` works internally. Here's a simplified implementation that writes directly to a stream:

```csharp
public class StreamBufferWriter : IBufferWriter<byte>
{
    private readonly Stream _stream;
    private byte[] _buffer = new byte[4096];
    
    public void Advance(int count)
    {
        _stream.Write(_buffer, 0, count);
    }
    
    public Memory<byte> GetMemory(int sizeHint = 0)
    {
        if (sizeHint > _buffer.Length)
            Array.Resize(ref _buffer, sizeHint);
        return _buffer;
    }
    
    public Span<byte> GetSpan(int sizeHint = 0) => GetMemory(sizeHint).Span;
}

// Usage - same interface as ArrayBufferWriter
var streamWriter = new StreamBufferWriter(fileStream);
var utf8Writer = new Utf8JsonWriter(streamWriter);
JsonSerializer.Serialize(utf8Writer, data); // Writes directly to stream
```

## Efficient Text Building

```csharp
public static string BuildQuery(Dictionary<string, string> parameters)
{
    var buffer = new ArrayBufferWriter<byte>();
    var writer = new Utf8JsonWriter(buffer);
    
    writer.WriteStartObject();
    foreach (var (key, value) in parameters)
    {
        writer.WriteString(key, value);
    }
    writer.WriteEndObject();
    
    return Encoding.UTF8.GetString(buffer.WrittenSpan);
}
```

## Memory Management Benefits

```csharp
// Avoids multiple allocations during building
var buffer = new ArrayBufferWriter<char>();
buffer.Write("SELECT * FROM users WHERE ");

foreach (var condition in conditions)
{
    buffer.Write(condition);
    buffer.Write(" AND ");
}

// Remove trailing " AND "
var query = buffer.WrittenSpan[..^5].ToString();
```

## Common Patterns and Pitfalls

```csharp
public static void WriteVariableData(IBufferWriter<byte> writer)
{
    // Pattern 1: Variable-length encoding
    Span<byte> lengthBytes = writer.GetSpan(4);
    int dataLength = EncodeLength(lengthBytes, out int lengthBytesUsed);
    writer.Advance(lengthBytesUsed); // Only advance by actual bytes used
    
    // Pattern 2: Writing collections
    foreach (var item in items)
    {
        Span<byte> itemSpan = writer.GetSpan(EstimateItemSize(item));
        int itemBytesWritten = EncodeItem(item, itemSpan);
        writer.Advance(itemBytesWritten); // Critical: advance per item
    }
}

// WRONG: Don't advance before writing
void BadExample(IBufferWriter<byte> writer)
{
    Span<byte> span = writer.GetSpan(100);
    writer.Advance(50); // WRONG: advancing before writing
    WriteData(span); // Data might not fit in remaining space
}

// RIGHT: Write first, then advance
void GoodExample(IBufferWriter<byte> writer)
{
    Span<byte> span = writer.GetSpan(100);
    int written = WriteData(span);
    writer.Advance(written); // Correct: advance by actual amount
}
```

## Choosing WrittenSpan vs WrittenMemory

```csharp
public async Task ProcessBuffer(ArrayBufferWriter<byte> buffer)
{
    BuildData(buffer);
    
    // Use WrittenSpan for:
    // - Immediate processing
    // - When you need maximum performance
    // - Stack-based operations
    if (NeedsImmediateProcessing())
    {
        ReadOnlySpan<byte> data = buffer.WrittenSpan;
        ValidateData(data); // Fast, no allocations
        ComputeChecksum(data);
    }
    
    // Use WrittenMemory for:
    // - Async operations
    // - Storing for later use
    // - Passing across async boundaries
    if (NeedsAsyncProcessing())
    {
        ReadOnlyMemory<byte> data = buffer.WrittenMemory;
        await SendOverNetworkAsync(data); // Can cross await
        await WriteToFileAsync(data);
    }
    
    // Convert when needed
    ReadOnlyMemory<byte> memory = buffer.WrittenMemory;
    ReadOnlySpan<byte> span = memory.Span; // Get span from memory
}
```

## ArrayBufferWriter vs StringBuilder

For string building, choose based on your specific needs:

**StringBuilder:** Simple string building with formatting
```csharp
var report = new StringBuilder();
report.AppendLine($"User: {user.Name}");
report.AppendFormat("Total: ${0:F2}", total);
return report.ToString();
```

**ArrayBufferWriter&lt;char&gt;:** Modern API integration and span access
```csharp
var buffer = new ArrayBufferWriter<char>();
var writer = new CustomTextWriter(buffer); // Takes IBufferWriter<char>
writer.WriteFormatted(data);

ReadOnlySpan<char> content = buffer.WrittenSpan; // Direct span access
return content.ToString();
```

Use `StringBuilder` for convenience and readability. Use `ArrayBufferWriter&lt;char&gt;` when you need `IBufferWriter&lt;T&gt;` compatibility or span processing before string conversion.

`ArrayBufferWriter<T>` provides controlled buffer growth with minimal allocations, perfect for building data before final consumption by serializers, network APIs, or file operations. Always advance by actual bytes written, and choose `WrittenSpan` for immediate use or `WrittenMemory` for async scenarios.