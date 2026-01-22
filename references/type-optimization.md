# Type Optimization

## Structs vs Classes

### When to Use Structs

- Size ≤16 bytes (measure for your case)
- Short-lived, method-scoped
- Represents a single value or small group of related values
- Stored in arrays frequently (contiguous memory layout)
- Want to avoid GC pressure

### When NOT to Use Structs

- Large types (copying cost > allocation cost)
- Needs to be mutated in multiple places
- Requires inheritance
- Reference semantics by design

### Readonly Structs

Prevents defensive copies:

```csharp
public readonly struct PlayerInput
{
    public readonly int PlayerId;
    public readonly float MoveX;
    public readonly float MoveY;
    
    public PlayerInput(int id, float x, float y) => (PlayerId, MoveX, MoveY) = (id, x, y);
}
```

---

## Ref, In, and Ref Readonly

### `in` - Large readonly struct parameters

```csharp
// Bad - copies 64 bytes
public float Determinant(Matrix4x4 matrix) { }

// Good - passes by reference
public float Determinant(in Matrix4x4 matrix) { }
```

### `ref` - Mutate the original

```csharp
public void ApplyDamage(ref PlayerState state, int damage)
{
    state.Health -= damage;
}
```

### `ref readonly` - Return reference without mutation

```csharp
public ref readonly Vector3 GetPosition(int entityId)
{
    return ref _positions[entityId];
}
```

### Ref locals - Avoid repeated indexing

```csharp
// Bad - indexes twice
entities[i].Position = newPos;
entities[i].Velocity = newVel;

// Good - single index
ref var entity = ref entities[i];
entity.Position = newPos;
entity.Velocity = newVel;
```

### Defensive Copy Warning

Non-readonly structs on readonly fields create copies:

```csharp
public struct MutablePoint { public int X; public void Move() => X++; }

readonly MutablePoint _point;

void Example()
{
    _point.Move();  // Creates defensive copy! X doesn't change.
}
// Fix: Make struct readonly, or don't use readonly field
```

---

## Ref Structs

Stack-only types that can wrap stackalloc or native memory.

### Constraints

Cannot:
- Be boxed to `object` or interfaces
- Be array elements
- Be fields in classes/non-ref structs
- Be captured in lambdas
- Be used in async methods or iterators
- Implement interfaces (before C# 13)

### Example: SpanReader

```csharp
public ref struct SpanReader
{
    private ReadOnlySpan<byte> _buffer;
    private int _position;

    public SpanReader(ReadOnlySpan<byte> buffer) => (_buffer, _position) = (buffer, 0);

    public byte ReadByte() => _buffer[_position++];

    public ReadOnlySpan<byte> ReadBytes(int count)
    {
        var slice = _buffer.Slice(_position, count);
        _position += count;
        return slice;
    }
}
```

### Scoped keyword (.NET 7+)

Guarantees span won't escape:

```csharp
void Process(scoped ReadOnlySpan<byte> data)
{
    // Compiler ensures 'data' doesn't get stored anywhere
}
```

---

## Sealed Classes

### Why Sealing Improves Performance

- JIT devirtualizes method calls → direct calls
- Enables method inlining
- Eliminates vtable lookups
- Better type check assumptions

### When to Seal

```csharp
// Internal implementation - seal it
internal sealed class ConnectionPool { }

// Private nested class - seal it
private sealed class CacheEntry { }

// Leaf in inheritance hierarchy - seal it
public sealed class FastHandler : BaseHandler { }

// Seal specific overrides
public class Derived : Base
{
    public sealed override void Process() { }  // Prevents further overrides
}

// Records - seal when not inheriting
public sealed record PlayerEvent(int PlayerId, string Action);
```

### When NOT to Seal

- Public API designed for extension
- Library classes consumers may inherit
- Intended base classes

---

## Boxing Avoidance

### Common Boxing Scenarios

**Interface calls on structs:**

```csharp
IComparable<MyStruct> boxed = new MyStruct();  // Boxing!
void Process(IComparable<MyStruct> item) { }
Process(new MyStruct());  // Boxing!
```

**Non-generic collections:**

```csharp
ArrayList list = new();
list.Add(42);  // Boxing!
```

**String concatenation:**

```csharp
int count = 5;
string s = "Count: " + count;  // Boxing!
// Fix:
string s = $"Count: {count}";  // No boxing (.NET 6+)
```

**Non-overridden object methods:**

```csharp
struct Point { public int X, Y; }
var p = new Point();
p.ToString();  // Boxing! (calls object.ToString)
```

### Avoiding Boxing

**Override object methods on structs:**

```csharp
public readonly struct Point : IEquatable<Point>
{
    public readonly int X, Y;
    
    public override string ToString() => $"({X}, {Y})";
    public override int GetHashCode() => HashCode.Combine(X, Y);
    public override bool Equals(object? obj) => obj is Point p && Equals(p);
    public bool Equals(Point other) => X == other.X && Y == other.Y;
}
```

**Use generic constraints:**

```csharp
// Bad - boxes structs
void Process(IComparable item) { }

// Good - no boxing
void Process<T>(T item) where T : IComparable<T> { }
```

**Use typed comparers:**

```csharp
// Bad
bool equal = object.Equals(a, b);  // Boxing!

// Good
bool equal = EqualityComparer<T>.Default.Equals(a, b);
int cmp = Comparer<T>.Default.Compare(a, b);
```

**Enum flags:**

```csharp
[Flags] enum Perms { Read = 1, Write = 2 }
var p = Perms.Read | Perms.Write;

// Pre-.NET Core 2.1 - boxes
bool can = p.HasFlag(Perms.Read);

// Always fast
bool can = (p & Perms.Read) != 0;
```

**Generic math (.NET 7+):**

```csharp
T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum;
}
```
