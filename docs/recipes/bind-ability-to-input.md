---
title: "Recipe: Bind Ability to Input"
description: Quick-reference checklist for wiring an Enhanced Input Action to a Gameplay Ability.
---

# Recipe: Bind Ability to Input

**Goal:** Press a button, activate an ability.

**Prerequisites:** Enhanced Input plugin enabled. A character with an ASC. An ability to bind. See [Input Binding](../gameplay-abilities/input-binding.md) for the full architecture and complete code examples.

---

## Quick Version

There are two approaches to input binding in GAS. Pick the one that fits your project -- both are covered in detail on the [Input Binding](../gameplay-abilities/input-binding.md) page.

| | **InputID** | **Tag-Based** |
|:---|:---|:---|
| Complexity | Lower | Higher |
| Scalability | Fixed enum slots | Unlimited tags |
| Ability task support | Automatic | Manual (`AbilitySpecInputPressed/Released`) |
| Works well for | Fixed input layouts | Dynamic ability sets, loadouts |

---

## Steps (InputID Approach)

### 1. Create the Input Action

1. Content Browser > right-click > **Input > Input Action**
2. Name it: `IA_Attack`
3. Set **Value Type** to `Digital (Bool)`

### 2. Add to Input Mapping Context

1. Open your IMC (e.g., `IMC_Default`)
2. Add a mapping: **Input Action** = `IA_Attack`, **Key** = Left Mouse Button

### 3. Define an Input Enum (C++)

If you haven't already, create an `EAbilityInput` enum with entries for each bindable slot (PrimaryAttack, Dodge, Jump, etc.). See [Input Binding -- Step 1](../gameplay-abilities/input-binding.md#input-id) for the full enum.

### 4. Map InputAction to InputID

Add an entry to your character's `AbilityInputBindings` array (in Class Defaults or C++):

| Input Action | InputID |
|:---|:---|
| `IA_Attack` | PrimaryAttack |

### 5. Wire SetupPlayerInputComponent (C++)

Your character class needs a `SetupPlayerInputComponent` override that loops through the bindings and calls `AbilityLocalInputPressed` / `AbilityLocalInputReleased` on the ASC. See [Input Binding -- Steps 3-4](../gameplay-abilities/input-binding.md#input-id) for the complete code.

### 6. Grant the Ability with InputID

When granting the ability, pass the `InputID` to the `FGameplayAbilitySpec` constructor:

```cpp
FGameplayAbilitySpec Spec(AbilityClass, 1, static_cast<int32>(EAbilityInput::PrimaryAttack), this);
AbilitySystemComponent->GiveAbility(Spec);
```

### 7. Test

1. PIE
2. Press the bound key
3. Verify the ability activates (`showdebug abilitysystem`)
4. Release the key -- verify held abilities respond to release
5. Check Stamina/cooldown if the ability has cost/cooldown effects

---

## Checklist

- [ ] Input Action created (`IA_`)
- [ ] Input Action added to Input Mapping Context with a key
- [ ] Input enum defined with an entry for this ability
- [ ] InputAction mapped to InputID (character array or data asset)
- [ ] `SetupPlayerInputComponent` wires the binding loop (C++)
- [ ] Ability granted with the correct InputID
- [ ] Tested in PIE

---

## Using Tag-Based Routing Instead?

Follow the same first two steps (create IA, add to IMC), then:

- Create an `InputTag` (e.g., `InputTag.Attack`)
- Map the IA to the tag in your character's input config
- Grant the ability with the tag added to `DynamicSpecSourceTags`
- **Call `AbilitySpecInputPressed`/`Released`** in your routing callbacks -- without this, `WaitInputRelease` breaks silently

See [Input Binding -- Tag-Based Routing](../gameplay-abilities/input-binding.md#tag-based) for the complete walkthrough and [The Critical Detail](../gameplay-abilities/input-binding.md#spec-input-pressed) for why the `AbilitySpecInputPressed` call matters.

## Related

- [Input Binding](../gameplay-abilities/input-binding.md) -- full architecture, both approaches, complete code
- [Add an Ability](add-ability.md) -- creating and granting abilities
- [Net Execution Policies](../networking/net-execution-policies.md) -- how input interacts with networking
