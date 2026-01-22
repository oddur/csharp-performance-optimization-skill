# C# Performance Optimization Skill

A Claude skill for C# performance optimization - reducing allocations, improving algorithmic complexity, and writing high-performance .NET code.

## Installation

### Claude Code

```bash
# Clone to your global skills directory
git clone https://github.com/oddur/csharp-performance-optimization-skill.git ~/.claude/skills/csharp-performance-optimization

# Or for project-specific installation
git clone https://github.com/oddur/csharp-performance-optimization-skill.git .claude/skills/csharp-performance-optimization
```

### Claude.ai

1. Download this repository as a ZIP
2. Go to **Settings > Capabilities**
3. Ensure **Code execution and file creation** is enabled
4. Click **Upload skill** and select the ZIP file
5. Toggle the skill ON

## What's Included

The skill covers:

- **Algorithmic Complexity** — BigO analysis, acceleration structures (indexes, spatial hashing, bloom filters, tries)
- **Memory Allocation** — Pooling, stackalloc, Span/Memory, lazy initialization
- **Type Optimization** — Structs, ref/in/ref readonly, sealed classes, boxing avoidance
- **Closures and LINQ** — Static lambdas, loop alternatives for hot paths
- **String Optimization** — StringBuilder, string.Create, interning, FrozenSet
- **Async and Threading** — ValueTask, ConfigureAwait, lock selection
- **JIT Optimization** — Inlining, throw helpers, bounds check elimination, SIMD
- **Reflection Avoidance** — Caching, compiled delegates, source generators

## Usage

Once installed, the skill automatically triggers when you ask about C# performance topics. It will:

1. Ask clarifying questions about goals, collection sizes, and pain points
2. Create a feature branch and GitHub PR
3. Post benchmark results as PR comments
4. Ensure CI passes before completion

## Structure

```
csharp-performance-optimization/
├── SKILL.md                              # Main skill file
└── references/
    ├── algorithmic-complexity.md         # BigO, acceleration structures
    ├── memory-allocation.md              # Pooling, Span, stackalloc
    ├── type-optimization.md              # Structs, sealed, boxing
    ├── closures-and-linq.md              # Closure avoidance, LINQ alternatives
    ├── string-optimization.md            # StringBuilder, interning
    ├── async-and-threading.md            # ValueTask, locks
    ├── jit-optimization.md               # Inlining, SIMD
    └── reflection-avoidance.md           # Source generators, caching
```

## Requirements

- Claude Pro, Max, Team, or Enterprise plan (for Claude.ai)
- Claude Code (any version)
- Code execution enabled

## License

MIT
