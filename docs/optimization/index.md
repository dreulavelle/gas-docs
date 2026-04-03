---
title: Optimization
description: GAS performance optimization — replication, cue batching, attribute proxy, and lazy loading. Read this after your game works.
---

# Optimization

!!! warning "Only read this after your game works"
    Premature optimization is the root of all evil. Get your abilities, effects, and cues working correctly first. Then profile. Then optimize what the profiler says is slow. Don't guess.

If you're here because you've profiled and identified GAS as a bottleneck (or you're planning for large player counts), this section covers the main optimization levers.

## What to Optimize, and When

| Symptom | Likely Cause | Page |
|:---|:---|:---|
| High bandwidth per player | Too much GE replication | [Replication Optimization](replication-optimization.md) |
| RPC saturation / dropped cues | Too many cue RPCs | [Cue Batching](cue-batching.md) |
| Bandwidth scales badly with player count | Need minimal replication + proxy | [Attribute Proxy](attribute-proxy.md) |
| Hitches when cues first play | Sync loading cue notifies | [Lazy Loading](lazy-loading.md) |
| Long initial load times | Loading all cues at startup | [Lazy Loading](lazy-loading.md) |

## The Optimization Hierarchy

In rough order of impact and ease of implementation:

1. **Choose the right replication mode.** Mixed for most games, Minimal for large-scale. This is the biggest single win.

2. **Batch cue RPCs.** Use `FScopedGameplayCueSendContext` whenever applying multiple effects.

3. **Make cues unreliable.** They already are by default -- don't override to reliable unless you have a specific need.

4. **Reduce active effect count.** Combine effects where possible. An effect with 3 modifiers is cheaper than 3 effects with 1 modifier each.

5. **Pre-allocate cue actors.** Set `NumPreallocatedInstances` on frequently used Actor/Looping cues to avoid runtime spawning.

6. **Async load cues.** Don't sync-load missing cues at runtime. Configure the manager for async loading.

7. **Implement the replication proxy.** For 50+ player games, move replication to a proxy actor with better relevancy control.

## Profiling GAS

### Network Profiler

The built-in network profiler shows RPC counts and bandwidth per actor. Look for:

- `GameplayCue` RPCs (should be batched, unreliable)
- `AbilitySystem` RPCs (activation, prediction)
- `FActiveGameplayEffect` replication bandwidth

### Stat Commands

```
stat net                    // General network stats
stat game                   // Game thread time
stat AbilitySystem          // GAS-specific stats (if available)
```

### Unreal Insights

Use Unreal Insights for detailed CPU profiling of GAS operations:

- Effect application and removal
- Modifier evaluation
- Cue routing and spawning
- Attribute aggregation

## In This Section

- [Replication Optimization](replication-optimization.md) -- choosing and configuring replication modes
- [Cue Batching](cue-batching.md) -- reducing cue RPC overhead
- [Attribute Proxy](attribute-proxy.md) -- the replication proxy for large player counts
- [Lazy Loading](lazy-loading.md) -- async loading cue notifies and ability classes
