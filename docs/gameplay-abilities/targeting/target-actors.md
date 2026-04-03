---
title: Target Actors
description: AGameplayAbilityTargetActor — the world actors that perform targeting logic, produce target data, and provide visual feedback during ability targeting.
---

# Target Actors

`AGameplayAbilityTargetActor` is a world actor spawned during ability targeting. It performs the actual targeting logic — running traces, doing overlap checks, or waiting for player input — and produces `FGameplayAbilityTargetData` when targeting is confirmed or cancelled.

!!! note "Performance warning from Epic"
    The engine source includes a prominent warning: target actors are spawned once per ability activation and "in their default form are not very efficient." For most games, you will want to subclass and optimize, or implement targeting logic directly in a game-specific actor to avoid the actor spawn cost. That said, they are excellent learning tools and work fine for prototyping.

## The Base Class: AGameplayAbilityTargetActor

### Lifecycle

1. **StartTargeting** — called when the target actor begins targeting. Receives the owning ability. Set up your initial state here.
2. **Tick** — for actors that continuously update (e.g., traces that follow the crosshair), targeting logic runs each tick.
3. **ConfirmTargeting** — the player confirmed. Produce your target data and broadcast it.
4. **ConfirmTargetingAndContinue** — produce target data but keep the actor alive for additional confirmations.
5. **CancelTargeting** — the player cancelled. Broadcast the cancel delegate.

```cpp
// Key virtual functions
virtual void StartTargeting(UGameplayAbility* Ability);
virtual void ConfirmTargeting();
virtual void ConfirmTargetingAndContinue();
virtual void CancelTargeting();
```

### Key Properties

| Property | Type | Purpose |
|:---|:---|:---|
| `ShouldProduceTargetDataOnServer` | `bool` | If `true`, the server generates its own target data instead of using what the client sends. Useful for server-authoritative targeting. |
| `bDestroyOnConfirmation` | `bool` | If `true`, the actor is destroyed after confirmation. Set to `false` for "CustomMulti" targeting that produces data multiple times. |
| `StartLocation` | `FGameplayAbilityTargetingLocationInfo` | Where targeting originates (actor transform, socket, literal position). |
| `Filter` | `FGameplayTargetDataFilterHandle` | Filters candidate targets. See [Reticles and Filters](reticles-and-filters.md). |
| `ReticleClass` | `TSubclassOf<AGameplayAbilityWorldReticle>` | The reticle actor to spawn for visual feedback. |
| `ReticleParams` | `FWorldReticleParameters` | Parameters passed to the reticle (e.g., AoE scale). |
| `bDebug` | `bool` | Draw debug visualization for the targeting actor. |
| `SourceActor` | `AActor*` | The actor that owns/initiated targeting. |

### Delegates

Target actors broadcast two delegates:

```cpp
FAbilityTargetData TargetDataReadyDelegate;   // Fires with target data on confirm
FAbilityTargetData CanceledDelegate;           // Fires on cancel
```

The `WaitTargetData` [ability task](../ability-tasks.md) binds to these delegates and forwards them back to the ability as `ValidData` and `Cancelled` outputs.

### Integration with WaitTargetData

The typical usage pattern is through the `WaitTargetData` task:

```cpp
void UMyAbility::ActivateAbility(...)
{
    auto* Task = UAbilityTask_WaitTargetData::WaitTargetData(
        this,
        NAME_None,
        EGameplayTargetingConfirmation::UserConfirmed,
        AGameplayAbilityTargetActor_SingleLineTrace::StaticClass());

    Task->ValidData.AddDynamic(this, &ThisClass::OnTargetDataReady);
    Task->Cancelled.AddDynamic(this, &ThisClass::OnTargetDataCancelled);
    Task->ReadyForActivation();
}
```

The task handles spawning the target actor, calling `StartTargeting`, binding to confirm/cancel inputs, and cleaning up.

## Built-in Target Actor Types

### AGameplayAbilityTargetActor_Trace (Abstract Base)

The base class for all trace-based targeting. It handles aiming from the player controller's viewpoint, clipping traces to ability range, and producing `FGameplayAbilityTargetData_SingleTargetHit`.

**Key properties:**

| Property | Purpose |
|:---|:---|
| `MaxRange` | Maximum trace distance |
| `TraceProfile` | Collision profile to use for the trace |
| `bTraceAffectsAimPitch` | Whether the trace result affects the aiming pitch |

**Key methods:**

- `AimWithPlayerController` — calculates trace start/end from the player's camera
- `ClipCameraRayToAbilityRange` — clips the trace to `MaxRange` from the source
- `LineTraceWithFilter` / `SweepWithFilter` — trace functions that apply the target data filter
- `PerformTrace` — pure virtual, implemented by subclasses

### AGameplayAbilityTargetActor_SingleLineTrace

The simplest trace actor. Performs a single line trace from the player's camera through the crosshair, out to `MaxRange`. Produces a single `FGameplayAbilityTargetData_SingleTargetHit`.

Good for: hitscan weapons, single-target abilities, click-to-target.

### AGameplayAbilityTargetActor_GroundTrace

Traces to find a point on the ground. Useful for "place AoE on the ground" targeting like MOBA ground-target abilities.

Extends the trace functionality to project the hit point onto a valid ground surface and can validate placement against navigation or other constraints.

### AGameplayAbilityTargetActor_Radius

Selects all actors within a given radius of the source location. Does not use traces — instead performs an overlap check.

```cpp
UPROPERTY(BlueprintReadWrite, EditAnywhere,
    meta = (ExposeOnSpawn = true), Category = Radius)
float Radius;
```

Produces `FGameplayAbilityTargetData_ActorArray` with all overlapping actors that pass the filter.

Good for: AoE abilities, "all enemies in range" selection.

### AGameplayAbilityTargetActor_ActorPlacement

Used for "place an actor at this location" targeting. Works with the `AGameplayAbilityWorldReticle_ActorVisualization` reticle to show a preview of what will be placed.

Good for: turret placement, ward placement, building construction.

## Creating Custom Target Actors

For most projects, you will want at least one custom target actor. Here is the pattern:

```cpp
UCLASS(Blueprintable)
class ATargetActor_ConeTrace : public AGameplayAbilityTargetActor
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite,
        meta = (ExposeOnSpawn = true), Category = "Cone")
    float ConeHalfAngle = 30.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite,
        meta = (ExposeOnSpawn = true), Category = "Cone")
    float ConeRange = 500.0f;

    virtual void StartTargeting(UGameplayAbility* Ability) override;
    virtual void ConfirmTargetingAndContinue() override;

private:
    TArray<TWeakObjectPtr<AActor>> PerformConeCheck();
};
```

```cpp
void ATargetActor_ConeTrace::StartTargeting(UGameplayAbility* Ability)
{
    Super::StartTargeting(Ability);
    // Setup, spawn reticle, etc.
}

void ATargetActor_ConeTrace::ConfirmTargetingAndContinue()
{
    TArray<TWeakObjectPtr<AActor>> HitActors = PerformConeCheck();

    // Filter results
    TArray<TWeakObjectPtr<AActor>> FilteredActors;
    for (const auto& Actor : HitActors)
    {
        if (Filter.FilterPassesForActor(Actor.Get()))
        {
            FilteredActors.Add(Actor);
        }
    }

    // Build target data
    FGameplayAbilityTargetData_ActorArray* ActorData =
        new FGameplayAbilityTargetData_ActorArray();
    ActorData->TargetActorArray = FilteredActors;

    FGameplayAbilityTargetDataHandle Handle;
    Handle.Add(ActorData);

    // Broadcast to the waiting task
    TargetDataReadyDelegate.Broadcast(Handle);
}
```

### Tips for Custom Target Actors

- Always apply the `Filter` to your results before broadcasting
- Use `ShouldProduceTargetData()` to check whether this actor should produce data (respects client/server authority settings)
- Override `OnReplicatedTargetDataReceived` if you need to validate client data on the server
- Set `bDestroyOnConfirmation = false` if the ability needs multiple rounds of targeting
- For tick-based targeting (continuous crosshair updates), override `Tick` and update your reticle position each frame
