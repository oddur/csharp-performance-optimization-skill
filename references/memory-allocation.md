# Memory Allocation Optimization

## Preallocate Collection Sizes

```csharp
// Bad - resizes multiple times
var list = new List<int>();

// Good - single allocation
var list = new List<int>(expectedCount);

// Good - with growth buffer
var list = new List<int>((int)(expectedCount * 1.2));
```

## Lazy Initialization

Defer allocation until needed:

```csharp
private List<Error>? _errors;

public void AddError(Error error)
{
    _errors ??= new List<Error>();
    _errors.Add(error);
}
```

## ArrayPool<T>

For temporary arrays:

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
try
{
    // use buffer (may be larger than requested)
    ProcessData(buffer.AsSpan(0, actualLength));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

## ObjectPool<T>

For complex reusable objects:

```csharp
private static readonly ObjectPool<StringBuilder> Pool =
    new DefaultObjectPoolProvider().CreateStringBuilderPool();

public string BuildMessage()
{
    var sb = Pool.Get();
    try
    {
        sb.Append("Header: ");
        // ... build string
        return sb.ToString();
    }
    finally
    {
        Pool.Return(sb);
    }
}
```

## Stackalloc

For small, method-scoped buffers:

```csharp
// Simple case - known small size
Span<byte> buffer = stackalloc byte[256];

// With fallback for larger sizes
const int StackThreshold = 1024;
byte[]? pooled = null;
Span<byte> buffer = length <= StackThreshold
    ? stackalloc byte[length]
    : (pooled = ArrayPool<byte>.Shared.Rent(length));
try
{
    // use buffer
}
finally
{
    if (pooled != null)
        ArrayPool<byte>.Shared.Return(pooled);
}
```

**Constraints:**
- Keep under ~1KB to avoid stack overflow
- Lifetime must not exceed method scope
- Cannot be used in async methods or iterators

## Span<T> and Memory<T>

### Span<T> - Stack-only views

```csharp
void ProcessData(ReadOnlySpan<byte> data)
{
    // Works with any contiguous memory source
}

// All work without allocation:
ProcessData(stackalloc byte[64]);
ProcessData(myArray);
ProcessData(myArray.AsSpan(10, 20));  // Slice
ProcessData(new Span<byte>(nativePtr, length));
```

### Memory<T> - Heap-storable views

Use when you need to store a view or use in async:

```csharp
class BufferHolder
{
    private Memory<byte> _buffer;  // Can store Memory, not Span
    
    public async Task ProcessAsync(Memory<byte> data)
    {
        _buffer = data;  // OK
        await Task.Delay(100);
        Process(_buffer.Span);  // Access Span when needed
    }
}
```

## IReadOnlyList<T> for Parameters

Communicates read-only intent without copying:

```csharp
void ProcessItems(IReadOnlyList<int> items)
{
    for (int i = 0; i < items.Count; i++)
        Console.WriteLine(items[i]);  // O(1) indexed access
}

int[] array = { 1, 2, 3 };
ProcessItems(array);  // No copy - arrays implement IReadOnlyList<T>

List<int> list = new() { 1, 2, 3 };
ProcessItems(list);   // No copy
```

**For hot paths, prefer ReadOnlySpan<T>** — avoids interface dispatch overhead.

## IDisposable

Forward Dispose to owned disposables:

```csharp
public class ResourceHolder : IDisposable
{
    private readonly Stream _stream;
    private readonly DbConnection _connection;
    private bool _disposed;

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        
        _stream?.Dispose();
        _connection?.Dispose();
    }
}
```

**Use `using` statements:**

```csharp
using var stream = File.OpenRead(path);
using var reader = new StreamReader(stream);
// Automatically disposed at end of scope
```

## Disposable Ref Struct Pattern

For pooled memory with automatic cleanup:

```csharp
public ref struct RentedBuffer<T>
{
    private T[]? _array;
    private readonly int _length;

    public RentedBuffer(int length)
    {
        _array = ArrayPool<T>.Shared.Rent(length);
        _length = length;
    }

    public Span<T> Span => _array.AsSpan(0, _length);

    public void Dispose()
    {
        if (_array != null)
        {
            ArrayPool<T>.Shared.Return(_array);
            _array = null;
        }
    }
}

// Usage
using var buffer = new RentedBuffer<byte>(1024);
Process(buffer.Span);
```
