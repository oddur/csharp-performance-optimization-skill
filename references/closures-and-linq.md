# Closures and LINQ Optimization

## Why Closures Are Expensive

- Compiler generates a hidden class to hold captured variables
- Each invocation allocates a new instance on the heap
- Creates GC pressure in hot paths

## Recognizing Closures

Any lambda referencing outer-scope variables:

```csharp
int threshold = 100;
var filtered = items.Where(x => x.Value > threshold);  // Captures 'threshold'

var mapped = items.Select(x => x.Value + this.Offset);  // Captures 'this'
```

## Avoiding Closures

### Static Lambdas (.NET 5+)

Compiler error if you accidentally capture:

```csharp
items.Where(static x => x.Value > 100);  // Safe - no captures possible
```

### Replace with Loops

```csharp
// Bad - closure over 'playerId'
var player = players.FirstOrDefault(p => p.Id == playerId);

// Good - no allocation
Player? player = null;
foreach (var p in players)
{
    if (p.Id == playerId)
    {
        player = p;
        break;
    }
}
```

### Cache Non-Capturing Delegates

```csharp
// Bad - new delegate each call
events.Subscribe(e => HandleEvent(e));

// Good - cached delegate
private static readonly Action<Event> CachedHandler = HandleEvent;
events.Subscribe(CachedHandler);
```

### Avoid Capturing `this`

```csharp
// Bad - captures 'this', prevents GC
timer.Elapsed += (s, e) => this.DoWork();

// Good - explicit method
timer.Elapsed += OnTimerElapsed;
```

### Async Method Captures

Each captured variable extends state machine allocation:

```csharp
async Task ProcessAsync(int id)
{
    var context = GetContext();  // Captured in state machine
    await Task.Delay(100);
    Use(context);
}
```

## When Closures Are Acceptable

- Not in a hot path
- Readability outweighs allocation cost
- Lambda created once and reused (static field)

---

## LINQ in Hot Paths

### Why LINQ Allocates

- Iterator objects for each method
- Delegate objects for predicates
- Potential boxing for value types

### Common Offenders

Replace these in hot paths:

| LINQ | Loop Alternative |
|------|------------------|
| `.Any(predicate)` | `foreach` with early return |
| `.First(predicate)` | `foreach` with early return |
| `.Where().ToList()` | `foreach` with `Add()` |
| `.Select().ToArray()` | Pre-sized array + loop |
| `.Count(predicate)` | `foreach` with counter |

### Examples

**Any:**

```csharp
// Bad
bool hasMatch = items.Any(x => x.Id == targetId);

// Good
bool hasMatch = false;
foreach (var item in items)
{
    if (item.Id == targetId) { hasMatch = true; break; }
}
```

**First:**

```csharp
// Bad
var match = items.FirstOrDefault(x => x.Id == targetId);

// Good
Item? match = null;
foreach (var item in items)
{
    if (item.Id == targetId) { match = item; break; }
}
```

**Select + ToArray:**

```csharp
// Bad
var ids = items.Select(x => x.Id).ToArray();

// Good
var ids = new int[items.Count];
for (int i = 0; i < items.Count; i++)
    ids[i] = items[i].Id;
```

### Non-Allocating Alternatives

```csharp
// Span.Contains - vectorized
ReadOnlySpan<int> span = array;
bool found = span.Contains(42);

// Array.IndexOf - optimized
int index = Array.IndexOf(array, value);

// List.Find - still allocates delegate, but faster than LINQ
var item = list.Find(x => x.Id == id);
```

## When LINQ is Fine

- Cold paths (startup, configuration)
- Readability is priority
- Small collections with infrequent access
- Prototyping / non-production code
