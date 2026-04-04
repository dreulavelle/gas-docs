---
title: Connecting GAS to UI
description: Best practices for driving UMG widgets from GAS data — attribute listeners, cooldown displays, buff icons, damage numbers, and reactive UI patterns that never poll.
---

# Connecting GAS to UI

Displaying ability system data in your UI is one of the most common tasks in a GAS project -- and one of the easiest to get wrong. Health bars, cooldown sweeps, buff icons, damage numbers... they all need GAS data, and they all need it without tanking your frame rate.

This page covers the industry-standard techniques for building reactive UI that *listens* to GAS events instead of polling for changes.

## The Golden Rule

!!! danger "Never read attributes in Tick"
    UI should **react** to GAS events, not poll for changes. If your widget has a `Tick` function that calls `GetNumericAttribute` every frame, you're doing it wrong.

    The problems with polling:

    - **Wasteful** -- you're checking a value 60+ times per second when it might change once every few seconds
    - **Scales poorly** -- 10 widgets polling 3 attributes each = 30 attribute lookups per frame, for no reason
    - **Misses context** -- polling gives you the current value but not *what changed it* or *by how much*

    GAS has a rich delegate and async task system specifically designed for this. Use it.

The correct approach uses three mechanisms, depending on the situation:

| Mechanism | Best For | Works From |
|:---|:---|:---|
| `UAbilityAsync_WaitAttributeChanged` | Attribute bars (health, stamina, mana) | Any Blueprint -- widgets, actors, anything |
| `GetGameplayAttributeValueChangeDelegate` | C++ widgets that need `FOnAttributeChangeData` details | C++ only |
| `RegisterGameplayTagEvent` / `UAbilityAsync_WaitGameplayTagCountChanged` | Buff icons, status indicators, cooldown state | Any Blueprint or C++ |

---

## Health / Stamina / Mana Bars -- Listening to Attribute Changes

This is the most common UI task: a progress bar that reflects the current value of an attribute. The recommended approach uses **async ability tasks** -- specifically `UAbilityAsync_WaitAttributeChanged`.

!!! info "Async tasks vs. Ability Tasks"
    Don't confuse these with `UAbilityTask` subclasses (like `WaitForAttributeChange`). Ability Tasks only work inside an active ability. **Async ability tasks** (`UAbilityAsync` subclasses) work from *anywhere* -- widgets, actors, game modes, whatever. They inherit from `UCancellableAsyncAction`, not `UAbilityTask`. This is what makes them perfect for UI.

=== "Blueprint (UMG Widget)"
    In your widget's **Construct** event:

    **1. Get the ASC and set initial values**

    ```
    Event Construct
        |
        v
    Get Owning Player Pawn
        |
        v
    Get Ability System Component (from pawn)
        |-- [Store as variable: ASCRef (weak reference)]
        |
        v
    Get Numeric Attribute Value (Health)
        |-- Set Progress Bar Percent = Health / MaxHealth
        |
        v
    Get Numeric Attribute Value (Stamina)
        |-- Set Progress Bar Percent = Stamina / MaxStamina
    ```

    !!! warning "Handle initial state"
        The listener won't fire until the attribute *changes*. If your widget is created after the character already has 80/100 health, you need to read the current value on Construct. Otherwise the bar starts empty (or full) until the first change.

    **2. Create async listeners**

    ```
    (continuing from Construct)
        |
        v
    Wait for Attribute Changed (TargetActor: Pawn, Attribute: Health)
        |-- Changed → OnHealthChanged (custom event)
        |-- [Store as variable: HealthListener]
        |
        v
    Wait for Attribute Changed (TargetActor: Pawn, Attribute: Stamina)
        |-- Changed → OnStaminaChanged (custom event)
        |-- [Store as variable: StaminaListener]
    ```

    **3. Handle the callback**

    ```
    Event OnHealthChanged (Attribute, NewValue, OldValue)
        |
        v
    Set Progress Bar Percent = NewValue / MaxHealth
    ```

    Repeat the same pattern for Stamina and Mana. Each attribute gets its own async task and callback.

    **4. Clean up on Destruct**

    ```
    Event Destruct
        |
        v
    HealthListener → End Action
    StaminaListener → End Action
    ```

    !!! tip "Why End Action?"
        `UAbilityAsync` tasks are garbage collected when nothing references them, but calling `EndAction` explicitly unregisters the delegate immediately. This prevents callbacks firing on a widget that's being torn down -- which can crash.

=== "C++"
    The C++ approach uses the ASC's native delegate directly, which gives you the full `FOnAttributeChangeData` struct:

    ```cpp
    // UMyHealthBarWidget.h
    #pragma once

    #include "CoreMinimal.h"
    #include "Blueprint/UserWidget.h"
    #include "AttributeSet.h"
    #include "GameplayEffectTypes.h"
    #include "MyHealthBarWidget.generated.h"

    class UAbilitySystemComponent;
    class UProgressBar;

    UCLASS()
    class UMyHealthBarWidget : public UUserWidget
    {
        GENERATED_BODY()

    public:
        /** Call this after the widget is created and the ASC is available */
        UFUNCTION(BlueprintCallable, Category = "GAS|UI")
        void InitializeWithASC(UAbilitySystemComponent* InASC);

    protected:
        virtual void NativeDestruct() override;

    private:
        void OnHealthChanged(const FOnAttributeChangeData& ChangeData);
        void OnMaxHealthChanged(const FOnAttributeChangeData& ChangeData);
        void UpdateHealthBar();

        UPROPERTY(meta = (BindWidget))
        TObjectPtr<UProgressBar> HealthBar;

        TWeakObjectPtr<UAbilitySystemComponent> ASCRef;
        float CachedHealth = 0.f;
        float CachedMaxHealth = 1.f;

        FDelegateHandle HealthChangedHandle;
        FDelegateHandle MaxHealthChangedHandle;
    };
    ```

    ```cpp
    // UMyHealthBarWidget.cpp
    #include "MyHealthBarWidget.h"
    #include "AbilitySystemComponent.h"
    #include "Components/ProgressBar.h"

    void UMyHealthBarWidget::InitializeWithASC(
        UAbilitySystemComponent* InASC)
    {
        if (!InASC)
        {
            return;
        }

        ASCRef = InASC;

        // Read current values (handle initial state)
        bool bFound = false;
        CachedHealth = InASC->GetGameplayAttributeValue(
            UMyAttributeSet::GetHealthAttribute(), bFound);
        CachedMaxHealth = InASC->GetGameplayAttributeValue(
            UMyAttributeSet::GetMaxHealthAttribute(), bFound);
        UpdateHealthBar();

        // Bind to future changes
        HealthChangedHandle = InASC
            ->GetGameplayAttributeValueChangeDelegate(
                UMyAttributeSet::GetHealthAttribute())
            .AddUObject(this, &UMyHealthBarWidget::OnHealthChanged);

        MaxHealthChangedHandle = InASC
            ->GetGameplayAttributeValueChangeDelegate(
                UMyAttributeSet::GetMaxHealthAttribute())
            .AddUObject(this, &UMyHealthBarWidget::OnMaxHealthChanged);
    }

    void UMyHealthBarWidget::NativeDestruct()
    {
        // Always unbind -- prevents dangling delegate calls
        if (UAbilitySystemComponent* ASC = ASCRef.Get())
        {
            ASC->GetGameplayAttributeValueChangeDelegate(
                UMyAttributeSet::GetHealthAttribute())
                .Remove(HealthChangedHandle);
            ASC->GetGameplayAttributeValueChangeDelegate(
                UMyAttributeSet::GetMaxHealthAttribute())
                .Remove(MaxHealthChangedHandle);
        }

        Super::NativeDestruct();
    }

    void UMyHealthBarWidget::OnHealthChanged(
        const FOnAttributeChangeData& ChangeData)
    {
        CachedHealth = ChangeData.NewValue;
        UpdateHealthBar();
    }

    void UMyHealthBarWidget::OnMaxHealthChanged(
        const FOnAttributeChangeData& ChangeData)
    {
        CachedMaxHealth = ChangeData.NewValue;
        UpdateHealthBar();
    }

    void UMyHealthBarWidget::UpdateHealthBar()
    {
        if (HealthBar && CachedMaxHealth > 0.f)
        {
            HealthBar->SetPercent(CachedHealth / CachedMaxHealth);
        }
    }
    ```

    **What you get from `FOnAttributeChangeData`:**

    | Field | Type | Description |
    |:---|:---|:---|
    | `Attribute` | `FGameplayAttribute` | Which attribute changed |
    | `NewValue` | `float` | The new value after modification |
    | `OldValue` | `float` | The value before modification |
    | `GEModData` | `const FGameplayEffectModCallbackData*` | Full context -- which GE caused the change, who applied it, etc. Can be null. |

    The `GEModData` field is powerful for advanced UI -- you can determine *who* dealt the damage, *which effect* caused the change, and use that to color-code damage numbers or show source icons.

---

## Cooldown Display -- Showing Time Remaining

Cooldown UI needs to show a shrinking progress indicator (radial wipe, linear bar, countdown text) while an ability is on cooldown. There are two approaches.

### Approach A: Tag-Based with Local Timer (Recommended)

Listen for the cooldown tag being added or removed, then drive a local tween. This is event-driven at the boundaries and uses a lightweight local interpolation for the smooth visual.

=== "Blueprint"
    ```
    Event Construct
        |
        v
    Wait Gameplay Tag Count Changed on Actor
        (TargetActor: OwnerPawn, Tag: Cooldown.Ability.BasicAttack)
        |
        +-- TagCountChanged (TagCount)
            |
            +-- [TagCount > 0] → Start Cooldown Display
            |       |
            |       v
            |   Get Cooldown Time Remaining and Duration
            |       (from the ability instance on the ASC)
            |       |
            |       v
            |   Set CooldownDuration variable
            |   Set CooldownStartTime = GetGameTimeSinceCreation()
            |   Set bOnCooldown = true
            |   Start local Tick update (or use a Timeline)
            |
            +-- [TagCount == 0] → End Cooldown Display
                    |
                    v
                Set bOnCooldown = false
                Set Progress = 0 (fully available)
    ```

    For the smooth visual update during cooldown, use a **Timeline** or a simple `Tick` that interpolates:

    ```
    Widget Tick (only while bOnCooldown)
        |
        v
    Elapsed = GetGameTimeSinceCreation() - CooldownStartTime
    Progress = 1.0 - (Elapsed / CooldownDuration)
    Set Radial Progress = Clamp(Progress, 0, 1)
    ```

    !!! note "Tick is OK here"
        Yes, we just said never poll in Tick. But this is different -- we're not *reading from GAS* in Tick. We're interpolating a local timer that was *started by a GAS event*. The Tick only runs while cooldown is active, and it's doing pure math -- no ASC calls. This is the standard pattern for smooth cooldown visuals.

=== "C++"
    ```cpp
    void UAbilitySlotWidget::InitializeForAbility(
        UAbilitySystemComponent* InASC,
        FGameplayTag InCooldownTag)
    {
        ASCRef = InASC;
        CooldownTag = InCooldownTag;

        // Listen for cooldown tag changes
        CooldownTagHandle = InASC->RegisterGameplayTagEvent(
            CooldownTag,
            EGameplayTagEventType::NewOrRemoved)
            .AddUObject(this,
                &UAbilitySlotWidget::OnCooldownTagChanged);
    }

    void UAbilitySlotWidget::OnCooldownTagChanged(
        const FGameplayTag Tag, int32 NewCount)
    {
        if (NewCount > 0)
        {
            // Cooldown started -- get duration from the ability
            float TimeRemaining = 0.f;
            float TotalDuration = 0.f;

            // Find the ability spec and query cooldown
            if (UAbilitySystemComponent* ASC = ASCRef.Get())
            {
                TArray<FGameplayAbilitySpec*> MatchingSpecs;
                // Find specs that have this cooldown tag
                // (you'd typically store the ability spec handle)
                if (AbilitySpecHandle.IsValid())
                {
                    FGameplayAbilitySpec* Spec =
                        ASC->FindAbilitySpecFromHandle(
                            AbilitySpecHandle);
                    if (Spec && Spec->Ability)
                    {
                        Spec->Ability
                            ->GetCooldownTimeRemainingAndDuration(
                                AbilitySpecHandle,
                                ASC->AbilityActorInfo.Get(),
                                TimeRemaining,
                                TotalDuration);
                    }
                }
            }

            CooldownDuration = TotalDuration;
            CooldownStartTime =
                GetWorld()->GetTimeSeconds();
            bOnCooldown = true;
        }
        else
        {
            // Cooldown ended
            bOnCooldown = false;
            UpdateCooldownVisual(0.f);
        }
    }

    void UAbilitySlotWidget::NativeTick(
        const FGeometry& MyGeometry, float InDeltaTime)
    {
        Super::NativeTick(MyGeometry, InDeltaTime);

        if (bOnCooldown && CooldownDuration > 0.f)
        {
            const float Elapsed =
                GetWorld()->GetTimeSeconds() - CooldownStartTime;
            const float Progress =
                FMath::Clamp(
                    1.f - (Elapsed / CooldownDuration), 0.f, 1.f);
            UpdateCooldownVisual(Progress);

            if (Progress <= 0.f)
            {
                bOnCooldown = false;
            }
        }
    }
    ```

### Approach B: Periodic Query (Simpler, Fine for Single-Player)

If you don't need frame-perfect cooldown visuals, you can poll at a low frequency. This is simpler but less efficient for multiplayer with many ability slots.

```cpp
// In a timer callback (every 0.1 seconds, not every frame)
float TimeRemaining = 0.f;
float TotalDuration = 0.f;

AbilityInstance->GetCooldownTimeRemainingAndDuration(
    SpecHandle,
    ASC->AbilityActorInfo.Get(),
    TimeRemaining,
    TotalDuration);

float Progress = (TotalDuration > 0.f)
    ? (TimeRemaining / TotalDuration)
    : 0.f;

UpdateCooldownVisual(Progress);
```

**Key function signatures** (from `UGameplayAbility`):

```cpp
// Returns seconds remaining on the active cooldown
float GetCooldownTimeRemaining() const;

// Returns both remaining time and the original total duration
virtual void GetCooldownTimeRemainingAndDuration(
    FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    float& TimeRemaining,
    float& CooldownDuration) const;

// Returns the tags that represent this ability's cooldown
virtual const FGameplayTagContainer* GetCooldownTags() const;
```

---

## Buff/Debuff Icons -- Tag-Based UI State

Buff and debuff bars track active status effects by listening for tag changes. The pattern is straightforward: when a tag from a known category appears, show an icon. When it's removed, hide it.

### The Listener

Use `UAbilityAsync_WaitGameplayTagCountChanged` for each tag you care about, or `RegisterGameplayTagEvent` in C++.

=== "Blueprint"
    ```
    Event Construct
        |
        v
    For each buff tag in BuffTagList (array of FGameplayTag):
        |
        v
    Wait Gameplay Tag Count Changed on Actor
        (TargetActor: Pawn, Tag: current buff tag)
        |
        +-- TagCountChanged (TagCount)
            |
            +-- [TagCount > 0] → AddOrUpdateBuffIcon(Tag, TagCount)
            +-- [TagCount == 0] → RemoveBuffIcon(Tag)
    ```

=== "C++"
    ```cpp
    void UBuffBarWidget::RegisterBuffTag(
        UAbilitySystemComponent* ASC,
        FGameplayTag BuffTag,
        TObjectPtr<UTexture2D> Icon)
    {
        FDelegateHandle Handle = ASC->RegisterGameplayTagEvent(
            BuffTag,
            EGameplayTagEventType::AnyCountChange)
            .AddUObject(this,
                &UBuffBarWidget::OnBuffTagChanged);

        // Store for cleanup
        BuffTagHandles.Add(BuffTag, Handle);
        BuffTagIcons.Add(BuffTag, Icon);
    }

    void UBuffBarWidget::OnBuffTagChanged(
        const FGameplayTag Tag, int32 NewCount)
    {
        if (NewCount > 0)
        {
            // Add or update the icon
            if (UTexture2D** Icon = BuffTagIcons.Find(Tag))
            {
                ShowBuffIcon(Tag, *Icon, NewCount);
            }
        }
        else
        {
            HideBuffIcon(Tag);
        }
    }
    ```

### The Buff Bar Widget Structure

A typical buff bar widget has:

| Component | Type | Purpose |
|:---|:---|:---|
| `BuffContainer` | `UHorizontalBox` or `UWrapBox` | Holds buff icon slots |
| `BuffSlotClass` | `TSubclassOf<UUserWidget>` | The widget class for a single icon |
| `ActiveBuffSlots` | `TMap<FGameplayTag, UUserWidget*>` | Maps tags to active slot widgets |

**On tag added (count > 0):**

1. Check `ActiveBuffSlots` -- if tag already has a slot, update the stack count overlay
2. If no slot exists, create one from `BuffSlotClass`, add it to `BuffContainer`, register in the map
3. Set the icon texture and stack count text

**On tag removed (count == 0):**

1. Find the slot in `ActiveBuffSlots`
2. Remove it from `BuffContainer`
3. Remove from the map

!!! tip "Use AnyCountChange for stack tracking"
    `EGameplayTagEventType::AnyCountChange` fires every time the tag count changes -- not just when it goes to/from zero. This is essential for stack count overlays. `NewOrRemoved` only fires on the 0-to-1 and 1-to-0 transitions, so you'd miss stack count updates.

---

## Ability State Indicators (Ready / On Cooldown / Blocked)

Action bar slots need to show whether an ability is available, on cooldown, blocked by tags, or missing resources. The temptation is to call `CanActivateAbility` every frame -- resist it.

### Reactive State Machine

Instead of polling `CanActivateAbility`, listen to the individual conditions and derive state:

```
enum EAbilitySlotState
{
    Ready,        // Can activate
    OnCooldown,   // Cooldown tag present
    Blocked,      // Blocking tag present (stunned, silenced, etc.)
    NoResource,   // Not enough stamina/mana (attribute below cost)
};
```

**Drive state from listeners:**

| Condition | Listener | State Transition |
|:---|:---|:---|
| Cooldown tag appears | `RegisterGameplayTagEvent(CooldownTag)` | `Ready -> OnCooldown` |
| Cooldown tag removed | Same delegate, count == 0 | `OnCooldown -> Ready` |
| Blocking tag appears | `RegisterGameplayTagEvent(BlockingTag)` | `Any -> Blocked` |
| Resource drops below cost | `GetGameplayAttributeValueChangeDelegate` | `Ready -> NoResource` |
| Resource rises above cost | Same delegate | `NoResource -> Ready` |

**Visual mapping:**

| State | Visual |
|:---|:---|
| Ready | Normal icon, full brightness |
| OnCooldown | Darkened icon + radial sweep overlay |
| Blocked | Red tint or lock icon overlay |
| NoResource | Desaturated icon, pulsing resource cost text |

This is more setup than polling `CanActivateAbility`, but it's zero-cost per frame and gives you granular control over the visual feedback for each failure reason.

---

## Damage Numbers / Combat Text

Floating damage numbers are a staple of action games. The cleanest approach uses **Gameplay Cues** -- keeping the damage pipeline completely decoupled from the UI.

### The Gameplay Cue Approach (Recommended)

**1. Trigger a cue from the damage pipeline**

In your `PostGameplayEffectExecute` (after calculating final damage), execute a Gameplay Cue:

```cpp
void UMyAttributeSet::PostGameplayEffectExecute(
    const FGameplayEffectModCallbackData& Data)
{
    // ... damage processing ...

    const float FinalDamage = GetPendingDamage();
    SetPendingDamage(0.f);
    SetHealth(FMath::Clamp(
        GetHealth() - FinalDamage, 0.f, GetMaxHealth()));

    // Trigger a Gameplay Cue for the damage number
    if (FinalDamage > 0.f)
    {
        FGameplayCueParameters CueParams;
        CueParams.RawMagnitude = FinalDamage;
        CueParams.Location =
            Data.Target.AbilityActorInfo->AvatarActor
                ->GetActorLocation();

        Data.Target.AbilityActorInfo
            ->AbilitySystemComponent.Get()
            ->ExecuteGameplayCue(
                FGameplayTag::RequestGameplayTag(
                    FName("GameplayCue.UI.DamageNumber")),
                CueParams);
    }
}
```

**2. Create the Gameplay Cue handler**

Create a Blueprint class inheriting from `AGameplayCueNotify_Actor` (or use `UGameplayCueNotify_Static` if you don't need a persistent actor). Name it `GC_UI_DamageNumber` and set its Gameplay Cue Tag to `GameplayCue.UI.DamageNumber`.

In the cue's `OnExecute` event:

```
OnExecute (Parameters)
    |
    v
Get RawMagnitude from Parameters → This is the damage value
Get Location from Parameters → This is where to spawn the number
    |
    v
Create Widget (DamageNumberWidgetClass)
    |-- Set damage text = Format as int
    |-- Set color (red for damage, green for healing)
    |
    v
Add to Viewport (or use a WidgetComponent in world space)
    |
    v
Play float-up animation (move Y, fade alpha over 1-2 seconds)
    |
    v
Remove from Parent when animation completes
```

**`FGameplayCueParameters` fields useful for damage numbers:**

| Field | Type | Use |
|:---|:---|:---|
| `RawMagnitude` | `float` | The damage/healing amount |
| `NormalizedMagnitude` | `float` | Magnitude normalized 0-1 (for scaling effects) |
| `Location` | `FVector` | World position for the number |
| `Normal` | `FVector` | Hit direction (for directional pop) |
| `EffectCauser` | `TWeakObjectPtr<AActor>` | Who caused the damage |

!!! tip "Why Gameplay Cues?"
    The damage pipeline doesn't know about UI. The UI doesn't know about the damage pipeline. The Gameplay Cue is the bridge -- it's GAS's built-in mechanism for "something happened, someone might want to react visually." This means your damage processing code never imports a widget header, and your widget code never imports an attribute set header. Clean separation.

### Alternative: Delegate from PostGameplayEffectExecute

If you don't want to use Gameplay Cues, you can define a custom delegate on your AttributeSet or a subsystem and broadcast from `PostGameplayEffectExecute`. The HUD widget binds to this delegate.

```cpp
// On your attribute set or a game subsystem
DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(
    FOnDamageDealt,
    float, DamageAmount,
    AActor*, TargetActor,
    AActor*, InstigatorActor);

UPROPERTY(BlueprintAssignable)
FOnDamageDealt OnDamageDealt;
```

This works, but it's tightly coupled compared to Gameplay Cues. The cue approach also handles networking automatically (cues can be replicated or executed locally based on your cue notify configuration), while a custom delegate requires manual replication consideration.

---

## Common Mistakes

### 1. Reading Attributes in Tick

We've beaten this one, but it bears repeating. Don't do this:

```cpp
// BAD -- runs every frame, even when nothing changed
void UMyWidget::NativeTick(const FGeometry& Geo, float Delta)
{
    Super::NativeTick(Geo, Delta);
    bool bFound;
    float Health = ASC->GetGameplayAttributeValue(
        UMyAttributeSet::GetHealthAttribute(), bFound);
    HealthBar->SetPercent(Health / MaxHealth);
}
```

Use a delegate. Your health bar only needs to update when health actually changes.

### 2. Holding Strong References to ASC

```cpp
// BAD -- prevents GC of the pawn/ASC after death/respawn
UPROPERTY()
TObjectPtr<UAbilitySystemComponent> ASCRef;

// GOOD -- weak pointer allows GC, check validity before use
TWeakObjectPtr<UAbilitySystemComponent> ASCRef;
```

When a pawn dies and is destroyed, you want the garbage collector to clean it up. A `UPROPERTY()` `TObjectPtr` on a widget keeps the ASC (and its owner) alive. Use `TWeakObjectPtr` and check `IsValid()` before accessing.

### 3. Not Handling Initial State

Your widget is created. The character already has 73/100 health. Your attribute listener hasn't fired yet because nothing changed since the widget was created.

**Always read current values on widget initialization**, then start listening for changes. The listener covers future changes; the initial read covers the present.

### 4. Not Cleaning Up Listeners

```cpp
// If you don't remove the delegate handle, the callback fires
// on a destroyed widget → crash
void UMyWidget::NativeDestruct()
{
    if (UAbilitySystemComponent* ASC = ASCRef.Get())
    {
        ASC->GetGameplayAttributeValueChangeDelegate(
            UMyAttributeSet::GetHealthAttribute())
            .Remove(HealthChangedHandle);
    }
    Super::NativeDestruct();
}
```

For Blueprint async tasks (`UAbilityAsync` subclasses), call `EndAction` on Destruct. For C++ delegates, remove the handle. For `RegisterGameplayTagEvent`, call `UnregisterGameplayTagEvent` with the stored `FDelegateHandle`.

### 5. Forgetting MaxHealth Changes

Your health bar shows `Health / MaxHealth`. You listen for Health changes -- great. But what if a buff increases MaxHealth? The bar's percentage changes even though Health didn't. Listen to *both* attributes.

---

## Quick Reference: Which Listener for What

| UI Element | What to Listen To | Recommended Mechanism |
|:---|:---|:---|
| Health/Stamina/Mana bar | Attribute value + max value | `UAbilityAsync_WaitAttributeChanged` or `GetGameplayAttributeValueChangeDelegate` |
| Cooldown sweep | Cooldown tag (add/remove) | `UAbilityAsync_WaitGameplayTagCountChanged` or `RegisterGameplayTagEvent` |
| Buff/debuff icons | Status tags (any count change) | `RegisterGameplayTagEvent` with `AnyCountChange` |
| Ability slot state | Cooldown tag + blocking tags + resource attribute | Multiple listeners, state machine |
| Damage numbers | Gameplay Cue execution | `GameplayCue.UI.DamageNumber` handler |
| Stack count overlay | Status tag count changes | `RegisterGameplayTagEvent` with `AnyCountChange` |

---

## Related Pages

- [Async Ability Tasks](../gameplay-abilities/async-ability-tasks.md) -- full reference for `UAbilityAsync` tasks
- [Gameplay Cues Overview](../core-concepts/gameplay-cues-overview.md) -- how cues work, cue types, cue parameters
- [Cue Parameters](../gameplay-cues/cue-parameters.md) -- `FGameplayCueParameters` field reference
- [Attributes and Attribute Sets](../core-concepts/attributes-and-attribute-sets.md) -- attribute definitions, `PostGameplayEffectExecute`
- [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) -- how the cooldown system works under the hood
- [Buff/Debuff System](buff-debuff-system.md) -- tag categories, stacking, UI representation
- [Tag Architecture](tag-design.md) -- designing your tag hierarchy for clean UI bindings
- [UI Data](../gameplay-effects/ui-data.md) -- effect display names, descriptions, and icons for UI
