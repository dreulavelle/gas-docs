---
title: Instancing Policy
description: Understanding EGameplayAbilityInstancingPolicy — NonInstanced, InstancedPerActor, and InstancedPerExecution — and how your choice affects what an ability can do.
---

# Instancing Policy

Every Gameplay Ability has an **Instancing Policy** that controls how the ability object is allocated in memory when activated. This is not just a performance knob — it fundamentally determines what your ability can and cannot do. Choose wrong and you will hit confusing crashes, replication failures, or subtle state bugs. The policy affects everything from how the [ability lifecycle](lifecycle-and-activation.md) works to whether you can use [ability tasks](ability-tasks.md).

The policy is set via the `InstancingPolicy` property in the ability's constructor or in Blueprint defaults:

```cpp
InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
```

## The Three Policies

| Policy | Object Lifetime | State | Replication | Tasks | Recommended |
|:---|:---|:---|:---|:---|:---|
| `InstancedPerActor` | One instance per ASC, reused across activations | Yes — persists between activations | Yes (RPCs, replicated properties) | Yes | **Default choice** |
| `InstancedPerExecution` | New instance each activation | Yes — per activation only | Not supported | Yes | Niche use cases |
| `NonInstanced` | Operates on the CDO, never instanced | No | No (no RPCs, no replicated properties) | **No** | **Deprecated in 5.5** |

### InstancedPerActor (Recommended Default) { #instanced-per-actor }

This is the policy Epic recommends and the one you should reach for unless you have a specific reason not to. One instance of the ability is created when the ability is granted to an ASC, and that same instance is reused every time the ability activates.

**What you get:**

- Member variables that persist between activations (useful for counting activations, storing state)
- Full support for `UAbilityTask` (the primary mechanism for async ability logic)
- Replication support — you can declare `UPROPERTY(Replicated)` members and use RPCs
- Access to `GetCurrentActorInfo()`, `GetCurrentActivationInfo()`, and other instanced-only functions

**Things to be aware of:**

- Since the instance is reused, you must reset any per-activation state at the start of `ActivateAbility`
- Only one activation can be active at a time (per instance). The `ActiveCount` on the spec tracks this
- If you want to re-trigger an already-active ability, set `bRetriggerInstancedAbility = true` — this ends the current activation and immediately starts a new one

```cpp
UMyAbility::UMyAbility()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
}
```

### InstancedPerExecution { #instanced-per-execution }

A brand new instance is created every time the ability activates. When the ability ends, the instance is destroyed.

**What you get:**

- Clean state every activation — no need to worry about leftover data from previous activations
- Multiple simultaneous activations are possible (the spec's `ActiveCount` can be > 1)
- Full task support

**What you lose:**

- Replication is **not supported** — you cannot use replicated properties or RPCs on the ability instance
- Higher memory allocation cost (one `NewObject` per activation)
- No persistent state between activations

**When to use it:**

InstancedPerExecution is useful when you genuinely need multiple simultaneous instances of the same ability (rare) or when the conceptual cleanliness of a fresh state per activation outweighs the allocation cost. In most projects, InstancedPerActor handles these cases just fine with a state reset at the top of `ActivateAbility`.

!!! note "Historical Default"
    `InstancedPerExecution` was the original engine default in earlier UE versions. Many older tutorials and sample projects use it. If you are starting fresh, there is no reason to prefer it over `InstancedPerActor`.

### NonInstanced (Deprecated) { #non-instanced }

With `NonInstanced`, the ability is never instantiated. All logic runs on the **Class Default Object** (CDO). Every activation, every actor, shares the same object.

!!! warning "Deprecated in UE 5.5"
    `NonInstanced` is marked `UE_DEPRECATED_FORGAME(5.5)` in the engine source. Epic recommends migrating to `InstancedPerActor`. It still compiles but you should avoid it for new work.

**What you lose — almost everything:**

- **No member variable state.** The CDO is shared, so any member variable you write to will be overwritten by the next activation (or by another actor's activation).
- **No replication.** No replicated properties, no RPCs.
- **No Ability Tasks.** Tasks require an instanced ability to bind to. Using tasks with a NonInstanced ability will crash or silently fail.
- **No `GetCurrentActorInfo()`.** You must use the handle/actorinfo parameters passed to each function.

**What it was good for:**

NonInstanced abilities have near-zero memory overhead. In a world with thousands of NPCs each carrying dozens of abilities, that can matter. The engine's built-in `UGameplayAbility_Montage` is NonInstanced as an example.

**The common mistake:**

```cpp
// DO NOT DO THIS — will crash or corrupt state
UMyAbility::UMyAbility()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::NonInstanced;
}

void UMyAbility::ActivateAbility(...)
{
    // This task needs an instanced ability to bind to!
    auto* Task = UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
        this, NAME_None, MyMontage);  // CRASH or undefined behavior
    Task->ReadyForActivation();
}
```

If you find yourself needing tasks, switch to `InstancedPerActor`.

## Comparison Table

Here is a more detailed side-by-side:

| Capability | NonInstanced | InstancedPerActor | InstancedPerExecution |
|:---|:---:|:---:|:---:|
| Member variables | CDO only (shared!) | Per-actor, persists | Per-activation, fresh |
| Ability Tasks | No | Yes | Yes |
| Replicated properties | No | Yes | No |
| RPCs | No | Yes | No |
| Multiple simultaneous activations | N/A | No (one at a time) | Yes |
| `GetCurrentActorInfo()` | No | Yes | Yes |
| `GetCurrentAbilitySpec()` | No | Yes | Yes |
| Memory per granted ability | Zero (CDO) | One UObject per ASC | One UObject per activation |
| Cleanup on end | N/A | State persists | Instance destroyed |

## Choosing Your Policy

For the vast majority of abilities, the decision is straightforward:

```
Do you need tasks, state, or replication?
├─ Yes → InstancedPerActor (this is almost always the answer)
└─ No, and memory is critical (1000+ NPCs)
    └─ Even then, InstancedPerActor is fine. NonInstanced is deprecated.
```

Use `InstancedPerExecution` only if you have a concrete need for multiple simultaneous activations of the same ability class on the same actor. In practice, this is rare — combo attacks, multi-charge abilities, and similar patterns can usually be modeled as a single `InstancedPerActor` ability with internal state tracking.

!!! tip "Project-wide convention"
    Pick `InstancedPerActor` as your project default. Set it in a shared base class that all your abilities inherit from. This avoids the "forgot to set it" problem entirely:

    ```cpp
    UMyProjectAbilityBase::UMyProjectAbilityBase()
    {
        InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
        NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    }
    ```

## Related Pages

- [Lifecycle and Activation](lifecycle-and-activation.md) -- how instancing policy affects the activation flow, state management, and EndAbility
- [Ability Tasks](ability-tasks.md) -- tasks require an instanced ability and will not work with NonInstanced
- [Ability Sets](ability-sets.md) -- where instancing policy interacts with how abilities are granted and reused
