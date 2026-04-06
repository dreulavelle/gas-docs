---
title: Input Binding
description: How to connect Enhanced Input to Gameplay Abilities -- the InputID approach, tag-based routing (Lyra pattern), anti-patterns, and complete working examples.
---

# Input Binding

Connecting player input to ability activation is one of the first things you need to do in any GAS project. GAS does not ship with a turnkey Enhanced Input integration -- you build the bridge yourself. The good news is that there are only two correct patterns, and both are straightforward once you understand the moving parts.

This page is the canonical deep-dive. It covers both approaches end-to-end with complete code examples, explains the critical details that are easy to get wrong, and gives you enough context to choose the right approach for your project.

## Before You Choose

There are two correct ways to wire Enhanced Input to GAS. Both work. Both support input-aware ability tasks like `WaitInputRelease`. They differ in complexity and how well they scale to large, dynamic ability sets.

| | **InputID** | **Tag-Based Routing** |
|:---|:---|:---|
| **Complexity** | Lower -- uses built-in `FGameplayAbilitySpec::InputID` field | Higher -- requires a custom ASC subclass |
| **Scalability** | Fixed enum of input slots | Unlimited -- tags are data, not code |
| **Data-driven** | Enum values are defined in C++ | Tags can be created in the editor or `.ini` files |
| **Ability task support** | Automatic -- `AbilityLocalInputPressed/Released` updates spec input state | Manual -- you must call `AbilitySpecInputPressed/Released` yourself |
| **C++ required** | Yes -- `SetupPlayerInputComponent` is the binding loop | Yes -- same reason |
| **Used by** | GAS Companion plugin, many shipped indie games | Lyra, larger studio projects with dynamic ability sets |
| **Works well for** | Projects with a fixed set of input slots (RPG hotbar, action game) | Projects where abilities are added/removed dynamically and inputs are configured per-loadout |

!!! warning "Both approaches require C++"
    The binding loop lives in `SetupPlayerInputComponent`, which is a C++ virtual function on `ACharacter` / `APawn`. You cannot set this up purely in Blueprint. The abilities themselves can still be Blueprint classes -- only the input bridge needs C++.

!!! danger "Anti-pattern: direct TryActivateAbility from input callbacks"
    You might be tempted to skip the plumbing and call `TryActivateAbilityByClass()` or `TryActivateAbilityByTag()` directly from your input callback. This works for fire-and-forget abilities, but it **breaks input-aware ability tasks** like `WaitInputRelease` and `WaitInputPress`. Because the ASC never learns about the input state, those tasks will wait forever.

    If your project only has instant abilities (press to fire, never hold), you can get away with this. But the moment you need a held ability, a channeled ability, or confirm/cancel inputs, you will need one of the two approaches below.

## Approach 1: InputID { #input-id }

### Overview

Every `FGameplayAbilitySpec` has an `int32 InputID` field. When you grant an ability, you assign it an integer. When the player presses a button, you call `AbilitySystemComponent->AbilityLocalInputPressed(InputID)`, and the ASC finds all specs with that `InputID` and handles activation, release forwarding, and task notification automatically.

This is the simplest path. The engine does most of the work -- you just need to define an enum, map your input actions to enum values, and wire the callbacks.

### Step 1: Define an Input Enum

Create an enum that represents your bindable input slots. Each entry maps to an integer that the ASC uses to match abilities to inputs.

```cpp
UENUM(BlueprintType)
enum class EAbilityInput : uint8
{
    None            UMETA(DisplayName = "None"),
    Confirm         UMETA(DisplayName = "Confirm"),
    Cancel          UMETA(DisplayName = "Cancel"),
    PrimaryAttack   UMETA(DisplayName = "Primary Attack"),
    SecondaryAttack UMETA(DisplayName = "Secondary Attack"),
    Dodge           UMETA(DisplayName = "Dodge"),
    Jump            UMETA(DisplayName = "Jump"),
    Ability1        UMETA(DisplayName = "Ability 1"),
    Ability2        UMETA(DisplayName = "Ability 2"),
    // Add more as needed
};
```

!!! note "None vs INDEX_NONE"
    The `None` entry should be at index 0. Abilities that are not bound to any input should use `INDEX_NONE` (-1) when granted -- not the `None` enum value. `INDEX_NONE` tells the ASC "this ability has no input binding."

### Step 2: Map Enhanced Input Actions to InputIDs

You need a way to associate each `UInputAction` asset with an enum value. A simple struct array on your character works well:

```cpp
USTRUCT(BlueprintType)
struct FAbilityInputBinding
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    TObjectPtr<UInputAction> InputAction;

    UPROPERTY(EditAnywhere)
    EAbilityInput InputID = EAbilityInput::None;
};
```

On your character class:

```cpp
UPROPERTY(EditDefaultsOnly, Category = "Input")
TArray<FAbilityInputBinding> AbilityInputBindings;
```

In the editor, populate this array:

| Input Action | InputID |
|:---|:---|
| `IA_PrimaryAttack` | PrimaryAttack |
| `IA_SecondaryAttack` | SecondaryAttack |
| `IA_Dodge` | Dodge |
| `IA_Jump` | Jump |

### Step 3: Bind in SetupPlayerInputComponent

Loop through your bindings and wire each input action to press/release callbacks:

```cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    UEnhancedInputComponent* EIC = CastChecked<UEnhancedInputComponent>(PlayerInputComponent);

    for (const FAbilityInputBinding& Binding : AbilityInputBindings)
    {
        if (Binding.InputAction)
        {
            EIC->BindAction(Binding.InputAction, ETriggerEvent::Started, this,
                &ThisClass::OnAbilityInputPressed, Binding.InputID);

            EIC->BindAction(Binding.InputAction, ETriggerEvent::Completed, this,
                &ThisClass::OnAbilityInputReleased, Binding.InputID);
        }
    }
}
```

### Step 4: Route Callbacks to the ASC

The callbacks are simple one-liners. `AbilityLocalInputPressed` and `AbilityLocalInputReleased` handle everything -- finding matching specs, activating abilities, forwarding input state to active abilities, and notifying tasks like `WaitInputRelease`.

```cpp
void AMyCharacter::OnAbilityInputPressed(EAbilityInput InputID)
{
    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->AbilityLocalInputPressed(static_cast<int32>(InputID));
    }
}

void AMyCharacter::OnAbilityInputReleased(EAbilityInput InputID)
{
    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->AbilityLocalInputReleased(static_cast<int32>(InputID));
    }
}
```

### Step 5: Grant Abilities with InputIDs

When granting an ability, pass the `InputID` as the third constructor argument to `FGameplayAbilitySpec`:

```cpp
void AMyCharacter::GrantAbility(TSubclassOf<UGameplayAbility> AbilityClass, EAbilityInput InputID)
{
    if (!AbilitySystemComponent || !AbilityClass) return;

    FGameplayAbilitySpec Spec(
        AbilityClass,
        1,                                    // Level
        static_cast<int32>(InputID),          // InputID
        this                                  // Source object
    );

    AbilitySystemComponent->GiveAbility(Spec);
}
```

For abilities that are not bound to any input (activated by events, tags, or direct code calls), pass `INDEX_NONE`:

```cpp
FGameplayAbilitySpec Spec(AbilityClass, 1, INDEX_NONE, this);
```

### Complete Example { #inputid-complete-example }

Here is the full character header and implementation showing all pieces together.

**MyCharacter.h:**

```cpp
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "AbilitySystemInterface.h"
#include "InputAction.h"
#include "MyCharacter.generated.h"

UENUM(BlueprintType)
enum class EAbilityInput : uint8
{
    None,
    Confirm,
    Cancel,
    PrimaryAttack,
    SecondaryAttack,
    Dodge,
    Jump,
    Ability1,
    Ability2,
};

USTRUCT(BlueprintType)
struct FAbilityInputBinding
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    TObjectPtr<UInputAction> InputAction;

    UPROPERTY(EditAnywhere)
    EAbilityInput InputID = EAbilityInput::None;
};

UCLASS()
class AMyCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    AMyCharacter();

    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

protected:
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
    virtual void BeginPlay() override;

    UPROPERTY(VisibleAnywhere, Category = "Abilities")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    /** Maps input actions to ability input IDs. Configure in the editor. */
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    TArray<FAbilityInputBinding> AbilityInputBindings;

    /** Abilities to grant on startup. */
    UPROPERTY(EditDefaultsOnly, Category = "Abilities")
    TArray<TSubclassOf<UGameplayAbility>> StartupAbilities;

private:
    void OnAbilityInputPressed(EAbilityInput InputID);
    void OnAbilityInputReleased(EAbilityInput InputID);
};
```

**MyCharacter.cpp:**

```cpp
#include "MyCharacter.h"
#include "AbilitySystemComponent.h"
#include "EnhancedInputComponent.h"

AMyCharacter::AMyCharacter()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("ASC"));
}

UAbilitySystemComponent* AMyCharacter::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}

void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();

    if (!HasAuthority() || !AbilitySystemComponent) return;

    // Grant startup abilities with their configured input IDs
    for (const TSubclassOf<UGameplayAbility>& AbilityClass : StartupAbilities)
    {
        // Default to INDEX_NONE; override with a proper InputID when needed
        FGameplayAbilitySpec Spec(AbilityClass, 1, INDEX_NONE, this);
        AbilitySystemComponent->GiveAbility(Spec);
    }

    // Set up confirm/cancel for targeting and WaitConfirmCancel tasks
    AbilitySystemComponent->GenericConfirmInputID = static_cast<int32>(EAbilityInput::Confirm);
    AbilitySystemComponent->GenericCancelInputID = static_cast<int32>(EAbilityInput::Cancel);
}

void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    UEnhancedInputComponent* EIC = CastChecked<UEnhancedInputComponent>(PlayerInputComponent);

    for (const FAbilityInputBinding& Binding : AbilityInputBindings)
    {
        if (Binding.InputAction)
        {
            EIC->BindAction(Binding.InputAction, ETriggerEvent::Started, this,
                &ThisClass::OnAbilityInputPressed, Binding.InputID);

            EIC->BindAction(Binding.InputAction, ETriggerEvent::Completed, this,
                &ThisClass::OnAbilityInputReleased, Binding.InputID);
        }
    }
}

void AMyCharacter::OnAbilityInputPressed(EAbilityInput InputID)
{
    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->AbilityLocalInputPressed(static_cast<int32>(InputID));
    }
}

void AMyCharacter::OnAbilityInputReleased(EAbilityInput InputID)
{
    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->AbilityLocalInputReleased(static_cast<int32>(InputID));
    }
}
```

!!! tip "Granting with InputID from data"
    The example above grants all startup abilities with `INDEX_NONE`. In practice, you would either create a struct that pairs an ability class with an `EAbilityInput` value, or use an [Ability Set](ability-sets.md) data asset that includes the input binding. The pattern is the same -- pass the `InputID` to the `FGameplayAbilitySpec` constructor.

## Approach 2: Tag-Based Routing { #tag-based }

### Overview

Instead of mapping abilities to integer slots, you map them to `FGameplayTag` values like `InputTag.Attack` or `InputTag.Dodge`. A data asset or array maps `UInputAction` assets to tags, and a custom ASC subclass handles the routing.

This pattern scales well to projects where abilities are added and removed dynamically -- weapon swaps, loadouts, talent trees. It is the approach used by Lyra and many larger-scale GAS projects. It requires more initial setup than InputID, but the flexibility pays off as your project grows.

### Step 1: Define Input Tags

Register your input tags in a `.ini` file or through **Project Settings > Gameplay Tags**:

```ini
; DefaultGameplayTags.ini
+GameplayTagList=(Tag="InputTag.PrimaryAttack",DevComment="Primary attack input")
+GameplayTagList=(Tag="InputTag.SecondaryAttack",DevComment="Secondary attack input")
+GameplayTagList=(Tag="InputTag.Dodge",DevComment="Dodge input")
+GameplayTagList=(Tag="InputTag.Jump",DevComment="Jump input")
+GameplayTagList=(Tag="InputTag.Confirm",DevComment="Confirm input")
+GameplayTagList=(Tag="InputTag.Cancel",DevComment="Cancel input")
+GameplayTagList=(Tag="InputTag.Ability1",DevComment="Ability slot 1")
+GameplayTagList=(Tag="InputTag.Ability2",DevComment="Ability slot 2")
```

!!! tip "Tag naming"
    Using the `InputTag` namespace keeps input tags separate from ability tags, state tags, and other tag families. See [Tag Architecture](../patterns/tag-design.md) for naming conventions.

### Step 2: Create an Input Config (Data Asset or Array)

You need a data structure that maps `UInputAction` assets to `FGameplayTag` values. A simple struct array on your character works. For larger projects, a `UDataAsset` subclass is cleaner because designers can swap configs without touching the character.

**The mapping struct:**

```cpp
USTRUCT(BlueprintType)
struct FAbilityInputAction
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    TObjectPtr<UInputAction> InputAction;

    UPROPERTY(EditAnywhere, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};
```

**Option A -- Array on the character:**

```cpp
UPROPERTY(EditDefaultsOnly, Category = "Input")
TArray<FAbilityInputAction> AbilityInputActions;
```

**Option B -- Data asset (scales better):**

```cpp
UCLASS()
class UAbilityInputConfig : public UDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    TArray<FAbilityInputAction> InputActions;

    /** Find the tag for a given input action. Returns an empty tag if not found. */
    FGameplayTag FindTagForAction(const UInputAction* Action) const
    {
        for (const FAbilityInputAction& Entry : InputActions)
        {
            if (Entry.InputAction == Action)
            {
                return Entry.InputTag;
            }
        }
        return FGameplayTag();
    }
};
```

### Step 3: Bind in SetupPlayerInputComponent

Same loop structure as Approach 1 -- iterate your mappings and bind press/release:

```cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    UEnhancedInputComponent* EIC = CastChecked<UEnhancedInputComponent>(PlayerInputComponent);

    for (const FAbilityInputAction& Entry : AbilityInputActions)
    {
        if (Entry.InputAction)
        {
            EIC->BindAction(Entry.InputAction, ETriggerEvent::Started, this,
                &ThisClass::OnAbilityInputPressed, Entry.InputTag);

            EIC->BindAction(Entry.InputAction, ETriggerEvent::Completed, this,
                &ThisClass::OnAbilityInputReleased, Entry.InputTag);
        }
    }
}
```

### Step 4: Route Callbacks to the ASC

This is where tag-based routing diverges from InputID. There is no built-in `AbilityLocalInputPressed` for tags -- you need to iterate the activatable abilities yourself and match on tags.

```cpp
void AMyCharacter::OnAbilityInputPressed(FGameplayTag InputTag)
{
    if (!AbilitySystemComponent) return;

    for (FGameplayAbilitySpec& Spec : AbilitySystemComponent->GetActivatableAbilities())
    {
        if (Spec.GetDynamicSpecSourceTags().HasTagExact(InputTag))
        {
            // Notify the spec that its input was pressed (critical for ability tasks)
            AbilitySystemComponent->AbilitySpecInputPressed(Spec);

            if (!Spec.IsActive())
            {
                AbilitySystemComponent->TryActivateAbility(Spec.Handle);
            }
        }
    }
}

void AMyCharacter::OnAbilityInputReleased(FGameplayTag InputTag)
{
    if (!AbilitySystemComponent) return;

    for (FGameplayAbilitySpec& Spec : AbilitySystemComponent->GetActivatableAbilities())
    {
        if (Spec.GetDynamicSpecSourceTags().HasTagExact(InputTag) && Spec.IsActive())
        {
            AbilitySystemComponent->AbilitySpecInputReleased(Spec);
        }
    }
}
```

!!! note "Array mutation during iteration"
    `TryActivateAbility` can cause abilities to be granted or removed (via activation effects or events), which may mutate the array you are iterating. The engine's `AbilityLocalInputPressed` guards against this with `ABILITYLIST_SCOPE_LOCK()`. For production code, consider moving this logic into a custom ASC subclass and using the same lock, or collecting matching spec handles into a local array before activating.

### Step 5: Grant Abilities with Tags

When granting an ability, add the input tag to the spec's `DynamicSpecSourceTags`. This is how the routing loop in Step 4 finds the right ability:

```cpp
void AMyCharacter::GrantAbilityWithTag(
    TSubclassOf<UGameplayAbility> AbilityClass,
    FGameplayTag InputTag,
    int32 Level)
{
    if (!AbilitySystemComponent || !AbilityClass) return;

    FGameplayAbilitySpec Spec(AbilityClass, Level, INDEX_NONE, this);
    Spec.GetDynamicSpecSourceTags().AddTag(InputTag);

    AbilitySystemComponent->GiveAbility(Spec);
}
```

### The Critical Detail: AbilitySpecInputPressed/Released { #spec-input-pressed }

This is the most common mistake in tag-based routing, and it fails silently.

When you call `AbilityLocalInputPressed(InputID)` (Approach 1), the engine internally calls `AbilitySpecInputPressed` on matching specs. This updates the spec's internal input state, which is what ability tasks like `WaitInputRelease` and `WaitInputPress` check.

With tag-based routing, **you are responsible for calling `AbilitySpecInputPressed` and `AbilitySpecInputReleased` yourself**. If you skip these calls and only call `TryActivateAbility`, the ability will activate, but:

- `WaitInputRelease` will never fire -- it is waiting for the spec's input state to change, and nobody told the spec that input was pressed
- `WaitInputPress` will never fire for the same reason
- `InputReleased` overrides on the ability will never be called
- Confirm/cancel inputs may not work as expected

Look at the code in Step 4 above -- the `AbilitySpecInputPressed(Spec)` call on the press callback and `AbilitySpecInputReleased(Spec)` on the release callback are not optional. Without them, held abilities, channeled abilities, and any ability that uses input tasks will break.

!!! danger "Silent failure"
    This bug is particularly insidious because the ability still activates. Everything looks like it's working until someone adds a `WaitInputRelease` task to an ability and it hangs forever. If your tag-based input was working before and suddenly stops working for a new ability, check that `AbilitySpecInputPressed`/`Released` are being called.

### Complete Example { #tag-routing-complete-example }

Here is the full character implementation for tag-based routing.

**MyCharacter.h:**

```cpp
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "AbilitySystemInterface.h"
#include "GameplayTagContainer.h"
#include "InputAction.h"
#include "MyCharacter.generated.h"

USTRUCT(BlueprintType)
struct FAbilityInputAction
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    TObjectPtr<UInputAction> InputAction;

    UPROPERTY(EditAnywhere, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};

UCLASS()
class AMyCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    AMyCharacter();

    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

protected:
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
    virtual void BeginPlay() override;

    UPROPERTY(VisibleAnywhere, Category = "Abilities")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    /** Maps input actions to gameplay tags. Configure in the editor. */
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    TArray<FAbilityInputAction> AbilityInputActions;

private:
    void OnAbilityInputPressed(FGameplayTag InputTag);
    void OnAbilityInputReleased(FGameplayTag InputTag);
};
```

**MyCharacter.cpp:**

```cpp
#include "MyCharacter.h"
#include "AbilitySystemComponent.h"
#include "EnhancedInputComponent.h"

AMyCharacter::AMyCharacter()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("ASC"));
}

UAbilitySystemComponent* AMyCharacter::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}

void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();

    // Grant abilities here or via an ability set — see the Ability Sets page
}

void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    UEnhancedInputComponent* EIC = CastChecked<UEnhancedInputComponent>(PlayerInputComponent);

    for (const FAbilityInputAction& Entry : AbilityInputActions)
    {
        if (Entry.InputAction)
        {
            EIC->BindAction(Entry.InputAction, ETriggerEvent::Started, this,
                &ThisClass::OnAbilityInputPressed, Entry.InputTag);

            EIC->BindAction(Entry.InputAction, ETriggerEvent::Completed, this,
                &ThisClass::OnAbilityInputReleased, Entry.InputTag);
        }
    }
}

void AMyCharacter::OnAbilityInputPressed(FGameplayTag InputTag)
{
    if (!AbilitySystemComponent) return;

    for (FGameplayAbilitySpec& Spec : AbilitySystemComponent->GetActivatableAbilities())
    {
        if (Spec.GetDynamicSpecSourceTags().HasTagExact(InputTag))
        {
            AbilitySystemComponent->AbilitySpecInputPressed(Spec);

            if (!Spec.IsActive())
            {
                AbilitySystemComponent->TryActivateAbility(Spec.Handle);
            }
        }
    }
}

void AMyCharacter::OnAbilityInputReleased(FGameplayTag InputTag)
{
    if (!AbilitySystemComponent) return;

    for (FGameplayAbilitySpec& Spec : AbilitySystemComponent->GetActivatableAbilities())
    {
        if (Spec.GetDynamicSpecSourceTags().HasTagExact(InputTag) && Spec.IsActive())
        {
            AbilitySystemComponent->AbilitySpecInputReleased(Spec);
        }
    }
}
```

!!! tip "Custom ASC subclass"
    In larger projects, you can move the tag-matching logic into a custom `UAbilitySystemComponent` subclass with `AbilityInputTagPressed(FGameplayTag)` and `AbilityInputTagReleased(FGameplayTag)` methods. This keeps the character class clean and makes the input routing reusable across different character types. Lyra takes this approach.

## Forwarding Input to Active Abilities

When an ability is already active and the player presses or releases its bound input, the ASC forwards those events to the ability instance via virtual functions:

```cpp
virtual void InputPressed(const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo);

virtual void InputReleased(const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo);
```

You can override these in your ability to react to input while the ability is running. The built-in `UGameplayAbility_CharacterJump` uses `InputReleased` to stop the jump when the player releases the button.

Alternatively, use the `WaitInputPress` and `WaitInputRelease` [ability tasks](ability-tasks.md), which wrap this into the task pattern with delegates. These are the more common choice because they integrate cleanly with Blueprint and the ability task flow.

!!! info "How input forwarding works internally"
    When `AbilitySpecInputPressed` is called on a spec, the ASC iterates all active instances of that ability and calls `InputPressed` on each. It also notifies any running `WaitInputPress` tasks. The same pattern applies for `AbilitySpecInputReleased`. This is why calling `AbilitySpecInputPressed`/`Released` is essential in tag-based routing -- without it, this entire forwarding chain never fires.

## Confirm and Cancel Inputs

GAS has built-in support for generic "confirm" and "cancel" inputs. These are used by the [targeting system](targeting/index.md) and tasks like `WaitConfirmCancel`.

On the ASC, you specify which input IDs map to confirm and cancel:

```cpp
AbilitySystemComponent->GenericConfirmInputID = static_cast<int32>(EAbilityInput::Confirm);
AbilitySystemComponent->GenericCancelInputID = static_cast<int32>(EAbilityInput::Cancel);
```

When the player presses the confirm input, the ASC calls `LocalInputConfirm()`, which signals any active tasks that are waiting for confirmation. Cancel works the same way via `LocalInputCancel()`.

!!! note "Confirm/Cancel with tag-based routing"
    The `GenericConfirmInputID` and `GenericCancelInputID` properties are integers, so they use the InputID system regardless of your routing approach. If you are using tag-based routing for everything else, you still need to call `AbilityLocalInputPressed` with the confirm/cancel integer IDs -- or call `LocalInputConfirm()` / `LocalInputCancel()` directly from your input callbacks.

## The bReplicateInputDirectly Flag

`UGameplayAbility` has a `bReplicateInputDirectly` flag:

```cpp
UPROPERTY(EditDefaultsOnly, Category = Input)
bool bReplicateInputDirectly;
```

When `true`, input press/release events are always replicated to the server, regardless of whether the ability is currently predicting. This is useful for abilities that need the server to know about input state even when the ability is not actively predicting -- for example, a held ability where the server needs to know when the player releases.

## Legacy Direct Binding (Avoid) { #legacy-binding }

In older GAS tutorials, you may see code that binds abilities directly to the ASC using the old input system:

```cpp
// OLD APPROACH — do not use for new projects
AbilitySystemComponent->BindAbilityActivationToInputComponent(
    InputComponent,
    FGameplayAbilityInputBinds(
        "Confirm", "Cancel",
        "EMyAbilityInput",
        static_cast<int32>(EMyAbilityInput::Confirm),
        static_cast<int32>(EMyAbilityInput::Cancel)));
```

This approach ties directly into the legacy `UInputComponent` and requires a specific enum setup. It still works but does not integrate with Enhanced Input and is more rigid than either of the approaches above.

## Related Pages

- [Ability Tasks](ability-tasks.md) -- WaitInputPress, WaitInputRelease, and other async tasks that depend on input state
- [Ability Sets](ability-sets.md) -- granting abilities with input bindings as part of a cohesive loadout
- [Lifecycle and Activation](lifecycle-and-activation.md) -- what happens after input triggers TryActivateAbility
- [Targeting](targeting/index.md) -- confirm/cancel inputs for the targeting system
- [Bind Ability to Input](../recipes/bind-ability-to-input.md) -- condensed step-by-step recipe
- [Tag Architecture](../patterns/tag-design.md) -- InputTag namespace conventions and tag design principles
