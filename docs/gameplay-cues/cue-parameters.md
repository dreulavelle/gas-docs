---
title: Cue Parameters
description: FGameplayCueParameters fields, passing custom data to cues, IGameplayCueInterface for actor-level handling, and local vs replicated execution.
---

# Cue Parameters

Every Gameplay Cue event carries an `FGameplayCueParameters` struct that provides context about what triggered the cue. This is how your cue knows where the hit landed, who caused it, how strong the effect was, and any custom data your game needs to pass along.

## FGameplayCueParameters Fields

Here's the full struct from `GameplayEffectTypes.h`:

| Field | Type | Description |
|:---|:---|:---|
| `NormalizedMagnitude` | `float` | Magnitude of the source GE, normalized 0-1. Use for "how strong" (0 = min, 1 = max). |
| `RawMagnitude` | `float` | Raw final magnitude of the source GE. Use for displaying numbers. |
| `EffectContext` | `FGameplayEffectContextHandle` | The effect context -- hit result, source/target info, custom data. |
| `MatchedTagName` | `FGameplayTag` | The tag that matched this specific cue handler (not replicated). |
| `OriginalTag` | `FGameplayTag` | The original tag before any translation (not replicated). |
| `AggregatedSourceTags` | `FGameplayTagContainer` | All source tags from the effect spec. |
| `AggregatedTargetTags` | `FGameplayTagContainer` | All target tags from the effect spec. |
| `Location` | `FVector_NetQuantize10` | World location of the cue event. |
| `Normal` | `FVector_NetQuantizeNormal` | Impact normal direction. |
| `Instigator` | `TWeakObjectPtr<AActor>` | The actor that owns the source ASC. |
| `EffectCauser` | `TWeakObjectPtr<AActor>` | The physical actor that caused the effect (weapon, projectile). |
| `SourceObject` | `TWeakObjectPtr<const UObject>` | The object the effect was created from. |
| `PhysicalMaterial` | `TWeakObjectPtr<const UPhysicalMaterial>` | Physical material of the hit surface. |
| `GameplayEffectLevel` | `int32` | Level of the source Gameplay Effect. |
| `AbilityLevel` | `int32` | Level of the source Gameplay Ability. |

### Automatic Population

When a cue is triggered by a Gameplay Effect, the parameters are populated automatically by `UAbilitySystemGlobals::InitGameplayCueParameters_GESpec`. The effect spec's context, magnitude, tags, and level information all flow through.

When triggered manually, you construct the parameters yourself:

```cpp
FGameplayCueParameters CueParams;
CueParams.Location = HitResult.ImpactPoint;
CueParams.Normal = HitResult.ImpactNormal;
CueParams.PhysicalMaterial = HitResult.PhysMaterial.Get();
CueParams.Instigator = GetOwner();
CueParams.EffectCauser = WeaponActor;
CueParams.NormalizedMagnitude = 0.75f;
CueParams.RawMagnitude = DamageAmount;

ASC->ExecuteGameplayCue(GameplayCueTag, CueParams);
```

### Using Parameters in Blueprint

In your cue notify's event graph, the `Parameters` input pin gives you access to all fields. Common patterns:

- **Spawn particles at hit location**: Use `Parameters.Location` and `Parameters.Normal` for position and rotation
- **Scale effect by magnitude**: Use `Parameters.NormalizedMagnitude` to drive particle size or sound volume
- **Vary by surface**: Check `Parameters.PhysicalMaterial` to play different sounds on metal vs wood
- **Show damage numbers**: Use `Parameters.RawMagnitude` for the number to display

## Passing Custom Data via Effect Context

The built-in `FGameplayCueParameters` fields cover common cases, but games often need to pass additional data -- damage type enums, combo counters, element types, critical hit flags. The way to do this is through a custom `FGameplayEffectContext` subclass.

### Step 1: Create Your Custom Context

```cpp
USTRUCT()
struct FMyGameplayEffectContext : public FGameplayEffectContext
{
    GENERATED_BODY()

    /** Custom data */
    bool bIsCriticalHit = false;
    float DamageTypeMultiplier = 1.0f;
    FGameplayTag DamageTypeTag;

    /** Required: tells the engine how to duplicate this struct */
    virtual UScriptStruct* GetScriptStruct() const override
    {
        return FMyGameplayEffectContext::StaticStruct();
    }

    /** Required: net serialization */
    virtual bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess) override;

    /** Required: duplicate for prediction */
    virtual FMyGameplayEffectContext* Duplicate() const override
    {
        FMyGameplayEffectContext* NewContext = new FMyGameplayEffectContext();
        *NewContext = *this;
        NewContext->AddActors(Actors);
        if (GetHitResult())
        {
            NewContext->AddHitResult(*GetHitResult(), true);
        }
        return NewContext;
    }
};
```

### Step 2: Override AllocGameplayEffectContext

In your `UAbilitySystemGlobals` subclass:

```cpp
FGameplayEffectContext* UMyAbilitySystemGlobals::AllocGameplayEffectContext() const
{
    return new FMyGameplayEffectContext();
}
```

### Step 3: Set Custom Data When Creating Effects

```cpp
FGameplayEffectContextHandle ContextHandle = ASC->MakeEffectContext();
FMyGameplayEffectContext* MyContext =
    static_cast<FMyGameplayEffectContext*>(ContextHandle.Get());
MyContext->bIsCriticalHit = true;
MyContext->DamageTypeTag = CombatTags::DamageType_Fire;

// Apply effect with this context
FGameplayEffectSpecHandle SpecHandle =
    ASC->MakeOutgoingGameplayEffectSpec(DamageEffectClass, Level, ContextHandle);
ASC->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
```

### Step 4: Read Custom Data in Cues

In your cue notify (C++ or Blueprint via a helper library):

```cpp
bool UGC_DamageImpact::OnExecute_Implementation(
    AActor* MyTarget, const FGameplayCueParameters& Parameters) const
{
    const FMyGameplayEffectContext* MyContext =
        static_cast<const FMyGameplayEffectContext*>(
            Parameters.EffectContext.Get());

    if (MyContext && MyContext->bIsCriticalHit)
    {
        // Play extra flashy crit effect
    }
    return false;
}
```

!!! tip "Blueprint access"
    Create a static `UBlueprintFunctionLibrary` with helper functions that cast the context and expose your custom fields to Blueprint. This keeps your cue Blueprints clean.

## IGameplayCueInterface

The `IGameplayCueInterface` lets actors handle Gameplay Cue events directly, without going through a separate cue notify class. The ASC (`UAbilitySystemComponent`) already implements this interface.

### How It Works

When the `GameplayCueManager` routes a cue, it checks whether the target actor implements `IGameplayCueInterface` **before** looking up a cue notify class. If the interface is implemented, it calls `HandleGameplayCue` on the actor.

Key methods:

```cpp
class IGameplayCueInterface
{
    // Handle a single cue event
    virtual void HandleGameplayCue(UObject* Self, FGameplayTag GameplayCueTag,
        EGameplayCueEvent::Type EventType,
        const FGameplayCueParameters& Parameters);

    // Can this actor accept this cue right now?
    virtual bool ShouldAcceptGameplayCue(UObject* Self, FGameplayTag GameplayCueTag,
        EGameplayCueEvent::Type EventType,
        const FGameplayCueParameters& Parameters);

    // Forward to parent class handler
    virtual void ForwardGameplayCueToParent();
};
```

### Blueprint Handling via UFunction Matching

The interface supports automatic routing to Blueprint functions based on tag names. If your actor has a function named after the cue tag (with dots replaced by underscores), it will be called automatically:

```
Tag: GameplayCue.Hero.FireballImpact
Function: GameplayCue_Hero_FireballImpact
```

This is done through `DispatchBlueprintCustomHandler`. Call `ForwardGameplayCueToParent()` inside that function if you want the system to continue checking for more generic handlers up the tag hierarchy.

### When to Use Interface vs Notify

| Approach | Use When |
|:---|:---|
| **Cue Notify classes** | Reusable feedback shared across many actors; designer-configured VFX |
| **IGameplayCueInterface** | Actor-specific handling; when the response depends heavily on the actor's own state |

Most projects use cue notify classes for the bulk of their cues and reserve interface handling for special cases like player-specific UI feedback or AI-specific responses.

## Local vs Replicated Execution

Cues can execute through two paths, and the choice affects both gameplay feel and bandwidth.

### Replicated Cues (Default)

When triggered through the ASC's standard functions (`ExecuteGameplayCue`, `AddGameplayCue`), cues go through the replication pipeline:

1. Server applies the effect/triggers the cue
2. An RPC (usually unreliable multicast) is sent to clients
3. Clients invoke the cue locally

This ensures all players see the same feedback, but adds a round-trip delay for the instigating client.

### Local Cues

Triggered with `Local` variants:

```cpp
ASC->ExecuteGameplayCueLocal(Tag, Params);
ASC->AddGameplayCueLocal(Tag, Params);
ASC->RemoveGameplayCueLocal(Tag);
```

Or through the static helpers on `UGameplayCueManager`:

```cpp
UGameplayCueManager::AddGameplayCue_NonReplicated(Actor, Tag, Params);
UGameplayCueManager::RemoveGameplayCue_NonReplicated(Actor, Tag, Params);
UGameplayCueManager::ExecuteGameplayCue_NonReplicated(Actor, Tag, Params);
```

Local cues execute immediately on the calling machine with no replication. Use them for:

- **First-person feedback** (screen shake, camera effects, UI flashes) that only the local player should see
- **Predicted cues** where you want instant feedback on the owning client while waiting for server confirmation
- **Cosmetic-only local events** (footstep dust, animation-driven particles)

### Prediction and Cues

When an ability is predicted (see [Prediction](../networking/prediction.md)), the cues triggered during the prediction window are automatically treated with special care:

- The client plays the cue immediately (predictively)
- When the server confirms, the replicated cue arrives -- but the system checks the `FPredictionKey` and suppresses the duplicate
- If the server rejects the prediction, the cue is rolled back (for Add/Remove cues)

This happens automatically for cues triggered by predicted gameplay effects. You don't need to manually manage prediction for most cue scenarios.

## Related Pages

- [Cue Notify Types](cue-types.md) -- the six cue classes
- [Cue Manager](cue-manager.md) -- routing and loading
- [Prediction](../networking/prediction.md) -- how prediction keys interact with cues
- [Execution Calculations](../gameplay-effects/execution-calculations.md) -- where magnitude values originate
