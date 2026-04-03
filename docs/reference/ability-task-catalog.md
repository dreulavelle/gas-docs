---
icon: material/format-list-group
---

# Ability Task Catalog

Complete catalog of all built-in `UAbilityTask` and `UAbilityAsync` classes in UE 5.7. Tasks are organized by category. For concepts, see [Ability Tasks](../gameplay-abilities/ability-tasks.md) and [Async Ability Tasks](../gameplay-abilities/async-ability-tasks.md).

All tasks live in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/`.

---

## Wait Tasks

Tasks that pause ability execution until a condition is met or a timeout fires.

| Class | Description | Key Parameters | Output Delegates |
|---|---|---|---|
| `AbilityTask_WaitDelay` | Waits for a specified duration | `Time` (float) | `OnFinish` |
| `AbilityTask_WaitCancel` | Waits until the ability is cancelled | -- | `OnCancel` |
| `AbilityTask_WaitConfirm` | Waits for the server to confirm a predicted action | -- | `OnConfirm` |
| `AbilityTask_WaitConfirmCancel` | Waits for either confirm or cancel | -- | `OnConfirm`, `OnCancel` |
| `AbilityTask_WaitInputPress` | Waits for the ability's input to be pressed | `bTestAlreadyPressed` (bool) | `OnPress` |
| `AbilityTask_WaitInputRelease` | Waits for the ability's input to be released | `bTestAlreadyReleased` (bool) | `OnRelease` |
| `AbilityTask_WaitMovementModeChange` | Waits for the character's movement mode to change | `NewMode` (EMovementMode) | `OnChange` |
| `AbilityTask_WaitOverlap` | Waits for a primitive component overlap event | -- | `OnOverlap` |
| `AbilityTask_WaitVelocityChange` | Waits until the character's velocity meets a threshold condition | `Direction` (FVector), `MinimumMagnitude` (float) | `OnVelocityChg` |

---

## Wait Ability Tasks

Tasks that wait for other abilities to activate or commit.

| Class | Description | Key Parameters | Output Delegates |
|---|---|---|---|
| `AbilityTask_WaitAbilityActivate` | Waits for any ability matching a tag query to activate | `WithTag`/`WithoutTag` (FGameplayTag), `Query` (FGameplayTagQuery), `bTriggerOnce` | `OnActivate` |
| `AbilityTask_WaitAbilityCommit` | Waits for any ability matching a tag query to commit | `WithTag`/`WithoutTag` (FGameplayTag), `Query` (FGameplayTagQuery), `bTriggerOnce` | `OnCommit` |

---

## Gameplay Effect Tasks

Tasks that react to Gameplay Effect application, removal, and stacking.

| Class | Description | Key Parameters | Output Delegates |
|---|---|---|---|
| `AbilityTask_WaitGameplayEffectApplied_Self` | Waits for a GE matching criteria to be applied to the owning ASC | `SourceFilter` (FGameplayTargetDataFilterHandle), `SourceTagRequirements`/`TargetTagRequirements` (FGameplayTagRequirements), `TriggerOnce` | `OnApplied` |
| `AbilityTask_WaitGameplayEffectApplied_Target` | Waits for a GE matching criteria to be applied to another ASC | Same as Self variant | `OnApplied` |
| `AbilityTask_WaitGameplayEffectRemoved` | Waits for a specific active GE (by handle) to be removed | `Handle` (FActiveGameplayEffectHandle) | `OnRemoved`, `InvalidHandle` |
| `AbilityTask_WaitGameplayEffectStackChange` | Waits for a specific active GE's stack count to change | `Handle` (FActiveGameplayEffectHandle) | `OnChange`, `InvalidHandle` |
| `AbilityTask_WaitGameplayEffectBlockedImmunity` | Waits for a GE application to be blocked by immunity on the owning ASC | `SourceTagRequirements`/`TargetTagRequirements` (FGameplayTagRequirements) | `Blocked` |

---

## Gameplay Tag Tasks

Tasks that react to tag changes on the owning ASC.

| Class | Description | Key Parameters | Output Delegates |
|---|---|---|---|
| `AbilityTask_WaitGameplayTag` (Added) | Waits for a specific tag to be added to the ASC | `Tag` (FGameplayTag), `TriggerOnce` | `Added` |
| `AbilityTask_WaitGameplayTag` (Removed) | Waits for a specific tag to be removed from the ASC | `Tag` (FGameplayTag), `TriggerOnce` | `Removed` |
| `AbilityTask_WaitGameplayTagCountChanged` | Waits for a specific tag's count on the ASC to change | `Tag` (FGameplayTag) | `OnCountChanged` (includes new count) |
| `AbilityTask_WaitGameplayTagQuery` | Waits for a tag query to become satisfied or unsatisfied on the ASC | `TagQuery` (FGameplayTagQuery), `Condition` (trigger on match, unmatch, or either) | `Triggered` |

!!! note "Added vs. Removed"
    `AbilityTask_WaitGameplayTag` has two static factory functions: `WaitGameplayTagAdd` and `WaitGameplayTagRemove`. They share the same class but configure different internal behavior.

---

## Attribute Tasks

Tasks that react to attribute value changes.

| Class | Description | Key Parameters | Output Delegates |
|---|---|---|---|
| `AbilityTask_WaitAttributeChange` | Waits for any change to a specific attribute | `Attribute` (FGameplayAttribute), `WithTag`/`WithoutTag` (FGameplayTag) | `OnChange` |
| `AbilityTask_WaitAttributeChangeThreshold` | Waits for an attribute to cross a threshold | `Attribute` (FGameplayAttribute), `ComparisonType` (EWaitAttributeChangeComparison), `ComparisonValue` (float) | `OnChange` |
| `AbilityTask_WaitAttributeChangeRatioThreshold` | Waits for the ratio of two attributes to cross a threshold | `NumeratorAttribute`, `DenominatorAttribute` (FGameplayAttribute), `ComparisonType`, `ComparisonValue` | `OnChange` |

---

## Event Tasks

Tasks that wait for gameplay events.

| Class | Description | Key Parameters | Output Delegates |
|---|---|---|---|
| `AbilityTask_WaitGameplayEvent` | Waits for a gameplay event with a specific tag to be sent to the owning ASC | `EventTag` (FGameplayTag), `bOnlyTriggerOnce`, `bOnlyMatchExact` | `EventReceived` (provides `FGameplayEventData`) |
| `AbilityTask_WaitTargetData` | Spawns a `AGameplayAbilityTargetActor` and waits for target data | `TargetClass` (TSubclassOf), `ConfirmationType` | `ValidData`, `Cancelled` |

---

## Action Tasks

Tasks that perform actions and report completion.

| Class | Description | Key Parameters | Output Delegates |
|---|---|---|---|
| `AbilityTask_PlayMontageAndWait` | Plays an animation montage and waits for it to finish | `MontageToPlay`, `Rate`, `StartSection`, `bStopWhenAbilityEnds` | `OnCompleted`, `OnBlendOut`, `OnInterrupted`, `OnCancelled` |
| `AbilityTask_PlayAnimAndWait` | Plays a `UAnimAsset` (non-montage) and waits for it to finish | `AnimAsset`, `bStopWhenAbilityEnds` | `OnCompleted`, `OnInterrupted`, `OnCancelled` |
| `AbilityTask_SpawnActor` | Spawns an actor (with begin-spawn/finish-spawn pattern for setting properties) | `SpawnClass`, `TargetData` (FGameplayAbilityTargetDataHandle) | `Success` (provides spawned actor), `DidNotSpawn` |
| `AbilityTask_MoveToLocation` | Lerps the avatar to a target location over time | `TargetLocation` (FVector), `Duration`, `CurveFloat` (optional) | `OnTargetLocationReached` |
| `AbilityTask_VisualizeTargeting` | Spawns and updates a targeting reticle actor until the task ends | `TargetClass`, `TargetData` | -- (controlled externally) |
| `AbilityTask_StartAbilityState` | Enters a named ability state that auto-ends when the task is destroyed | `StateName` (FName), `bEndCurrentState` | `OnStateEnded`, `OnStateInterrupted` |
| `AbilityTask_NetworkSyncPoint` | Synchronizes ability execution between client and server | `SyncType` (EAbilityTaskNetSyncType: BothWait, OnlyServerWait, OnlyClientWait) | `OnSync` |
| `AbilityTask_Repeat` | Calls a delegate repeatedly at a fixed interval | `TimeBetweenActions` (float), `TotalActionCount` (int32) | `OnPerformAction` (provides action number), `OnFinished` |

---

## Root Motion Tasks

Tasks that apply root motion forces to characters. All inherit from `AbilityTask_ApplyRootMotion_Base`.

| Class | Description | Key Parameters | Output Delegates |
|---|---|---|---|
| `AbilityTask_ApplyRootMotionConstantForce` | Applies a constant force in a direction | `WorldDirection` (FVector), `Strength`, `Duration`, `bIsAdditive`, `StrengthOverTime` (UCurveFloat) | `OnFinish` |
| `AbilityTask_ApplyRootMotionJumpForce` | Applies a jump arc | `Rotation` (FRotator), `Distance`, `Height`, `Duration`, `MinimumLandedTriggerTime` | `OnFinish`, `OnLanded` |
| `AbilityTask_ApplyRootMotionMoveToActorForce` | Moves toward a target actor | `TargetActor`, `TargetLocationOffset`, `OffsetAlignment`, `Duration`, `bSetNewMovementMode` | `OnFinish` |
| `AbilityTask_ApplyRootMotionMoveToForce` | Moves toward a target location | `TargetLocation` (FVector), `Duration`, `bSetNewMovementMode`, `bRestrictSpeedToExpected` | `OnFinish` |
| `AbilityTask_ApplyRootMotionRadialForce` | Applies a radial force (push/pull) from a point | `Location` (FVector), `LocationActor` (AActor), `Strength`, `Duration`, `Radius`, `bIsPush` | `OnFinish` |

---

## Async Ability Tasks (UAbilityAsync)

These tasks inherit from `UAbilityAsync` instead of `UAbilityTask`. They do **not** require an active ability and can be used from any Blueprint (widgets, AI controllers, etc.). They bind to an ASC via a target actor parameter.

All async tasks live in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Async/`.

| Class | Description | Key Parameters | Output Delegates |
|---|---|---|---|
| `UAbilityAsync_WaitAttributeChanged` | Monitors an attribute for changes on a target ASC | `TargetActor`, `Attribute` (FGameplayAttribute), `bOnlyTriggerOnce` | `Changed` (provides `FGameplayAttribute`, NewValue, OldValue) |
| `UAbilityAsync_WaitGameplayEffectApplied` | Monitors a target ASC for GE application events | `TargetActor`, `SourceTagRequirements`/`TargetTagRequirements` (FGameplayTagRequirements), `bTriggerOnce` | `OnApplied` |
| `UAbilityAsync_WaitGameplayEvent` | Monitors a target ASC for gameplay events matching a tag | `TargetActor`, `EventTag` (FGameplayTag), `bOnlyTriggerOnce`, `bOnlyMatchExact` | `EventReceived` (provides `FGameplayEventData`) |
| `UAbilityAsync_WaitGameplayTag` | Monitors a target ASC for a tag being added or removed | `TargetActor`, `Tag` (FGameplayTag), `TagChangeType` (Added/Removed), `bOnlyTriggerOnce` | `OnTagChanged` |
| `UAbilityAsync_WaitGameplayTagCountChanged` | Monitors a target ASC for a tag count change | `TargetActor`, `Tag` (FGameplayTag) | `TagCountChanged` (provides new count) |
| `UAbilityAsync_WaitGameplayTagQuery` | Monitors a target ASC until a tag query is satisfied/unsatisfied | `TargetActor`, `TagQuery` (FGameplayTagQuery), `TriggerCondition` | `Triggered` |

!!! tip "When to use Async vs. regular tasks"
    Use `UAbilityAsync` when you need to monitor GAS state from outside an ability — for example, updating a health bar widget when an attribute changes. Use `UAbilityTask` when the monitoring is part of ability logic and should end when the ability ends.
