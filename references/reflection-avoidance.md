# Reflection Avoidance

## Why Reflection is Expensive

- Runtime type lookups
- No JIT optimization or inlining
- Allocates `MethodInfo`, `PropertyInfo`, etc.
- Parameter arrays for `Invoke`
- Boxing for value type arguments

---

## Cache Reflection Results

Never call `GetMethod`/`GetProperty` repeatedly:

```csharp
// Bad - lookup every call
public object GetValue(object obj, string propertyName)
{
    return obj.GetType().GetProperty(propertyName)?.GetValue(obj);
}

// Good - cached lookup
private static readonly ConcurrentDictionary<(Type, string), PropertyInfo?> _cache = new();

public object? GetValue(object obj, string propertyName)
{
    var key = (obj.GetType(), propertyName);
    var prop = _cache.GetOrAdd(key, k => k.Item1.GetProperty(k.Item2));
    return prop?.GetValue(obj);
}
```

---

## Compiled Delegates

Replace `MethodInfo.Invoke` with delegates:

```csharp
// Bad - slow reflection
MethodInfo method = typeof(MyClass).GetMethod("Process")!;
object result = method.Invoke(instance, new object[] { arg1, arg2 });

// Good - compiled delegate
private static readonly Func<MyClass, int, string, object> _process;

static MyClass()
{
    var method = typeof(MyClass).GetMethod("Process")!;
    _process = (Func<MyClass, int, string, object>)
        Delegate.CreateDelegate(typeof(Func<MyClass, int, string, object>), method);
}

// Fast invocation
object result = _process(instance, arg1, arg2);
```

---

## Expression Trees

Compile property accessors:

```csharp
public static class PropertyAccessor<T, TValue>
{
    public static Func<T, TValue> CreateGetter(string propertyName)
    {
        var param = Expression.Parameter(typeof(T), "obj");
        var property = Expression.Property(param, propertyName);
        return Expression.Lambda<Func<T, TValue>>(property, param).Compile();
    }

    public static Action<T, TValue> CreateSetter(string propertyName)
    {
        var param = Expression.Parameter(typeof(T), "obj");
        var value = Expression.Parameter(typeof(TValue), "value");
        var property = Expression.Property(param, propertyName);
        var assign = Expression.Assign(property, value);
        return Expression.Lambda<Action<T, TValue>>(assign, param, value).Compile();
    }
}

// Create once, use many times
var getName = PropertyAccessor<Person, string>.CreateGetter("Name");
var setName = PropertyAccessor<Person, string>.CreateSetter("Name");

string name = getName(person);  // Fast!
setName(person, "New");         // Fast!
```

---

## Source Generators

Prefer compile-time code generation.

### JSON Serialization

```csharp
[JsonSerializable(typeof(Person))]
[JsonSerializable(typeof(List<Person>))]
public partial class MyJsonContext : JsonSerializerContext { }

// Fast - no reflection
string json = JsonSerializer.Serialize(person, MyJsonContext.Default.Person);
Person? p = JsonSerializer.Deserialize(json, MyJsonContext.Default.Person);
```

### Regex (.NET 7+)

```csharp
// Bad - runtime compilation
Regex regex = new(@"\d{3}-\d{4}", RegexOptions.Compiled);

// Good - build-time compilation
[GeneratedRegex(@"\d{3}-\d{4}")]
private static partial Regex PhoneRegex();

bool match = PhoneRegex().IsMatch("555-1234");
```

### Logging

```csharp
// Bad - reflection and boxing
_logger.LogInformation("Player {Id} scored {Points}", playerId, points);

// Good - source generated
[LoggerMessage(Level = LogLevel.Information, Message = "Player {Id} scored {Points}")]
public static partial void LogPlayerScored(ILogger logger, int id, int points);

LogPlayerScored(_logger, playerId, points);  // No boxing!
```

---

## Generic Type Caching

Cache per-type data in generic static fields:

```csharp
public static class TypeCache<T>
{
    public static readonly int Size = Marshal.SizeOf<T>();
    public static readonly bool IsValueType = typeof(T).IsValueType;
    public static readonly string Name = typeof(T).Name;
}

// No reflection after first access per T
int size = TypeCache<MyStruct>.Size;
```

---

## Activator.CreateInstance Alternatives

```csharp
// Bad - reflection
var obj = Activator.CreateInstance(type);

// Better - generic constraint
T Create<T>() where T : new() => new T();

// Better - cached compiled expression
private static readonly ConcurrentDictionary<Type, Func<object>> _factories = new();

public static object Create(Type type)
{
    return _factories.GetOrAdd(type, t =>
    {
        var ctor = Expression.New(t);
        return Expression.Lambda<Func<object>>(ctor).Compile();
    })();
}
```

---

## Detection

Look for these patterns in code:

- `GetMethod`, `GetProperty`, `GetType().Get*`
- `MethodInfo.Invoke`, `PropertyInfo.GetValue/SetValue`
- `Activator.CreateInstance`
- `dynamic` keyword (compiles to reflection)
