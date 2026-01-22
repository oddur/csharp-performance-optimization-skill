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
- **[references/advanced-patterns.md](references/advanced-patterns.md)** — Decision flowcharts, multi-level buffer pools, circular buffers, batch processing, benchmarking, anti-patterns

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
- **dotnet-trace** — CPU sampling and thread time profiling (see below)
- **dotnet-counters** — Runtime GC pressure monitoring
- **Rider/ReSharper** — Highlights closures, boxing, unsealed classes
- **PerfView** — Detailed allocation and GC analysis
- **Heap Allocation Analyzer** — Roslyn analyzer for compile-time detection

---

## Using dotnet-trace for CPU Profiling

### Installation

```bash
dotnet tool install -g dotnet-trace
```

### Available Profiles

```bash
dotnet-trace list-profiles
```

| Profile | Use Case | Platform |
|---------|----------|----------|
| `dotnet-sampled-thread-time` | Wall clock time analysis | All |
| `cpu-sampling` | CPU usage measurement | Linux only |
| `gc-collect` | GC collection tracking (low overhead) | All |
| `gc-verbose` | GC + allocation sampling | All |

### Tracing a Process

#### Option 1: Launch and trace (recommended for benchmarks)

```bash
# Build first (important!)
dotnet build -c Release --framework net9.0

# Trace the built executable directly
dotnet-trace collect \
  --profile dotnet-sampled-thread-time \
  -o trace_output.nettrace \
  -- ./bin/Release/net9.0/YourApp [args]
```

**Important:** Don't use `dotnet run` through `dotnet-trace` — argument parsing issues occur. Always build first and trace the executable directly.

#### Option 2: Attach to running process

```bash
# Find process ID
dotnet-trace ps

# Attach with duration limit
dotnet-trace collect -p <PID> --duration 00:00:30
```

### Converting Traces for Visualization

```bash
# Convert to speedscope format (viewable in browser)
dotnet-trace convert trace_output.nettrace --format speedscope

# View at https://www.speedscope.app/
```

### Profiling Hot Paths (Recommended Approach)

**Do NOT trace BenchmarkDotNet directly** — it has massive startup overhead (assembly compilation, reflection, job scheduling) that dominates the trace. You'll capture BenchmarkDotNet infrastructure instead of your actual code.

**Instead, create a standalone profiling harness:**

```csharp
// Program.cs - minimal console app that directly exercises hot path
using System.Diagnostics;

int iterations = args.Length > 0 ? int.Parse(args[0]) : 100;
int batchSize = args.Length > 1 ? int.Parse(args[1]) : 5000;

// Setup (mirrors your benchmark setup)
var myService = new MyService();

// Warmup - JIT everything first
for (int i = 0; i < 10; i++)
{
    myService.HotPathMethod();
}

Console.WriteLine("Starting profiled run...");
var sw = Stopwatch.StartNew();

// The actual code you want to profile
for (int iter = 0; iter < iterations; iter++)
{
    for (int i = 0; i < batchSize; i++)
    {
        myService.HotPathMethod();
    }
}

sw.Stop();
Console.WriteLine($"Completed {iterations * batchSize:N0} ops in {sw.Elapsed.TotalSeconds:F2}s");
```

**Then trace it:**

```bash
# Build the profiler
dotnet build -c Release

# Trace with dotnet-trace
dotnet-trace collect \
  --profile dotnet-sampled-thread-time \
  -o profile.nettrace \
  -- ./bin/Release/net9.0/MyProfiler 100 5000

# Convert and view
dotnet-trace convert profile.nettrace --format speedscope
# Open at https://www.speedscope.app/
```

**Why this works better:**
- No BenchmarkDotNet overhead (compilation, reflection, scheduling)
- Trace captures only your code
- Fast iteration — run in seconds, not minutes
- Easy to adjust parameters for longer/shorter traces

### BenchmarkDotNet: Use for Measurement, Not Profiling

BenchmarkDotNet is excellent for **measuring** performance (ops/sec, allocations) but poor for **profiling** (finding hot spots). Use both tools together:

| Tool | Use For |
|------|---------|
| BenchmarkDotNet | Reliable measurements, before/after comparison |
| Standalone profiler + dotnet-trace | Finding where CPU time is spent |

### Quick Reference

```bash
# Install
dotnet tool install -g dotnet-trace

# List running .NET processes
dotnet-trace ps

# Trace with 30-second duration
dotnet-trace collect -p <PID> --duration 00:00:30

# Trace with specific providers
dotnet-trace collect -p <PID> --providers Microsoft-DotNETCore-SampleProfiler

# Convert to speedscope
dotnet-trace convert trace.nettrace --format speedscope

# Convert to chromium format (for Chrome DevTools)
dotnet-trace convert trace.nettrace --format chromium
```

### Interpreting Results in Speedscope

1. **Left Heavy view** — Shows call tree sorted by total time (find hottest paths)
2. **Sandwich view** — Shows callers and callees of selected function
3. **Time Order view** — Shows execution timeline (find phases)

Look for:
- Functions with high "self time" (CPU-bound work)
- Deep call stacks with high "total time" (algorithmic issues)
- Frequent short calls (consider inlining or batching)
