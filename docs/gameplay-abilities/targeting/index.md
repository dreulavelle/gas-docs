---
title: Targeting
description: Overview of the GAS targeting system — target data, target actors, reticles, and filters.
---

# Targeting

GAS includes a full targeting subsystem for abilities that need player-driven or automated target selection. It handles everything from "click to select a target" to "aim at the ground and show a radius indicator." The system is built from three main pieces:

- **Target Data** — the payload that describes _what_ was targeted (actors, locations, hit results)
- **Target Actors** — the world actors that perform the actual targeting logic (traces, overlap checks, visual feedback)
- **Reticles and Filters** — visual feedback during targeting and rules for which actors are valid targets

## When to Use the Targeting System

The built-in targeting system is most useful when you need **interactive targeting** — the player aims, selects, confirms, and the ability reacts. Think MOBA-style ground targeting, RTS unit selection, or Diablo-style "point and click."

For simpler cases, you often do not need the full targeting system at all:

| Scenario | What To Use |
|:---|:---|
| Melee attack hitting what is in front of the character | Line/sphere trace in the ability, wrap results in `FGameplayAbilityTargetData_SingleTargetHit` |
| Projectile that applies effects on hit | Trace or overlap in the projectile class, send results as a gameplay event |
| AoE around the caster | `FGameplayAbilityTargetData_ActorArray` with actors from an overlap check |
| Interactive ground targeting with preview | Full targeting system: Target Actor + Reticle + WaitTargetData task |
| Player selects a target with a crosshair | `AGameplayAbilityTargetActor_SingleLineTrace` with a reticle |

!!! tip "Keep it simple"
    The targeting system is powerful but heavy — it spawns actors, manages replication, and has a multi-step confirm/cancel flow. If a simple trace in your `ActivateAbility` does the job, use that. Reach for the full system when you genuinely need interactive targeting with visual feedback.

## The Three Pieces

### Target Data — The Payload

`FGameplayAbilityTargetData` is a polymorphic struct that carries targeting results. It can hold hit results, actor lists, locations, or custom data. Target data is what flows from the targeting step to the effect application step.

[Read more about Target Data](target-data.md)

### Target Actors — The Targeting Logic

`AGameplayAbilityTargetActor` is a world actor spawned during targeting. It performs traces, overlap checks, or other targeting logic, and produces `FGameplayAbilityTargetData` when the player confirms. The built-in types cover line traces, ground traces, radius selection, and actor placement.

[Read more about Target Actors](target-actors.md)

### Reticles and Filters — Visuals and Validation

`AGameplayAbilityWorldReticle` provides visual feedback during targeting (the circle on the ground, the crosshair on the target). `FGameplayTargetDataFilter` controls which actors pass the targeting filter (exclude self, require a specific class, etc.).

[Read more about Reticles and Filters](reticles-and-filters.md)

## The Targeting Flow

Here is how the pieces fit together in a typical interactive targeting scenario:

```
1. Ability activates
2. Ability spawns a WaitTargetData task
     └─ Task spawns or references a Target Actor
3. Target Actor runs targeting logic each tick
     └─ Shows a Reticle for visual feedback
     └─ Applies a Filter to candidate targets
4. Player confirms (or cancels)
5. Target Actor produces FGameplayAbilityTargetData
6. WaitTargetData task fires ValidData (or Cancelled) delegate
7. Ability receives target data and applies effects
8. Ability ends
```

The `WaitTargetData` [ability task](../ability-tasks.md) is the bridge between the ability and the target actor. It handles spawning the target actor, binding to its confirm/cancel delegates, and forwarding the resulting data back to the ability.

## Targeting and Networking

Target data is designed to replicate. `FGameplayAbilityTargetDataHandle` implements `NetSerialize` and can be sent from client to server. The `ShouldProduceTargetDataOnServer` flag on target actors controls whether the server generates its own targeting data or trusts what the client sends.

For multiplayer games, you will typically:

- Let the client run the targeting visualization
- Send the confirmed target data to the server
- Have the server validate and apply effects

The `OnReplicatedTargetDataReceived` virtual on target actors gives the server a chance to validate/sanitize incoming target data.

## Suggested Reading Order

1. [Target Data](target-data.md) — understand the payload format first
2. [Target Actors](target-actors.md) — the actors that produce target data
3. [Reticles and Filters](reticles-and-filters.md) — visual feedback and target validation
