---
title: GE Components (5.3+)
description: The modular UGameplayEffectComponent architecture introduced in UE 5.3 — how it works, all 11 built-in components, writing custom components, and migration from monolithic GEs.
---

# GE Components (5.3+)

Starting in Unreal Engine 5.3, Gameplay Effects moved from a monolithic class with dozens of properties to a **modular component-based architecture**. Instead of configuring everything through flat properties on `UGameplayEffect`, you now add `UGameplayEffectComponent` subclasses that each handle one specific behavior.

This is one of the most significant architectural changes to GAS in recent years, and it's not well-documented elsewhere. Let's fix that.

## Why the Change?

The old `UGameplayEffect` class had grown unwieldy. Tag requirements, granted tags, blocked ability tags, immunity queries, granted abilities, additional effects, custom application requirements — all of it was crammed into one class as flat properties. This created several problems:

- **Hard to extend** — Adding new behavior meant modifying the engine class or creating fragile subclasses
- **Hard to read** — Opening a GE in the editor showed dozens of properties, most of which were irrelevant to any particular effect
- **Hard to maintain** — All the logic for different features was tangled together in the GE's application and execution code

The component system solves these problems by decomposing behavior into discrete, self-contained classes. Each component registers for the callbacks it needs and handles its own logic. The `UGameplayEffect` class itself becomes a thin container.

## How It Works

### Architecture

`UGameplayEffectComponent` is an abstract base class. Components are **instanced subobjects** that live _within_ a `UGameplayEffect` (the GE is their Outer). Since GE assets are typically data-only Blueprints, components are configured in the editor — you add them to the `GEComponents` array on the GE.

From the source:

```cpp
UCLASS(Abstract, Const, DefaultToInstanced, EditInlineNew, CollapseCategories,
       Within=GameplayEffect, MinimalAPI)
class UGameplayEffectComponent : public UObject
```

Key points:

- Components are **Const** — they are asset data, not runtime instances. One component exists for all applied instances of that GE.
- Components are **native-only** for now. You write custom components in C++, not Blueprint. (The callback registration mechanism requires native code.)
- Components should **not store mutable runtime state**. Since one component instance serves all active instances of the GE, there's no per-instance storage on the component itself.

### Callback-Based Design

Rather than a large virtual API, components opt into behavior by overriding a small set of callbacks:

| Callback | When It Fires |
|:---|:---|
| `CanGameplayEffectApply` | Before application — return `false` to block |
| `OnActiveGameplayEffectAdded` | When a duration/infinite GE is added to the Active GE Container (including via replication). Return `false` to inhibit |
| `OnGameplayEffectExecuted` | When an instant or periodic GE executes (authority only) |
| `OnGameplayEffectApplied` | When a GE is first applied or stacked (not on replication, not periodic) |
| `OnGameplayEffectChanged` | When the owning GE asset is modified (editor-time) |

Components typically register additional delegates from within these callbacks — for example, registering for tag change events in `OnActiveGameplayEffectAdded` and cleaning up in an `OnRemoved` callback.

### The GEComponents Array

On `UGameplayEffect`, components live in:

```cpp
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Instanced, Category = "GameplayEffect")
TArray<TObjectPtr<UGameplayEffectComponent>> GEComponents;
```

In C++, you can find or add components:

```cpp
// Find a component
const UTargetTagsGameplayEffectComponent* TagsComp = MyGE->FindComponent<UTargetTagsGameplayEffectComponent>();

// Add a component (typically only during construction/migration)
UAbilitiesGameplayEffectComponent& AbilitiesComp = MyGE->FindOrAddComponent<UAbilitiesGameplayEffectComponent>();
```

## Built-In Components

UE 5.7 ships with 11 built-in components. Here's every one of them.

---

### AssetTagsGameplayEffectComponent

**Display Name:** "Tags This Effect Has (Asset Tags)"

Tags that the GE itself _has_. These do **not** transfer to the target actor — they are for querying and matching. Other systems can ask "does the target have any active effects with tag `Buff.Fire`?" and these tags are what gets checked.

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `InheritableAssetTags` | `FInheritedTagContainer` | Asset tags, with inheritance support from parent GE classes |

**Example:** Tag your damage GE with `Damage.Fire` so an immunity component on another effect can match against it.

---

### TargetTagsGameplayEffectComponent

**Display Name:** "Grant Tags to Target Actor"

Tags that are **granted to the target** while this GE is active. When the effect is removed, the tags are removed.

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `InheritableGrantedTagsContainer` | `FInheritedTagContainer` | Tags to grant, with inheritance support |

**Example:** A stun effect grants `State.Stunned` to the target. Abilities that check for that tag can react accordingly.

!!! warning "Duration Only"
    Granting tags only makes sense for duration/infinite effects. Instant effects can't grant tags because they don't persist. The editor will warn you if you misconfigure this.

---

### TargetTagRequirementsGameplayEffectComponent

**Display Name:** "Require Tags to Apply/Continue This Effect"

Controls when the effect can apply and whether it should stay active based on the target's tags. This is the component that drives [inhibition](duration-and-lifecycle.md#inhibition).

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `ApplicationTagRequirements` | `FGameplayTagRequirements` | Tags the target must/must not have at application time. Fail = GE doesn't apply. |
| `OngoingTagRequirements` | `FGameplayTagRequirements` | Tags the target must/must not have while active. Fail = GE becomes inhibited. |
| `RemovalTagRequirements` | `FGameplayTagRequirements` | If the target's tags match these requirements, the GE is removed entirely. |

**Example:** A regeneration buff that requires `State.InCombat` to be absent. When the player enters combat, the regen inhibits. When they leave combat, it uninhibits. If they gain `State.Dead`, a removal requirement removes it permanently.

See [Tags and Requirements](tags-and-requirements.md) for extensive coverage.

---

### BlockAbilityTagsGameplayEffectComponent

**Display Name:** "Block Abilities with Tags"

While this GE is active, abilities with matching tags on the target are **blocked from activating**.

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `InheritableBlockedAbilityTagsContainer` | `FInheritedTagContainer` | Ability tags to block, with inheritance support |

**Example:** A silence effect that blocks all abilities tagged `Ability.Magic`.

---

### CancelAbilityTagsGameplayEffectComponent

**Display Name:** "Cancel Abilities with Tags"

Cancels currently active abilities on the target based on tags. Supports two modes:

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `ComponentMode` | `ECancelAbilityTagsGameplayEffectComponentMode` | `OnApplication` or `OnExecution` |
| `InheritableCancelAbilitiesWithTagsContainer` | `FInheritedTagContainer` | Cancel abilities that _have_ these tags |
| `InheritableCancelAbilitiesWithoutTagsContainer` | `FInheritedTagContainer` | Cancel abilities that _don't have_ these tags |

The `OnApplication` mode cancels abilities once when the GE is applied. The `OnExecution` mode cancels abilities on each periodic execution — useful for a recurring interrupt effect.

**Example:** A hard CC effect that cancels all active channeled abilities on application.

---

### AbilitiesGameplayEffectComponent

**Display Name:** "Grant Gameplay Abilities"

Grants abilities to the target while the GE is active. Abilities are removed when the GE is removed (based on the configured removal policy).

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `GrantAbilityConfigs` | `TArray<FGameplayAbilitySpecConfig>` | List of abilities to grant |

Each `FGameplayAbilitySpecConfig` specifies:

- `Ability` — The ability class to grant
- `LevelScalableFloat` — What level to grant it at (defaults to 1.0)
- `InputID` — Optional input binding
- `RemovalPolicy` — What happens when the GE is removed: `CancelAbilityImmediately`, `RemoveAbilityOnEnd`, or `DoNothing`

**Example:** An equipment effect that grants a special parry ability while the shield is equipped.

!!! info "Inhibition-Aware"
    Granted abilities are removed when the GE is inhibited and re-granted when it's uninhibited.

---

### AdditionalEffectsGameplayEffectComponent

**Display Name:** "Apply Additional Effects"

Applies other GEs when this effect is applied or completes.

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `bOnApplicationCopyDataFromOriginalSpec` | `bool` | Copy SetByCaller data to the additional effect specs (defaults to `false` for backwards compatibility) |
| `OnApplicationGameplayEffects` | `TArray<FConditionalGameplayEffect>` | GEs to apply when this effect is applied (with optional source tag requirements) |
| `OnCompleteAlways` | `TArray<TSubclassOf<UGameplayEffect>>` | GEs to apply when this effect ends, regardless of how |
| `OnCompleteNormal` | `TArray<TSubclassOf<UGameplayEffect>>` | GEs to apply only on natural expiration |
| `OnCompletePrematurely` | `TArray<TSubclassOf<UGameplayEffect>>` | GEs to apply only on premature removal |

**Example:** A "Berserk" buff that applies a damage bonus on application and a fatigue debuff when it expires normally.

---

### ChanceToApplyGameplayEffectComponent

**Display Name:** "Chance To Apply This Effect"

Adds a probability gate to effect application.

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `ChanceToApplyToTarget` | `FScalableFloat` | Probability (0.0 = never, 1.0 = always) |

**Example:** A 25% chance to apply a critical bleed effect on hit.

---

### CustomCanApplyGameplayEffectComponent

**Display Name:** "Custom Can Apply This Effect"

Delegates the application check to one or more custom `UGameplayEffectCustomApplicationRequirement` classes.

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `ApplicationRequirements` | `TArray<TSubclassOf<UGameplayEffectCustomApplicationRequirement>>` | Custom classes that decide if the GE can apply |

**Example:** A buff that should only apply if the target has a specific item in their inventory — logic that can't be expressed with tags alone.

---

### ImmunityGameplayEffectComponent

**Display Name:** "Immunity to Other Effects"

While active, this GE blocks the application of other GEs that match the configured queries. Covered in detail in [Immunity](immunity.md).

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `ImmunityQueries` | `TArray<FGameplayEffectQuery>` | Queries that define which incoming effects to block |

**Example:** A fire shield that grants immunity to all effects tagged `Damage.Fire`.

---

### RemoveOtherGameplayEffectComponent

**Display Name:** "Remove Other Effects"

When applied, removes other active GEs that match the configured queries.

**Key Properties:**

| Property | Type | Description |
|:---|:---|:---|
| `RemoveGameplayEffectQueries` | `TArray<FGameplayEffectQuery>` | Queries that define which active effects to remove |

**Example:** A "Purify" spell that removes all active poison effects (those with tag `Debuff.Poison`).

---

## Writing Custom Components

To create your own component, subclass `UGameplayEffectComponent`:

```cpp
UCLASS(DisplayName = "My Custom Behavior", MinimalAPI)
class UMyCustomGEComponent : public UGameplayEffectComponent
{
    GENERATED_BODY()

public:
    // Block application if our custom condition isn't met
    virtual bool CanGameplayEffectApply(
        const FActiveGameplayEffectsContainer& ActiveGEContainer,
        const FGameplayEffectSpec& GESpec) const override
    {
        // Your custom logic here
        return true;
    }

    // Set up behavior when the effect is added
    virtual bool OnActiveGameplayEffectAdded(
        FActiveGameplayEffectsContainer& ActiveGEContainer,
        FActiveGameplayEffect& ActiveGE) const override
    {
        // Register delegates, set up state, etc.
        // Return false to inhibit the effect
        return true;
    }

    // Respond to execution (instant or periodic)
    virtual void OnGameplayEffectExecuted(
        FActiveGameplayEffectsContainer& ActiveGEContainer,
        FGameplayEffectSpec& GESpec,
        FPredictionKey& PredictionKey) const override
    {
        // React to execution
    }

private:
    UPROPERTY(EditDefaultsOnly, Category = "My Settings")
    float MyCustomValue = 1.0f;
};
```

Key things to remember:

1. **No mutable state on the component.** One component instance serves all active instances of this GE. If you need per-instance state, store it elsewhere (e.g., on the `FActiveGameplayEffect`'s event set, or bound as extra parameters on a delegate).

2. **Register for cleanup.** If you register delegates in `OnActiveGameplayEffectAdded`, make sure to unregister when the effect is removed. The `FActiveGameplayEffectEvents` on each `FActiveGameplayEffect` provides an `OnRemoved` delegate.

3. **Native only.** Components must be written in C++. The callback registration pattern doesn't work from Blueprint.

4. **Editor support.** Override `IsDataValid` to provide editor-time validation of your component's configuration.

## The GameplayEffectVersion Enum and Migration

The `EGameplayEffectVersion` enum tracks the schema version of a GE asset:

```cpp
enum class EGameplayEffectVersion : uint8
{
    Monolithic,           // Pre-5.3 (before components existed)
    Modular53,            // UE 5.3 — most properties moved to components
    AbilitiesComponent53, // Granted abilities moved into AbilitiesGameplayEffectComponent

    Current = AbilitiesComponent53
};
```

When you open an old (Monolithic) GE in UE 5.3+, the engine automatically runs conversion functions that:

1. Read the deprecated properties
2. Create the appropriate components
3. Copy the data into the components
4. Set the version to Current

This happens in `PostLoad` via helper functions like `ConvertAssetTagsComponent`, `ConvertImmunityComponent`, etc. The old properties are kept around (marked `UPROPERTY(meta = (DeprecatedProperty))`) to support the migration, but new code should always use components.

!!! tip "Resave Your Assets"
    After upgrading to 5.3+, it's a good idea to resave your GE assets. This ensures the migration runs and the version is updated, avoiding repeated conversion on every load.

## Mapping Old Properties to New Components

For anyone migrating from pre-5.3 or reading older documentation:

| Old Monolithic Property | New Component |
|:---|:---|
| `InheritableGameplayEffectTags` | `UAssetTagsGameplayEffectComponent` |
| `InheritableOwnedTagsContainer` | `UTargetTagsGameplayEffectComponent` |
| `InheritableBlockedAbilityTagsContainer` | `UBlockAbilityTagsGameplayEffectComponent` |
| `OngoingTagRequirements` | `UTargetTagRequirementsGameplayEffectComponent` |
| `ApplicationTagRequirements` | `UTargetTagRequirementsGameplayEffectComponent` |
| `RemovalTagRequirements` | `UTargetTagRequirementsGameplayEffectComponent` |
| `RemoveGameplayEffectsWithTags` / `RemoveGameplayEffectQuery` | `URemoveOtherGameplayEffectComponent` |
| `GrantedApplicationImmunityTags` / `GrantedApplicationImmunityQuery` | `UImmunityGameplayEffectComponent` |
| `ChanceToApplyToTarget` | `UChanceToApplyGameplayEffectComponent` |
| `ApplicationRequirements` | `UCustomCanApplyGameplayEffectComponent` |
| `ConditionalGameplayEffects` / `PrematureExpirationEffectClasses` / `RoutineExpirationEffectClasses` | `UAdditionalEffectsGameplayEffectComponent` |
| `GrantedAbilities` | `UAbilitiesGameplayEffectComponent` |

## What's Next?

Now that you understand the component architecture, dive into the tag-based configuration that several components use: [Tags and Requirements](tags-and-requirements.md). Or explore specific components in depth: [Immunity](immunity.md).

## Related Pages

- [GE Component Catalog](../reference/ge-component-catalog.md) -- complete reference for all UGameplayEffectComponent subclasses with properties and callbacks
