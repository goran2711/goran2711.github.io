---
layout: post
title: "Span: Zero-Copy Memory Operations"
date: 2024-08-27
---

`Span<T>` is a stack-allocated ref struct that provides type-safe access to contiguous memory without allocation. It's essentially a pointer with length and bounds checking.

**Available since:** .NET Core 2.1 / .NET Standard 2.1 / C# 7.2

## Core Concept

```csharp
// All these create spans over the same memory
int[] array = {1, 2, 3, 4, 5};
Span<int> span1 = array;
Span<int> span2 = array.AsSpan();
Span<int> span3 = new Span<int>(array, 1, 3); // elements 2,3,4
```

## Stack Allocation

```csharp
// Allocates on stack, not heap
Span<byte> buffer = stackalloc byte[256];
ProcessData(buffer);

void ProcessData(Span<byte> data)
{
    // Works with any contiguous memory source
    for (int i = 0; i < data.Length; i++)
        data[i] = (byte)(i % 256);
}
```

## Slicing Without Copying

```csharp
ReadOnlySpan<char> text = "Hello, World!";
ReadOnlySpan<char> hello = text[..5];     // "Hello"
ReadOnlySpan<char> world = text[7..12];   // "World"

// No string allocations occurred
```

## Parsing Performance

```csharp
// Old way - allocates substring
string ExtractValue(string input)
{
    int start = input.IndexOf('=') + 1;
    return input.Substring(start);
}

// New way - zero allocations
ReadOnlySpan<char> ExtractValue(ReadOnlySpan<char> input)
{
    int start = input.IndexOf('=') + 1;
    return input[start..];
}

// If you need to parse the extracted value
int ExtractNumberValue(ReadOnlySpan<char> input)
{
    ReadOnlySpan<char> numberSpan = ExtractValue(input);
    return int.Parse(numberSpan); // Parse directly from span
}
```

## Memory Pattern Matching

```csharp
static bool StartsWithIgnoreCase(ReadOnlySpan<char> text, ReadOnlySpan<char> prefix)
{
    return text.StartsWith(prefix, StringComparison.OrdinalIgnoreCase);
}

// Usage
ReadOnlySpan<char> data = GetLargeText();
if (StartsWithIgnoreCase(data, "HTTP/"))
{
    // Process without allocating substrings
}
```

## Ref Structs and Stack-Only Types

`Span<T>` is a ref struct, meaning it can only live on the stack, never on the heap. This enables zero-allocation memory views but comes with constraints.

```csharp
// Ref struct - stack allocated, contains references to memory
public readonly ref struct TextParser
{
    private readonly ReadOnlySpan<char> _text;
    
    public TextParser(ReadOnlySpan<char> text) => _text = text;
    
    public ReadOnlySpan<char> ExtractWord(int index)
    {
        // Can safely return spans - all stack allocated
        int start = FindWordStart(index);
        int end = FindWordEnd(start);
        return _text[start..end];
    }
}

// Usage - everything stays on stack
ReadOnlySpan<char> input = "Hello world from spans";
var parser = new TextParser(input);
ReadOnlySpan<char> word = parser.ExtractWord(1); // "world"
```

## Ref Struct Constraints

```csharp
public class DataProcessor
{
    // ILLEGAL - ref structs cannot be fields
    // private Span<byte> _buffer;
    
    // ILLEGAL - cannot be in async methods
    // public async Task ProcessAsync(Span<byte> data) { }
    
    // ILLEGAL - cannot be generic type arguments
    // List<Span<byte>> spans;
    
    // LEGAL - local variables and parameters only
    public void ProcessData(Span<byte> data)
    {
        Span<byte> localBuffer = stackalloc byte[256];
        ProcessBuffer(data, localBuffer);
    }
    
    private void ProcessBuffer(Span<byte> input, Span<byte> output)
    {
        // Both spans are stack-allocated references
        input.CopyTo(output);
    }
}
```

## Custom Ref Structs with Spans

```csharp
public readonly ref struct BufferWriter
{
    private readonly Span<byte> _buffer;
    private readonly int _position;
    
    public BufferWriter(Span<byte> buffer, int position = 0)
    {
        _buffer = buffer;
        _position = position;
    }
    
    public BufferWriter WriteInt32(int value)
    {
        BinaryPrimitives.WriteInt32LittleEndian(_buffer[_position..], value);
        return new BufferWriter(_buffer, _position + 4);
    }
    
    public BufferWriter WriteBytes(ReadOnlySpan<byte> data)
    {
        data.CopyTo(_buffer[_position..]);
        return new BufferWriter(_buffer, _position + data.Length);
    }
}

// Usage - fluent API with zero allocations
Span<byte> buffer = stackalloc byte[1024];
var writer = new BufferWriter(buffer)
    .WriteInt32(42)
    .WriteBytes("Hello"u8);
```

## ReadOnlySpan vs Span

`ReadOnlySpan<T>` provides immutable access to memory, preventing accidental modifications and enabling broader usage scenarios.

```csharp
// ReadOnlySpan - immutable view
public static int CountWords(ReadOnlySpan<char> text)
{
    int count = 0;
    bool inWord = false;
    
    foreach (char c in text) // Can iterate directly
    {
        if (char.IsWhiteSpace(c))
            inWord = false;
        else if (!inWord)
        {
            inWord = true;
            count++;
        }
    }
    return count;
}

// Works with any source
string text = "Hello world";
char[] array = {'H', 'i'};
ReadOnlySpan<char> fromString = text; // Implicit conversion
ReadOnlySpan<char> fromArray = array;
ReadOnlySpan<char> fromLiteral = "Direct literal";

int words1 = CountWords(fromString);
int words2 = CountWords("Inline text"); // Direct usage
```

## ReadOnlySpan String Operations

```csharp
public static bool IsValidEmail(ReadOnlySpan<char> email)
{
    int atIndex = email.IndexOf('@');
    if (atIndex <= 0 || atIndex == email.Length - 1)
        return false;
        
    ReadOnlySpan<char> local = email[..atIndex];
    ReadOnlySpan<char> domain = email[(atIndex + 1)..];
    
    return domain.IndexOf('.') > 0 && 
           !local.StartsWith('.') && 
           !domain.EndsWith('.');
}

// Zero allocations for validation
bool valid = IsValidEmail("user@example.com");
```

## Span vs ReadOnlySpan Conversion

```csharp
Span<char> mutableSpan = stackalloc char[100];
ReadOnlySpan<char> readOnlyView = mutableSpan; // Implicit conversion

// Cannot go backwards - compilation error
// Span<char> mutable = readOnlySpan; // ERROR

public void ProcessData(Span<char> buffer)
{
    // Can pass mutable span to readonly methods
    int length = CalculateLength(buffer); // Span -> ReadOnlySpan
    
    // Modify after readonly operations
    buffer[0] = char.ToUpper(buffer[0]);
}

private int CalculateLength(ReadOnlySpan<char> text)
{
    return text.TrimEnd().Length;
}
```

## ReadOnlySpan with String Literals

```csharp
// UTF-8 string literals create ReadOnlySpan<byte>
ReadOnlySpan<byte> utf8Data = "Hello, 世界"u8;

// Useful for protocol headers, magic numbers
ReadOnlySpan<byte> pngHeader = [0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A];

public static bool IsPngFile(ReadOnlySpan<byte> fileData)
{
    return fileData.StartsWith(pngHeader);
}
```

The ref struct limitation ensures `Span<T>` operations remain on the stack, preventing accidental heap allocations while enabling safe, high-performance memory operations. Use `ReadOnlySpan<T>` when you only need read access - it's safer and works in more scenarios.