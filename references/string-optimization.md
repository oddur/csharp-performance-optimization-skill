# String Optimization

## StringBuilder

### When to Use

- Concatenating in a loop
- More than 3-4 concatenations
- Unknown number of concatenations at compile time

### When NOT to Use

- 2-3 strings → `+` or interpolation is faster
- All parts known at compile time → `const` or interpolation
- Hot path → consider stack-allocated buffers

### Preallocate Capacity

```csharp
var sb = new StringBuilder(estimatedLength);
```

### Pool for High-Frequency Usage

```csharp
private static readonly ObjectPool<StringBuilder> Pool =
    new DefaultObjectPoolProvider().CreateStringBuilderPool();

public string Build()
{
    var sb = Pool.Get();
    try
    {
        // build string
        return sb.ToString();
    }
    finally
    {
        Pool.Return(sb);
    }
}
```

---

## Modern String APIs (.NET 6+)

### string.Create

Single allocation, no intermediate buffers:

```csharp
string result = string.Create(16, (id, timestamp), (span, state) =>
{
    state.id.TryFormat(span, out int written);
    span[written++] = '-';
    state.timestamp.TryFormat(span.Slice(written), out _);
});
```

### Interpolated String Handlers

Zero allocation if condition is false:

```csharp
// No allocation if logging disabled
logger.LogDebug($"Player {playerId} moved to {position}");
```

### Span<char> with Stackalloc

For small, method-scoped strings:

```csharp
Span<char> buffer = stackalloc char[64];
if (value.TryFormat(buffer, out int written))
{
    ReadOnlySpan<char> result = buffer.Slice(0, written);
    // use result within method scope
}
```

---

## String Interning and Freezing

### Static Readonly for Known Strings

String literals are automatically interned:

```csharp
public static class MessageTypes
{
    public static readonly string PlayerMove = "player_move";
    public static readonly string PlayerSpawn = "player_spawn";
}

// Reference equality works
if (ReferenceEquals(msg, MessageTypes.PlayerMove)) { }
```

### string.Intern for Runtime Strings

For strings that repeat frequently at runtime:

```csharp
// Identifiers from network/config/database
string interned = string.Intern(parsedIdentifier);
```

**Do NOT intern:**
- Unbounded user input (interned strings never GC'd)
- Unique or infrequent strings
- Short-lived, method-scoped strings

### FrozenSet/FrozenDictionary (.NET 8+)

For read-heavy lookup tables:

```csharp
private static readonly FrozenSet<string> ValidCommands =
    new[] { "move", "attack", "inventory" }.ToFrozenSet(StringComparer.Ordinal);

if (ValidCommands.Contains(command)) { }
```

### StringComparer.Ordinal

Always use for lookups in hot paths:

```csharp
// Fast - ordinal comparison
var dict = new Dictionary<string, int>(StringComparer.Ordinal);

// Slow - culture-aware
var dict = new Dictionary<string, int>();  // Default comparer
```

### Custom String Pool

For bounded, high-churn scenarios with GC eligibility:

```csharp
public class StringPool
{
    private readonly ConcurrentDictionary<string, WeakReference<string>> _pool = new();

    public string Intern(string value)
    {
        if (_pool.TryGetValue(value, out var weakRef) && weakRef.TryGetTarget(out var existing))
            return existing;
        
        _pool[value] = new WeakReference<string>(value);
        return value;
    }
}
```

---

## Array vs Dictionary for String Lookups

| String Length | Crossover Point |
|---------------|-----------------|
| Short (< 10 chars) | ~6-10 elements |
| Long (> 50 chars) | ~3-5 elements |

For tiny sets of known strings, array scan can beat dictionary:

```csharp
private static readonly string[] ValidTypes = { "int", "string", "bool" };

// May be faster than HashSet for 3 elements
bool isValid = Array.IndexOf(ValidTypes, type) >= 0;
```
