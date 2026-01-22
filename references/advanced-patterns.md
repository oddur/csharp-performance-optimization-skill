# Advanced Performance Patterns

Advanced patterns from high-performance .NET codebases (e.g., Microsoft Garnet) for scenarios requiring maximum throughput and minimal latency.

---

## Decision Flowcharts

### Memory Management Decision Tree

```
Need to manage memory?
│
├─► Temporary buffer for single operation?
│   ├─► Size known at compile time & < 1KB?
│   │   └─► Use: stackalloc + Span<T>
│   ├─► Size known at runtime & < 64KB?
│   │   └─► Use: ArrayPool<T>.Shared.Rent()
│   └─► Size > 64KB or variable?
│       └─► Use: MemoryPool<T>.Shared.Rent()
│
├─► Reusing objects across operations?
│   ├─► Simple object, single type?
│   │   └─► Use: ObjectPool<T> or ConcurrentStack<T>
│   ├─► Multiple buffer sizes?
│   │   └─► Use: Multi-level pool (power-of-2 sizing)
│   └─► Expensive to create, frequently used?
│       └─► Use: Lazy<T> singleton + pool
│
└─► Zero-copy data access?
    ├─► Read-only access?
    │   └─► Use: ReadOnlySpan<T> or ReadOnlyMemory<T>
    └─► Need to modify?
        └─► Use: Span<T> or Memory<T>
```

### Concurrency Pattern Decision Tree

```
Need thread-safe operations?
│
├─► Simple counter or flag?
│   └─► Use: Interlocked operations
│
├─► Collection access pattern?
│   ├─► Many readers, few writers?
│   │   └─► Use: ReaderWriterLockSlim or custom SWMR lock
│   ├─► Equal reads and writes?
│   │   └─► Use: ConcurrentDictionary<K,V>
│   └─► Queue/stack pattern?
│       └─► Use: ConcurrentQueue<T> or ConcurrentStack<T>
│
├─► Lock-free required?
│   ├─► Compare-and-swap operation?
│   │   └─► Use: Interlocked.CompareExchange
│   └─► Ownership tracking?
│       └─► Use: AtomicOwner pattern (CAS on 64-bit)
│
└─► Async coordination?
    ├─► Wait for single event?
    │   └─► Use: TaskCompletionSource + RunContinuationsAsynchronously
    ├─► Producer/consumer queue?
    │   └─► Use: ConcurrentQueue + SemaphoreSlim
    └─► Wait for counter to reach zero?
        └─► Use: AsyncCountDown with ValueTask
```

### Async Pattern Decision Tree

```
Implementing async method?
│
├─► Fast path completes synchronously most times?
│   └─► Use: ValueTask<T> (zero allocation on sync path)
│
├─► Always async?
│   └─► Use: Task<T>
│
├─► Need to signal completion?
│   ├─► Single waiter?
│   │   └─► Use: TaskCompletionSource
│   └─► Multiple waiters?
│       └─► Use: SemaphoreSlim or ManualResetEventSlim
│
└─► Pooling async resources?
    └─► Use: SemaphoreSlim for availability + ConcurrentQueue for items
```

### Data Structure Decision Tree

```
Choosing a data structure?
│
├─► Fixed-size buffer with wraparound?
│   ├─► Size is power of 2?
│   │   └─► Use: Circular buffer with bitwise AND modulo
│   └─► Arbitrary size?
│       └─► Use: Standard modulo (slower)
│
├─► Hash table requirements?
│   ├─► Need custom equality/hash?
│   │   └─► Use: sealed IEqualityComparer<T> implementation
│   └─► Concurrent access?
│       └─► Use: ConcurrentDictionary with static readonly comparer
│
├─► Compact struct storage?
│   ├─► Need bit-level packing?
│   │   └─► Use: StructLayout(Explicit) with FieldOffset
│   ├─► Need union/overlay?
│   │   └─► Use: StructLayout(Explicit) with overlapping offsets
│   └─► Prevent padding?
│       └─► Use: StructLayout(Sequential, Pack=1)
│
└─► Caching computed values?
    ├─► Per-thread cache?
    │   └─► Use: [ThreadStatic] or ThreadLocal<T>
    ├─► Global cache with invalidation?
    │   └─► Use: WeakReference for entries
    └─► Lazy initialization?
        └─► Use: Lazy<T>
```

### Performance Optimization Decision Tree

```
Where's the bottleneck?
│
├─► Method call overhead?
│   ├─► Hot path method?
│   │   └─► Add: [MethodImpl(AggressiveInlining)]
│   ├─► Virtual dispatch?
│   │   └─► Use: sealed class or struct generic parameter
│   └─► Interface dispatch?
│       └─► Use: struct with interface constraint
│
├─► Allocation pressure (GC)?
│   ├─► Temporary objects?
│   │   └─► Use: stackalloc, Span<T>, object pooling
│   ├─► String operations?
│   │   └─► Use: ReadOnlySpan<char>, stackalloc char[]
│   └─► Boxing value types?
│       └─► Use: generic constraints, avoid object parameters
│
├─► Branch misprediction?
│   ├─► Common case vs rare case?
│   │   └─► Put common case first, use early returns
│   ├─► Switch with many cases?
│   │   └─► Use: bitwise flags + masks instead
│   └─► Null checks?
│       └─► Use: [DisallowNull], [NotNull] attributes
│
├─► Loop overhead?
│   ├─► Known iteration count?
│   │   └─► Help JIT eliminate bounds checks
│   ├─► Bulk byte operations?
│   │   └─► Use: Vector<T>, TensorPrimitives
│   └─► String/span comparison?
│       └─► Use: SequenceEqual, word-level comparison
│
└─► I/O bound?
    ├─► Batching possible?
    │   └─► Use: Batch operations, scatter-gather
    ├─► Buffer reuse?
    │   └─► Use: ArrayPool, pre-allocated buffers
    └─► Async?
        └─► Use: ValueTask for sync-completing paths
```

### Error Handling Decision Tree

```
Handling errors?
│
├─► Expected failure (not exceptional)?
│   ├─► Simple success/failure?
│   │   └─► Use: bool return with out parameter
│   ├─► Need failure details?
│   │   └─► Use: bool return with multiple out params
│   └─► Need result or error?
│       └─► Use: struct with IsSuccess flag
│
├─► Exceptional failure?
│   ├─► Helper method for throwing?
│   │   └─► Add: [DoesNotReturn] attribute
│   └─► Cold path?
│       └─► Add: [MethodImpl(NoInlining)] to throw helper
│
└─► Validation?
    ├─► Debug-only?
    │   └─► Use: Debug.Assert or [Conditional("DEBUG")]
    └─► Production validation?
        └─► Use: if + ThrowHelper pattern
```

---

## Pattern Selection Quick Reference

| Symptom | Likely Pattern |
|---------|----------------|
| High allocation rate | Object pooling, Span<T>, stackalloc |
| GC pauses | ArrayPool, reduce allocations, struct instead of class |
| Lock contention | Interlocked, SWMR lock, ConcurrentDictionary |
| Virtual call overhead | Sealed classes, struct generics |
| Cache misses | Power-of-2 sizing, StructLayout for alignment |
| Slow string ops | ReadOnlySpan<byte>, u8 literals |
| Branch misprediction | Bitwise flags, early returns for common case |
| Method call overhead | AggressiveInlining on hot paths |
| Async overhead | ValueTask for sync-completing paths |
| Loop bounds checks | Fixed-size iterations, span slicing |

---

## Multi-Level Buffer Pools

Power-of-2 sized buffer pools using concurrent queues:

```csharp
public class BufferPool
{
    private readonly ConcurrentQueue<byte[]>[] pools = new ConcurrentQueue<byte[]>[32];
    private int totalReferences;

    public BufferPool()
    {
        for (int i = 0; i < pools.Length; i++)
            pools[i] = new ConcurrentQueue<byte[]>();
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public byte[] Rent(int size)
    {
        int level = BitOperations.Log2((uint)size);
        if (pools[level].TryDequeue(out var buffer))
        {
            Interlocked.Increment(ref totalReferences);
            return buffer;
        }
        return new byte[1 << level];
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Return(byte[] buffer)
    {
        int level = BitOperations.Log2((uint)buffer.Length);
        pools[level].Enqueue(buffer);
        Interlocked.Decrement(ref totalReferences);
    }
}
```

---

## Try-Before-Realloc Pattern

Optimistic write with fallback for buffer management:

```csharp
public ref struct ResponseWriter
{
    private Span<byte> buffer;
    private int position;

    public void WriteString(string value)
    {
        while (!TryWriteString(value))
            Reallocate(value.Length * 2);
    }

    private bool TryWriteString(string value)
    {
        int required = value.Length + 5;  // +overhead
        if (buffer.Length - position < required)
            return false;

        // Write directly to buffer
        buffer[position++] = (byte)'$';
        // ... write length and content ...
        return true;
    }

    private void Reallocate(int extraHint)
    {
        int newSize = (int)BitOperations.RoundUpToPowerOf2(
            (uint)(buffer.Length + extraHint));
        var newBuffer = new byte[newSize];
        buffer.Slice(0, position).CopyTo(newBuffer);
        buffer = newBuffer;
    }
}
```

---

## ReadOnlySpan<byte> u8 Literals

Protocol constants as static data (no heap allocation):

```csharp
public static class CmdStrings
{
    // Compiled as static data - no heap allocation
    public static ReadOnlySpan<byte> OK => "OK"u8;
    public static ReadOnlySpan<byte> PONG => "PONG"u8;
    public static ReadOnlySpan<byte> QUEUED => "QUEUED"u8;
    public static ReadOnlySpan<byte> NIL => "$-1\r\n"u8;
    public static ReadOnlySpan<byte> CRLF => "\r\n"u8;
}

// Usage - zero allocation comparison
public static bool TryParseCommand(ReadOnlySpan<byte> input, out Command cmd)
{
    if (input.SequenceEqual("PING"u8))
    {
        cmd = Command.Ping;
        return true;
    }
    cmd = default;
    return false;
}
```

---

## Bitwise Status Codes

Replace switch statements with bitwise operations:

```csharp
[Flags]
public enum StatusCode : byte
{
    Found = 0,
    NotFound = 1,
    Pending = 2,
    Error = 3,
    Expired = 0x10,

    BasicMask = 0x0F
}

public readonly struct Status
{
    private readonly StatusCode code;

    // Bitwise operations instead of switch statements
    public bool Found => (code & StatusCode.BasicMask) == StatusCode.Found;
    public bool NotFound => (code & StatusCode.BasicMask) == StatusCode.NotFound;
    public bool IsPending => code == StatusCode.Pending;
    public bool IsExpired => (code & StatusCode.Expired) != 0;
}
```

---

## Circular Buffer with Bitwise Modulo

Power-of-2 sizing enables O(1) wraparound with bitwise AND:

```csharp
public sealed class CircularBuffer<T>
{
    private const int Capacity = 0xFFF;  // 4095 = 2^12 - 1
    private readonly T[] items = new T[Capacity + 1];
    private int head, tail;

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Enqueue(T value)
    {
        int next = (tail + 1) & Capacity;  // Bitwise AND instead of modulo
        if (next == head)
            throw new InvalidOperationException("Buffer full");
        items[tail] = value;
        tail = next;
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public T Dequeue()
    {
        if (head == tail)
            throw new InvalidOperationException("Buffer empty");
        int oldHead = head;
        head = (head + 1) & Capacity;  // O(1) wraparound
        var item = items[oldHead];
        items[oldHead] = default!;
        return item;
    }

    public int Count => (tail - head) & Capacity;
}
```

---

## Batch Processing (Scatter-Gather)

Process items in batches using pooled buffers:

```csharp
public readonly struct BatchProcessor<T>
{
    public void ProcessBatch(ReadOnlySpan<T> items, Action<T> processor)
    {
        // Rent buffer for results
        var results = ArrayPool<(T item, bool success)>.Shared.Rent(items.Length);
        try
        {
            // Process and gather results
            int completed = 0;
            foreach (var item in items)
            {
                results[completed++] = (item, TryProcess(item, processor));
            }

            // Handle failures
            for (int i = 0; i < completed; i++)
            {
                if (!results[i].success)
                    HandleFailure(results[i].item);
            }
        }
        finally
        {
            ArrayPool<(T, bool)>.Shared.Return(results);
        }
    }

    private bool TryProcess(T item, Action<T> processor)
    {
        try { processor(item); return true; }
        catch { return false; }
    }

    private void HandleFailure(T item) { /* log or retry */ }
}
```

---

## Exception-Free Error Paths

Boolean returns with out parameters for expected failures:

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static bool TryReadInt(ReadOnlySpan<byte> span, out int value)
{
    value = 0;
    if (span.IsEmpty)
        return false;  // No exception for expected failure

    int sign = 1;
    int i = 0;

    if (span[0] == '-')
    {
        sign = -1;
        i = 1;
    }

    for (; i < span.Length; i++)
    {
        int digit = span[i] - '0';
        if ((uint)digit > 9)
            return false;  // Invalid character - no exception
        value = value * 10 + digit;
    }

    value *= sign;
    return true;
}
```

With multiple error details:

```csharp
public static bool TryParse(
    ReadOnlySpan<byte> input,
    out long value,
    out int bytesConsumed,
    out bool overflow)
{
    value = 0;
    bytesConsumed = 0;
    overflow = false;

    // ... parsing logic ...

    if (value > long.MaxValue / 10)
    {
        overflow = true;
        return false;
    }

    return true;
}
```

---

## AsyncQueue Pattern

Producer/consumer queue with async dequeue:

```csharp
public class AsyncQueue<T>
{
    private readonly ConcurrentQueue<T> queue = new();
    private readonly SemaphoreSlim signal = new(0);

    public void Enqueue(T item)
    {
        queue.Enqueue(item);
        signal.Release();
    }

    public async ValueTask<T> DequeueAsync(CancellationToken ct = default)
    {
        await signal.WaitAsync(ct);
        queue.TryDequeue(out var item);
        return item!;
    }

    public bool TryDequeue(out T? item) => queue.TryDequeue(out item);

    public int Count => queue.Count;
}
```

---

## Benchmarking Best Practices

### BenchmarkDotNet Setup

```csharp
[MemoryDiagnoser]  // Track allocations
public class BasicOperations
{
    [ParamsSource(nameof(OperationParamsProvider))]
    public OperationParams Params { get; set; }

    public IEnumerable<OperationParams> OperationParamsProvider()
    {
        yield return new OperationParams { BatchSize = 100 };
        yield return new OperationParams { BatchSize = 1000 };
    }

    [GlobalSetup]
    public void Setup()
    {
        // Initialize resources once before all benchmarks
    }

    [GlobalCleanup]
    public void Cleanup() { }

    [Benchmark]
    public void Operation()
    {
        // Benchmark this method
    }
}
```

### Throughput Measurement

```csharp
public class ThroughputBench
{
    private long totalOperations;
    private volatile bool running = true;
    private readonly Stopwatch stopwatch = new();

    public void Run(int numThreads, int durationMs)
    {
        var threads = new Thread[numThreads];
        var barrier = new Barrier(numThreads);

        for (int i = 0; i < numThreads; i++)
        {
            threads[i] = new Thread(() => Worker(barrier));
            threads[i].Start();
        }

        stopwatch.Start();
        Thread.Sleep(durationMs);
        running = false;

        foreach (var t in threads) t.Join();
        stopwatch.Stop();

        double seconds = stopwatch.ElapsedMilliseconds / 1000.0;
        double opsPerSec = Interlocked.Read(ref totalOperations) / seconds;
        Console.WriteLine($"Throughput: {opsPerSec:N0} ops/sec");
    }

    private void Worker(Barrier barrier)
    {
        barrier.SignalAndWait();  // Synchronized start
        long localOps = 0;

        while (running)
        {
            DoOperation();
            localOps++;
        }

        Interlocked.Add(ref totalOperations, localOps);
    }
}
```

### Latency Percentiles with HdrHistogram

```csharp
using HdrHistogram;

public class LatencyBench
{
    private readonly LongHistogram[] threadHistograms;

    public LatencyBench(int numThreads)
    {
        threadHistograms = new LongHistogram[numThreads];
        for (int i = 0; i < numThreads; i++)
            threadHistograms[i] = new LongHistogram(1, TimeStamp.Seconds(100), 2);
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    private void RecordLatency(int threadId, long elapsedTicks)
    {
        threadHistograms[threadId].RecordValue(elapsedTicks);
    }

    public void PrintPercentiles()
    {
        var combined = new LongHistogram(1, TimeStamp.Seconds(100), 2);
        foreach (var h in threadHistograms)
            combined.Add(h);

        double scale = OutputScalingFactor.TimeStampToMicroseconds;
        Console.WriteLine($"  P50: {combined.GetValueAtPercentile(50) / scale:F2} us");
        Console.WriteLine($"  P95: {combined.GetValueAtPercentile(95) / scale:F2} us");
        Console.WriteLine($"  P99: {combined.GetValueAtPercentile(99) / scale:F2} us");
        Console.WriteLine($"P99.9: {combined.GetValueAtPercentile(99.9) / scale:F2} us");
    }
}
```

### High-Precision Timing

```csharp
// Avoid Stopwatch.Start()/Stop() overhead
long startTimestamp = Stopwatch.GetTimestamp();
DoOperation();
long elapsed = Stopwatch.GetTimestamp() - startTimestamp;
```

### Warmup Pattern

```csharp
public void Run()
{
    // Phase 1: JIT warmup (not measured)
    for (int i = 0; i < 1000; i++)
        DoOperation();

    Thread.Sleep(100);  // Let JIT settle

    // Phase 2: GC stabilization
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();

    // Phase 3: Actual measurement
    var sw = Stopwatch.StartNew();
    for (int i = 0; i < 100000; i++)
        DoOperation();
    sw.Stop();
}
```

---

## Anti-Patterns to Avoid

### Memory Anti-Patterns

```csharp
// BAD: Allocating in hot loops
for (int i = 0; i < 1000000; i++)
{
    var buffer = new byte[1024];  // Allocation every iteration!
    Process(buffer);
}

// GOOD: Rent from pool
var buffer = ArrayPool<byte>.Shared.Rent(1024);
try
{
    for (int i = 0; i < 1000000; i++)
        Process(buffer);
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

```csharp
// BAD: String concatenation in loops
string result = "";
foreach (var item in items)
    result += item.ToString();  // Creates new string each iteration!

// GOOD: Use StringBuilder
var sb = new StringBuilder();
foreach (var item in items)
    sb.Append(item);
string result = sb.ToString();
```

```csharp
// BAD: Boxing value types
void Process(object value) { }
Process(42);  // Boxes int

// GOOD: Use generics
void Process<T>(T value) { }
Process(42);  // No boxing
```

### Async Anti-Patterns

```csharp
// BAD: Task<T> when sync is common
public async Task<int> GetValueAsync()
{
    if (cache.TryGetValue(key, out var value))
        return value;  // Still allocates Task!
    return await LoadAsync();
}

// GOOD: ValueTask for sync fast-path
public ValueTask<int> GetValueAsync()
{
    if (cache.TryGetValue(key, out var value))
        return new ValueTask<int>(value);  // No allocation
    return new ValueTask<int>(LoadAsync());
}
```

```csharp
// BAD: TaskCompletionSource without async flag
var tcs = new TaskCompletionSource<int>();  // Inline continuation risk

// GOOD: Use RunContinuationsAsynchronously
var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
```

### Concurrency Anti-Patterns

```csharp
// BAD: Lock on public object
public readonly object SyncRoot = new();
lock (SyncRoot) { }  // Anyone can deadlock this!

// GOOD: Lock on private object
private readonly object syncRoot = new();
lock (syncRoot) { }
```

```csharp
// BAD: Check then act
if (queue.Count > 0)
{
    var item = queue.Dequeue();  // Race condition!
}

// GOOD: Atomic operations
if (queue.TryDequeue(out var item))
{
    Process(item);
}
```

### Collection Anti-Patterns

```csharp
// BAD: LINQ in hot paths
var first = items.Where(x => x.IsValid).FirstOrDefault();  // Allocates!

// GOOD: Manual loop
Item? first = null;
foreach (var item in items)
{
    if (item.IsValid) { first = item; break; }
}
```

```csharp
// BAD: Dictionary lookup twice
if (dict.ContainsKey(key))
{
    var value = dict[key];  // Two lookups!
}

// GOOD: Single lookup
if (dict.TryGetValue(key, out var value))
{
    Process(value);
}
```

### Exception Anti-Patterns

```csharp
// BAD: Exceptions for control flow
try
{
    var value = dict[key];  // Throws if not found
}
catch (KeyNotFoundException) { }

// GOOD: TryGet pattern
if (dict.TryGetValue(key, out var value))
    Process(value);
```

### Inlining Anti-Patterns

```csharp
// BAD: AggressiveInlining on large methods
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public void LargeMethod()  // Code bloat!
{
    // 100+ lines...
}

// GOOD: Only inline small hot methods
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public int Add(int a, int b) => a + b;
```

```csharp
// BAD: Cold code blocking inlining
public void Process()
{
    if (unlikely)
    {
        // Error handling here prevents inlining
    }
    // Hot path
}

// GOOD: Separate cold path
public void Process()
{
    if (unlikely)
        HandleError();  // Separate method
    // Hot path now inlinable
}

[MethodImpl(MethodImplOptions.NoInlining)]
private void HandleError() { /* cold path */ }
```

### Span Anti-Patterns

```csharp
// BAD: Converting Span to array
void Process(Span<byte> data)
{
    var array = data.ToArray();  // Allocates!
    InternalProcess(array);
}

// GOOD: Keep as Span
void Process(Span<byte> data)
{
    InternalProcess(data);  // No allocation
}
```

---

## Anti-Pattern Quick Reference

| Anti-Pattern | Impact | Fix |
|--------------|--------|-----|
| Allocating in loops | GC pressure | Pool or reuse |
| String concatenation | O(n²) allocations | StringBuilder |
| Boxing value types | Heap allocation | Generics |
| Task<T> for sync paths | Allocation | ValueTask<T> |
| Lock on public object | Deadlock risk | Private lock |
| LINQ in hot paths | Enumerator alloc | Manual loops |
| Double dictionary lookup | 2x hash cost | TryGetValue |
| Exceptions for control flow | 1000x slower | Try* pattern |
| AggressiveInlining on large | Code bloat | Small methods only |
| Span.ToArray() | Allocation | Keep as Span |

---

## Additional Patterns

### SingleWriterMultiReaderLock

```csharp
public struct ReaderWriterLock
{
    private int state;  // Negative = write lock, Positive = reader count

    public bool TryReadLock()
    {
        int current = state;
        if (current >= 0 && Interlocked.CompareExchange(ref state, current + 1, current) == current)
            return true;
        return false;
    }

    public void ReadUnlock() => Interlocked.Decrement(ref state);

    public bool TryWriteLock() =>
        Interlocked.CompareExchange(ref state, int.MinValue, 0) == 0;

    public void WriteUnlock() =>
        Interlocked.CompareExchange(ref state, 0, int.MinValue);
}
```

### Bit Packing for Compact Storage

```csharp
public readonly struct PackedEntry
{
    private readonly long value;

    // Pack multiple values into single long
    public int Address => (int)(value & 0x7FFFFFFF);           // 31 bits
    public ushort Tag => (ushort)((value >> 31) & 0x7FFF);     // 15 bits
    public bool IsValid => (value >> 46) != 0;                  // 1 bit

    public PackedEntry(int address, ushort tag, bool valid) =>
        value = address | ((long)tag << 31) | ((valid ? 1L : 0L) << 46);
}
```

### Cached Enum Parsing

```csharp
public static class EnumUtils
{
    private static readonly Dictionary<Type, Dictionary<string, object>> cache = new();

    public static bool TryParse<T>(string value, out T result) where T : struct, Enum
    {
        var type = typeof(T);
        if (!cache.TryGetValue(type, out var dict))
        {
            dict = new Dictionary<string, object>(StringComparer.OrdinalIgnoreCase);
            foreach (T enumValue in Enum.GetValues<T>())
                dict[enumValue.ToString()] = enumValue;
            cache[type] = dict;
        }

        if (dict.TryGetValue(value, out var obj))
        {
            result = (T)obj;
            return true;
        }
        result = default;
        return false;
    }
}
```

### Struct Layout with FieldOffset

```csharp
[StructLayout(LayoutKind.Explicit, Size = 16)]
public struct CompactEntry
{
    [FieldOffset(0)] public long Key;
    [FieldOffset(8)] public int Value;
    [FieldOffset(12)] public int Flags;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct PackedHeader
{
    public byte Type;      // 1 byte
    public byte Flags;     // 1 byte
    public ushort Length;  // 2 bytes
    // Total: 4 bytes (no padding)
}
```

### DoesNotReturn Attribute

```csharp
public static class ThrowHelper
{
    [DoesNotReturn]
    public static void ThrowArgumentNull(string paramName)
        => throw new ArgumentNullException(paramName);

    [DoesNotReturn]
    public static void ThrowInvalidOperation(string message)
        => throw new InvalidOperationException(message);
}

// Usage - JIT knows code after this won't execute
public int Parse(ReadOnlySpan<byte> input)
{
    if (input.IsEmpty)
        ThrowHelper.ThrowArgumentNull(nameof(input));

    return ParseCore(input);  // JIT optimizes knowing we reach here only if non-empty
}
```

---

## Quick Reference Tables

### When to Use What

| Scenario | Pattern |
|----------|---------|
| Temporary buffers | stackalloc, ArrayPool<T>.Shared |
| Reusable objects | ObjectPool<T>, ConcurrentStack<T> |
| Variable-length data | Memory<T>, IMemoryOwner<T> |
| Zero-copy parsing | ReadOnlySpan<T>, Span<T> |
| Reader-heavy locks | ReaderWriterLockSlim |
| Lock-free counters | Interlocked operations |
| Bulk byte ops | TensorPrimitives, Vector<T> |
| Fast async | ValueTask<T>, pre-allocated TCS |
| Per-thread caching | [ThreadStatic] fields |
| String constants | ReadOnlySpan<byte> with u8 suffix |
| Status checking | Bitwise flags instead of switch |
| Ring buffers | Power-of-2 size with bitwise AND |
| Error handling | Boolean returns, out parameters |
| Buffer writes | Try-before-realloc pattern |

### Key Attributes

| Attribute | Purpose |
|-----------|---------|
| `[MethodImpl(AggressiveInlining)]` | Force JIT inlining |
| `[MethodImpl(NoInlining)]` | Prevent inlining (cold paths) |
| `[DoesNotReturn]` | Mark throw helpers |
| `[StructLayout(Explicit)]` | Precise field layout |
| `[ThreadStatic]` | Thread-local storage |
| `readonly struct` | Prevent defensive copies |
| `ref struct` | Stack-only allocation |

### Benchmarking Tools

| Purpose | Tool/Pattern |
|---------|--------------|
| Micro-benchmarks | BenchmarkDotNet + [MemoryDiagnoser] |
| Latency percentiles | HdrHistogram |
| Throughput | Stopwatch + Interlocked counters |
| High-precision timing | Stopwatch.GetTimestamp() |
| Thread sync | Barrier for synchronized start |
| GC consistency | GC.Collect() between iterations |
| JIT warmup | Run N iterations before measuring |
