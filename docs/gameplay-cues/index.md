---
title: Gameplay Cues Deep Dive
description: Comprehensive guide to Gameplay Cues in UE 5.7 — cue types, parameters, the manager, and animation/sequencer integration.
---

# Gameplay Cues Deep Dive

Gameplay Cues are the cosmetic layer of GAS. They exist for one purpose: turning gameplay state changes into audiovisual feedback -- particles, sounds, screen shakes, camera effects, UI popups. They are the reason your fireball doesn't just subtract numbers silently.

If you haven't already, read the [Gameplay Cues Overview](../core-concepts/gameplay-cues-overview.md) first. This section assumes you understand the basic concepts -- Execute vs Add/Remove, tag routing, local vs replicated -- and goes much deeper.

!!! danger "The Golden Rule (One More Time)"
    **No gameplay logic in cues.** Your game must function identically with every cue disabled. If you're tempted to put a health check, a tag grant, or a damage calculation inside a cue, stop. That belongs in a Gameplay Effect or Ability.

## What's in This Section

### [Cue Notify Types](cue-types.md)

UE 5.7 ships **six** cue notify classes, not four. This page covers all of them in depth -- when to use each, their lifecycle events, key properties, and practical examples. Includes a decision matrix to pick the right one fast.

### [Cue Parameters](cue-parameters.md)

Every cue receives an `FGameplayCueParameters` struct with context about what triggered it. This page breaks down every field, explains how to pass custom data via a custom `FGameplayEffectContext`, and covers `IGameplayCueInterface` for handling cues directly on actors.

### [Cue Manager](cue-manager.md)

The `UGameplayCueManager` is the routing backbone. It decides which cue notify to invoke, handles async loading, manages the actor recycle pool, and supports batching. Learn how to configure scan paths, preload assets, use the `GameplayCueTranslator`, and override the manager for custom routing.

### [Cue Translator](cue-translator.md)

Dynamic cue tag remapping — show different VFX based on weapon type or context without separate cue assets.

### [AnimNotify and Sequencer](anim-notify-and-sequencer.md)

Cues can be triggered from Animation Montages and Level Sequences, not just from Effects and Abilities. This page covers `UAnimNotify_GameplayCue`, `UAnimNotify_GameplayCueState`, and the Sequencer gameplay cue track/sections.

## Architecture at a Glance

Here's how a cue flows through the system:

```
Gameplay Effect applied (or manual call)
    │
    ▼
UAbilitySystemComponent
    │ Routes based on replication mode
    ▼
UGameplayCueManager::HandleGameplayCue()
    │
    ├─ ShouldSuppressGameplayCues()    → can reject
    ├─ TranslateGameplayCue()          → tag remapping
    ├─ IGameplayCueInterface check     → actor-level handling
    └─ RouteGameplayCue()              → finds and invokes the right notify
        │
        ▼
    GameplayCueSet lookup by tag
        │
        ▼
    Cue Notify class (Static, Burst, Actor, etc.)
        │
        ▼
    Your Blueprint/C++ event handlers fire
```

The key insight: you rarely interact with the manager directly. You set a tag on your effect, create a cue notify with a matching tag, and the system connects them. The deep dive pages below are for when you need to customize that pipeline.

## Key Source Files

| File | What's Inside |
|:---|:---|
| `GameplayCueNotify_Static.h` | `UGameplayCueNotify_Static` -- non-instanced base |
| `GameplayCueNotify_Burst.h` | `UGameplayCueNotify_Burst` -- one-shot with spawn config |
| `GameplayCueNotify_BurstLatent.h` | `AGameplayCueNotify_BurstLatent` -- one-shot with latent support |
| `GameplayCueNotify_Actor.h` | `AGameplayCueNotify_Actor` -- stateful, recycled |
| `GameplayCueNotify_Looping.h` | `AGameplayCueNotify_Looping` -- duration-based loops |
| `GameplayCueNotify_HitImpact.h` | `UGameplayCueNotify_HitImpact` -- legacy, deprecated |
| `GameplayCueManager.h` | `UGameplayCueManager` -- routing, loading, recycling |
| `GameplayCueInterface.h` | `IGameplayCueInterface` -- actor-level cue handling |
| `GameplayCueNotifyTypes.h` | Spawn conditions, placement, burst/looping effect structs |
| `GameplayEffectTypes.h` | `FGameplayCueParameters` definition |
| `AnimNotify_GameplayCue.h` | Anim notify classes |
| `Sequencer/MovieSceneGameplayCueTrack.h` | Sequencer track |
| `Sequencer/MovieSceneGameplayCueSections.h` | Sequencer sections and key struct |
