---
title: Data-Driven Attributes
description: How to initialize and scale GAS attributes from DataTables, CurveTables, and Gameplay Effects — the three approaches to data-driven stats in UE 5.7.
---

# Data-Driven Attributes

You've defined your attributes in C++ — but hardcoding `InitHealth(100.f)` in the constructor doesn't scale. What about per-character-class stats? Per-level scaling? Designer-editable values without recompiling?

This page covers three approaches to populating attributes from data, starting with the recommended approach and progressing to more specialized options. Every production game needs at least one of these — pick the one that matches your needs and grow from there.

---

## Which Approach Should I Use?

| Need | Best Approach |
|:---|:---|
| Simple flat values, no level scaling | DataTable + `FAttributeMetaData` |
| Values that scale with level, designer-editable | Instant GE + `FScalableFloat` + CurveTable |
| Per-class stats with level scaling (Warrior vs Mage) | `FAttributeSetInitter` + CurveTable |
| Quick prototyping | Constructor defaults or DataTable |
| Production game | Instant GE or `FAttributeSetInitter` |

!!! warning "Epic's own verdict on InitStats"
    The engine comment on `UAbilitySystemComponent::InitStats` reads:

    > *"Not well supported, a gameplay effect with curve table references may be a better solution."*

    Lyra uses the Instant GE approach. If you're starting a new project, that's the path of least resistance.

---

## Approach A: Instant GE with Override Modifiers (Recommended)

This is what Lyra and most modern UE5 projects use. You create an Instant Gameplay Effect with Override modifiers that set each attribute's base value. Pair it with a CurveTable for per-level scaling, and you get a fully data-driven, designer-editable, replication-aware initialization system.

### 1. Create the Gameplay Effect

=== "Blueprint"

    1. **Content Browser** > right-click > **Blueprint Class** > search for **Gameplay Effect**
    2. Name it `GE_InitAttributes` (or per-class: `GE_InitAttributes_Warrior`)
    3. Set **Duration Policy** to **Instant**

=== "C++"

    For a programmatic approach, create the GE as a Blueprint asset and reference it:

    ```cpp
    UPROPERTY(EditDefaultsOnly, Category = "Attributes")
    TSubclassOf<UGameplayEffect> InitAttributesEffect;
    ```

### 2. Add Override Modifiers

Add one modifier per attribute you want to initialize:

| Property | Value |
|:---|:---|
| **Attribute** | `MyAttributeSet.Health` |
| **Modifier Op** | **Override** |
| **Modifier Magnitude** > Magnitude Calculation Type | **Scalable Float** |
| **Modifier Magnitude** > Scalable Float Magnitude | Your value (e.g., `100.0`) |

!!! info "Why Override, not Add"
    `Override` (`EGameplayModOp::Override`) **sets** the base value directly. `AddBase` would *add* to the existing value — and since attributes default to zero, it might seem equivalent. But if the init effect is ever re-applied (level-up recalculation, respawn), `Add` would stack on top of the previous value. `Override` is idempotent — it always sets the value to exactly what you specify.

Repeat for each attribute: MaxHealth, Stamina, MaxStamina, AttackPower, Defense, and so on. Yes, it's one modifier per attribute — that's the correct pattern.

### 3. FScalableFloat and Level Scaling

Each modifier's magnitude is an `FScalableFloat`, which evaluates as:

```
FinalValue = Value * CurveTable[RowName].Eval(Level)
```

For **flat initialization** (no level scaling), just set the `Value` field directly. Leave the curve reference empty.

For **per-level scaling**, set `Value` to `1.0` and reference a CurveTable row that maps level to the actual stat value. The multiplication of `1.0 * CurveValue` gives you the curve value directly.

See [Scalable Floats](../reference/scalable-float.md) for the full reference on `FScalableFloat`.

### 4. CurveTable Setup

=== "In the Editor"

    1. **Content Browser** > Add > Miscellaneous > **Curve Table**
    2. Choose **Rich Curve** (supports interpolation) or **Simple Curve** (linear)
    3. Add rows for each attribute you want to scale:
        - `MaxHealth`, `MaxStamina`, `AttackPower`, `Defense`, etc.
    4. Add columns for levels 1 through N
    5. Fill in the values

=== "From CSV"

    Create a CSV file and import it:

    ```csv
    ---,1,5,10,15,20
    MaxHealth,100,200,300,400,500
    MaxStamina,50,80,120,160,200
    AttackPower,10,20,30,40,50
    Defense,5,10,15,20,25
    ```

    Import: **Content Browser** > right-click > **Import** > select the CSV > choose **CurveTable** as the destination type.

### 5. Applying at Startup

=== "Blueprint"

    The simplest Blueprint approach: configure a **Startup Effects** array on your ASC or character class, and apply them in `BeginPlay` or `PossessedBy`.

    Many projects add an array of GE classes to their character base class:

    ```cpp
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attributes")
    TArray<TSubclassOf<UGameplayEffect>> StartupEffects;
    ```

    Then apply them in Blueprint's `BeginPlay` or `PossessedBy` event.

=== "C++"

    ```cpp
    void AMyCharacter::PossessedBy(AController* NewController)
    {
        Super::PossessedBy(NewController);

        if (AbilitySystemComponent)
        {
            AbilitySystemComponent->InitAbilityActorInfo(this, this);

            for (const TSubclassOf<UGameplayEffect>& EffectClass : StartupEffects)
            {
                if (EffectClass)
                {
                    FGameplayEffectContextHandle Context =
                        AbilitySystemComponent->MakeEffectContext();
                    Context.AddSourceObject(this);

                    FGameplayEffectSpecHandle Spec =
                        AbilitySystemComponent->MakeOutgoingSpec(
                            EffectClass, CharacterLevel, Context);

                    if (Spec.IsValid())
                    {
                        AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(
                            *Spec.Data.Get());
                    }
                }
            }
        }
    }
    ```

### 6. Setting the Level

The GE spec has a `Level` field. When you call `MakeOutgoingSpec(EffectClass, Level, Context)`, that second parameter is the level that gets passed to every `FScalableFloat` evaluation inside the effect.

If your CurveTable maps level 1-20 to stat values, and you pass level 10, every ScalableFloat modifier will evaluate its curve at 10.

```cpp
// Level comes from your character's progression system
float CharacterLevel = GetCharacterLevel();

FGameplayEffectSpecHandle Spec =
    AbilitySystemComponent->MakeOutgoingSpec(
        InitAttributesEffect, CharacterLevel, Context);
```

!!! tip "Re-apply on level-up"
    Since we're using Override modifiers, you can safely re-apply the same GE at the new level whenever the character levels up. The Override operation will set the base values to the new level's curve values.

### 7. Why This Is Better Than DataTable

| Concern | DataTable | Instant GE |
|:---|:---|:---|
| Level scaling | No | Yes (via CurveTable) |
| GAS pipeline | Bypassed | Full integration |
| Replication | Manual | Handled by GAS |
| Designer tooling | DataTable editor | Full GE editor + curves |
| Re-initialization | Requires manual code | Re-apply with new level |
| Per-character variation | Separate tables | Separate GEs or separate curves |

---

## Approach B: DataTable + FAttributeMetaData

The simplest way to get attribute values out of C++ and into an editable asset. You create a DataTable with the `FAttributeMetaData` row struct, fill in values, and the engine applies them at startup.

### 1. Create the DataTable

=== "Editor"

    1. **Content Browser** > right-click > **Miscellaneous** > **Data Table**
    2. When prompted for a Row Struct, select **`AttributeMetaData`** (search for it — it's under the GameplayAbilities plugin)
    3. Name it something like `DT_DefaultAttributes`

=== "C++"

    You can also load an existing DataTable asset by soft reference:

    ```cpp
    UPROPERTY(EditDefaultsOnly, Category = "Attributes")
    TSoftObjectPtr<UDataTable> DefaultAttributeTable;
    ```

### 2. Row Naming Convention

Each row in the table corresponds to a single attribute. The row name **must** follow this format:

```
{AttributeSetClassName}.{AttributeName}
```

Where `AttributeSetClassName` is the class name **without the U prefix**. The engine uses `Property->GetOwnerVariant().GetName()` internally, which strips the `U`.

| Attribute Set Class | Attribute | Row Name |
|:---|:---|:---|
| `UMyAttributeSet` | `Health` | `MyAttributeSet.Health` |
| `UMyAttributeSet` | `MaxHealth` | `MyAttributeSet.MaxHealth` |
| `UVitalAttributeSet` | `Stamina` | `VitalAttributeSet.Stamina` |

### 3. FAttributeMetaData Fields

The struct has five fields, but only one actually does anything meaningful:

| Field | Type | Default | Actually Used? |
|:---|:---|:---|:---|
| `BaseValue` | `float` | 0.0 | **Yes** — this is the value that gets applied to the attribute |
| `MinValue` | `float` | 0.0 | Stored but **not enforced** by the engine |
| `MaxValue` | `float` | 1.0 | Stored but **not enforced** by the engine |
| `DerivedAttributeInfo` | `FString` | `""` | **Not used** by the engine |
| `bCanStack` | `bool` | `false` | **Not used** by the engine |

!!! danger "MinValue and MaxValue do NOT clamp your attributes"
    This trips up nearly everyone. The `MinValue` and `MaxValue` fields in `FAttributeMetaData` are metadata only — the engine **never** reads them to enforce clamping. If you need attributes to stay within bounds, implement that in `PreAttributeChange` and `PreAttributeBaseChange`. See [Attributes and Attribute Sets](attributes-and-attribute-sets.md#2-preattributechange) for the callback chain.

### 4. Three Ways to Apply

#### DefaultStartingData on the ASC (Blueprint)

The easiest way — no C++ needed. Configure it directly in the Blueprint details panel of your character or any actor with an ASC:

=== "Blueprint"

    1. Select the **Ability System Component** in the details panel
    2. Find the **Default Starting Data** array (under the **AttributeTest** category)
    3. Add an entry:
        - **Attributes**: your `UAttributeSet` subclass (e.g., `MyAttributeSet`)
        - **Default Starting Table**: your `FAttributeMetaData` DataTable

    This runs automatically during `OnRegister` — no code needed.

=== "C++"

    You can also populate `DefaultStartingData` in your ASC subclass constructor:

    ```cpp
    UMyAbilitySystemComponent::UMyAbilitySystemComponent()
    {
        FAttributeDefaults Defaults;
        Defaults.Attributes = UMyAttributeSet::StaticClass();
        // DataTable loaded via ConstructorHelpers or set in Blueprint
        Defaults.DefaultStartingTable = MyDataTable;
        DefaultStartingData.Add(Defaults);
    }
    ```

#### InitStats on the ASC (C++)

Call this from code when you need explicit control over timing:

```cpp
// Creates the attribute set (if it doesn't exist) and initializes from the table
AbilitySystemComponent->InitStats(UMyAttributeSet::StaticClass(), MyDataTable);
```

This is a convenience wrapper that calls `GetOrCreateAttributeSubobject` and then `InitFromMetaDataTable` internally.

#### InitFromMetaDataTable on the Attribute Set (C++)

If you already have a reference to the attribute set instance, call it directly:

```cpp
UMyAttributeSet* AttributeSet = Cast<UMyAttributeSet>(
    AbilitySystemComponent->GetAttributeSubobject(UMyAttributeSet::StaticClass()));

if (AttributeSet)
{
    AttributeSet->InitFromMetaDataTable(MyDataTable);
}
```

This iterates over every `FGameplayAttributeData` property on the set, looks up a matching row by name, and sets both `BaseValue` and `CurrentValue` to the table's `BaseValue`.

### 5. Limitations

- **Flat values only** — no level scaling, no curves
- **MinValue/MaxValue not enforced** — they're purely informational
- **Epic considers it "not well supported"** — the comment in the source is explicit
- **No GAS pipeline integration** — values are set directly, bypassing effects, callbacks, and replication events
- **No per-character variation** — unless you create a separate DataTable per character class

For prototyping or games with simple, flat stats, this works fine. For anything with level scaling or per-class variation, use Approach A or C.

---

## Approach C: FAttributeSetInitter + CurveTable (Most Powerful)

The global attribute initialization system. This approach initializes ALL attribute sets on an ASC in one call, based on a "group" name (think character class) and a level. Different groups = different stat progressions = different character archetypes, all from a single CurveTable.

### 1. What It Is

`FAttributeSetInitter` is a global system managed by `UAbilitySystemGlobals`. You configure one or more CurveTables in Project Settings. At runtime, you call `InitAttributeSetDefaults` with a group name and level, and it initializes every attribute on the ASC that has a matching row in the table.

### 2. CurveTable Row Naming

Rows follow a three-part dotted format:

```
{GroupName}.{AttributeSetName}.{AttributeName}
```

| Part | Description | Example |
|:---|:---|:---|
| **GroupName** | Arbitrary name for the character class/archetype | `Default`, `Warrior`, `Mage` |
| **AttributeSetName** | Partial match on the `UAttributeSet` class name | `MyAttributeSet`, `VitalSet` |
| **AttributeName** | Exact match on the property name | `MaxHealth`, `AttackPower` |

!!! note "Partial matching on AttributeSetName"
    The engine uses a **partial match** on the class name. If your class is `UMyGameHealthSet`, the row name `Health` would match because `"Health"` is contained within `"MyGameHealthSet"`. This is convenient but can be surprising — be specific enough to avoid ambiguity.

Example CurveTable rows:

```
Default.MyAttributeSet.MaxHealth
Default.MyAttributeSet.MaxStamina
Default.MyAttributeSet.AttackPower
Warrior.MyAttributeSet.MaxHealth
Warrior.MyAttributeSet.AttackPower
Mage.MyAttributeSet.MaxHealth
Mage.MyAttributeSet.MaxMana
```

### 3. CurveTable Column Format

Columns represent levels. Column 1 = level 1, column 2 = level 2, and so on.

```csv
---,1,2,3,4,5,10,15,20
Default.MyAttributeSet.MaxHealth,100,110,120,130,140,200,280,400
Default.MyAttributeSet.AttackPower,10,11,12,13,14,20,30,50
Warrior.MyAttributeSet.MaxHealth,120,135,150,165,180,260,360,500
Warrior.MyAttributeSet.AttackPower,15,17,19,21,23,35,50,75
Mage.MyAttributeSet.MaxHealth,80,88,96,104,112,160,220,320
Mage.MyAttributeSet.MaxMana,100,115,130,145,160,250,360,500
```

### 4. Configuration

Set the CurveTable references in **Project Settings** > **Plugins** > **Gameplay Abilities** (the `UGameplayAbilitiesDeveloperSettings` panel):

- **Global Attribute Set Defaults Tables**: add your CurveTable asset(s) here

!!! warning "Deprecated path"
    Prior to 5.5, this was configured directly on `UAbilitySystemGlobals` via the `GlobalAttributeSetDefaultsTableNames` property. That path is deprecated. Use `UGameplayAbilitiesDeveloperSettings` (Project Settings > Plugins > Gameplay Abilities) instead.

After changing this setting, the editor requires a restart (`ConfigRestartRequired`).

### 5. Runtime Call

```cpp
#include "GameplayAbilitiesModule.h"
#include "AbilitySystemGlobals.h"

void AMyCharacter::InitializeAttributes()
{
    if (AbilitySystemComponent)
    {
        FName GroupName = TEXT("Warrior"); // Or "Mage", "Default", etc.
        int32 Level = GetCharacterLevel();

        IGameplayAbilitiesModule::Get()
            .GetAbilitySystemGlobals()
            ->GetAttributeSetInitter()
            ->InitAttributeSetDefaults(
                AbilitySystemComponent, GroupName, Level, true);
    }
}
```

The last parameter `bInitialInit` should be `true` for the first initialization and `false` for subsequent re-initializations (e.g., level-up).

### 6. Group Fallback

If the specified group doesn't define a row for a particular attribute, the system falls back to the `Default` group. `"Default"` is hardcoded as the fallback group name.

This means you only need to define per-class rows for attributes that actually differ. Shared attributes (like base movement speed) can live in `Default` and every group inherits them.

```
Default.MyAttributeSet.MoveSpeed       → used by ALL groups unless overridden
Warrior.MyAttributeSet.MaxHealth       → Warrior-specific override
Mage.MyAttributeSet.MaxHealth          → Mage-specific override
```

### 7. When to Use This

- **RPGs** with multiple character classes that have fundamentally different stat progressions
- **MOBAs** with hero-specific stat curves (each hero is a "group")
- **Any game** where different characters have different stat growth but share the same attribute sets
- Projects that want a **single source of truth** for all attribute initialization across the game

If your game has one character class or no level scaling, this is overkill — use Approach B instead.

---

## Practical Example: RPG Character with Per-Level Stats

Let's walk through a complete setup using **Approach B** (Instant GE + CurveTable), since it's the recommended approach for most projects.

### The Goal

A character with these stats that scale from level 1 to level 20:

| Attribute | Level 1 | Level 10 | Level 20 |
|:---|:---|:---|:---|
| MaxHealth | 100 | 300 | 500 |
| Health | 100 | 300 | 500 |
| MaxStamina | 50 | 120 | 200 |
| Stamina | 50 | 120 | 200 |
| AttackPower | 10 | 30 | 50 |
| Defense | 5 | 15 | 25 |

### Step 1: CurveTable CSV

Create `CT_CharacterStats.csv`:

```csv
---,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20
MaxHealth,100,121,142,163,184,205,226,247,268,300,321,342,363,384,405,426,447,468,489,500
MaxStamina,50,58,66,74,82,90,98,106,114,120,128,136,144,152,160,168,176,184,192,200
AttackPower,10,12,14,16,18,20,22,24,26,30,32,34,36,38,40,42,44,46,48,50
Defense,5,6,7,8,9,10,11,12,13,15,16,17,18,19,20,21,22,23,24,25
```

Import this into the editor as a **Curve Table** asset.

### Step 2: Gameplay Effect Blueprint

Create `GE_InitStats` (Gameplay Effect Blueprint):

- **Duration Policy**: Instant
- **Modifiers** (6 entries):

| # | Attribute | Operation | Magnitude Type | Value | Curve Row |
|:---|:---|:---|:---|:---|:---|
| 1 | `MyAttributeSet.MaxHealth` | Override | Scalable Float | 1.0 | `CT_CharacterStats.MaxHealth` |
| 2 | `MyAttributeSet.Health` | Override | Scalable Float | 1.0 | `CT_CharacterStats.MaxHealth` |
| 3 | `MyAttributeSet.MaxStamina` | Override | Scalable Float | 1.0 | `CT_CharacterStats.MaxStamina` |
| 4 | `MyAttributeSet.Stamina` | Override | Scalable Float | 1.0 | `CT_CharacterStats.MaxStamina` |
| 5 | `MyAttributeSet.AttackPower` | Override | Scalable Float | 1.0 | `CT_CharacterStats.AttackPower` |
| 6 | `MyAttributeSet.Defense` | Override | Scalable Float | 1.0 | `CT_CharacterStats.Defense` |

Note that `Health` and `Stamina` reference the same curve rows as their `Max` counterparts — we want current values to start at max.

### Step 3: C++ Startup Code

```cpp
// MyCharacter.h
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attributes")
TSubclassOf<UGameplayEffect> InitStatsEffect;

UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Attributes")
float DefaultCharacterLevel = 1.0f;

// MyCharacter.cpp
void AMyCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);

    if (!AbilitySystemComponent || !InitStatsEffect)
    {
        return;
    }

    // IMPORTANT: InitAbilityActorInfo must be called before applying effects
    AbilitySystemComponent->InitAbilityActorInfo(this, this);

    FGameplayEffectContextHandle Context =
        AbilitySystemComponent->MakeEffectContext();
    Context.AddSourceObject(this);

    FGameplayEffectSpecHandle Spec =
        AbilitySystemComponent->MakeOutgoingSpec(
            InitStatsEffect, DefaultCharacterLevel, Context);

    if (Spec.IsValid())
    {
        AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(
            *Spec.Data.Get());
    }
}
```

In your character Blueprint, set `InitStatsEffect` to `GE_InitStats` and `DefaultCharacterLevel` to the starting level.

### Step 4: Level-Up

When the character levels up, re-apply the same effect at the new level:

```cpp
void AMyCharacter::OnLevelUp(int32 NewLevel)
{
    if (!AbilitySystemComponent || !InitStatsEffect)
    {
        return;
    }

    FGameplayEffectContextHandle Context =
        AbilitySystemComponent->MakeEffectContext();
    Context.AddSourceObject(this);

    FGameplayEffectSpecHandle Spec =
        AbilitySystemComponent->MakeOutgoingSpec(
            InitStatsEffect, static_cast<float>(NewLevel), Context);

    if (Spec.IsValid())
    {
        AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(
            *Spec.Data.Get());
    }
}
```

Because every modifier uses Override, this safely overwrites the old base values with the new level's values. Active duration/infinite effects (buffs, debuffs) are unaffected — their modifiers still apply on top of the new base.

---

## Common Mistakes

!!! bug "Using the U prefix in DataTable row names"
    If your attribute set class is `UMyAttributeSet`, the row name must be `MyAttributeSet.Health` — **not** `UMyAttributeSet.Health`. The engine calls `Property->GetOwnerVariant().GetName()`, which returns the class name without the U prefix. If you include it, the row lookup silently fails and your attributes stay at zero.

!!! bug "Expecting MinValue/MaxValue to enforce clamping"
    `FAttributeMetaData::MinValue` and `MaxValue` are stored but **never read by the engine** for enforcement. They're metadata fields that were never fully implemented. To clamp attributes, implement `PreAttributeChange` (for CurrentValue) and `PreAttributeBaseChange` (for BaseValue) in your `UAttributeSet` subclass. See [Attributes and Attribute Sets](attributes-and-attribute-sets.md#2-preattributechange).

!!! bug "Forgetting to set the GE level"
    The second parameter to `MakeOutgoingSpec` is the effect level. If you omit it or leave it at `0`, every ScalableFloat will evaluate its curve at level 0 — which is often 0 or undefined. Always pass the character's actual level.

    ```cpp
    // WRONG — defaults to level 0
    auto Spec = ASC->MakeOutgoingSpec(EffectClass, 0, Context);

    // RIGHT — uses the character's level
    auto Spec = ASC->MakeOutgoingSpec(EffectClass, CharacterLevel, Context);
    ```

!!! bug "Using AddBase instead of Override for initialization"
    `AddBase` adds to the existing value. If the init effect is applied twice (respawn, re-init on level-up), the values stack: 100 + 100 = 200 HP on the second application. `Override` always sets the base to exactly the specified value regardless of current state.

!!! bug "Not calling InitAbilityActorInfo before applying effects"
    If you apply the init GE before `InitAbilityActorInfo`, the ASC doesn't know who its owner and avatar are. Effects can silently fail or produce incorrect results. Always call `InitAbilityActorInfo` first:

    ```cpp
    AbilitySystemComponent->InitAbilityActorInfo(OwnerActor, AvatarActor);
    // NOW apply effects
    ```

---

## Related Pages

- [Attributes and Attribute Sets](attributes-and-attribute-sets.md) -- the core concepts behind `FGameplayAttributeData`, the callback chain, and clamping
- [Scalable Floats](../reference/scalable-float.md) -- full reference on `FScalableFloat`, curve evaluation, and `GetValueAtLevel`
- [Modifiers](../gameplay-effects/modifiers.md) -- deep dive on the Override operation and how modifiers aggregate
- [Project Setup](../getting-started/project-setup.md) -- initial ASC and AttributeSet configuration
- [Starter Attributes](../patterns/starter-attributes.md) -- a complete 50+ attribute schema for action RPG projects
- [AbilitySystemGlobals](../reference/ability-system-globals.md) -- reference for global configuration including `FAttributeSetInitter`
