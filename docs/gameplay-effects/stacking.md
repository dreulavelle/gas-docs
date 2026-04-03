---
title: Stacking
description: How Gameplay Effects stack — stacking types, limits, overflow, duration refresh, period reset, and expiration policies with practical examples.
---

# Stacking

Stacking controls what happens when the same Gameplay Effect class is applied more than once. Does it create a separate instance each time? Does it increment a counter? Does it refresh the duration? The stacking configuration determines all of this.

!!! info "Stacking Only Applies to Duration/Infinite Effects"
    Instant effects are executed and gone — they never exist in the Active GE Container, so stacking doesn't apply to them. Stacking is strictly for effects that have a duration or are infinite.

## Stacking Types

The `EGameplayEffectStackingType` enum defines three modes:

### None (No Stacking)

```cpp
EGameplayEffectStackingType::None
```

Each application creates a **separate independent instance** in the Active GE Container. If you apply the same buff three times, you get three distinct active effects, each with its own duration timer and modifier contribution.

This is the default and the simplest. It's appropriate when you want effects to be fully independent — for example, separate DoT effects from different sources that all tick independently.

### AggregateBySource (Stack Per Source)

```cpp
EGameplayEffectStackingType::AggregateBySource
```

Multiple applications of the same GE **from the same source** are combined into a single instance with a stack count. Applications from different sources create separate instances.

**Example:** Player A applies Poison to a boss. Applying Poison again from Player A increments the stack. Player B also applying Poison creates a separate stack. The boss has two Poison instances — one from each source — each with their own stack count.

### AggregateByTarget (Stack Per Target)

```cpp
EGameplayEffectStackingType::AggregateByTarget
```

Multiple applications of the same GE on the **same target** are combined into a single instance, regardless of who applied it. Only one instance ever exists per target.

**Example:** Anyone applying Sunder Armor to a boss increments the same stack count. There's only ever one Sunder Armor instance on the boss, no matter how many different players are applying it.

## Stack Limit

`StackLimitCount` sets the maximum number of stacks. A value of 0 or -1 means **no limit** — stacks can grow indefinitely.

When an application would push the stack count above the limit, overflow handling kicks in (see below).

## Stacking Policies

When a stack is successfully added (not overflowing), several policies control what happens:

### Duration Refresh Policy

`EGameplayEffectStackingDurationPolicy` controls what happens to the duration timer when a new stack is added:

| Policy | Behavior |
|:---|:---|
| `RefreshOnSuccessfulApplication` | The duration resets to full on each successful stack application |
| `NeverRefresh` | The duration continues from its original start time, never resets |
| `ExtendDuration` | The new spec's duration is added onto the current remaining time |

### Period Reset Policy

`EGameplayEffectStackingPeriodPolicy` controls what happens to the periodic timer:

| Policy | Behavior |
|:---|:---|
| `ResetOnSuccessfulApplication` | Progress toward the next periodic tick is discarded; the timer restarts |
| `NeverReset` | The periodic timer continues undisturbed |

### Stack Expiration Policy

`EGameplayEffectStackingExpirationPolicy` controls what happens when the duration timer expires:

| Policy | Behavior |
|:---|:---|
| `ClearEntireStack` | The entire effect (all stacks) is removed |
| `RemoveSingleStackAndRefreshDuration` | One stack is removed and the duration resets. The effect slowly "unwinds" one stack at a time |
| `RefreshDuration` | The duration resets without removing any stacks. This effectively makes the effect infinite — you would handle stack removal manually via `OnStackCountChange` |

### bFactorInStackCount

When `true`, modifier magnitudes are automatically multiplied by the current stack count. When `false`, the stack count has no effect on magnitudes (you would handle scaling manually, perhaps in an [Execution Calculation](execution-calculations.md)).

## Overflow

When an application attempts to add a stack beyond `StackLimitCount`:

1. **OverflowEffects** — These GEs are applied to the target when overflow occurs. This fires whether or not the overflow application itself succeeds.

2. **bDenyOverflowApplication** — If `true`, the overflowing application is rejected. The duration and context are **not** refreshed. If `false`, the application "succeeds" (duration/context may refresh depending on policies) but the stack count stays at the limit.

3. **bClearStackOnOverflow** — If `true` (and `bDenyOverflowApplication` is also `true`), the entire stack is cleared on overflow. This creates a "burst" pattern: stacks build up to the limit, the next application clears them all and triggers an OverflowEffect.

## Practical Examples

### Stacking DoT (Poison Stacks)

A poison that stacks up to 5 times, with damage scaling per stack:

| Setting | Value |
|:---|:---|
| Duration Policy | Has Duration (10s) |
| Period | 1.0s |
| Stacking Type | AggregateByTarget |
| Stack Limit Count | 5 |
| Stack Duration Refresh | RefreshOnSuccessfulApplication |
| Stack Period Reset | NeverReset |
| Stack Expiration | ClearEntireStack |
| bFactorInStackCount | true |

The modifier does -5 HP per tick. At 3 stacks, it does -15 per tick. Reapplying refreshes the 10s timer. Maxes at 5 stacks.

### Non-Stacking Buff That Refreshes

A speed boost that doesn't stack but refreshes its duration when reapplied:

| Setting | Value |
|:---|:---|
| Duration Policy | Has Duration (5s) |
| Stacking Type | AggregateByTarget |
| Stack Limit Count | 1 |
| Stack Duration Refresh | RefreshOnSuccessfulApplication |
| Stack Expiration | ClearEntireStack |
| bFactorInStackCount | false |

This is a common pattern. Effectively, you get one instance of the buff that refreshes its timer every time it's reapplied. Using stacking with a limit of 1 is the standard way to implement "refresh on reapply" in GAS.

### Stacking Armor Debuff (Sunder)

An armor debuff that stacks per source, decays one stack at a time:

| Setting | Value |
|:---|:---|
| Duration Policy | Has Duration (8s) |
| Stacking Type | AggregateBySource |
| Stack Limit Count | 3 |
| Stack Duration Refresh | RefreshOnSuccessfulApplication |
| Stack Period Reset | NeverReset |
| Stack Expiration | RemoveSingleStackAndRefreshDuration |
| bFactorInStackCount | true |

Each source can apply up to 3 stacks. When the timer expires, one stack drops and the timer resets. The debuff gradually fades rather than disappearing all at once.

### Burst Pattern (Combo Finisher)

A charge mechanic where stacks build up, and on overflow the accumulated energy releases:

| Setting | Value |
|:---|:---|
| Duration Policy | Infinite |
| Stacking Type | AggregateByTarget |
| Stack Limit Count | 4 |
| bDenyOverflowApplication | true |
| bClearStackOnOverflow | true |
| Overflow Effects | GE_ComboFinisher (instant) |

Each hit adds a stack. On the 5th hit (overflow), all stacks are cleared and `GE_ComboFinisher` is applied. The finisher GE can read how many stacks were cleared if needed.

## Tips

!!! warning "Stacking + Prediction"
    Stacking can be tricky with client prediction. The client might predict a stack increment before the server confirms it, leading to brief visual inconsistencies. Test your stacking effects in multiplayer early.

!!! tip "Stack Count in ExecCalcs"
    Inside an [Execution Calculation](execution-calculations.md), you can read the stack count from `ExecutionParams.GetOwningSpec().GetStackCount()`. This lets you write custom scaling formulas that go beyond simple linear multiplication.

!!! note "No Stacking for Instant Effects"
    If you need "stacking" behavior for instant effects (like accumulating damage), you'll need to manage that state yourself — typically in an attribute or a custom gameplay effect component.
