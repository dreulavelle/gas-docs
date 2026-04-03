---
title: Duration and Lifecycle
description: How Gameplay Effects are created, applied, live inside the Active GE Container, and eventually expire or get removed.
---

# Duration and Lifecycle

Every Gameplay Effect has a lifecycle: it gets created as a spec, applied to a target, potentially lives for a while doing its thing, and eventually goes away. Understanding this lifecycle is essential — it drives how you think about modifiers, stacking, tags, and just about everything else in GAS.

## Duration Policy

The `DurationPolicy` property on `UGameplayEffect` is an `EGameplayEffectDurationType` enum with three values:

### Instant

```cpp
EGameplayEffectDurationType::Instant
```

Instant effects are **executed** — they modify the **base value** of an attribute and are immediately discarded. They never enter the Active Gameplay Effects Container. Think of them like a permanent stamp: you apply a heal of +50 HP, the base health goes up by 50, and the effect is gone.

!!! info "Base vs. Current Value"
    Attributes have a **base value** and a **current value**. Instant effects change the base value directly. Duration/Infinite effects contribute to the current value through modifiers while active, leaving the base value untouched.

### Has Duration

```cpp
EGameplayEffectDurationType::HasDuration
```

Duration effects live on the target for a specified amount of time. The duration is defined by a `FGameplayEffectModifierMagnitude`, which means it can be a simple float, driven by an attribute, a custom calculation, or a [SetByCaller](set-by-caller.md) value.

While active, the effect's modifiers contribute to the **current value** of attributes. When the effect expires, those modifier contributions are removed, and the attribute reverts.

### Infinite

```cpp
EGameplayEffectDurationType::Infinite
```

Infinite effects are just duration effects with `INFINITE_DURATION` (-1.0). They live on the target until something explicitly removes them — another effect, a tag change, or direct C++ code.

These are commonly used for passive buffs, auras, equipment bonuses, and anything that needs to persist until a specific condition removes it.

## Periodic Effects

Any Duration or Infinite effect can also be **periodic**. When you set a `Period` greater than 0, the effect will **execute** at each period interval — just like an instant effect firing repeatedly on a timer.

```
Duration GE with Period = 2.0s and Duration = 10.0s
├─ Applied (t=0) — effect added to container
├─ Execute (t=0) — if bExecutePeriodicEffectOnApplication is true
├─ Execute (t=2)
├─ Execute (t=4)
├─ Execute (t=6)
├─ Execute (t=8)
└─ Expired (t=10) — effect removed from container
```

Key properties:

| Property | Type | Description |
|:---|:---|:---|
| `Period` | `FScalableFloat` | Time in seconds between periodic executions |
| `bExecutePeriodicEffectOnApplication` | `bool` | If true, an execution happens immediately when the effect is first applied |
| `PeriodicInhibitionPolicy` | `EGameplayEffectPeriodInhibitionRemovedPolicy` | What happens when an inhibited periodic effect becomes uninhibited |

The `PeriodicInhibitionPolicy` has three options:

- **NeverReset** — The period timer keeps ticking even while inhibited. When uninhibited, the next execution happens at the next scheduled time as if nothing happened.
- **ResetPeriod** — The period timer resets. The next execution will be one full period after uninhibition.
- **ExecuteAndResetPeriod** — Executes immediately on uninhibition, then resets the timer.

!!! tip "Periodic Effects Are Both Added and Executed"
    A periodic effect is _added_ to the Active GE Container (because it has duration), **and** it _executes_ at every period. This is important because modifiers on a periodic effect contribute to the current value while it's active, but the periodic execution can also directly change base values or trigger execution calculations.

## The Active Gameplay Effects Container

The `FActiveGameplayEffectsContainer` is the data structure on every `UAbilitySystemComponent` that holds all currently active (non-instant) Gameplay Effects. When people say "the GE is active," they mean it lives here.

The container is responsible for:

- Tracking all active GE instances
- Managing their durations and periodic timers
- Evaluating modifier aggregation across all active effects
- Handling stacking logic
- Replicating active effects to clients
- Firing callbacks when effects are added, removed, or inhibited

### FActiveGameplayEffect

Each entry in the container is an `FActiveGameplayEffect`. This struct wraps:

```cpp
struct FActiveGameplayEffect : public FFastArraySerializerItem
{
    FActiveGameplayEffectHandle Handle;    // Unique identifier
    FGameplayEffectSpec Spec;              // The runtime spec (what GE, what level, what context)
    FPredictionKey PredictionKey;          // For client prediction
    float StartServerWorldTime;            // When this started on the server
    float StartWorldTime;                  // When this started locally
    bool bIsInhibited;                     // Is the effect currently dormant?

    // Timer handles for duration and period
    FTimerHandle PeriodHandle;
    FTimerHandle DurationHandle;

    // Granted ability handles
    TArray<FGameplayAbilitySpecHandle> GrantedAbilityHandles;
};
```

### FActiveGameplayEffectHandle

When a GE is applied, you get back an `FActiveGameplayEffectHandle`. This is a lightweight identifier that you can store and use later to query, modify, or remove the effect.

```cpp
// Apply a GE and store the handle
FActiveGameplayEffectHandle BuffHandle = ASC->ApplyGameplayEffectSpecToSelf(SpecHandle);

// Later, remove it
ASC->RemoveActiveGameplayEffect(BuffHandle);
```

The handle is globally unique (within a process) and can be used to look up the owning ASC. It is **not** replicated as-is — the server and client will have different handle values for the same logical effect. Replication uses the `FFastArraySerializer` infrastructure instead.

!!! warning "Handle Validity"
    Always check handle validity before using it. An invalid handle means the effect was never successfully applied (or was already removed).

    ```cpp
    if (BuffHandle.IsValid())
    {
        // Safe to use
    }
    ```

## Effect Application Flow

Here is the high-level flow when a Gameplay Effect is applied:

1. **Create Spec** — `MakeOutgoingGameplayEffectSpec` creates an `FGameplayEffectSpec` from the GE class, capturing source attributes and tags.

2. **CanApply Check** — Each `UGameplayEffectComponent` on the GE gets a `CanGameplayEffectApply` call. If any returns false, application is rejected. This is where tag requirements, chance-to-apply, and custom application logic run.

3. **Apply** — If all checks pass:
    - **Instant**: The effect is _executed_. Modifiers permanently alter base attribute values. Execution calculations run. The effect is never stored.
    - **Duration/Infinite**: The effect is _added_ to the Active GE Container as an `FActiveGameplayEffect`. Modifiers affect the current value of attributes while the effect is active.

4. **OnActiveGameplayEffectAdded** — For duration/infinite effects, each GE Component's `OnActiveGameplayEffectAdded` callback fires. This is where components set up their behavior (registering tag listeners, granting abilities, etc.).

5. **Inhibition Check** — Even after being added, the effect might be **inhibited** if ongoing tag requirements aren't met. An inhibited effect is in the container but dormant — its modifiers don't apply.

6. **Periodic Execution** — If the effect has a period, it executes at each interval.

7. **Removal** — The effect is removed when:
    - Its duration expires naturally
    - It's explicitly removed via code (`RemoveActiveGameplayEffect`)
    - A tag change triggers its removal requirements
    - Another effect removes it (via `RemoveOtherGameplayEffectComponent`)
    - The owning ASC is destroyed

## Inhibition

Inhibition is one of the more subtle concepts in GAS. An effect can be **added** to the container but **inhibited** — meaning it exists but does nothing. Its modifiers are not applied, its granted tags are not active, and its granted abilities are not usable.

Inhibition is driven by **ongoing tag requirements** (configured via `UTargetTagRequirementsGameplayEffectComponent`). When the target's tags change and no longer satisfy the ongoing requirements, the effect becomes inhibited. When the tags change back and the requirements are met again, the effect becomes uninhibited.

```
Example: A "Rage Mode" buff that requires the tag State.Enraged

1. State.Enraged is granted → Rage Mode uninhibits → modifiers apply
2. State.Enraged is removed → Rage Mode inhibits → modifiers stop applying
3. State.Enraged is granted again → Rage Mode uninhibits → modifiers apply again
```

This is different from removal. An inhibited effect is still in the container and still ticking its duration timer. It can come back. A removed effect is gone.

!!! note "Inhibition vs. Removal"
    Use ongoing tag requirements (inhibition) when an effect should toggle on and off based on conditions. Use removal tag requirements when the effect should be permanently removed.

See [Tags and Requirements](tags-and-requirements.md) for the full picture.

## Effect Removal

Effects can be removed in several ways:

### Duration Expiration

Duration effects naturally expire when their timer runs out. When this happens, the `OnCompleteNormal` effects from the `UAdditionalEffectsGameplayEffectComponent` will fire (if configured).

### Explicit Removal

```cpp
// Remove by handle
ASC->RemoveActiveGameplayEffect(Handle);

// Remove by handle with stack count
ASC->RemoveActiveGameplayEffect(Handle, StacksToRemove);

// Remove all effects matching a query
FGameplayEffectQuery Query = FGameplayEffectQuery::MakeQuery_MatchAnyOwningTags(TagContainer);
ASC->RemoveActiveEffectsWithGrantedTags(TagContainer);
```

### Tag-Based Removal

The `UTargetTagRequirementsGameplayEffectComponent` supports `RemovalTagRequirements` — if the target gains tags that match these requirements, the effect is removed outright (not just inhibited).

### Removal by Another Effect

The `URemoveOtherGameplayEffectComponent` lets an effect remove other active effects that match specific queries when it's applied. This is great for "dispel" mechanics.

### FGameplayEffectRemovalInfo

When an effect is removed, callbacks receive an `FGameplayEffectRemovalInfo` struct that tells you why it was removed and gives you access to the spec. The `UAdditionalEffectsGameplayEffectComponent` uses this to distinguish between normal expiration and premature removal.

## FGameplayEffectSpec

The spec is the **runtime instance data** of a Gameplay Effect. The `UGameplayEffect` class is an asset (immutable at runtime) — the spec wraps it with mutable runtime state:

- The GE definition it came from
- The level it was created at
- Who created it (effect context — instigator, causer, etc.)
- Captured attribute snapshots
- SetByCaller magnitudes
- Dynamic tags
- Computed modifier magnitudes
- Stack count

Specs are created via `MakeOutgoingGameplayEffectSpec` and can be modified before application. See [Dynamic Effects](dynamic-effects.md) for the full story on spec manipulation.

## What's Next?

Now that you understand how effects live and die, the next question is: what do they actually _do_ to attributes? That's covered in [Modifiers](modifiers.md).
