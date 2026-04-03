---
title: "Recipe: Bind Ability to Input"
description: Step-by-step procedure for wiring an Input Action to a Gameplay Ability via InputTag.
---

# Recipe: Bind Ability to Input

**Goal:** Press a button, activate an ability. Using the Enhanced Input + InputTag pattern.

**Prerequisites:** Enhanced Input plugin enabled. An ability to bind. See [Input Binding](../gameplay-abilities/input-binding.md) for the concepts.

---

## Steps

### 1. Create the Input Action (IA)

1. Content Browser > right-click > **Input > Input Action**
2. Name it: `IA_Attack`
3. Set **Value Type** to `bool` (for press/release) or `Axis1D`/`Axis2D` for analog
4. Save

### 2. Add to Input Mapping Context (IMC)

1. Open your IMC (e.g., `IMC_Default`)
2. Click **+** to add a mapping
3. Set the **Input Action** to `IA_Attack`
4. Assign a key (e.g., Left Mouse Button)
5. Add modifiers/triggers as needed (e.g., **Pressed** trigger for single activation)
6. Save

### 3. Create the InputTag

If it doesn't already exist, create the tag:

```ini
# DefaultGameplayTags.ini
+GameplayTagList=(Tag="InputTag.Attack",DevComment="Primary attack input")
```

Or add it in **Project Settings > GameplayTags > Manage Gameplay Tags**.

### 4. Map InputAction to InputTag

This step depends on your input binding implementation. The common approach uses a data asset or array that maps IA to tag:

=== "Data-Driven (Recommended)"

    Create a data asset or DataTable entry:

    ```cpp
    USTRUCT(BlueprintType)
    struct FInputActionTagBinding
    {
        GENERATED_BODY()

        UPROPERTY(EditAnywhere)
        TObjectPtr<UInputAction> InputAction;

        UPROPERTY(EditAnywhere, Meta = (Categories = "InputTag"))
        FGameplayTag InputTag;
    };
    ```

    In your input data:
    ```
    IA_Attack → InputTag.Attack
    IA_Dodge  → InputTag.Dodge
    IA_Jump   → InputTag.Jump
    ```

=== "In Code"

    In your character's input setup:

    ```cpp
    void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
    {
        UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(PlayerInputComponent);

        for (const FInputActionTagBinding& Binding : InputBindings)
        {
            EIC->BindAction(Binding.InputAction, ETriggerEvent::Triggered,
                this, &AMyCharacter::OnAbilityInputPressed, Binding.InputTag);
            EIC->BindAction(Binding.InputAction, ETriggerEvent::Completed,
                this, &AMyCharacter::OnAbilityInputReleased, Binding.InputTag);
        }
    }

    void AMyCharacter::OnAbilityInputPressed(FGameplayTag InputTag)
    {
        ASC->AbilityLocalInputPressed(InputTag);
    }

    void AMyCharacter::OnAbilityInputReleased(FGameplayTag InputTag)
    {
        ASC->AbilityLocalInputReleased(InputTag);
    }
    ```

### 5. Set InputTag on the Ability

Open your ability Blueprint (e.g., `GA_LightAttack`):

1. In Class Defaults, find the **InputTag** property (or your project's equivalent)
2. Set it to `InputTag.Attack`

!!! note "Implementation varies"
    The exact property name depends on your base ability class. Some projects use `AbilityInputAction`, `ActivationTag`, or a custom property. The concept is the same: the ability declares which input tag activates it.

### 6. Grant the Ability

Make sure the ability is granted to the character (see [Add an Ability](add-ability.md)). When granting, the InputTag is used to associate the ability spec with the input:

```cpp
FGameplayAbilitySpec Spec(AbilityClass, Level, INDEX_NONE, this);
// If using the input tag approach, the ability's CDO InputTag
// is read during activation matching
ASC->GiveAbility(Spec);
```

### 7. Test

1. PIE
2. Press the bound key (Left Mouse Button)
3. Verify the ability activates
4. Release the key -- verify the ability handles release if needed (for hold abilities)
5. Check `showdebug abilitysystem` to see the ability in the active/granted list

---

## Checklist

- [ ] Input Action created (IA\_)
- [ ] Input Action added to Input Mapping Context
- [ ] InputTag created (e.g., `InputTag.Attack`)
- [ ] IA mapped to InputTag (data asset or code)
- [ ] Ability's InputTag property set
- [ ] Ability granted to character
- [ ] Tested in PIE

## Related

- [Input Binding](../gameplay-abilities/input-binding.md) -- concepts and architecture
- [Add an Ability](add-ability.md) -- creating and granting abilities
- [Net Execution Policies](../networking/net-execution-policies.md) -- how input interacts with networking
