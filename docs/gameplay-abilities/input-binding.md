---
title: Input Binding
description: How to connect player input to Gameplay Abilities — Enhanced Input integration, InputID, tag-based routing, and the legacy binding approach.
---

# Input Binding

Connecting player input to ability activation is one of the first things you will need to do in any GAS project. There are several approaches, ranging from the simple built-in `InputID` system to fully custom Enhanced Input integration with tag-based routing. This page covers all of them.

## The Landscape

There are broadly three approaches to ability input binding:

1. **Tag-based routing with Enhanced Input** — the modern, recommended approach
2. **InputID on FGameplayAbilitySpec** — the built-in integer-based system
3. **Legacy direct binding** — the old ASC-level input binding (avoid for new projects)

Most projects end up using approach #1, often combined with #2 for simple cases. Let's look at each.

## Enhanced Input Integration (Recommended) { #enhanced-input }

Enhanced Input is Unreal's current input system, and the recommended way to handle input in UE 5.x. GAS does not ship with a turnkey Enhanced Input integration — you build the bridge yourself. The good news is that it is straightforward.

### The Pattern: Input Action to Gameplay Tag Map

The most common pattern is a data-driven map from `UInputAction` to `FGameplayTag`, stored on your character or a data asset. When an input action fires, you look up the corresponding tag and activate any ability that matches.

```cpp
USTRUCT(BlueprintType)
struct FAbilityInputMapping
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    TObjectPtr<UInputAction> InputAction;

    UPROPERTY(EditAnywhere, meta = (Categories = "Input"))
    FGameplayTag InputTag;
};
```

Your character (or a component) holds an array of these mappings and sets up Enhanced Input bindings in `SetupPlayerInputComponent`:

```cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    UEnhancedInputComponent* EIC = CastChecked<UEnhancedInputComponent>(PlayerInputComponent);

    for (const FAbilityInputMapping& Mapping : AbilityInputMappings)
    {
        EIC->BindAction(Mapping.InputAction, ETriggerEvent::Started, this,
            &ThisClass::OnAbilityInputPressed, Mapping.InputTag);

        EIC->BindAction(Mapping.InputAction, ETriggerEvent::Completed, this,
            &ThisClass::OnAbilityInputReleased, Mapping.InputTag);
    }
}
```

The callbacks then route to the ASC:

```cpp
void AMyCharacter::OnAbilityInputPressed(FGameplayTag InputTag)
{
    if (!AbilitySystemComponent) return;

    for (FGameplayAbilitySpec& Spec : AbilitySystemComponent->GetActivatableAbilities())
    {
        if (Spec.GetDynamicSpecSourceTags().HasTagExact(InputTag))
        {
            AbilitySystemComponent->TryActivateAbility(Spec.Handle);
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

When granting abilities, add the input tag to the spec's dynamic source tags:

```cpp
FGameplayAbilitySpec Spec(AbilityClass, Level, INDEX_NONE, this);
Spec.GetDynamicSpecSourceTags().AddTag(InputTag);
AbilitySystemComponent->GiveAbility(Spec);
```

!!! tip "Why tags instead of integers?"
    Tags are self-documenting, hierarchical, and designer-friendly. `Input.Ability.Primary` is much clearer than `InputID = 3`. Tags also compose well with the rest of GAS — you can query them, filter by them, and use them in tag queries.

### Handling Press, Release, and Held

Enhanced Input gives you fine-grained trigger events:

| Trigger Event | Use For |
|:---|:---|
| `Started` | Ability activation on press |
| `Completed` | Notify active abilities that input was released |
| `Triggered` | Continuous/held input (e.g., channeled abilities) |
| `Ongoing` | Input is held but hasn't triggered yet |

For a basic "press to activate, release to end" pattern:

- Bind `Started` to `TryActivateAbility`
- Bind `Completed` to `AbilitySpecInputReleased`
- In your ability, use `WaitInputRelease` task to know when the player lets go

For held/channeled abilities, you can use the `Triggered` event to know the input is still held, or simply rely on the `WaitInputRelease` task inside the ability.

## InputID on FGameplayAbilitySpec { #input-id }

The simplest built-in approach. Every `FGameplayAbilitySpec` has an `int32 InputID` field. When you grant an ability, you can assign an integer input ID:

```cpp
FGameplayAbilitySpec Spec(AbilityClass, 1, static_cast<int32>(EMyAbilityInput::Ability1));
AbilitySystemComponent->GiveAbility(Spec);
```

The engine's `UGameplayAbilitySet` uses this approach — its `FGameplayAbilityBindInfo` pairs an `EGameplayAbilityInputBinds` enum value with an ability class.

To activate by input ID, you need to wire your input system to call:

```cpp
AbilitySystemComponent->AbilityLocalInputPressed(InputID);
AbilitySystemComponent->AbilityLocalInputReleased(InputID);
```

The ASC then finds specs matching that `InputID` and attempts activation.

!!! note "InputID defaults"
    If you do not set an `InputID`, it defaults to `INDEX_NONE` (-1), which means the ability is not bound to any input. This is fine for abilities activated by events, tags, or direct code calls.

### Confirm and Cancel Inputs

GAS has built-in support for "confirm" and "cancel" generic inputs. These are used by the [targeting system](targeting/index.md) and tasks like `WaitConfirmCancel`.

On the ASC, you can specify which input IDs map to confirm/cancel:

```cpp
AbilitySystemComponent->GenericConfirmInputID = static_cast<int32>(EMyInput::Confirm);
AbilitySystemComponent->GenericCancelInputID = static_cast<int32>(EMyInput::Cancel);
```

When the player presses the confirm input, the ASC calls `LocalInputConfirm()`, which signals any active tasks that are waiting for confirmation.

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

This approach ties directly into the legacy `UInputComponent` and requires a specific enum setup. It still works but does not integrate well with Enhanced Input and is more rigid than the tag-based approach.

## Forwarding Input to Active Abilities

When an ability is already active and the player presses or releases its bound input, the ASC forwards those events to the ability instance via:

```cpp
virtual void InputPressed(const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo);

virtual void InputReleased(const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo);
```

You can override these in your ability to react to input while the ability is running. The built-in `UGameplayAbility_CharacterJump` uses `InputReleased` to stop the jump when the player releases the button.

Alternatively, use the `WaitInputPress` and `WaitInputRelease` [ability tasks](ability-tasks.md), which wrap this into the task pattern with delegates.

## The bReplicateInputDirectly Flag

`UGameplayAbility` has a `bReplicateInputDirectly` flag:

```cpp
UPROPERTY(EditDefaultsOnly, Category = Input)
bool bReplicateInputDirectly;
```

When `true`, input press/release events are always replicated to the server, regardless of whether the ability is currently predicting. This is useful for abilities that need the server to know about input state even when the ability is not actively predicting (e.g., a held ability where the server needs to know when the player releases).

## Putting It Together

Here is a practical setup combining Enhanced Input with tag-based routing:

1. **Define input tags** — `Input.Ability.Primary`, `Input.Ability.Secondary`, `Input.Ability.Dodge`, etc.
2. **Create input action assets** — one per bindable action in Enhanced Input
3. **Map actions to tags** — via a data asset or character property array
4. **Grant abilities with tags** — add input tags to specs when granting
5. **Route input to ASC** — Enhanced Input callbacks look up tags and activate matching abilities
6. **Handle release** — forward release events so active abilities and tasks can respond

This approach is flexible, data-driven, and cleanly separates input configuration from ability logic.
