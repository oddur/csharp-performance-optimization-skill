# Algorithmic Complexity and Acceleration Structures

## Understand Your Data First

Before optimizing, answer these questions:

- How many items? 10? 1,000? 1,000,000?
- Ratio of reads to writes?
- Ratio of inserts to lookups to deletes?
- How often does this code execute?
- Access patterns: sequential, random, clustered?

**These numbers determine everything:**

```
N = 100 entities, lookup once per frame:
  Linear scan: 100 comparisons/frame → probably fine
  Dictionary: 1 hash + lookup/frame → unnecessary complexity

N = 10,000 entities, lookup 100 times per frame:
  Linear scan: 1,000,000 comparisons/frame → unacceptable
  Dictionary: 100 hash + lookups/frame → necessary
```

## Complexity Mismatches to Fix

| Code Pattern | Hidden Complexity | Fix |
|--------------|-------------------|-----|
| Nested loops over growing collections | O(n × m) | Index one collection |
| `.Contains()` in a loop | O(n × m) | Use HashSet |
| "Find all X near Y" scanning everything | O(n × m) | Spatial partitioning |
| Sorting on every query | O(n log n) per query | Maintain sorted structure |
| Duplicate detection comparing all pairs | O(n²) | HashSet or Bloom filter |
| String concatenation in loop | O(n²) total chars | StringBuilder |

## Complexity Cheat Sheet

| Operation | Array/List | Dictionary/HashSet | SortedSet |
|-----------|------------|-------------------|-----------|
| Index by position | O(1) | N/A | N/A |
| Find by key | O(n) | O(1) | O(log n) |
| Contains | O(n) | O(1) | O(log n) |
| Insert at end | O(1)* | O(1)* | O(log n) |
| Insert at position | O(n) | N/A | N/A |
| Remove by value | O(n) | O(1) | O(log n) |
| Min/Max | O(n) | O(n) | O(1) |

*Amortized — occasional resize is O(n)

---

## Acceleration Structures

### Dictionary Index

Replace linear searches with O(1) lookups:

```csharp
// Bad - O(n) every lookup
Player? FindPlayer(List<Player> players, int id)
{
    foreach (var p in players)
        if (p.Id == id) return p;
    return null;
}

// Good - O(1) lookup
private readonly Dictionary<int, Player> _playersById = new();
Player? FindPlayer(int id) => _playersById.GetValueOrDefault(id);
```

**Multiple indexes for different query patterns:**

```csharp
public class PlayerRegistry
{
    private readonly Dictionary<int, Player> _byId = new();
    private readonly Dictionary<string, Player> _byName = new();
    private readonly Dictionary<int, HashSet<Player>> _byZoneId = new();

    public void Add(Player player)
    {
        _byId[player.Id] = player;
        _byName[player.Name] = player;
        if (!_byZoneId.TryGetValue(player.ZoneId, out var set))
            _byZoneId[player.ZoneId] = set = new HashSet<Player>();
        set.Add(player);
    }
}
```

**FrozenDictionary for static data (.NET 8+):**

```csharp
private static readonly FrozenDictionary<string, ItemDefinition> _itemDefs =
    LoadItemDefinitions().ToFrozenDictionary(x => x.Id);
```

### Spatial Hash (Grid)

For "find nearby" queries:

```csharp
public class SpatialHash<T>
{
    private readonly Dictionary<(int, int), List<T>> _cells = new();
    private readonly Func<T, Vector2> _getPosition;
    private readonly float _cellSize;

    public SpatialHash(float cellSize, Func<T, Vector2> getPosition)
    {
        _cellSize = cellSize;
        _getPosition = getPosition;
    }

    private (int, int) GetCell(Vector2 pos) =>
        ((int)MathF.Floor(pos.X / _cellSize), (int)MathF.Floor(pos.Y / _cellSize));

    public void Insert(T item)
    {
        var cell = GetCell(_getPosition(item));
        if (!_cells.TryGetValue(cell, out var list))
            _cells[cell] = list = new List<T>();
        list.Add(item);
    }

    public IEnumerable<T> QueryRadius(Vector2 center, float radius)
    {
        int cellRadius = (int)MathF.Ceiling(radius / _cellSize);
        var centerCell = GetCell(center);

        for (int dx = -cellRadius; dx <= cellRadius; dx++)
        for (int dy = -cellRadius; dy <= cellRadius; dy++)
        {
            var cell = (centerCell.Item1 + dx, centerCell.Item2 + dy);
            if (_cells.TryGetValue(cell, out var list))
                foreach (var item in list)
                    if (Vector2.DistanceSquared(center, _getPosition(item)) <= radius * radius)
                        yield return item;
        }
    }

    public void Clear() => _cells.Clear();
}
```

**Spatial structure selection:**

| Structure | Best For | Complexity |
|-----------|----------|------------|
| Spatial Hash / Grid | Uniform distribution | O(1) insert, O(k) query |
| Quadtree / Octree | Non-uniform, hierarchical | O(log n) |
| BVH | Static geometry, raycasting | O(log n) query |
| k-d Tree | Nearest neighbor | O(log n) average |

### Bloom Filter

Probabilistic "definitely not in set" checks:

```csharp
public class BloomFilter
{
    private readonly BitArray _bits;
    private readonly int _hashCount;
    private readonly int _size;

    public BloomFilter(int expectedItems, double falsePositiveRate = 0.01)
    {
        _size = (int)Math.Ceiling(-expectedItems * Math.Log(falsePositiveRate) / (Math.Log(2) * Math.Log(2)));
        _hashCount = (int)Math.Round((double)_size / expectedItems * Math.Log(2));
        _bits = new BitArray(_size);
    }

    public void Add(ReadOnlySpan<byte> item)
    {
        var (h1, h2) = GetHashes(item);
        for (int i = 0; i < _hashCount; i++)
            _bits[Math.Abs((int)((h1 + i * h2) % _size))] = true;
    }

    public bool MayContain(ReadOnlySpan<byte> item)
    {
        var (h1, h2) = GetHashes(item);
        for (int i = 0; i < _hashCount; i++)
            if (!_bits[Math.Abs((int)((h1 + i * h2) % _size))]) 
                return false;  // Definitely not in set
        return true;  // Probably in set
    }

    private static (long, long) GetHashes(ReadOnlySpan<byte> item) =>
        (XxHash64.HashToUInt64(item), XxHash64.HashToUInt64(item, seed: 42));
}
```

**Use cases:** Skip expensive lookups, cache admission, duplicate detection in streams.

### Trie (Prefix Search)

For autocomplete, prefix matching:

```csharp
public class Trie
{
    private class Node { public Dictionary<char, Node>? Children; public bool IsTerminal; }
    private readonly Node _root = new();

    public void Add(string word)
    {
        var node = _root;
        foreach (char c in word)
        {
            node.Children ??= new();
            if (!node.Children.TryGetValue(c, out var child))
                node.Children[c] = child = new Node();
            node = child;
        }
        node.IsTerminal = true;
    }

    public IEnumerable<string> GetByPrefix(string prefix)
    {
        var node = _root;
        foreach (char c in prefix)
            if (node.Children == null || !node.Children.TryGetValue(c, out node!))
                yield break;
        foreach (var word in Collect(node, prefix))
            yield return word;
    }

    private IEnumerable<string> Collect(Node node, string prefix)
    {
        if (node.IsTerminal) yield return prefix;
        if (node.Children == null) yield break;
        foreach (var (c, child) in node.Children)
            foreach (var word in Collect(child, prefix + c))
                yield return word;
    }
}
```

### LRU Cache

Bounded memory with hot data retention:

```csharp
public class LruCache<TKey, TValue> where TKey : notnull
{
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<(TKey Key, TValue Value)>> _map;
    private readonly LinkedList<(TKey Key, TValue Value)> _list = new();

    public LruCache(int capacity)
    {
        _capacity = capacity;
        _map = new(capacity);
    }

    public bool TryGet(TKey key, out TValue value)
    {
        if (_map.TryGetValue(key, out var node))
        {
            _list.Remove(node);
            _list.AddFirst(node);
            value = node.Value.Value;
            return true;
        }
        value = default!;
        return false;
    }

    public void Set(TKey key, TValue value)
    {
        if (_map.TryGetValue(key, out var existing))
        {
            _list.Remove(existing);
            _map.Remove(key);
        }
        else if (_map.Count >= _capacity)
        {
            var last = _list.Last!;
            _map.Remove(last.Value.Key);
            _list.RemoveLast();
        }
        var node = _list.AddFirst((key, value));
        _map[key] = node;
    }
}
```

## Structure Selection Summary

| Problem | Structure | Speedup |
|---------|-----------|---------|
| Find by ID | Dictionary / FrozenDictionary | O(n) → O(1) |
| Find nearby | Spatial Hash / Quadtree | O(n) → O(k) |
| Check if seen | Bloom Filter / HashSet | O(n) → O(1) |
| Range queries | Interval Tree / SortedSet | O(n) → O(log n + k) |
| Autocomplete | Trie | O(nm) → O(m + k) |
| Keep top K | SortedSet / PriorityQueue | O(n) → O(log n) |
| Bounded cache | LRU Cache | Prevents unbounded growth |
