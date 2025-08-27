---
layout: post
title: "Efficient String Processing in C#"
date: 2024-08-27
---

String allocations are one of the biggest sources of garbage collection pressure in C# applications. Understanding allocation patterns and modern alternatives can dramatically improve performance.

**Available since:** Various - .NET Framework 1.0 to .NET 8

## The String Allocation Problem

```csharp
// Each operation creates a new string object
public string BuildMessage(string name, int count)
{
    // 4 string allocations for a simple operation
    return "Hello " + name + ", you have " + count + " items";
}

// Hot path example - allocates in tight loop
public List<string> ProcessItems(string[] items)
{
    var results = new List<string>();
    foreach (var item in items)
    {
        // Each iteration allocates multiple strings
        results.Add("Item: " + item.ToUpper() + " [processed]");
    }
    return results;
}
```

## StringBuilder for Multiple Concatenations

```csharp
// Use StringBuilder for multiple operations
public string BuildQuery(Dictionary<string, string> parameters)
{
    var query = new StringBuilder("SELECT * FROM users WHERE ");
    
    bool first = true;
    foreach (var (key, value) in parameters)
    {
        if (!first) query.Append(" AND ");
        query.Append(key).Append(" = '").Append(value).Append("'");
        first = false;
    }
    
    return query.ToString(); // Single final allocation
}

// Pre-size StringBuilder when possible
public string BuildLargeString(IEnumerable<string> parts)
{
    var estimated = parts.Sum(p => p.Length) + 100; // Add buffer
    var builder = new StringBuilder(estimated);
    
    foreach (var part in parts)
        builder.Append(part);
        
    return builder.ToString();
}
```

## String Interpolation vs Concatenation

```csharp
// String interpolation - efficient for simple cases
public string FormatUser(string name, int age)
{
    return $"User: {name}, Age: {age}"; // Single allocation
}

// Avoid complex expressions in interpolation
public string BadInterpolation(User user)
{
    // Multiple allocations within interpolation
    return $"User: {user.Name.ToUpper().Trim()}, Email: {user.Email.ToLower()}";
}

// Better - extract to variables
public string GoodInterpolation(User user)
{
    string name = user.Name.ToUpper().Trim();
    string email = user.Email.ToLower();
    return $"User: {name}, Email: {email}";
}
```

## ReadOnlySpan for Zero-Allocation Processing

```csharp
// Traditional approach - multiple allocations
public bool IsValidEmailOld(string email)
{
    string trimmed = email.Trim();
    string lower = trimmed.ToLower();
    int atIndex = lower.IndexOf('@');
    
    if (atIndex <= 0) return false;
    
    string local = lower.Substring(0, atIndex);
    string domain = lower.Substring(atIndex + 1);
    
    return domain.Contains('.') && !local.StartsWith('.');
}

// Modern approach - zero allocations
public bool IsValidEmail(ReadOnlySpan<char> email)
{
    ReadOnlySpan<char> trimmed = email.Trim();
    
    int atIndex = trimmed.IndexOf('@');
    if (atIndex <= 0) return false;
    
    ReadOnlySpan<char> local = trimmed[..atIndex];
    ReadOnlySpan<char> domain = trimmed[(atIndex + 1)..];
    
    return domain.IndexOf('.') > 0 && 
           !local.StartsWith(".") && 
           !domain.EndsWith(".");
}
```

## String.Create for Custom Formatting

```csharp
// Efficient custom formatting without intermediate allocations
public static string FormatFileSize(long bytes)
{
    const int maxLength = 20; // "999.99 GB" is about 9 chars
    
    return String.Create(maxLength, bytes, (span, value) =>
    {
        string unit;
        double size;
        
        if (value >= 1_000_000_000)
        {
            size = value / 1_073_741_824.0; // 1024^3
            unit = " GB";
        }
        else if (value >= 1_000_000)
        {
            size = value / 1_048_576.0; // 1024^2
            unit = " MB";
        }
        else
        {
            size = value / 1024.0;
            unit = " KB";
        }
        
        // Format directly into the span
        bool success = size.TryFormat(span, out int written, "F2");
        if (success && written + unit.Length <= span.Length)
        {
            unit.AsSpan().CopyTo(span[written..]);
            span = span[..(written + unit.Length)];
        }
    });
}
```

## ArrayBufferWriter for Complex Building

```csharp
public static string BuildJsonArray(IEnumerable<object> items)
{
    var buffer = new ArrayBufferWriter<byte>();
    var writer = new Utf8JsonWriter(buffer);
    
    writer.WriteStartArray();
    foreach (var item in items)
    {
        JsonSerializer.Serialize(writer, item);
    }
    writer.WriteEndArray();
    
    // Single allocation for final string
    return Encoding.UTF8.GetString(buffer.WrittenSpan);
}
```

## Avoid String.Split When Possible

```csharp
// Allocates array and substrings
public List<string> ParseCsvOld(string line)
{
    return line.Split(',').ToList();
}

// Zero-allocation parsing with spans
public void ParseCsv(ReadOnlySpan<char> line, List<string> results)
{
    results.Clear();
    
    while (!line.IsEmpty)
    {
        int commaIndex = line.IndexOf(',');
        if (commaIndex == -1)
        {
            results.Add(line.ToString()); // Only allocate when storing
            break;
        }
        
        results.Add(line[..commaIndex].ToString());
        line = line[(commaIndex + 1)..];
    }
}
```

## String Pooling for Repeated Values

```csharp
// Use string interning for known repeated values
private static readonly ConcurrentDictionary<string, string> _stringPool = new();

public string GetCachedString(string input)
{
    return _stringPool.GetOrAdd(input, s => s);
}

// Or use built-in interning for compile-time constants
public string GetStatusMessage(int code)
{
    return code switch
    {
        200 => "OK",           // Interned at compile time
        404 => "Not Found",    // Interned at compile time
        500 => "Server Error", // Interned at compile time
        _ => $"Status: {code}" // Allocated
    };
}
```

## Memory<char> for Async String Building

```csharp
public async ValueTask<string> BuildStringAsync(IAsyncEnumerable<string> parts)
{
    var buffer = new ArrayBufferWriter<char>();
    
    await foreach (var part in parts)
    {
        part.AsSpan().CopyTo(buffer.GetSpan(part.Length));
        buffer.Advance(part.Length);
    }
    
    // Single allocation for final result
    return buffer.WrittenSpan.ToString();
}
```

## Performance Guidelines

**Use StringBuilder when:**
- Building strings with 3+ concatenations
- Appending in loops
- You know approximate final size

**Use ReadOnlySpan&lt;char&gt; when:**
- Parsing/validating without storing substrings
- Hot path string operations
- You don't need the result to cross async boundaries

**Use String.Create when:**
- Custom formatting logic
- You know exact final length
- Building complex formatted strings

**Avoid:**
- String concatenation in loops
- Excessive `.ToUpper()/.ToLower()` calls
- `.Substring()` in hot paths
- String.Split when you only need first/last part

Modern C# provides powerful tools for zero-allocation string processing. Choose the right approach based on your specific use case and performance requirements.