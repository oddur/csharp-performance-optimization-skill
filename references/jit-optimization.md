# JIT Optimization

## Inlining

### AggressiveInlining

For small, hot methods:

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static int Max(int a, int b) => a > b ? a : b;
```

**When to use:**
- Very small methods (few lines)
- Called in tight loops
- JIT isn't inlining automatically

**When NOT to use:**
- Large methods (hurts instruction cache)
- Methods that throw (JIT won't inline anyway)
- Virtual methods (can't inline without devirtualization)

### NoInlining

Prevent inlining (rare):

```csharp
[MethodImpl(MethodImplOptions.NoInlining)]
public void RarelyCalledSetup() { }
```

---

## Throw Helpers

JIT won't inline methods that throw. Move throws to helpers:

```csharp
// Bad - won't be inlined
public int GetValue(int index)
{
    if ((uint)index >= (uint)_length)
        throw new ArgumentOutOfRangeException(nameof(index));
    return _array[index];
}

// Good - can be inlined
public int GetValue(int index)
{
    if ((uint)index >= (uint)_length)
        ThrowIndexOutOfRange(index);
    return _array[index];
}

[DoesNotReturn]
private static void ThrowIndexOutOfRange(int index)
{
    throw new ArgumentOutOfRangeException(nameof(index), index, "Index out of range.");
}
```

Use `[DoesNotReturn]` attribute to tell compiler the method never returns.

---

## Bounds Check Elimination

### Patterns JIT Recognizes

```csharp
// JIT eliminates bounds checks
for (int i = 0; i < array.Length; i++)
{
    Process(array[i]);  // JIT knows i < Length
}

// Span iteration - no bounds checks
foreach (var item in array.AsSpan())
{
    Process(item);
}
```

### Patterns That Prevent Elimination

```csharp
// Conditional access may prevent elimination
for (int i = 0; i < array.Length; i++)
{
    if (SomeCondition())
        Process(array[i]);  // JIT may not eliminate
}
```

### uint Cast Trick

Combine lower and upper bound check:

```csharp
// Checks i >= 0 AND i < length in one comparison
if ((uint)index < (uint)array.Length)
{
    // bounds check eliminated
    return array[index];
}
```

---

## SIMD and Vectorization

### Automatic Vectorization

JIT vectorizes some operations:

```csharp
ReadOnlySpan<byte> span = data;
bool found = span.Contains((byte)42);      // Vectorized
int index = span.IndexOf((byte)42);        // Vectorized
bool equal = span1.SequenceEqual(span2);   // Vectorized
```

### Manual Vectorization

```csharp
using System.Runtime.Intrinsics;

public static int SumVectorized(ReadOnlySpan<int> values)
{
    int sum = 0;
    int i = 0;

    if (Vector256.IsHardwareAccelerated && values.Length >= Vector256<int>.Count)
    {
        var vsum = Vector256<int>.Zero;
        int lastBlock = values.Length - (values.Length % Vector256<int>.Count);

        for (; i < lastBlock; i += Vector256<int>.Count)
        {
            var v = Vector256.Create(values.Slice(i, Vector256<int>.Count));
            vsum = Vector256.Add(vsum, v);
        }
        sum = Vector256.Sum(vsum);
    }

    // Scalar remainder
    for (; i < values.Length; i++)
        sum += values[i];

    return sum;
}
```

### Check Hardware Support

```csharp
if (Vector256.IsHardwareAccelerated)
{
    // Use 256-bit vectors
}
else if (Vector128.IsHardwareAccelerated)
{
    // Fall back to 128-bit
}
else
{
    // Scalar fallback
}
```

---

## Devirtualization

### How Sealing Helps

```csharp
// Virtual call - vtable lookup
public class Handler { public virtual void Process() { } }

// Direct call - JIT can inline
public sealed class FastHandler : Handler
{
    public sealed override void Process() { }
}
```

### Verify with Disassembly

```csharp
[DisassemblyDiagnoser]
public class SealedBenchmarks
{
    // Check generated assembly for call vs callvirt
}
```

---

## Hot/Cold Code Separation

Keep hot path linear, move cold code out:

```csharp
public void Process(Data data)
{
    // Hot path - common case
    if (data.IsValid)
    {
        DoFastPath(data);
        return;
    }
    
    // Cold path - separate method
    HandleInvalidData(data);
}

[MethodImpl(MethodImplOptions.NoInlining)]
private void HandleInvalidData(Data data)
{
    // Error handling, logging, etc.
}
```
