---
icon: material/puzzle
---

# GE Component Catalog

`UGameplayEffectComponent` subclasses are the modular building blocks of Gameplay Effects, introduced in UE 5.3. Each component adds a specific behavior to its owning GE. Components are native-only (no Blueprint subclassing) and stateless at runtime — they live on the GE CDO, not on individual instances.

For concepts, see [GE Components](../gameplay-effects/ge-components.md). For the base class API, see [UGameplayEffectComponent](#base-class).

---

## Component Summary

| Class | Display Name | Purpose | Key Callbacks |
|---|---|---|---|
| [`UAbilitiesGameplayEffectComponent`](#grant-gameplay-abilities) | Grant Gameplay Abilities | Grants abilities to the target while the GE is active | `OnActiveGameplayEffectAdded` |
| [`UAdditionalEffectsGameplayEffectComponent`](#apply-additional-effects) | Apply Additional Effects | Applies secondary GEs on application, completion, or premature removal | `OnActiveGameplayEffectAdded`, `OnGameplayEffectApplied` |
| [`UAssetTagsGameplayEffectComponent`](#asset-tags) | Tags This Effect Has (Asset Tags) | Tags the GE asset owns (used for queries, not granted to actors) | `OnGameplayEffectChanged` |
| [`UBlockAbilityTagsGameplayEffectComponent`](#block-abilities-with-tags) | Block Abilities with Tags | Prevents abilities with matching tags from activating while GE is active | `OnGameplayEffectChanged` |
| [`UCancelAbilityTagsGameplayEffectComponent`](#cancel-abilities-with-tags) | Cancel Abilities with Tags | Cancels active abilities with (or without) matching tags | `OnGameplayEffectApplied`, `OnGameplayEffectExecuted` |
| [`UChanceToApplyGameplayEffectComponent`](#chance-to-apply) | Chance To Apply This Effect | Adds a probability gate to GE application | `CanGameplayEffectApply` |
| [`UCustomCanApplyGameplayEffectComponent`](#custom-can-apply) | Custom Can Apply This Effect | Delegates application checks to custom `UGameplayEffectCustomApplicationRequirement` classes | `CanGameplayEffectApply` |
| [`UImmunityGameplayEffectComponent`](#immunity) | Immunity to Other Effects | Blocks application of other GEs matching query criteria while active | `OnActiveGameplayEffectAdded` |
| [`URemoveOtherGameplayEffectComponent`](#remove-other-effects) | Remove Other Effects | Removes other active GEs matching query criteria on application | `OnGameplayEffectApplied` |
| [`UTargetTagRequirementsGameplayEffectComponent`](#target-tag-requirements) | Require Tags to Apply/Continue This Effect | Controls application, ongoing validity, and removal based on target tags | `CanGameplayEffectApply`, `OnActiveGameplayEffectAdded` |
| [`UTargetTagsGameplayEffectComponent`](#grant-tags-to-target) | Grant Tags to Target Actor | Grants tags to the target actor while the GE is active | `OnGameplayEffectChanged` |

---

## Base Class

`UGameplayEffectComponent` (Abstract) provides the callback interface that subclasses override:

| Virtual Method | When Called | Return |
|---|---|---|
| `CanGameplayEffectApply()` | Before application — all components must return `true` | `bool` — `false` blocks application |
| `OnActiveGameplayEffectAdded()` | When a duration/infinite GE is added to the `FActiveGameplayEffectsContainer` (including via replication) | `bool` — `false` inhibits the effect |
| `OnGameplayEffectExecuted()` | When an instant GE executes (server only). Also fires each period for periodic effects. | `void` |
| `OnGameplayEffectApplied()` | When a GE is initially applied or stacked. Not periodic, not via replication. | `void` |
| `OnGameplayEffectChanged()` | When the owning GE asset is modified in the editor | `void` |

---

## Component Details

### Grant Gameplay Abilities

**Class:** `UAbilitiesGameplayEffectComponent`

Grants abilities to the target while the GE is active. Abilities are added when the GE becomes uninhibited and removed based on the configured `RemovalPolicy`.

| Property | Type | Description |
|---|---|---|
| `GrantAbilityConfigs` | `TArray<FGameplayAbilitySpecConfig>` | Array of abilities to grant |

Each `FGameplayAbilitySpecConfig` contains:

| Field | Type | Default | Description |
|---|---|---|---|
| `Ability` | `TSubclassOf<UGameplayAbility>` | -- | The ability class to grant |
| `LevelScalableFloat` | `FScalableFloat` | `1.0` | Level of the granted ability (supports [curve scaling](scalable-float.md)) |
| `InputID` | `int32` | `INDEX_NONE` | Input ID to bind the ability to |
| `RemovalPolicy` | `EGameplayEffectGrantedAbilityRemovePolicy` | `CancelAbilityImmediately` | What happens when the GE is removed |

!!! warning "Duration required"
    This component only makes sense on `HasDuration` or `Infinite` effects. Instant effects are never "active" and cannot grant abilities.

---

### Apply Additional Effects

**Class:** `UAdditionalEffectsGameplayEffectComponent`

Applies secondary Gameplay Effects under various conditions.

| Property | Type | Description |
|---|---|---|
| `bOnApplicationCopyDataFromOriginalSpec` | `bool` (default: `false`) | Copy SetByCaller data from the parent GE spec to OnApplication specs |
| `OnApplicationGameplayEffects` | `TArray<FConditionalGameplayEffect>` | GEs applied when this effect applies (supports conditions) |
| `OnCompleteAlways` | `TArray<TSubclassOf<UGameplayEffect>>` | GEs applied when this effect ends, regardless of how |
| `OnCompleteNormal` | `TArray<TSubclassOf<UGameplayEffect>>` | GEs applied only when this effect expires naturally |
| `OnCompletePrematurely` | `TArray<TSubclassOf<UGameplayEffect>>` | GEs applied only when this effect is removed early |

---

### Asset Tags

**Class:** `UAssetTagsGameplayEffectComponent`

Tags that the GE asset itself "has." These tags are used for GE queries (e.g., immunity checks, `FGameplayEffectQuery`) but are **not** granted to any actor.

| Property | Type | Description |
|---|---|---|
| `InheritableAssetTags` | `FInheritedTagContainer` | Asset tags with parent GE inheritance support |

---

### Block Abilities with Tags

**Class:** `UBlockAbilityTagsGameplayEffectComponent`

While this GE is active on a target, abilities with matching `AbilityTags` are blocked from activating.

| Property | Type | Description |
|---|---|---|
| `InheritableBlockedAbilityTagsContainer` | `FInheritedTagContainer` | Tags that block ability activation, with inheritance |

---

### Cancel Abilities with Tags

**Class:** `UCancelAbilityTagsGameplayEffectComponent`

Cancels currently active abilities based on tag matching. Supports two timing modes.

| Property | Type | Description |
|---|---|---|
| `ComponentMode` | `ECancelAbilityTagsGameplayEffectComponentMode` | `OnApplication` (once on apply) or `OnExecution` (each periodic tick) |
| `InheritableCancelAbilitiesWithTagsContainer` | `FInheritedTagContainer` | Cancel abilities that **have** these tags |
| `InheritableCancelAbilitiesWithoutTagsContainer` | `FInheritedTagContainer` | Cancel abilities that **do not have** these tags |

---

### Chance to Apply

**Class:** `UChanceToApplyGameplayEffectComponent`

Adds a random probability gate to GE application.

| Property | Type | Description |
|---|---|---|
| `ChanceToApplyToTarget` | `FScalableFloat` | Probability from 0.0 (never) to 1.0 (always). Supports [level scaling](scalable-float.md). |

---

### Custom Can Apply

**Class:** `UCustomCanApplyGameplayEffectComponent`

Delegates the "can this GE apply?" check to one or more custom C++ classes.

| Property | Type | Description |
|---|---|---|
| `ApplicationRequirements` | `TArray<TSubclassOf<UGameplayEffectCustomApplicationRequirement>>` | Custom requirement classes. All must return `true`. |

Each `UGameplayEffectCustomApplicationRequirement` subclass implements `CanApplyGameplayEffect()` with access to the GE definition, spec, and target ASC.

---

### Immunity

**Class:** `UImmunityGameplayEffectComponent`

While this GE is active, it blocks application of **other** GEs that match the configured queries. This is a global check registered on the ASC.

| Property | Type | Description |
|---|---|---|
| `ImmunityQueries` | `TArray<FGameplayEffectQuery>` | Queries matched against incoming GE specs. Any match blocks application. |

For concepts, see [Immunity](../gameplay-effects/immunity.md).

!!! warning "Duration required"
    Instant effects cannot grant immunity because they are not persistently active on the ASC.

---

### Remove Other Effects

**Class:** `URemoveOtherGameplayEffectComponent`

On application, removes any active GEs on the target that match the configured queries.

| Property | Type | Description |
|---|---|---|
| `RemoveGameplayEffectQueries` | `TArray<FGameplayEffectQuery>` | Queries matched against active GEs. Any match is removed. |

---

### Target Tag Requirements

**Class:** `UTargetTagRequirementsGameplayEffectComponent`

The most complex tag component. Controls three separate tag-based behaviors:

| Property | Type | Description |
|---|---|---|
| `ApplicationTagRequirements` | `FGameplayTagRequirements` | Target must meet these requirements for the GE to apply. Pass/fail at application time. |
| `OngoingTagRequirements` | `FGameplayTagRequirements` | While active, the GE is inhibited if the target stops meeting these requirements. Re-activates when met again. |
| `RemovalTagRequirements` | `FGameplayTagRequirements` | If the target meets these requirements, the GE is removed entirely (not just inhibited). Also prevents application. |

---

### Grant Tags to Target

**Class:** `UTargetTagsGameplayEffectComponent`

Grants tags to the target actor's owned tag set while the GE is active.

| Property | Type | Description |
|---|---|---|
| `InheritableGrantedTagsContainer` | `FInheritedTagContainer` | Tags granted to the target, with parent GE inheritance support |

These tags are removed when the GE expires or is removed.

## Related Pages

- [GE Components](../gameplay-effects/ge-components.md) -- concepts, the base class API, migration from monolithic properties, and how components interact
- [Tags and Requirements](../gameplay-effects/tags-and-requirements.md) -- how tag-based components control application, inhibition, and removal
