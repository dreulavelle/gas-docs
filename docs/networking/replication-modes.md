---
title: Replication Modes
description: Full, Mixed, and Minimal Gameplay Effect replication modes — what replicates, to whom, bandwidth impact, and how to choose.
---

# Replication Modes

The `EGameplayEffectReplicationMode` on `UAbilitySystemComponent` controls how Gameplay Effects, tags, and cues are replicated to clients. This is the single most impactful networking decision you'll make in your GAS project.

## The Three Modes

Set the mode on your ASC, typically in your character or player state constructor:

```cpp
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
```

### Full

```cpp
EGameplayEffectReplicationMode::Full
```

**Replicates full gameplay effect info to all clients.**

Every active GE, every attribute change, every tag, every cue -- replicated to every connected client. This is the simplest mode and the most bandwidth-intensive.

| What | Replicated To |
|:---|:---|
| Active Gameplay Effects (full spec) | All clients |
| Attributes | All clients |
| Gameplay Tags | All clients |
| Gameplay Cues | All clients (via RPC) |

**Good for:** Small-scale games (1-4 players), prototyping, single-player with occasional multiplayer.

**Bad for:** Anything with many players or many active effects. Bandwidth scales with `players * active_effects`.

### Mixed

```cpp
EGameplayEffectReplicationMode::Mixed
```

**Replicates minimal info to simulated proxies, full info to the owning client.**

The owning client (autonomous proxy) gets the full GE replication -- they need it for prediction reconciliation. Other clients (simulated proxies) only get the replicated tags and cues, not the full effect specs.

| What | Owner | Other Clients |
|:---|:---|:---|
| Active Gameplay Effects (full spec) | Yes | No |
| Attributes | Yes | Only if in the `AttributeSet`'s `GetLifetimeReplicatedProps` |
| Gameplay Tags | Yes | Yes (via `MinimalReplicationTags`) |
| Gameplay Cues | Yes | Yes (via RPC) |

**Good for:** Most multiplayer games (4-32 players). This is the recommended default for competitive and co-op games.

**Bad for:** Games where other clients need to inspect the full effect list on remote players (rare).

### Minimal

```cpp
EGameplayEffectReplicationMode::Minimal
```

**Replicates only what's absolutely necessary.**

No GE specs are replicated to anyone. Tags are replicated via the minimal proxy. Cues use the minimal replication proxy. You're responsible for ensuring clients have the state they need through other means (custom replication, RPCs).

| What | Owner | Other Clients |
|:---|:---|:---|
| Active Gameplay Effects | No | No |
| Attributes | Custom (you manage) | Custom |
| Gameplay Tags | Via minimal proxy | Via minimal proxy |
| Gameplay Cues | Via minimal proxy | Via minimal proxy |

!!! warning "Not for owned ASCs without extra work"
    Minimal mode doesn't replicate GE specs, which means prediction reconciliation won't work out of the box. Epic's docs note: "This does not work for Owned AbilitySystemComponents (Use Mixed instead)." If you use Minimal, the ASC should not be on a player-controlled pawn unless you've set up the replication proxy.

**Good for:** AI-controlled characters, large-scale games (50+ players) with proxy interface, dedicated server setups where clients don't need effect details.

## PlayerState Ownership

Where the ASC lives affects replication behavior. There are two common patterns:

### ASC on the Character/Pawn

```
AMyCharacter
  └── UAbilitySystemComponent
```

The ASC replicates with the character. When the character is destroyed (death, respawn), the ASC and all its state go with it. Simple but inflexible.

### ASC on the PlayerState

```
AMyPlayerState
  └── UAbilitySystemComponent

AMyCharacter
  └── (references PlayerState's ASC via IAbilitySystemInterface)
```

The ASC persists across character deaths/respawns since the PlayerState survives. Effects, attributes, and abilities survive respawn. This is the recommended pattern for most multiplayer games.

!!! note "Ownership matters for replication"
    UE replicates actor properties to the owning connection. The PlayerState is owned by its player's connection, so ASC properties on the PlayerState replicate correctly to the owning client. If you put the ASC on a pawn that's not owned by the connection, you'll have replication issues.

## Decision Framework

| Factor | Full | Mixed | Minimal |
|:---|:---:|:---:|:---:|
| Player count 1-4 | Great | Great | Overkill |
| Player count 4-16 | Fine | **Best** | If needed |
| Player count 16-64 | Risky | Good | **Best** |
| Player count 64+ | Don't | Maybe | **Required** |
| AI characters | Wasteful | Wasteful | **Best** |
| Prediction needed | Yes | Yes | Extra work |
| Bandwidth sensitive | No | Moderate | **Best** |
| Simplest setup | **Yes** | Moderate | Complex |

### By Genre

| Genre | Recommended Mode |
|:---|:---|
| Single-player / Co-op PvE | Full or Mixed |
| Arena Shooter (8-16 players) | Mixed |
| Battle Royale | Mixed for players, Minimal for AI |
| MMO-lite (50+ players) | Minimal with proxy |
| MOBA | Mixed |
| Turn-based | Full (bandwidth isn't a concern) |

## Per-ASC Configuration

You can mix modes in the same game. This is common:

```cpp
// Player characters -- need prediction support
PlayerASC->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

// AI enemies -- don't need full replication
AIASC->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);

// Important boss -- other players need to see its buffs
BossASC->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
```

## Bandwidth Impact

A rough mental model for bandwidth per ASC per frame:

| Mode | Active Effects (N) | Approximate Cost |
|:---|:---|:---|
| Full | N effects to all clients | `O(N * Clients)` |
| Mixed | N effects to owner only, tags+cues to others | `O(N + Clients * TagCount)` |
| Minimal | Tags+cues only | `O(Clients * TagCount)` |

The actual numbers depend heavily on your game (how many effects are active, how often they change, how many attributes replicate). Use Unreal's network profiler to measure.

## Related Pages

- [Prediction](prediction.md) -- how prediction interacts with replication modes
- [Net Execution Policies](net-execution-policies.md) -- controlling where abilities run
- [Replication Proxy](replication-proxy.md) -- advanced replication for Minimal mode
- [Replication Optimization](../optimization/replication-optimization.md) -- performance tuning
