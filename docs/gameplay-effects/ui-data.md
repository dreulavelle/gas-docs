---
title: Effect UI Data
icon: material/card-text
description: Attaching display information (names, descriptions, icons) to Gameplay Effects for UI representation.
---

# Effect UI Data

Every game with a buff bar, debuff tooltip, or status effect icon needs a way to associate *display information* with Gameplay Effects. What's the icon for "Burning"? What text should the tooltip show for "Speed Boost"? GAS solves this with the **UI Data** system — a component on the GE that carries purely cosmetic information for the UI to read.

UI Data has zero impact on gameplay logic. It's never consulted during effect application, modifier calculation, or tag processing. It exists solely so that your UI code has a clean, standardized place to find display information for a given effect.

## UGameplayEffectUIData

The base class for all UI data. As of UE 5.3, it derives from `UGameplayEffectComponent`, which means it's added to a GE the same way you add any other GE component — through the `GEComponents` array in the editor.

```cpp
UCLASS(Blueprintable, Abstract, EditInlineNew, CollapseCategories, MinimalAPI)
class UGameplayEffectUIData : public UGameplayEffectComponent
{
    GENERATED_BODY()
};
```

Key things to notice:

- **Blueprintable** — you can create Blueprint subclasses without touching C++
- **Abstract** — you can't use `UGameplayEffectUIData` directly; you must use a subclass
- **EditInlineNew** — instances are created inline on the GE asset (they're subobjects, not standalone assets)
- **Derives from UGameplayEffectComponent** — since 5.3, UI data *is* a GE component. Prior to 5.3, it was a standalone `UObject` subobject referenced by a property on the GE.

The base class has no properties of its own. It's a blank slate — subclasses add whatever display data your game needs.

## UGameplayEffectUIData_TextOnly

The engine ships one concrete subclass: `UGameplayEffectUIData_TextOnly`. It's intentionally minimal — just a single `FText` description field:

```cpp
UCLASS(DisplayName="UI Data (Text Only)", MinimalAPI)
class UGameplayEffectUIData_TextOnly : public UGameplayEffectUIData
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Data, meta = (MultiLine = "true"))
    FText Description;
};
```

This is useful for prototyping or games where effects only need a text description. For anything more (icons, display names, rarity, etc.), you'll write a custom subclass.

## Setting Up UI Data in the Editor

1. Open your Gameplay Effect Blueprint
2. In the **Details** panel, find the **GE Components** section (or the **Components** array)
3. Click the **+** button and select **UI Data (Text Only)** — or your custom subclass
4. Fill in the display fields (description text, icon, etc.)

That's it. The UI data is now baked into the GE asset.

!!! info "One UI Data per GE"
    While technically nothing prevents you from adding multiple UI Data components, it rarely makes sense. Your UI code will query for a specific subclass, and having multiple just creates ambiguity. Stick to one per GE.

## Reading UI Data from Blueprint

The `UAbilitySystemBlueprintLibrary` provides a dedicated function for accessing UI data:

```
GetGameplayEffectUIData(EffectClass, DataType) → UGameplayEffectUIData*
```

- `EffectClass` — the GE class (you can get this from an active effect handle or a spec)
- `DataType` — the specific `UGameplayEffectUIData` subclass you expect (e.g., your custom `UMyGameEffectUIData`)

The function searches the GE's components for a UI data component matching the requested type and returns it. If none is found, it returns `nullptr`.

### From an Active Effect Handle

The typical flow for a buff bar widget:

1. Get the active effect handles from the ASC (e.g., iterate `GetActiveGameplayEffects()`)
2. For each handle, call `GetGameplayEffectFromActiveEffectHandle` to get the GE CDO
3. Call `GetGameplayEffectUIData` with the GE's class and your UI data type
4. Cast (or just use — the `DeterminesOutputType` meta means Blueprint auto-casts for you)
5. Read the display properties (icon, name, description)

```cpp
// C++ equivalent
const UGameplayEffect* GE = GetGameplayEffectFromActiveEffectHandle(ActiveHandle);
if (GE)
{
    const UMyGameEffectUIData* UIData = Cast<UMyGameEffectUIData>(
        UAbilitySystemBlueprintLibrary::GetGameplayEffectUIData(
            GE->GetClass(), UMyGameEffectUIData::StaticClass()));
    if (UIData)
    {
        // Use UIData->Icon, UIData->DisplayName, UIData->Description, etc.
    }
}
```

## Writing a Custom UI Data Class

For most shipping games, `UGameplayEffectUIData_TextOnly` won't cut it. Here's how to create a richer UI data class:

```cpp
// MyGameEffectUIData.h
#pragma once

#include "GameplayEffectUIData.h"
#include "MyGameEffectUIData.generated.h"

UCLASS(DisplayName = "UI Data (My Game)", BlueprintType)
class MYGAME_API UMyGameEffectUIData : public UGameplayEffectUIData
{
    GENERATED_BODY()

public:
    /** Display name shown in buff bar and tooltips */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Display")
    FText DisplayName;

    /** Multiline description for tooltips */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Display", meta = (MultiLine = "true"))
    FText Description;

    /** Icon texture for the buff/debuff bar */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Display")
    TSoftObjectPtr<UTexture2D> Icon;

    /** Whether this should display as a buff (positive) or debuff (negative) */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Display")
    bool bIsDebuff = false;

    /** Maximum stacks to display (0 = don't show stack count) */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Display")
    int32 MaxDisplayStacks = 0;
};
```

Once compiled, "UI Data (My Game)" appears in the GE component dropdown. Designers can fill in the fields per-effect, and your UI code can query for `UMyGameEffectUIData` specifically.

!!! tip "Use TSoftObjectPtr for icons"
    Icons are often only needed when the UI is visible. Using `TSoftObjectPtr<UTexture2D>` keeps the texture unloaded until your widget actually requests it, reducing memory usage when many GE classes are loaded but not displayed.

## Practical Pattern: Buff Bar Widget

Here's the typical flow for a buff bar that reads UI data from active effects. This bridges GAS data with your UMG widgets.

```
1. Widget listens for OnActiveGameplayEffectAdded / OnAnyGameplayEffectRemoved on the ASC
2. On add: query UI data from the new effect, create a buff icon widget
3. On remove: find and destroy the corresponding icon widget
4. Each icon widget polls remaining duration via GetActiveGameplayEffectRemainingDuration
5. Display: Icon from UIData->Icon, tooltip from UIData->Description, timer overlay
```

The key insight is that UI Data gives you a **data-driven** path from GE asset to display. Designers can set up icons and descriptions in the GE Blueprint — no C++ changes needed when adding new buff types.

For the complete pattern including duration timers, stack counts, and debuff sorting, see [Connecting GAS to UI](../patterns/gas-to-ui.md).

## The 5.3 Migration

Before UE 5.3, UI data was a separate `UObject` subobject referenced via a `UIData` property on `UGameplayEffect`:

```cpp
// Pre-5.3 (deprecated property)
UPROPERTY(EditDefaultsOnly, Instanced, Category = Display)
UGameplayEffectUIData* UIData;
```

In 5.3+, `UGameplayEffectUIData` was reparented to derive from `UGameplayEffectComponent`. Existing GE assets should migrate automatically — the old `UIData` property is converted into a component in the `GEComponents` array.

If you're upgrading a pre-5.3 project and your UI data stops appearing, check that the component migrated correctly. You may need to re-add the UI data component manually on affected GEs.

## Further Reading

- [Blueprint Function Library](../reference/blueprint-library.md) — `GetGameplayEffectUIData` function reference
- [GE Components](ge-components.md) — the component architecture that UI Data plugs into
- [Connecting GAS to UI](../patterns/gas-to-ui.md) — full UI integration patterns including buff bars, cooldown displays, and attribute-driven widgets
