
# Attributes and Attribute Sets

Attributes are the numeric stats in your game — Health, Mana, AttackPower, Armor, MoveSpeed. In GAS, attributes live inside **Attribute Sets**, and they can only be modified during gameplay through **Gameplay Effects**. This constraint is what makes the entire effect pipeline work: every stat change flows through a consistent, auditable path.

## FGameplayAttributeData

Every attribute is stored as an `FGameplayAttributeData` struct, which holds two values:

```cpp
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health)
FGameplayAttributeData Health;
```

| Value | What It Is | Modified By |
|---|---|---|
| **BaseValue** | The permanent underlying value | **Instant** effects (damage, healing, leveling up) |
| **CurrentValue** | BaseValue + all active modifiers | **Duration/Infinite** effects add modifiers on top |

Think of BaseValue as "your actual stat" and CurrentValue as "your stat right now, with all buffs and debuffs applied."

When a duration effect that adds +50 MaxHealth expires, its modifier is removed and CurrentValue recalculates from BaseValue. The base was never touched — the buff is just *gone*. This is the magic that makes temporary modifiers work without cleanup code.

!!! note "You almost always read CurrentValue"
    When your game logic asks "how much health does this actor have?", you want `GetCurrentValue()`. BaseValue is useful for UI that shows "base vs modified" or for logic that needs the un-buffed number.

## UAttributeSet

Attributes don't float around on their own — they live inside a `UAttributeSet` subclass. The Attribute Set is a UObject that you register with the ASC:

```cpp
UCLASS()
class UMyAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Vital")
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealth, Category = "Vital")
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Mana, Category = "Vital")
    FGameplayAttributeData Mana;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Mana)
};
```

Registration happens in the character's constructor using `CreateDefaultSubobject`:

```cpp
AMyCharacter::AMyCharacter()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("ASC"));
    AttributeSet = CreateDefaultSubobject<UMyAttributeSet>(TEXT("AttributeSet"));
}
```

The ASC automatically discovers Attribute Sets that are subobjects of its owner. You don't need to manually register them.

## The ATTRIBUTE_ACCESSORS Macro

You'll see this macro on every attribute:

```cpp
ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)
```

It generates four helper functions:

| Generated Function | What It Does |
|---|---|
| `static FGameplayAttribute GetHealthAttribute()` | Returns the `FGameplayAttribute` handle — used in effect configurations and queries |
| `float GetHealth() const` | Returns the current value |
| `void SetHealth(float NewVal)` | Sets the base value directly (use sparingly — prefer effects) |
| `void InitHealth(float NewVal)` | Sets both base and current value — use only during initialization |

!!! warning "SetX vs Effects"
    `SetHealth()` sets the BaseValue directly and bypasses the entire effect pipeline — no callbacks, no replication, no prediction. Use it only for initialization or testing. During gameplay, always modify attributes through Gameplay Effects.

## The Attribute Callback Chain

When a Gameplay Effect modifies an attribute, a series of callbacks fire in a specific order. This chain is where you implement clamping, damage processing, death checks, and reactions. Understanding the order is critical.

### The Full Chain

Here's every callback, in the order they fire:

#### 1. PreGameplayEffectExecute

```cpp
bool UMyAttributeSet::PreGameplayEffectExecute(FGameplayEffectModCallbackData& Data)
{
    // Called BEFORE the effect executes
    // Return false to cancel the effect entirely
    return true;
}
```

Use this to reject an effect before it does anything. Example: blocking damage on an invulnerable target.

#### 2. PreAttributeChange

```cpp
void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    // Called BEFORE CurrentValue changes
    // Clamp the incoming value here
    if (Attribute == GetMaxHealthAttribute())
    {
        NewValue = FMath::Max(NewValue, 1.0f);
    }
}
```

!!! warning "CurrentValue only"
    `PreAttributeChange` affects **CurrentValue**, not BaseValue. If you clamp here, the base can still exceed your clamp. This catches modifier changes — if you need to clamp the base, use `PreAttributeBaseChange`.

#### 3. PreAttributeBaseChange / PostAttributeBaseChange

```cpp
void UMyAttributeSet::PreAttributeBaseChange(
    const FGameplayAttribute& Attribute, float& NewValue) const
{
    // Called BEFORE BaseValue changes (Instant effects)
    // Clamp base value here
}

void UMyAttributeSet::PostAttributeBaseChange(
    const FGameplayAttribute& Attribute, float OldValue, float NewValue) const
{
    // Called AFTER BaseValue changes
}
```

These fire specifically when the BaseValue is being modified (typically by Instant effects). Use `PreAttributeBaseChange` for base-level clamping.

#### 4. PostAttributeChange

```cpp
void UMyAttributeSet::PostAttributeChange(
    const FGameplayAttribute& Attribute, float OldValue, float NewValue)
{
    // Called AFTER the value changes (either base or current)
    // Good for reactions that don't need effect context
}
```

#### 5. PostGameplayEffectExecute

```cpp
void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    // Called AFTER the effect has executed
    // This is where your damage pipeline lives

    if (Data.EvaluatedData.Attribute == GetPendingDamageAttribute())
    {
        const float Damage = GetPendingDamage();
        SetPendingDamage(0.0f);  // Reset the meta attribute

        if (Damage > 0.0f)
        {
            const float NewHealth = GetHealth() - Damage;
            SetHealth(FMath::Clamp(NewHealth, 0.0f, GetMaxHealth()));

            if (GetHealth() <= 0.0f)
            {
                // Trigger death
            }
        }
    }
}
```

This is the most important callback for gameplay. It has full context about the effect that caused the change — source actor, target actor, effect spec, tags — everything you need to implement a damage pipeline, trigger death, or fire reactive abilities.

#### 6. OnAttributeAggregatorCreated

```cpp
void UMyAttributeSet::OnAttributeAggregatorCreated(
    const FGameplayAttribute& Attribute, FAggregator* NewAggregator) const
{
    // Called when a modifier aggregator is first created for an attribute
    // Use to set custom aggregator behavior (e.g., clamping rules)
}
```

This is advanced — most projects never override it. It lets you customize how modifiers are aggregated (e.g., "MoveSpeed can never go below 100 regardless of modifiers").

### Callback Summary

| Callback | When | Common Use |
|---|---|---|
| `PreGameplayEffectExecute` | Before effect runs | Reject/cancel effects |
| `PreAttributeChange` | Before CurrentValue changes | Clamp current value |
| `PreAttributeBaseChange` | Before BaseValue changes | Clamp base value |
| `PostAttributeBaseChange` | After BaseValue changes | React to base changes |
| `PostAttributeChange` | After either value changes | General reactions |
| `PostGameplayEffectExecute` | After effect completes | Damage pipeline, death, reactions |
| `OnAttributeAggregatorCreated` | First modifier on attribute | Custom aggregation rules |

## Meta Attributes

Meta attributes are transient "scratch pad" values used to pass data through the effect pipeline. The most common example: `PendingDamage`.

```cpp
// Not replicated — only exists on the server during effect processing
UPROPERTY(BlueprintReadOnly, Category = "Meta")
FGameplayAttributeData PendingDamage;
ATTRIBUTE_ACCESSORS(UMyAttributeSet, PendingDamage)
```

Here's the pattern:

1. A damage effect writes to `PendingDamage` (not directly to `Health`)
2. `PostGameplayEffectExecute` reads `PendingDamage`, applies armor/resistance/shields, then subtracts the final amount from `Health`
3. `PendingDamage` is reset to zero

Why not just write to Health directly? Because meta attributes give you a **processing step** between "damage was dealt" and "health changes." That's where your damage formula, resistances, shields, and death checks live.

!!! info "Meta attributes are not replicated"
    Meta attributes should have no `ReplicatedUsing` specifier. They exist only during server-side effect processing and are always reset to zero afterward. Clients never see them.

## Derived Attributes

An attribute can be automatically derived from other attributes using an **Infinite** Gameplay Effect with a modifier that references another attribute.

For example, to make `HealthRegen` automatically equal `Stamina * 0.1`:

1. Create an Infinite GE
2. Add a modifier targeting `HealthRegen`
3. Set the modifier to use `Attribute Based` magnitude, based on `Stamina` with coefficient 0.1

When Stamina changes, HealthRegen automatically recalculates. No manual wiring needed. See [Magnitude Calculations](../gameplay-effects/magnitude-calculations.md) for the full setup.

## Initializing Attributes

Attributes default to zero. You need to initialize them to meaningful values. There are three common approaches:

### Constructor Defaults

```cpp
UMyAttributeSet::UMyAttributeSet()
{
    InitHealth(100.0f);
    InitMaxHealth(100.0f);
    InitMana(50.0f);
}
```

Simple but inflexible — every instance gets the same values.

### DataTable (InitFromMetaDataTable)

Create a DataTable with row type `FAttributeMetaData` and initialize from it:

```cpp
// Typically called in BeginPlay or after ASC init
AttributeSet->InitFromMetaDataTable(MyDataTable, RowName);
```

This lets you define different stat profiles (warrior vs mage) without code changes. Great for data-driven designs.

### Startup Gameplay Effect

Apply an Instant Gameplay Effect at `BeginPlay` that sets all starting values. This is the most GAS-idiomatic approach — it flows through the full pipeline and is easy to configure per-character via DataAssets.

!!! tip "Recommendation"
    For prototyping, constructor defaults are fine. For production, use a DataTable or startup effect — they're easier to tune and allow per-character variation.

## Multiple Attribute Sets

You're not limited to one Attribute Set. You can split attributes across multiple sets:

```cpp
// In character constructor
VitalAttributes = CreateDefaultSubobject<UVitalAttributeSet>(TEXT("VitalAttributes"));
CombatAttributes = CreateDefaultSubobject<UCombatAttributeSet>(TEXT("CombatAttributes"));
MovementAttributes = CreateDefaultSubobject<UMovementAttributeSet>(TEXT("MovementAttributes"));
```

This is useful when different actor types share *some* attributes but not others. Every actor might have `VitalAttributeSet` (Health, MaxHealth), but only player characters have `CombatAttributeSet` (AttackPower, CritChance).

The ASC discovers all Attribute Sets on its owner — no extra registration needed.

## TickableAttributeSetInterface

If an attribute needs to update every frame (e.g., regeneration that ticks smoothly rather than in discrete effect periods), implement `ITickableAttributeSetInterface`:

```cpp
UCLASS()
class UMyRegenAttributeSet : public UAttributeSet, public ITickableAttributeSetInterface
{
    GENERATED_BODY()

public:
    virtual void Tick(float DeltaTime) override;
    virtual bool ShouldTick() const override;
};
```

Use this sparingly. In most cases, a periodic Gameplay Effect (one that ticks every N seconds) is a better and more GAS-idiomatic approach than per-frame attribute ticking.

## Attribute vs Blueprint Variable

A common question: "Should this stat be a GAS attribute or just a Blueprint variable?"

**Use a GAS Attribute when:**

- Gameplay Effects need to modify it (buffs, debuffs, damage)
- It should participate in the modifier pipeline (base value + modifiers)
- It needs to replicate through GAS
- Other GAS systems need to read it (magnitude calculations, execution calculations)

**Use a Blueprint variable when:**

- It's purely cosmetic (UI display name, icon reference)
- No GAS system ever reads or writes it
- It doesn't need the modifier pipeline (it's always set directly)

**Rule of thumb:** If any GAS system interacts with the value, make it an attribute. If it lives entirely outside of GAS, keep it a variable.

## Related Pages

- [Modifiers](../gameplay-effects/modifiers.md) -- how Gameplay Effects modify attribute values through the modifier pipeline
- [Starter Attributes](../patterns/starter-attributes.md) -- a complete 50+ attribute schema for action RPG projects
- [Add an Attribute](../recipes/add-attribute.md) -- step-by-step recipe for adding a new attribute to your project
