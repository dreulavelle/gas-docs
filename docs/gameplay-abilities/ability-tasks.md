---
title: Ability Tasks
description: Deep dive into UAbilityTask — the async building blocks for ability logic — plus a complete catalog of all 27+ built-in tasks and how to write custom ones.
---

# Ability Tasks

Ability Tasks (`UAbilityTask`) are the primary mechanism for doing asynchronous work inside a Gameplay Ability. They are small, self-contained operations that follow the pattern of "start something and wait until it finishes or is interrupted." Think of them as latent actions scoped to an ability's lifetime.

Need to play a montage and wait for it to finish? There is a task for that. Need to wait for a gameplay event? A task. Wait for an attribute to cross a threshold? Task. They are the bread and butter of ability implementation.

## How Tasks Work

### Lifecycle

Every ability task follows the same lifecycle:

1. **Construction** — a static factory function creates and configures the task
2. **Activation** — `ReadyForActivation()` (or `Activate()` internally) starts the task
3. **Running** — the task does its work, potentially ticking each frame
4. **Callbacks** — the task fires `BlueprintAssignable` delegates when things happen
5. **Destruction** — `OnDestroy()` cleans up (unregisters delegates, etc.)

The critical rule: **tasks are automatically cleaned up when their owning ability ends.** You do not need to manually end tasks in your `EndAbility` override — the system handles it. However, you _do_ need to handle the case where a task's callback fires and the ability should end.

### The Factory Pattern

Tasks are never created with `NewObject` directly. Instead, each task class provides a static factory function (often called `Create...` or the task name itself):

```cpp
UAbilityTask_PlayMontageAndWait* Task =
    UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
        this,              // Owning ability
        NAME_None,         // Task instance name
        MyMontage,         // Parameters...
        1.0f);

// Bind to output delegates
Task->OnCompleted.AddDynamic(this, &ThisClass::OnMontageCompleted);
Task->OnCancelled.AddDynamic(this, &ThisClass::OnMontageCancelled);
Task->OnInterrupted.AddDynamic(this, &ThisClass::OnMontageCancelled);

// IMPORTANT: Must call this to actually start the task!
Task->ReadyForActivation();
```

!!! warning "Don't forget ReadyForActivation"
    Creating a task does not start it. You must call `ReadyForActivation()` after binding your delegates. Forgetting this call is a common source of "my task does nothing" bugs.

### ShouldBroadcastAbilityTaskDelegates

Before firing any callback, tasks check `ShouldBroadcastAbilityTaskDelegates()`. This returns `false` if the ability has ended or the ASC is no longer valid. This prevents callbacks from firing into a dead ability — a safety net, not something you typically need to worry about.

### Instance Names

Tasks can be given instance names. This allows you to find and manipulate tasks by name from the ability:

- `ConfirmTaskByInstanceName(FName)` — sends a confirm signal to matching tasks
- `EndTaskByInstanceName(FName)` — ends matching tasks (next frame)
- `CancelTaskByInstanceName(FName)` — cancels matching tasks (next frame)

This is useful for complex abilities with multiple concurrent tasks where you need fine-grained control.

## Built-in Task Catalog

The engine ships with over 27 ability tasks. Here they are, organized by category.

### Wait Tasks

These tasks wait for something to happen, then fire a delegate.

| Task | What It Does | Key Outputs |
|:---|:---|:---|
| **WaitDelay** | Waits for a specified number of seconds | `OnFinish` |
| **WaitCancel** | Waits for the ability's cancel input (GenericCancel) | `OnCancel` |
| **WaitConfirm** | Waits for the ability's confirm input (GenericConfirm) | `OnConfirm` |
| **WaitConfirmCancel** | Combines confirm and cancel into one task | `OnConfirm`, `OnCancel` |
| **WaitInputPress** | Waits for the ability's bound input to be pressed | `OnPress` (receives time held) |
| **WaitInputRelease** | Waits for the ability's bound input to be released | `OnRelease` (receives time held) |
| **WaitMovementModeChange** | Waits for the character's movement mode to change | `OnChange` (receives new mode) |
| **WaitOverlap** | Waits for a primitive component overlap event | `OnOverlap` (receives overlap info) |
| **WaitVelocityChange** | Waits for the avatar's velocity to meet a threshold | `OnVelocityChg` |
| **WaitAbilityActivate** | Waits for another ability (matching tag query) to activate on this ASC | `OnActivate` (receives activated ability) |
| **WaitAbilityCommit** | Waits for another ability to commit | `OnCommit` (receives committed ability) |

### Gameplay Effect Tasks

| Task | What It Does | Key Outputs |
|:---|:---|:---|
| **WaitGameplayEffectApplied_Self** | Waits for a GE to be applied _to_ the owner, with tag filtering | `OnApplied` (source actor, spec handle, active handle) |
| **WaitGameplayEffectApplied_Target** | Waits for a GE that the owner applied _to others_, with tag filtering | `OnApplied` (target actor, spec handle, active handle) |
| **WaitGameplayEffectRemoved** | Waits for a specific active GE (by handle) to be removed | `OnRemoved`, `InvalidHandle` |
| **WaitGameplayEffectStackChange** | Waits for a specific active GE's stack count to change | `OnChange` (old count, new count), `InvalidHandle` |
| **WaitGameplayEffectBlockedImmunity** | Waits for a GE to be blocked by immunity on the owner | `Blocked` (blocked spec, active immunity handle) |

### Gameplay Tag Tasks

| Task | What It Does | Key Outputs |
|:---|:---|:---|
| **WaitGameplayTagAdded** | Waits for a specific tag to be added to the owner (or external target) | `Added` |
| **WaitGameplayTagRemoved** | Waits for a specific tag to be removed from the owner (or external target) | `Removed` |
| **WaitGameplayTagCountChanged** | Fires when the count of a specific tag changes (added or removed) | `TagCountChanged` (new count) |
| **WaitGameplayTagQuery** | Fires when a `FGameplayTagQuery` transitions between true/false on the owner | `Triggered` |

!!! info "Tag tasks and external targets"
    The WaitGameplayTag tasks accept an `OptionalExternalTarget` actor, letting you watch tags on someone other than the owning actor.

### Attribute Tasks

| Task | What It Does | Key Outputs |
|:---|:---|:---|
| **WaitAttributeChange** | Fires when a specific attribute changes, with optional tag filtering | `OnChange` |
| **WaitAttributeChangeThreshold** | Fires when an attribute crosses a comparison threshold (>, <, ==, etc.) | `OnChange` |
| **WaitAttributeChangeRatioThreshold** | Fires when the ratio of two attributes crosses a threshold (e.g., Health/MaxHealth < 0.25) | `OnChange` |

`WaitAttributeChange` supports comparison operators via the `WaitForAttributeChangeWithComparison` factory:

```cpp
auto* Task = UAbilityTask_WaitAttributeChange::WaitForAttributeChangeWithComparison(
    this,
    UMyAttributeSet::GetHealthAttribute(),
    FGameplayTag(),         // WithTag (optional source tag filter)
    FGameplayTag(),         // WithoutTag
    EWaitAttributeChangeComparison::LessThan,
    25.0f,                  // Comparison value
    true);                  // Trigger once
```

### Gameplay Event Tasks

| Task | What It Does | Key Outputs |
|:---|:---|:---|
| **WaitGameplayEvent** | Waits for a gameplay event with a matching tag to be sent to the owner's ASC | `EventReceived` (FGameplayEventData payload) |
| **WaitTargetData** | Spawns or uses a [Target Actor](targeting/target-actors.md) and waits for targeting data | `ValidData` (FGameplayAbilityTargetDataHandle), `Cancelled` |

`WaitGameplayEvent` is incredibly versatile. It is the primary mechanism for abilities that react to external events mid-execution (e.g., "wait for the hit event from the anim notify, then apply damage").

```cpp
auto* Task = UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
    this,
    FGameplayTag::RequestGameplayTag("Event.Montage.Hit"),
    nullptr,    // OptionalExternalTarget
    false,      // OnlyTriggerOnce — false to keep listening
    true);      // OnlyMatchExact

Task->EventReceived.AddDynamic(this, &ThisClass::OnHitEvent);
Task->ReadyForActivation();
```

### Action Tasks

| Task | What It Does | Key Parameters |
|:---|:---|:---|
| **PlayMontageAndWait** | Plays an anim montage and waits for completion, blend out, interruption, or cancellation | Montage, Rate, StartSection, bStopWhenAbilityEnds, StartTimeSeconds |
| **PlayAnimAndWait** | Simpler animation playback without montage features | AnimSequence |
| **SpawnActor** | Spawns an actor with deferred construction (supports ExposeOnSpawn) | ActorClass |
| **MoveToLocation** | Smoothly moves the avatar to a target location over time | TargetLocation, Duration, Curve |
| **VisualizeTargeting** | Visualizes targeting (spawns/updates a target actor each tick) | TargetActorClass, Duration |
| **StartAbilityState** | Creates a named state that can be ended externally | StateName, bEndCurrentState |
| **NetworkSyncPoint** | Waits for both client and server to reach the same point | OnSync, OnSyncFailed (enum: BothWait, OnlyServerWait, OnlyClientWait) |
| **Repeat** | Repeats a callback a specified number of times with a delay between each | TimeBetweenActions, TotalActionCount |

`PlayMontageAndWait` deserves special mention — it is by far the most commonly used task. Its delegates cover every outcome:

- `OnCompleted` — montage played to the end
- `OnBlendedIn` — montage blend-in finished (5.7+)
- `OnBlendOut` — montage started blending out
- `OnInterrupted` — another montage overwrote this one
- `OnCancelled` — the ability was cancelled

### Root Motion Tasks

These tasks apply root motion forces to the character, useful for dashes, knockbacks, lunges, and similar movement abilities.

| Task | What It Does | Key Parameters |
|:---|:---|:---|
| **ApplyRootMotion_Base** | Abstract base class for root motion tasks | Strength, Duration, bFinishOnMovementEnd |
| **ApplyRootMotionConstantForce** | Applies a constant directional force | WorldDirection, Strength, Duration, bIsAdditive |
| **ApplyRootMotionJumpForce** | Jump arc with separate horizontal and vertical control | Rotation, Distance, Height, Duration, Curve |
| **ApplyRootMotionMoveToActorForce** | Moves toward a target actor over time | TargetActor, StretchFactor, OffsetFromTarget |
| **ApplyRootMotionMoveToForce** | Moves toward a world location over time | TargetLocation, Duration, bSetNewMovementMode |
| **ApplyRootMotionRadialForce** | Applies a radial push/pull (explosion knockback, vortex pull) | Location, Radius, Strength |

All root motion tasks share common output delegates: `OnFinish` (completed normally) and typically a callback for when movement is blocked.

## Writing Custom Tasks

When the built-in tasks do not cover your use case, writing a custom one is straightforward. Here is a template:

```cpp
UCLASS()
class UAbilityTask_WaitForSomething : public UAbilityTask
{
    GENERATED_BODY()

public:
    // Output delegate — the "pin" that fires in Blueprint
    UPROPERTY(BlueprintAssignable)
    FMyTaskDelegate OnSomethingHappened;

    // Static factory function
    UFUNCTION(BlueprintCallable, Category = "Ability|Tasks",
        meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility",
                BlueprintInternalUseOnly = "TRUE"))
    static UAbilityTask_WaitForSomething* WaitForSomething(
        UGameplayAbility* OwningAbility,
        FName TaskInstanceName,
        float MyParameter);

    virtual void Activate() override;
    virtual void OnDestroy(bool bInOwnerFinished) override;

private:
    float MyParameter;
    FDelegateHandle MyCallbackHandle;
};
```

```cpp
UAbilityTask_WaitForSomething* UAbilityTask_WaitForSomething::WaitForSomething(
    UGameplayAbility* OwningAbility,
    FName TaskInstanceName,
    float MyParameter)
{
    UAbilityTask_WaitForSomething* Task =
        NewAbilityTask<UAbilityTask_WaitForSomething>(OwningAbility, TaskInstanceName);
    Task->MyParameter = MyParameter;
    return Task;
    // NOTE: Do NOT call Activate or fire delegates here!
}

void UAbilityTask_WaitForSomething::Activate()
{
    Super::Activate();

    // Register for whatever event you're waiting for
    // Example: subscribe to a delegate on a component
    if (AActor* Avatar = GetAvatarActor())
    {
        // MyCallbackHandle = SomeComponent->OnSomething.AddUObject(...);
    }
}

void UAbilityTask_WaitForSomething::OnDestroy(bool bInOwnerFinished)
{
    // ALWAYS unregister your callbacks here
    // if (MyCallbackHandle.IsValid()) { ... }

    Super::OnDestroy(bInOwnerFinished);
}
```

!!! tip "Checklist for custom tasks"
    1. Override `OnDestroy` and unregister all callbacks — this is called when the ability ends
    2. All setup logic goes in `Activate()`, not the factory function
    3. Never fire delegates from the factory function
    4. Check `ShouldBroadcastAbilityTaskDelegates()` before broadcasting
    5. Use `NewAbilityTask<T>()` to create the instance (handles init and debug tracking)

### Making Tasks Tick

If your task needs to do work every frame, override `TickTask`:

```cpp
// In the header
virtual void TickTask(float DeltaTime) override;

// In Activate()
bTickingTask = true;
```

Set `bTickingTask = true` before or during `Activate()` to opt in. The task will tick as long as it is alive.

## Task Lifetime and Auto-Cleanup

One of the best features of ability tasks is that they automatically clean up when their owning ability ends. You do not need to manually track and destroy tasks. The system calls `OnDestroy` on every active task when `EndAbility` fires.

This means:

- A montage task will stop the montage when the ability ends (if `bStopWhenAbilityEnds` is `true`)
- Delegate registrations in `OnDestroy` are cleaned up
- You do not need to call `EndTask()` on tasks from your `EndAbility` override

However, you _can_ manually end a task early by calling `EndTask()` on it, or by using the instance name functions on the ability.

## Tasks and Instancing Policy

Tasks require an instanced ability. They store a pointer to the owning `UGameplayAbility` instance and use it for delegate callbacks, lifecycle management, and ASC access.

- **InstancedPerActor** — tasks work perfectly
- **InstancedPerExecution** — tasks work perfectly
- **NonInstanced** — tasks do **not** work (the ability is a CDO, not a real instance)

See [Instancing Policy](instancing-policy.md) for more details on why this matters.
