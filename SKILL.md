---
name: csharp-performance-optimization
description: C# performance optimization guidelines for reducing allocations, improving algorithmic complexity, and writing high-performance .NET code. Use when reviewing or writing performance-critical C# code, optimizing hot paths, reducing GC pressure, or when the user asks about memory allocation, pooling, Span, Memory, ReadOnlySpan, boxing, closures, LINQ performance, struct vs class, sealed classes, async optimization, ValueTask, stackalloc, ArrayPool, or acceleration structures like spatial hashing and bloom filters.
---

# C# Performance Optimization

## Workflow

### 1. Research and Clarify

Before optimizing, ask clarifying questions:

- What are the performance goals? (latency target, throughput, memory budget)
- What are the current pain points? (GC pauses, high CPU, memory pressure)
- What are the collection sizes? (N = 10? 1,000? 1,000,000?)
- What are the access patterns? (read-heavy, write-heavy, mixed)
- How often does this code execute? (once per frame, per request, per message)
- Is this a hot path confirmed by profiling?

### 2. Create Branch and PR

- Create a feature branch for the optimization work
- Open a GitHub PR early to track progress
- Use a descriptive title: `perf: optimize player lookup with spatial hash`

### 3. Track Work in GitHub

- Post benchmark results as PR comments to document before/after
- Explain optimization rationale in comments
- Direct the user to review and respond on GitHub
- Answer questions in PR comments to maintain context

Example comment:
```
## Benchmark: PlayerLookup

| Method | N | Mean | Allocated |
|--------|---|------|-----------|
| Before | 10000 | 847.3 μs | 48 KB |
| After | 10000 | 12.4 μs | 0 B |

Replaced O(n) linear scan with Dictionary index. 68x faster, zero allocations.
```

### 4. Verify CI Passes

- Ensure all CI jobs pass before requesting review
- Fix any test failures introduced by optimizations
- Address compiler warnings

---

## Core Principles

1. **Analyze algorithmic complexity first** — Fix O(n²) before micro-optimizing
2. **Profile to find hot paths** — Don't guess, measure
3. **Benchmark before/after** — Verify optimizations actually help
4. **Skip optimizations that don't yield measurable benefits**
5. **Do not check in benchmarks** — They are artifacts to guide work, post results as PR comments

## Priority Order

| Priority | Category | Typical Speedup |
|----------|----------|-----------------|
| 1 | Algorithmic complexity (BigO) | 10-1000x |
| 2 | Acceleration structures (indexes, spatial) | 10-100x |
| 3 | Allocation reduction (pooling, stackalloc) | 2-10x |
| 4 | JIT optimization (inlining, devirtualization) | 1.2-3x |

## Quick Decision Trees

### Should I optimize this code?

```
Is it on a hot path? (runs frequently)
├─ No → Don't optimize
└─ Yes → Is it measurably slow?
    ├─ No → Don't optimize
    └─ Yes → Profile first, then optimize
```

### What's causing the slowdown?

```
Profile shows high CPU time
├─ In tight loops → Check algorithmic complexity, consider acceleration structures
├─ In method calls → Check for virtual dispatch, consider sealing
└─ In allocations → Check for boxing, closures, LINQ

Profile shows high allocations / GC pressure
├─ Many small objects → Pool them or use structs
├─ Temporary arrays → Use stackalloc or ArrayPool
├─ String building → Use StringBuilder or Span<char>
└─ Lambda allocations → Avoid closures, use static lambdas
```

### Collection selection

```
Need key-based lookup?
├─ N < 8 elements → Array + linear scan
├─ N >= 8, read-heavy, static → FrozenDictionary
└─ N >= 8, dynamic → Dictionary

Need "find nearby" queries?
└─ Use spatial partitioning (grid, quadtree)

Need duplicate detection?
├─ Exact, bounded → HashSet
└─ Probabilistic, large scale → Bloom filter
```

### Struct vs Class

```
Size <= 16 bytes AND short-lived AND value semantics?
├─ Yes → struct (mark readonly if immutable)
└─ No → class (mark sealed if no inheritance)
```

## Reference Files

Detailed patterns and code examples for each optimization category:

- **[references/algorithmic-complexity.md](references/algorithmic-complexity.md)** — BigO analysis, understanding data scale, acceleration structures (indexes, spatial partitioning, bloom filters, tries)
- **[references/memory-allocation.md](references/memory-allocation.md)** — Pooling, stackalloc, Span/Memory, lazy initialization, IDisposable
- **[references/type-optimization.md](references/type-optimization.md)** — Structs, ref/in/ref readonly, sealed classes, boxing avoidance
- **[references/closures-and-linq.md](references/closures-and-linq.md)** — Closure avoidance, static lambdas, LINQ alternatives
- **[references/string-optimization.md](references/string-optimization.md)** — StringBuilder, string.Create, interning, FrozenSet
- **[references/async-and-threading.md](references/async-and-threading.md)** — ValueTask, ConfigureAwait, lock selection
- **[references/jit-optimization.md](references/jit-optimization.md)** — Inlining, throw helpers, bounds check elimination, SIMD
- **[references/reflection-avoidance.md](references/reflection-avoidance.md)** — Caching, compiled delegates, source generators

## Checklists

### Hot Path Review Checklist

- [ ] Algorithmic complexity is appropriate for data size
- [ ] No nested loops over collections that grow together
- [ ] No `.Contains()` in loops (use HashSet or index)
- [ ] No LINQ in tight loops
- [ ] No closures capturing variables (use static lambdas)
- [ ] No boxing of value types
- [ ] Collections pre-allocated with known capacity
- [ ] Temporary arrays use stackalloc or ArrayPool
- [ ] Classes are sealed where possible
- [ ] Large structs passed with `in` keyword

### Allocation Reduction Checklist

- [ ] Use `ArrayPool<T>.Shared` for temporary arrays
- [ ] Use `stackalloc` for small, method-scoped buffers (< 1KB)
- [ ] Use `Span<T>` / `ReadOnlySpan<T>` instead of copying
- [ ] Use `StringBuilder` pool for frequent string building
- [ ] Cache delegates that don't capture state
- [ ] Override `ToString`, `Equals`, `GetHashCode` on structs
- [ ] Use generic constraints to avoid interface boxing

## Detection Tools

- **BenchmarkDotNet** — `[MemoryDiagnoser]` for allocation measurement
- **Rider/ReSharper** — Highlights closures, boxing, unsealed classes
- **dotnet-counters** — Runtime GC pressure monitoring
- **PerfView** — Detailed allocation and GC analysis
- **Heap Allocation Analyzer** — Roslyn analyzer for compile-time detection
