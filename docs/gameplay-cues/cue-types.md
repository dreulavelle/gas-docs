---
title: Cue Notify Types
description: All six Gameplay Cue Notify classes in UE 5.7 — Static, Burst, BurstLatent, Actor, Looping, and HitImpact — with lifecycle details and decision guidance.
---

# Cue Notify Types

UE 5.7 ships six Gameplay Cue Notify classes. The [overview page](../core-concepts/gameplay-cues-overview.md) introduced four of them -- here we cover all six in full, including the two newer types (`Burst` and `BurstLatent`) that provide a much better workflow than the original `Static` and `Actor` classes.

## Decision Matrix

Before diving into each type, here's the quick decision guide:

| Question | Answer | Use This |
|:---|:---|:---|
| One-shot effect, no editor VFX config needed? | Yes | **Static** |
| One-shot effect, want editor-configurable VFX/SFX? | Yes | **Burst** |
| One-shot effect, need Timelines or Delays? | Yes | **BurstLatent** |
| Persistent effect (starts and stops)? | Yes | **Looping** or **Actor** |
| Persistent with built-in looping FX management? | Yes | **Looping** |
| Persistent with fully custom lifecycle control? | Yes | **Actor** |
| Simple hit impact with particle + sound? | Yes | ~~HitImpact~~ (deprecated, use **Burst**) |

!!! tip "When in doubt"
    **Burst** for fire-and-forget. **Looping** for duration-based. These two cover 90% of use cases and have the best editor UX.

---

## UGameplayCueNotify_Static

**The lightest cue.** A non-instanced UObject (not an Actor) that handles cue events through its CDO. No actor is ever spawned -- the engine calls methods directly on the class default object.

### When to Use

- Triggering sounds or VFX from code that you don't need editor-exposed spawn configuration for
- Running pure logic in response to a cue (logging, analytics, UI notifications)
- Maximum performance -- zero actor spawn overhead

### Lifecycle Events

| Event | When It Fires |
|:---|:---|
| `OnExecute` | Instant GE applied or periodic tick |
| `OnActive` | Duration/Infinite GE first applied (only if client witnessed activation) |
| `WhileActive` | Duration/Infinite GE seen as active (including join-in-progress) |
| `OnRemove` | Duration/Infinite GE removed |
| `HandleGameplayCue` | Generic handler called for every event type |

### Key Properties

| Property | Type | Purpose |
|:---|:---|:---|
| `GameplayCueTag` | `FGameplayTag` | Tag this notify responds to |
| `IsOverride` | `bool` | If true, prevents parent tags from also firing |

### Inheritance

```
UObject
  └── UGameplayCueNotify_Static
        ├── UGameplayCueNotify_Burst        (child)
        └── UGameplayCueNotify_HitImpact    (child, deprecated)
```

### Practical Example

A simple static cue that plays a sound at the target's location:

```cpp
UCLASS()
class UGC_HitConfirm : public UGameplayCueNotify_Static
{
    GENERATED_BODY()

    virtual bool OnExecute_Implementation(AActor* MyTarget,
        const FGameplayCueParameters& Parameters) const override
    {
        if (MyTarget && HitSound)
        {
            UGameplayStatics::PlaySoundAtLocation(
                MyTarget, HitSound, Parameters.Location);
        }
        return false; // false = also run parent handlers
    }

    UPROPERTY(EditDefaultsOnly, Category = "Sound")
    TObjectPtr<USoundBase> HitSound;
};
```

!!! warning "No world context by default"
    Static cues are not actors, so they don't inherently have a world context. The engine provides one through `GetWorld()` during cue handling, but be careful with editor preview.

---

## UGameplayCueNotify_Burst

**The recommended one-shot cue.** Inherits from `UGameplayCueNotify_Static` (so it's still non-instanced and cheap), but adds editor-configurable spawn conditions, placement rules, and burst effects (Niagara, sounds, camera shakes, force feedback, decals).

Display name in the editor: **GCN Burst**.

### When to Use

- Hit impacts, damage numbers, ability cast effects
- Any one-shot VFX/SFX where you want designers to configure everything in the editor
- When you don't need Timelines, Delays, or any latent actions

### Lifecycle

Burst cues only respond to `OnExecute`. When triggered, the system:

1. Checks `DefaultSpawnCondition` (locally controlled policy, chance to play, surface filter)
2. Determines placement from `DefaultPlacementInfo` (attach to socket, use hit location, offset)
3. Spawns all effects in `BurstEffects` (particles, sounds, camera shakes, decals, force feedback)
4. Calls `OnBurst` Blueprint event with spawn results

### Key Properties

| Property | Type | Purpose |
|:---|:---|:---|
| `DefaultSpawnCondition` | `FGameplayCueNotify_SpawnCondition` | Should this cue play? (local control, chance, surface type) |
| `DefaultPlacementInfo` | `FGameplayCueNotify_PlacementInfo` | Where to spawn effects (socket, location, rotation) |
| `BurstEffects` | `FGameplayCueNotify_BurstEffects` | The actual VFX/SFX to spawn |

### Blueprint Events

| Event | Parameters | Notes |
|:---|:---|:---|
| `OnBurst` | `Target`, `Parameters`, `SpawnResults` | The main event -- fires after all effects spawn |

The `SpawnResults` struct contains references to spawned Niagara/audio/decal components, so you can modify them in the event graph if needed.

### Practical Example

In Blueprint:

1. Create a new Blueprint, parent class `GameplayCueNotify_Burst`
2. Set `GameplayCue Tag` to `GameplayCue.Hit.Physical`
3. In `BurstEffects`, add a Niagara system for spark particles and a sound for the impact
4. Set `DefaultPlacementInfo` to use hit location from parameters
5. Done -- no code needed

---

## AGameplayCueNotify_BurstLatent

**A one-shot cue that supports latent actions.** Unlike `Burst` (which is a UObject), `BurstLatent` is an Actor -- it gets spawned into the world. This means it can use Timelines, Delays, and other latent Blueprint nodes.

Display name in the editor: **GCN Burst Latent**.

### When to Use

- One-shot effects that need to animate over time (fade out, scale up then down)
- Effects that need a Timeline node
- When `Burst` almost works but you need just a bit more control over timing

### Lifecycle

Like `Burst`, this only responds to `OnExecute`. The difference is the actor persists until:

- It finishes its work and calls `K2_EndGameplayCue()` (or the auto-destroy delay fires)
- It's recycled back to the pool

### Key Properties

Same as Burst (`DefaultSpawnCondition`, `DefaultPlacementInfo`, `BurstEffects`), plus the full `AGameplayCueNotify_Actor` properties it inherits (auto-destroy, recycle settings).

| Property | Type | Purpose |
|:---|:---|:---|
| `BurstSpawnResults` | `FGameplayCueNotify_SpawnResult` | The spawned components, accessible in Blueprint for post-spawn modification |

### Blueprint Events

| Event | Parameters | Notes |
|:---|:---|:---|
| `OnBurst` | `Target`, `Parameters`, `SpawnResults` | Fires after effects spawn. You can chain latent actions from here. |

### Inheritance

```
AActor
  └── AGameplayCueNotify_Actor
        ├── AGameplayCueNotify_BurstLatent    (child)
        └── AGameplayCueNotify_Looping        (child)
```

!!! note "Cost vs Burst"
    BurstLatent spawns an actor, which is more expensive than Burst (a UObject). If you don't need latent actions, prefer Burst.

---

## AGameplayCueNotify_Actor

**The stateful base class.** An actor that is spawned into the world and persists for the duration of a gameplay effect. It handles the full lifecycle (Active, WhileActive, Execute for periodic ticks, Remove) and supports actor recycling to avoid spawn/destroy overhead.

### When to Use

- Any persistent visual that starts and stops (auras, shields, status overlays)
- When you need full actor functionality (components, tick, collision)
- When you need unique instances per instigator (beam effects)
- As a base class for fully custom cue actors

### Lifecycle Events

| Event | Blueprint Display Name | When It Fires |
|:---|:---|:---|
| `OnActive` | On Burst | GE with duration first applied (client witnessed activation) |
| `WhileActive` | On Become Relevant | GE seen as active (join-in-progress, relevancy change) |
| `OnExecute` | OnExecute | Periodic tick of a Duration/Infinite GE |
| `OnRemove` | On Cease Relevant | GE removed or expired |
| `HandleGameplayCue` | HandleGameplayCue | Generic handler for all event types |

!!! info "Display name differences"
    In 5.7, `OnActive` displays as "On Burst" and `WhileActive` as "On Become Relevant" in Blueprint. The C++ names haven't changed -- only the editor display names. This is an important source of confusion if you're reading older documentation.

### Key Properties

| Property | Type | Default | Purpose |
|:---|:---|:---|:---|
| `bAutoDestroyOnRemove` | `bool` | -- | Recycle/destroy when OnRemove fires |
| `AutoDestroyDelay` | `float` | 0 | Seconds to wait after remove before cleanup |
| `bAutoAttachToOwner` | `bool` | -- | Attach actor to the target |
| `bUniqueInstancePerInstigator` | `bool` | -- | Separate cue actor per instigator |
| `bUniqueInstancePerSourceObject` | `bool` | -- | Separate cue actor per source object |
| `bAllowMultipleOnActiveEvents` | `bool` | -- | Re-trigger OnActive if already active |
| `bAllowMultipleWhileActiveEvents` | `bool` | -- | Re-trigger WhileActive if already active |
| `NumPreallocatedInstances` | `int32` | 0 | How many to pre-spawn for the recycle pool |
| `IsOverride` | `bool` | -- | Override parent cues in hierarchy |

### Recycling

Actor cue notifies are **recycled**, not destroyed and re-created. When a cue finishes:

1. `Recycle()` is called -- the actor hides itself, disables tick, and resets state
2. The actor goes into the `GameplayCueManager`'s recycle pool
3. Next time a cue of the same class is needed, `ReuseAfterRecycle()` is called instead of spawning a new actor

Override `Recycle()` to return `false` if your cue can't be recycled (rare).

### Practical Example

A burning aura cue:

1. Create Blueprint, parent `GameplayCueNotify_Actor`
2. Add a Niagara component for fire particles as a child component
3. Set `GameplayCue Tag` to `GameplayCue.Status.Burning`
4. Set `bAutoAttachToOwner` = true
5. Set `bAutoDestroyOnRemove` = true, `AutoDestroyDelay` = 1.0 (let particles fade)
6. In `On Become Relevant`, activate the particle system
7. In `On Cease Relevant`, deactivate the particle system

---

## AGameplayCueNotify_Looping

**The recommended duration-based cue.** Inherits from `AGameplayCueNotify_Actor` and adds the same spawn-configuration workflow that `Burst` brought to one-shots: editor-configurable effects for application, looping, recurring (periodic ticks), and removal.

Display name in the editor: **GCN Looping**.

### When to Use

- Duration-based effects where you want designers to configure VFX/SFX in the editor
- Effects with distinct phases (start burst, looping particles, tick effects, removal burst)
- The standard recommendation for any persistent cue

### Lifecycle

The Looping cue maps its four effect categories to the Actor lifecycle:

| Phase | Effect Category | When |
|:---|:---|:---|
| **Application** | `ApplicationEffects` (burst) | OnActive -- when the effect first applies |
| **Looping** | `LoopingEffects` (persistent) | WhileActive -- starts and persists |
| **Recurring** | `RecurringEffects` (burst) | OnExecute -- each periodic tick |
| **Removal** | `RemovalEffects` (burst) | OnRemove -- when the effect ends |

### Blueprint Events

| Event | When |
|:---|:---|
| `OnApplication` | After application effects spawn |
| `OnLoopingStart` | After looping effects spawn |
| `OnRecurring` | After recurring effects spawn (each tick) |
| `OnRemoval` | After removal effects spawn |

Each event receives `Target`, `Parameters`, and the `SpawnResults` for that phase.

### Key Properties

| Property | Type | Purpose |
|:---|:---|:---|
| `DefaultSpawnCondition` | `FGameplayCueNotify_SpawnCondition` | Global spawn condition |
| `DefaultPlacementInfo` | `FGameplayCueNotify_PlacementInfo` | Global placement rules |
| `ApplicationEffects` | `FGameplayCueNotify_BurstEffects` | One-shot effects on start |
| `LoopingEffects` | `FGameplayCueNotify_LoopingEffects` | Persistent effects while active |
| `RecurringEffects` | `FGameplayCueNotify_BurstEffects` | One-shot effects per periodic tick |
| `RemovalEffects` | `FGameplayCueNotify_BurstEffects` | One-shot effects on end |

### Practical Example

A poison DoT cue:

1. Create Blueprint, parent `GameplayCueNotify_Looping`
2. Set `GameplayCue Tag` to `GameplayCue.Status.Poison`
3. **Application Effects**: A burst of green gas particles + poison apply sound
4. **Looping Effects**: Persistent dripping green particles attached to the target
5. **Recurring Effects**: A small green flash + tick sound (fires each DoT tick)
6. **Removal Effects**: A cleanse burst particle + sound
7. Set `DefaultPlacementInfo` to attach to the target's mesh

All of this is configurable in the editor with zero code.

---

## UGameplayCueNotify_HitImpact (Deprecated)

!!! warning "Deprecated"
    `UGameplayCueNotify_HitImpact` is marked as deprecated in the UE 5.7 source. The class itself says: "This class is deprecated. Use GCN Burst instead." If you have existing HitImpact cues, they'll still work, but all new cues should use `UGameplayCueNotify_Burst`.

A non-instanced cue (inherits from `Static`) with two hardcoded properties:

| Property | Type | Purpose |
|:---|:---|:---|
| `Sound` | `USoundBase*` | Sound to play at impact |
| `ParticleSystem` | `UParticleSystem*` | Legacy Cascade particle system to spawn |

It only handles `OnExecute` and spawns the particle/sound at the hit location from the parameters. The `Burst` class does everything this does and more, with Niagara support and configurable spawn conditions.

---

## Type Comparison Summary

| | Static | Burst | BurstLatent | Actor | Looping | HitImpact |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Is an Actor** | No | No | Yes | Yes | Yes | No |
| **Instanced** | No (CDO) | No (CDO) | Yes | Yes | Yes | No (CDO) |
| **Stateful** | No (shared CDO) | No | No | Yes | Yes | No |
| **Supports Latent Actions** | No | No | Yes | Yes | Yes | No |
| **Editor VFX Config** | No | Yes | Yes | Manual | Yes | Limited |
| **Recycled** | N/A | N/A | Yes | Yes | Yes | N/A |
| **Relative Cost** | Cheapest | Cheap | Medium | Medium | Medium | Cheap |
| **Deprecated** | No | No | No | No | No | **Yes** |

## Related Pages

- [Cue Parameters](cue-parameters.md) -- what data your cue receives
- [Cue Manager](cue-manager.md) -- how cues are routed, loaded, and managed
- [AnimNotify and Sequencer](anim-notify-and-sequencer.md) -- triggering cues from animations
- [Add a Gameplay Cue](../recipes/add-gameplay-cue.md) -- step-by-step recipe
- [Cue Batching](../optimization/cue-batching.md) -- performance optimization
