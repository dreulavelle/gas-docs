---
title: Blueprint Function Library
description: Complete reference for UAbilitySystemBlueprintLibrary and UGameplayCueFunctionLibrary — every Blueprint-callable GAS function in one place.
---

# Blueprint Function Library

`UAbilitySystemBlueprintLibrary` is the static function library that exposes GAS operations to Blueprints. Every function here is a `BlueprintCallable` or `BlueprintPure` static method — meaning you can call them from any Blueprint without needing a reference to any particular object. This is the primary Blueprint API surface for GAS.

If you've ever searched for "how do I do X in GAS from Blueprint," the answer is usually somewhere in this class. This page documents every public function, organized by category.

!!! tip "Ctrl+F is your friend"
    This is a dense reference page by design. Use your browser's search to find the function you need.

---

## Getting the ASC

The starting point for most Blueprint GAS work — you need an Ability System Component before you can do anything.

| Function | Description | Returns |
|:---|:---|:---|
| `GetAbilitySystemComponent` | Finds the ASC on an actor. Checks the `IAbilitySystemInterface` first, then falls back to a component search. | `UAbilitySystemComponent*` |

This is a `BlueprintPure` node with `DefaultToSelf = "Actor"`, so when you drag it out in a Blueprint, it defaults to the owning actor. For player characters that implement `IAbilitySystemInterface`, this is extremely fast — it just calls `GetAbilitySystemComponent()` on the interface. The component search fallback handles cases where the interface isn't implemented.

---

## Gameplay Events

Send data-rich events that can trigger abilities with the `GameplayEvent` activation method.

| Function | Description | Key Parameters |
|:---|:---|:---|
| `SendGameplayEventToActor` | Sends a gameplay event to the actor's ASC. If the ASC has abilities listening for the event tag, they activate. | `Actor`, `EventTag` (filtered to GameplayEventTagsCategory), `Payload` (`FGameplayEventData`) |

The `Payload` struct carries optional data: `Instigator`, `Target`, `TargetData`, `EventMagnitude`, `OptionalObject`, and tag containers. Abilities activated by events receive this payload in their `ActivateAbility` override.

!!! note
    If the actor has no ASC, the event silently does nothing. No error, no warning — it just returns.

---

## Tag Operations

Blueprint functions for querying and manipulating gameplay tags on an ASC. These are the Blueprint equivalents of `AddLooseGameplayTag`, `RemoveLooseGameplayTag`, etc.

### Adding and Removing Tags

| Function | Description | Key Parameters |
|:---|:---|:---|
| `AddLooseGameplayTags` | Adds tags to the actor's ASC. Optionally replicates. | `Actor`, `GameplayTags`, `bShouldReplicate` (default `false`) |
| `RemoveLooseGameplayTags` | Removes tags from the actor's ASC. Optionally replicates. | `Actor`, `GameplayTags`, `bShouldReplicate` (default `false`) |
| `AddGameplayTags` | Adds tags with explicit replication control via `EGameplayTagReplicationState`. | `Actor`, `GameplayTags`, `ReplicationRule` (default `None`) |
| `RemoveGameplayTags` | Removes tags with explicit replication control. | `Actor`, `GameplayTags`, `ReplicationRule` (default `None`) |

!!! warning "Deprecated vs Current"
    `AddLooseGameplayTags` / `RemoveLooseGameplayTags` use a boolean for replication control. `AddGameplayTags` / `RemoveGameplayTags` use `EGameplayTagReplicationState` which provides finer-grained control. Prefer the latter in new code.

### Tag Change Event Wrappers

These functions let Blueprints bind to tag change events on an ASC. This is a major quality-of-life feature in UE 5.7 — previously, listening for tag changes from Blueprint required custom C++ glue code or event dispatchers.

| Function | Description | Returns |
|:---|:---|:---|
| `BindEventWrapperToGameplayTagChanged` | Binds a delegate to fire when a single tag changes on the ASC. | `FGameplayTagChangedEventWrapperSpecHandle` |
| `BindEventWrapperToAnyOfGameplayTagsChanged` | Binds a delegate to fire when *any* tag in an array changes. | `FGameplayTagChangedEventWrapperSpecHandle` |
| `BindEventWrapperToAnyOfGameplayTagContainerChanged` | Same as above, but accepts an `FGameplayTagContainer` instead of an array. | `FGameplayTagChangedEventWrapperSpecHandle` |
| `UnbindAllGameplayTagChangedEventWrappersForHandle` | Unbinds all tag change events tied to a handle. | `void` |
| `UnbindGameplayTagChangedEventWrapperForHandle` | Unbinds a specific tag's event from a multi-tag handle. | `void` |

All bind functions accept these advanced parameters:

- `bExecuteImmediatelyIfTagApplied` (default `true`) — if the tag is already present when you bind, the delegate fires immediately
- `TagListeningPolicy` (default `NewOrRemoved`) — controls whether you hear every count change (`AnyCountChange`) or just presence transitions (`NewOrRemoved`)

Cache the returned `FGameplayTagChangedEventWrapperSpecHandle` and call one of the unbind functions when you're done listening.

---

## Attribute Access

Read attribute values from Blueprints. These are the primary way to display health bars, mana pools, and other attribute-driven UI.

| Function | Description | Returns |
|:---|:---|:---|
| `GetFloatAttribute` | Gets the current value of an attribute from an actor. | `float` (+ `bSuccessfullyFoundAttribute` out param) |
| `GetFloatAttributeFromAbilitySystemComponent` | Same, but takes an ASC directly instead of an actor. | `float` (+ `bSuccessfullyFoundAttribute` out param) |
| `GetFloatAttributeBase` | Gets the **base** value (before modifiers) from an actor. | `float` (+ `bSuccessfullyFoundAttribute` out param) |
| `GetFloatAttributeBaseFromAbilitySystemComponent` | Base value, taking an ASC directly. | `float` (+ `bSuccessfullyFoundAttribute` out param) |
| `EvaluateAttributeValueWithTags` | Evaluates the attribute with source/target tag context. | `float` (+ `bSuccess` out param) |
| `EvaluateAttributeValueWithTagsAndBase` | Same, but substitutes a custom base value instead of the real one. | `float` (+ `bSuccess` out param) |
| `IsValid` | Returns whether an `FGameplayAttribute` handle is valid. | `bool` |
| `EqualEqual_GameplayAttributeGameplayAttribute` | Equality check (`==` node). | `bool` |
| `NotEqual_GameplayAttributeGameplayAttribute` | Inequality check (`!=` node). | `bool` |
| `GetDebugStringFromGameplayAttribute` | Returns `"AttrSetName.AttrName"` for debugging. | `FString` |

!!! tip "Current vs Base"
    `GetFloatAttribute` returns the *current* value — base + all active modifiers. `GetFloatAttributeBase` returns the value before modifiers. For UI, you almost always want the current value. For calculating "how much of their max HP do they have left?", you may want the base.

---

## Target Data

Functions for creating, querying, and filtering `FGameplayAbilityTargetDataHandle` — the container GAS uses to pass targeting information between abilities, tasks, and the server.

### Creating Target Data

| Function | Description | Returns |
|:---|:---|:---|
| `AbilityTargetDataFromLocations` | Creates target data from source + destination locations. | `FGameplayAbilityTargetDataHandle` |
| `AbilityTargetDataFromHitResult` | Creates target data from a single `FHitResult`. | `FGameplayAbilityTargetDataHandle` |
| `AbilityTargetDataFromActor` | Creates target data pointing at a single actor. | `FGameplayAbilityTargetDataHandle` |
| `AbilityTargetDataFromActorArray` | Creates target data from an array of actors. `OneTargetPerHandle` controls whether they're packed into one or split. | `FGameplayAbilityTargetDataHandle` |
| `AppendTargetDataHandle` | Copies targets from one handle into another. | `FGameplayAbilityTargetDataHandle` |

### Querying Target Data

| Function | Description | Returns |
|:---|:---|:---|
| `GetDataCountFromTargetData` | Number of target data objects in the handle (not necessarily distinct targets). | `int32` |
| `GetActorsFromTargetData` | All actors targeted at a specific index. | `TArray<AActor*>` |
| `GetAllActorsFromTargetData` | All actors across all indices. | `TArray<AActor*>` |
| `DoesTargetDataContainActor` | Checks if a specific actor is targeted at an index. | `bool` |
| `TargetDataHasActor` | Whether the target data at an index has at least one actor. | `bool` |
| `TargetDataHasHitResult` | Whether the target data at an index has a hit result. | `bool` |
| `GetHitResultFromTargetData` | Gets the hit result at an index. | `FHitResult` |
| `TargetDataHasOrigin` | Whether the target data at an index has an origin. | `bool` |
| `GetTargetDataOrigin` | Gets the origin transform at an index. | `FTransform` |
| `TargetDataHasEndPoint` | Whether the target data at an index has an end point. | `bool` |
| `GetTargetDataEndPoint` | Gets the end point location at an index. | `FVector` |
| `GetTargetDataEndPointTransform` | Gets the end point as a full transform. | `FTransform` |

### Filtering

| Function | Description | Returns |
|:---|:---|:---|
| `FilterTargetData` | Returns a new handle with targets filtered by an `FGameplayTargetDataFilterHandle`. | `FGameplayAbilityTargetDataHandle` |
| `MakeFilterHandle` | Creates a filter handle from an `FGameplayTargetDataFilter` and a filter actor. | `FGameplayTargetDataFilterHandle` |

---

## Effect Spec Manipulation

Functions for creating and modifying `FGameplayEffectSpecHandle` before application. This is how you configure dynamic effect behavior in Blueprints — SetByCaller magnitudes, dynamic tags, stack counts, etc.

### Creating Specs

| Function | Description | Key Parameters |
|:---|:---|:---|
| `MakeSpecHandleByClass` | Creates an `FGameplayEffectSpecHandle` from a GE class. The preferred way to create specs in Blueprint. | `GameplayEffect` (class), `Instigator`, `EffectCauser`, `Level` |
| `MakeSpecHandle` | :material-alert: **Deprecated (5.5)** — use `MakeSpecHandleByClass` instead. Takes a GE object (must be CDO). | `InGameplayEffect`, `InInstigator`, `InEffectCauser`, `InLevel` |
| `CloneSpecHandle` | Clones an existing spec with a new instigator and effect causer. | `InNewInstigator`, `InEffectCauser`, `GameplayEffectSpecHandle_Clone` |

### Modifying Specs

| Function | Description | Returns |
|:---|:---|:---|
| `AssignTagSetByCallerMagnitude` | Sets a SetByCaller magnitude by gameplay tag. **This is the standard way to set SetByCaller values.** | `FGameplayEffectSpecHandle` |
| `AssignSetByCallerMagnitude` | Sets a SetByCaller magnitude by raw `FName`. Prefer the tag version. | `FGameplayEffectSpecHandle` |
| `SetDuration` | Manually overrides the duration on the spec. | `FGameplayEffectSpecHandle` |
| `AddGrantedTag` | Dynamically adds a single tag to the spec's granted tags. | `FGameplayEffectSpecHandle` |
| `AddGrantedTags` | Dynamically adds multiple tags to the spec's granted tags. | `FGameplayEffectSpecHandle` |
| `AddAssetTag` | Adds a single asset tag to the spec. | `FGameplayEffectSpecHandle` |
| `AddAssetTags` | Adds multiple asset tags to the spec. | `FGameplayEffectSpecHandle` |
| `SetStackCount` | Sets the spec's initial stack count (before application). | `FGameplayEffectSpecHandle` |
| `SetStackCountToMax` | Sets the spec's stack count to the GE's configured maximum. | `FGameplayEffectSpecHandle` |

### Querying Specs

| Function | Description | Returns |
|:---|:---|:---|
| `GetGrantedTags` | Returns all granted tags (asset + dynamic) from a spec. | `FGameplayTagContainer` |
| `GetAssetTags` | Returns all asset tags from a spec. | `FGameplayTagContainer` |
| `GetEffectContext` | Gets the `FGameplayEffectContextHandle` from a spec. | `FGameplayEffectContextHandle` |
| `GetModifiedAttributeMagnitude` | Gets the computed magnitude of change for an attribute from an *applied* spec. | `float` |
| `GetGameplayEffectFromSpecHandle` | Gets the GE CDO from a spec handle. | `const UGameplayEffect*` |
| `IsDurationGameplayEffectSpecHandle` | Returns `true` if the spec is for a Duration or Infinite effect (not Instant). | `bool` |
| `GetDurationPolicyFromGameplayEffectSpecHandle` | Returns the `EGameplayEffectDurationType` from the spec. | `EGameplayEffectDurationType` |

### Deprecated Linked Effect Functions

These were deprecated in UE 5.3. Use `UAdditionalGameplayEffectsComponent` on the GE asset instead.

| Function | Status |
|:---|:---|
| `AddLinkedGameplayEffectSpec` | :material-alert: Deprecated 5.3 |
| `AddLinkedGameplayEffect` | :material-alert: Deprecated 5.3 |
| `GetAllLinkedGameplayEffectSpecHandles` | :material-alert: Deprecated 5.3 |

---

## Active Effect Queries

Functions for querying effects that are currently active on an ASC. Use these to build buff bars, check remaining durations, and inspect active effect state.

| Function | Description | Returns |
|:---|:---|:---|
| `IsActiveGameplayEffectHandleValid` | Whether the handle has a valid value (even if the effect expired). | `bool` |
| `IsActiveGameplayEffectHandleActive` | Whether the handle points to an effect that is *currently active*. | `bool` |
| `GetAbilitySystemComponentFromActiveGameplayEffectHandle` | Gets the owning ASC from an active effect handle. | `UAbilitySystemComponent*` |
| `GetActiveGameplayEffectStackCount` | Current stack count. Returns 0 if the effect is no longer valid. | `int32` |
| `GetActiveGameplayEffectStackLimitCount` | Maximum stack count configured on the GE. | `int32` |
| `GetActiveGameplayEffectStartTime` | World time when the effect was added. | `float` |
| `GetActiveGameplayEffectExpectedEndTime` | Predicted expiration time. Can change if duration is modified. | `float` |
| `GetActiveGameplayEffectTotalDuration` | Total configured duration. | `float` |
| `GetActiveGameplayEffectRemainingDuration` | Time left until expiration (needs a `WorldContextObject`). | `float` |
| `GetActiveGameplayEffectDebugString` | Human-readable debug string. | `FString` |
| `GetGameplayEffectFromActiveEffectHandle` | Gets the GE CDO from an active effect. Read-only — use for icons, descriptions, etc. | `const UGameplayEffect*` |
| `EqualEqual_ActiveGameplayEffectHandle` | Equality check (`==` node). | `bool` |
| `NotEqual_ActiveGameplayEffectHandle` | Inequality check (`!=` node). | `bool` |

### UI Data Access

| Function | Description | Returns |
|:---|:---|:---|
| `GetGameplayEffectUIData` | Returns the `UGameplayEffectUIData` component from a GE class. The `DataType` parameter filters to a specific subclass. | `const UGameplayEffectUIData*` |

See [Effect UI Data](../gameplay-effects/ui-data.md) for details on the UI Data system.

---

## Gameplay Effect Asset Queries

Query the GE *class* (CDO) directly, without needing an active instance.

| Function | Description | Returns |
|:---|:---|:---|
| `GetGameplayEffectAssetTags` | Returns the GE's own asset tags (tags *on* the GE, not tags it grants). | `const FGameplayTagContainer&` |
| `GetGameplayEffectGrantedTags` | Returns tags the GE grants to the target actor. | `const FGameplayTagContainer&` |

---

## Effect Context

Functions for reading and modifying the `FGameplayEffectContextHandle` — the struct that carries metadata about who caused an effect, where it came from, and what hit.

| Function | Description | Returns |
|:---|:---|:---|
| `EffectContextIsValid` | Whether the context has been initialized. | `bool` |
| `EffectContextIsInstigatorLocallyControlled` | Whether the instigating ASC is locally controlled. | `bool` |
| `EffectContextGetHitResult` | Extracts the hit result from the context. | `FHitResult` |
| `EffectContextHasHitResult` | Whether the context contains a hit result. | `bool` |
| `EffectContextAddHitResult` | Adds a hit result to the context. `bReset` clears any existing hit result. | `void` |
| `EffectContextGetOrigin` | Gets the world location where the effect originated. | `FVector` |
| `EffectContextSetOrigin` | Sets the origin location. | `void` |
| `EffectContextGetInstigatorActor` | The actor that owns the instigating ASC. | `AActor*` |
| `EffectContextGetOriginalInstigatorActor` | The actor that *originally* started the chain (may differ from instigator for chained effects). | `AActor*` |
| `EffectContextGetEffectCauser` | The physical actor that caused the effect (e.g., a projectile, weapon). | `AActor*` |
| `EffectContextGetSourceObject` | The source object of the effect (often the ability CDO). | `UObject*` |

---

## Gameplay Cue Helpers

Functions for working with `FGameplayCueParameters` inside Gameplay Cue Blueprints. When you implement a cue notify, these help you extract useful data from the cue parameters.

| Function | Description | Returns |
|:---|:---|:---|
| `IsInstigatorLocallyControlled` | Whether the ability that triggered this cue is locally controlled. | `bool` |
| `IsInstigatorLocallyControlledPlayer` | Same, but also checks if it's a player (not AI). | `bool` |
| `GetActorCount` | Number of actors stored in the effect context. | `int32` |
| `GetActorByIndex` | Gets an actor from the context by index. | `AActor*` |
| `GetHitResult` | Extracts a hit result from the cue parameters. | `FHitResult` |
| `HasHitResult` | Whether the cue parameters contain a hit result. | `bool` |
| `ForwardGameplayCueToTarget` | Forwards the cue to another `IGameplayCueInterface` implementer. | `void` |
| `GetInstigatorActor` | The instigating actor. | `AActor*` |
| `GetInstigatorTransform` | The instigator's world transform. | `FTransform` |
| `GetOrigin` | The origin location. | `FVector` |
| `GetGameplayCueEndLocationAndNormal` | Best end location and normal — falls back from hit result to target actor. | `bool` (+ `Location`, `Normal` out params) |
| `GetGameplayCueDirection` | Best normalized direction from source to target. Useful for directional effects. | `bool` (+ `Direction` out param) |
| `DoesGameplayCueMeetTagRequirements` | Checks aggregated source/target tags against tag requirements. | `bool` |
| `MakeGameplayCueParameters` | Constructs a full `FGameplayCueParameters` struct from individual values. | `FGameplayCueParameters` |
| `BreakGameplayCueParameters` | Breaks a `FGameplayCueParameters` into individual values. The Blueprint "Break" node. | `void` (many out params) |

---

## Gameplay Ability Queries

| Function | Description | Returns |
|:---|:---|:---|
| `GetGameplayAbilityFromSpecHandle` | Gets the ability object from a spec handle. Sets `bIsInstance` to indicate whether it's an instanced ability or the CDO. | `const UGameplayAbility*` |
| `IsGameplayAbilityActive` | Whether an ability instance is currently active (activated, not yet ended). | `bool` |
| `HasAnyAbilitiesWithAssetTag` | Whether the ASC has any granted abilities with a specific asset tag. | `bool` |
| `EqualEqual_GameplayAbilitySpecHandle` | Equality check (`==` node) for ability spec handles. | `bool` |
| `NotEqual_GameplayAbilitySpecHandle` | Inequality check (`!=` node) for ability spec handles. | `bool` |

---

## Scalable Float Conversion

| Function | Description | Returns |
|:---|:---|:---|
| `Conv_ScalableFloatToFloat` | Converts an `FScalableFloat` to a `float` at a given level. Auto-cast node. | `float` |
| `Conv_ScalableFloatToDouble` | Same, but returns `double` precision. | `double` |

These are Blueprint auto-cast nodes — when you connect a Scalable Float pin to a float pin, the engine inserts this conversion automatically. See [Scalable Floats](scalable-float.md) for more on the `FScalableFloat` type.

---

## UGameplayCueFunctionLibrary

A separate, smaller Blueprint function library specifically for Gameplay Cues. These functions provide a clean way to fire cues directly on actors — especially useful when you want to trigger cues outside of ability/effect code.

| Function | Description | Key Parameters |
|:---|:---|:---|
| `MakeGameplayCueParametersFromHitResult` | Builds cue parameters from a hit result — populates location, normal, physical material, etc. | `HitResult` |
| `ExecuteGameplayCueOnActor` | Fires a one-shot "burst" cue on the target. If the actor has an ASC, it's authority-only and replicated. Without an ASC, it fires locally only. | `Target`, `GameplayCueTag`, `Parameters` |
| `AddGameplayCueOnActor` | Starts a looping cue on the target. Must be paired with `RemoveGameplayCueOnActor`. Same replication behavior as above. | `Target`, `GameplayCueTag`, `Parameters` |
| `RemoveGameplayCueOnActor` | Ends a looping cue started with `AddGameplayCueOnActor`. | `Target`, `GameplayCueTag`, `Parameters` |

!!! tip "When to use these vs ASC functions"
    The ASC has its own `ExecuteGameplayCue`, `AddGameplayCue`, and `RemoveGameplayCue` methods. The function library versions are useful when you want to fire cues on actors that *might not have an ASC* — the library handles the fallback to local-only execution automatically.

---

## Further Reading

- [Gameplay Effects Overview](../core-concepts/gameplay-effects-overview.md) — context for the effect-related functions
- [Target Data](../gameplay-abilities/targeting/target-data.md) — deep dive on the target data system
- [SetByCaller](../gameplay-effects/set-by-caller.md) — how `AssignTagSetByCallerMagnitude` fits into the bigger picture
- [Effect UI Data](../gameplay-effects/ui-data.md) — the `GetGameplayEffectUIData` function in practice
- [Scalable Floats](scalable-float.md) — the type behind the auto-cast nodes
