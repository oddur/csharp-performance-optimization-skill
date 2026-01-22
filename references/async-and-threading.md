# Async and Threading Optimization

## ValueTask<T>

Use when async methods often complete synchronously.

### When to Use

- Methods that frequently return cached/computed results
- Hot async paths where Task allocation matters

### Example

```csharp
public ValueTask<Data> GetDataAsync(int id)
{
    // Fast path - no allocation
    if (_cache.TryGetValue(id, out var data))
        return new ValueTask<Data>(data);

    // Slow path - actual async
    return new ValueTask<Data>(LoadDataAsync(id));
}
```

### Rules

- Never await the same ValueTask twice
- Don't store and reuse ValueTask
- Use `.AsTask()` if you need to store it

---

## ConfigureAwait(false)

### Use in Library Code

Avoids capturing synchronization context:

```csharp
public async Task<Data> FetchDataAsync()
{
    var response = await _client.GetAsync(url).ConfigureAwait(false);
    var content = await response.Content.ReadAsStringAsync().ConfigureAwait(false);
    return Parse(content);
}
```

### When to Use

- Library code that doesn't need UI context
- Backend services
- Any code that doesn't need to resume on original context

### When NOT to Use

- UI code that updates controls after await
- Code that depends on synchronization context

---

## Lock Selection

| Scenario | Choice |
|----------|--------|
| Simple exclusive access, sync | `lock` |
| Need `await` inside critical section | `SemaphoreSlim(1, 1)` |
| Throttling (max N concurrent) | `SemaphoreSlim(N, N)` |
| Read-heavy, infrequent writes, sync | `ReaderWriterLockSlim` |
| Read-heavy + async | `AsyncReaderWriterLock` (Nito.AsyncEx) |

### lock

```csharp
private readonly object _lock = new();

public void Update()
{
    lock (_lock)
    {
        // exclusive access
    }
}
```

### SemaphoreSlim for Async

```csharp
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task UpdateAsync()
{
    await _semaphore.WaitAsync();
    try
    {
        // exclusive access with await allowed
        await DoWorkAsync();
    }
    finally
    {
        _semaphore.Release();
    }
}
```

### SemaphoreSlim for Throttling

```csharp
// Max 10 concurrent database calls
private readonly SemaphoreSlim _dbThrottle = new(10, 10);

public async Task<Data> QueryAsync()
{
    await _dbThrottle.WaitAsync();
    try
    {
        return await _db.QueryAsync();
    }
    finally
    {
        _dbThrottle.Release();
    }
}
```

### ReaderWriterLockSlim

```csharp
private readonly ReaderWriterLockSlim _rwLock = new();

public Data Read()
{
    _rwLock.EnterReadLock();
    try
    {
        return _data;  // Multiple readers allowed
    }
    finally
    {
        _rwLock.ExitReadLock();
    }
}

public void Write(Data data)
{
    _rwLock.EnterWriteLock();
    try
    {
        _data = data;  // Exclusive access
    }
    finally
    {
        _rwLock.ExitWriteLock();
    }
}
```

---

## Thread-Local Storage

| Type | Use Case |
|------|----------|
| `[ThreadStatic]` | Simple static field per thread |
| `ThreadLocal<T>` | Per-thread with initialization |
| `AsyncLocal<T>` | Flows across async/await |

### ThreadStatic

```csharp
[ThreadStatic]
private static StringBuilder? _builder;

public static StringBuilder GetBuilder()
{
    return _builder ??= new StringBuilder();
}
```

### AsyncLocal

For context that flows across async calls:

```csharp
private static readonly AsyncLocal<RequestContext> _context = new();

public static RequestContext? Current
{
    get => _context.Value;
    set => _context.Value = value;
}
```

---

## Async State Machine Optimization

### Minimize Captured Variables

Each captured variable extends state machine size:

```csharp
// More captures = larger state machine
async Task ProcessAsync(int id)
{
    var a = GetA();  // captured
    var b = GetB();  // captured
    var c = GetC();  // captured
    await Task.Delay(100);
    Use(a, b, c);
}

// Better - compute after await if possible
async Task ProcessAsync(int id)
{
    await Task.Delay(100);
    var a = GetA();  // not captured
    Use(a);
}
```

### Avoid Async When Synchronous

```csharp
// Unnecessary async overhead
public async Task<int> GetValueAsync()
{
    return 42;  // Always synchronous
}

// Better - return completed task or use ValueTask
public ValueTask<int> GetValueAsync()
{
    return new ValueTask<int>(42);
}
```
