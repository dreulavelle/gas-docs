---
title: Async Ability Tasks
description: UAbilityAsync — the newer async task base class that works outside of abilities, usable from UI widgets, AI controllers, and anywhere with an ASC reference.
---

# Async Ability Tasks

`UAbilityAsync` is the newer alternative to [`UAbilityTask`](ability-tasks.md) for listening to ability system events. The key difference is scope: while `UAbilityTask` is tied to a specific ability's lifetime, `UAbilityAsync` is a standalone `UCancellableAsyncAction` that can be used from **anywhere** — UI widgets, AI controllers, HUDs, game mode code, or even other abilities.

If you need to listen for attribute changes in a health bar widget, or watch for tag changes in an AI behavior tree task, `UAbilityAsync` is the tool you want. For common UI integration patterns, see [GAS to UI](../patterns/gas-to-ui.md).

## How They Differ from UAbilityTask

| Feature | UAbilityTask | UAbilityAsync |
|:---|:---|:---|
| **Requires an active ability** | Yes — bound to the owning ability | No — just needs an actor with an ASC |
| **Auto-cleanup** | When the owning ability ends | When the Blueprint graph cleans up, or `EndAction()` is called |
| **Can be used in UI widgets** | No | Yes |
| **Can be used in AI controllers** | No | Yes |
| **Base class** | `UGameplayTask` | `UCancellableAsyncAction` |
| **Blueprint node style** | Latent action (within ability graph) | Async action (standalone node) |

!!! info "When to prefer UAbilityTask"
    If you are inside an ability and want the task to automatically clean up when the ability ends, use the regular `UAbilityTask` version. The async variants will **keep listening even after the ability ends**, which is usually not what you want inside ability logic. The engine source comments explicitly warn about this.

## The Built-in Async Tasks

The engine provides six async task types, each mirroring a corresponding `UAbilityTask` but without the ability scope requirement.

### WaitAttributeChanged

Fires whenever a specific attribute changes on the target actor's ASC.

```cpp
UFUNCTION(BlueprintCallable, meta = (DefaultToSelf = "TargetActor"))
static UAbilityAsync_WaitAttributeChanged* WaitForAttributeChanged(
    AActor* TargetActor,
    FGameplayAttribute Attribute,
    bool OnlyTriggerOnce = false);
```

**Output delegate:**

```cpp
// Delegate signature: (FGameplayAttribute Attribute, float NewValue, float OldValue)
UPROPERTY(BlueprintAssignable)
FAsyncWaitAttributeChangedDelegate Changed;
```

**Use case: Health bar UI widget**

This is the poster child for async tasks. A health bar widget does not own an ability — it just needs to know when health changes:

```cpp
void UHealthBarWidget::NativeConstruct()
{
    Super::NativeConstruct();

    if (AActor* OwnerActor = GetOwningPlayerPawn())
    {
        WaitTask = UAbilityAsync_WaitAttributeChanged::WaitForAttributeChanged(
            OwnerActor,
            UMyAttributeSet::GetHealthAttribute(),
            false);  // Keep listening

        WaitTask->Changed.AddDynamic(this, &ThisClass::OnHealthChanged);
        WaitTask->Activate();
    }
}

void UHealthBarWidget::OnHealthChanged(
    FGameplayAttribute Attribute, float NewValue, float OldValue)
{
    HealthBar->SetPercent(NewValue / MaxHealth);
}

void UHealthBarWidget::NativeDestruct()
{
    if (WaitTask)
    {
        WaitTask->EndAction();
    }
    Super::NativeDestruct();
}
```

!!! tip "Remember to EndAction"
    Unlike ability-scoped tasks, async tasks do not have an automatic cleanup trigger. Call `EndAction()` when you no longer need the listener (e.g., when the widget is destroyed).

### WaitGameplayEffectApplied

Fires when a Gameplay Effect is applied to the target actor, with extensive tag filtering options.

```cpp
UFUNCTION(BlueprintCallable, meta = (DefaultToSelf = "TargetActor"))
static UAbilityAsync_WaitGameplayEffectApplied* WaitGameplayEffectAppliedToActor(
    AActor* TargetActor,
    const FGameplayTargetDataFilterHandle SourceFilter,
    FGameplayTagRequirements SourceTagRequirements,
    FGameplayTagRequirements TargetTagRequirements,
    FGameplayTagRequirements AssetTagRequirements,
    FGameplayTagRequirements GrantedTagRequirements,
    bool TriggerOnce = false,
    bool ListenForPeriodicEffect = false);
```

**Output delegate:**

```cpp
// Delegate signature: (AActor* Source, FGameplayEffectSpecHandle SpecHandle,
//                      FActiveGameplayEffectHandle ActiveHandle)
UPROPERTY(BlueprintAssignable)
FOnAppliedDelegate OnApplied;
```

This is the most parameter-heavy async task. The tag requirement parameters let you filter by source tags, target tags, the effect's asset tags, and the tags the effect grants. This makes it useful for building systems that react to specific categories of effects (e.g., "show a damage number whenever a GE with the tag `Damage` is applied").

!!! note "Replication caveat"
    The engine source notes that effects themselves are not replicated — only the tags they grant, the attributes they change, and the gameplay cues they emit are replicated. This task only listens for effects applied locally (server-side or predicted client-side).

### WaitGameplayEvent

Fires when a gameplay event with a matching tag is sent to the target actor's ASC.

```cpp
UFUNCTION(BlueprintCallable, meta = (DefaultToSelf = "TargetActor"))
static UAbilityAsync_WaitGameplayEvent* WaitGameplayEventToActor(
    AActor* TargetActor,
    FGameplayTag EventTag,
    bool OnlyTriggerOnce = false,
    bool OnlyMatchExact = true);
```

**Output delegate:**

```cpp
// Delegate signature: (FGameplayEventData Payload)
UPROPERTY(BlueprintAssignable)
FEventReceivedDelegate EventReceived;
```

When `OnlyMatchExact` is `false`, it will trigger for child tags too (e.g., listening for `Event.Damage` will also fire on `Event.Damage.Fire`).

**Use case:** An AI controller listening for damage events to trigger threat assessment logic, or a game mode listening for kill events.

### WaitGameplayTagAdded / WaitGameplayTagRemoved

Two separate classes that fire when a specific tag is added to or removed from the target actor.

```cpp
// Added
static UAbilityAsync_WaitGameplayTagAdded* WaitGameplayTagAddToActor(
    AActor* TargetActor,
    FGameplayTag Tag,
    bool OnlyTriggerOnce = false);

// Removed
static UAbilityAsync_WaitGameplayTagRemoved* WaitGameplayTagRemoveFromActor(
    AActor* TargetActor,
    FGameplayTag Tag,
    bool OnlyTriggerOnce = false);
```

Both share the same abstract base class `UAbilityAsync_WaitGameplayTag` and the same delegate signature (no parameters, just a notification).

An important behavior: `WaitGameplayTagAdded` will **immediately fire** if the tag is already present when the task starts. Similarly, `WaitGameplayTagRemoved` will immediately fire if the tag is **not** present. This avoids race conditions where the tag changes before you start listening.

**Use case:** A UI element that shows/hides a "Stunned" indicator based on the stun tag.

### WaitGameplayTagCountChanged

Fires whenever the count of a specific tag changes on the target actor.

```cpp
static UAbilityAsync_WaitGameplayTagCountChanged*
    WaitGameplayTagCountChangedOnActor(
        AActor* TargetActor,
        FGameplayTag Tag);
```

**Output delegate:**

```cpp
// Delegate signature: (int32 TagCount)
UPROPERTY(BlueprintAssignable)
FAsyncWaitGameplayTagCountDelegate TagCountChanged;
```

Unlike the Added/Removed variants, this fires on **every** count change and provides the new count. Useful for stacking indicators in UI (e.g., showing "x3" next to a buff icon).

Note that this task does **not** have an `OnlyTriggerOnce` parameter — it always keeps listening. Call `EndAction()` when you are done.

### WaitGameplayTagQuery

The most flexible tag async task. Instead of watching a single tag, it evaluates a full `FGameplayTagQuery` and fires when the query result transitions.

```cpp
static UAbilityAsync_WaitGameplayTagQuery* WaitGameplayTagQueryOnActor(
    AActor* TargetActor,
    const FGameplayTagQuery TagQuery,
    const EWaitGameplayTagQueryTriggerCondition TriggerCondition
        = EWaitGameplayTagQueryTriggerCondition::WhenTrue,
    const bool bOnlyTriggerOnce = false);
```

`TriggerCondition` controls when the delegate fires:

- `WhenTrue` — fires when the query goes from false to true
- `WhenFalse` — fires when the query goes from true to false

**Use case:** Complex conditional UI, like "show the 'ready to combo' indicator when the character has tags A and B but not C."

## When to Use Async Tasks vs Regular Tasks

The rule of thumb is simple:

- **Inside an ability** — use `UAbilityTask` (scoped to the ability, auto-cleanup)
- **Outside an ability** — use `UAbilityAsync` (standalone, manual cleanup via `EndAction()`)

If you use `UAbilityAsync` inside an ability, the engine source warns that it will **keep listening after the ability ends**, which can cause unexpected behavior. The regular `UAbilityTask` equivalents are designed to stop when the ability stops.

## Creating Custom Async Tasks

Custom async tasks follow the same pattern as `UCancellableAsyncAction`:

```cpp
UCLASS()
class UAbilityAsync_WaitForMyThing : public UAbilityAsync
{
    GENERATED_BODY()

public:
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FMyDelegate, float, Value);

    UPROPERTY(BlueprintAssignable)
    FMyDelegate OnTriggered;

    UFUNCTION(BlueprintCallable, meta = (DefaultToSelf = "TargetActor",
              BlueprintInternalUseOnly = "TRUE"))
    static UAbilityAsync_WaitForMyThing* WaitForMyThing(
        AActor* TargetActor, float Threshold);

    virtual void Activate() override;
    virtual void EndAction() override;

private:
    float Threshold;
    FDelegateHandle Handle;
};
```

Override `Activate()` to register your callbacks and `EndAction()` to clean them up. Always call `ShouldBroadcastDelegates()` before firing your output delegates — it checks that the action and ASC are still valid.

## Related Pages

- [Ability Tasks](ability-tasks.md) -- the ability-scoped equivalent with automatic cleanup on EndAbility
- [GAS to UI](../patterns/gas-to-ui.md) -- patterns for connecting ability system state to UMG widgets using async tasks
- [Ability Task Catalog](../reference/ability-task-catalog.md) -- complete reference for all built-in UAbilityTask and UAbilityAsync classes
