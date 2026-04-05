
# Project Setup

This page walks through everything you need to enable GAS in a new or existing Unreal Engine 5.7 project. By the end, you'll have a compiling project with an Ability System Component, an Attribute Set, a base character class, and a base ability class — the full C++ foundation that all your Blueprint work will build on.

It's more code than the other Getting Started pages, but you only write it once. After this, your day-to-day work is almost entirely Blueprint and editor configuration.

!!! tip "Use your project name"
    Throughout this page, we use `YourProject` as a placeholder. Replace it with your actual project name (e.g., `Crucible`, `Mythos`, whatever you named it). Your module name is whatever appears in `Source/YourProject/`.

## 1. Module Dependencies (Build.cs)

GAS lives in several modules that aren't included by default. Open your project's `Build.cs` file and add them.

**File:** `Source/YourProject/YourProject.Build.cs`

```cs
using UnrealBuildTool;

public class YourProject : ModuleRules
{
    public YourProject(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicDependencyModuleNames.AddRange(new string[]
        {
            "Core",
            "CoreUObject",
            "Engine",
            "InputCore",
            "EnhancedInput",
            "GameplayAbilities",
            "GameplayTags",
            "GameplayTasks"
        });
    }
}
```

| Module | Why You Need It |
|---|---|
| `GameplayAbilities` | The core GAS module — abilities, effects, attribute sets, ASC |
| `GameplayTags` | Hierarchical tags used for state, queries, and communication |
| `GameplayTasks` | Ability Tasks (play montage and wait, wait for event, etc.) |
| `EnhancedInput` | UE 5.x input system — you'll bind abilities to input actions through this |

!!! warning "Don't forget GameplayTasks"
    This is a common oversight. If you skip `GameplayTasks`, everything compiles fine until you try to use any Ability Task node in Blueprint — then you get mysterious linker errors. Add it now.

## 2. Enable the Plugin (.uproject)

GAS ships as a plugin that needs to be explicitly enabled. Open your `.uproject` file and add it to the `Plugins` array.

**File:** `YourProject.uproject`

```json
{
    "Plugins": [
        {
            "Name": "GameplayAbilities",
            "Enabled": true
        }
    ]
}
```

If you already have a `Plugins` array with other entries, just add the `GameplayAbilities` entry to the existing array. After editing this file, regenerate your project files (right-click the `.uproject` file and select **Generate Visual Studio project files**, or run `GenerateProjectFiles.bat`).

## 3. AbilitySystemGlobals

This is the step most tutorials skip, and it causes the most mysterious bugs later.

`UAbilitySystemGlobals` is a singleton that GAS initializes once at startup. It holds global configuration — things like what class to use for `GameplayEffectContext`, how to initialize target data, and various global callbacks. If you don't call `InitGlobalData()`, some GAS features silently fail to initialize.

### Why Subclass It?

You'll eventually want to customize your `GameplayEffectContext` to carry extra data through the damage pipeline (hit result info, damage type tags, etc.). The way to do that is by overriding `AllocGameplayEffectContext()` in your own subclass. Setting this up now — even if you don't customize anything yet — saves you from a painful refactor later.

### Create the Subclass

=== "Header (.h)"
    ```cpp
    // Source/YourProject/Public/YourProjectAbilitySystemGlobals.h
    #pragma once

    #include "CoreMinimal.h"
    #include "AbilitySystemGlobals.h"
    #include "YourProjectAbilitySystemGlobals.generated.h"

    UCLASS()
    class YOURPROJECT_API UYourProjectAbilitySystemGlobals : public UAbilitySystemGlobals
    {
        GENERATED_BODY()

    public:
        /** Called once during engine init. Override for custom global setup. */
        virtual void InitGlobalData() override;
    };
    ```

=== "Source (.cpp)"
    ```cpp
    // Source/YourProject/Private/YourProjectAbilitySystemGlobals.cpp
    #include "YourProjectAbilitySystemGlobals.h"

    void UYourProjectAbilitySystemGlobals::InitGlobalData()
    {
        Super::InitGlobalData();

        // Future: custom GameplayEffectContext allocation,
        // global tag registration, etc.
    }
    ```

### Point GAS at Your Subclass

**File:** `Config/DefaultGame.ini`

```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
AbilitySystemGlobalsClassName="/Script/YourProject.YourProjectAbilitySystemGlobals"
```

!!! danger "Replace `YourProject` with your actual module name"
    The string after `/Script/` must exactly match your C++ module name (the one in your `Build.cs`). If it doesn't match, GAS silently falls back to the default `UAbilitySystemGlobals` and your overrides do nothing. No error, no warning — it just doesn't work.

### Call InitGlobalData

In UE 5.3+, `InitGlobalData()` is called automatically by the engine during startup via `UAbilitySystemGlobals::Get().InitGlobalData()`. In earlier versions, you had to call it manually from an AssetManager subclass.

!!! note "UE 5.3+ auto-initialization"
    If you're on UE 5.3 or later (which includes 5.7), you do **not** need to manually call `InitGlobalData()` — the engine handles it. If you're following a tutorial written for 4.x or early 5.x that tells you to create a custom AssetManager just for this call, you can skip that step.

## 4. The Attribute Set

Attributes are your character's numeric stats — Health, MaxHealth, Stamina, Damage, Armor, whatever your game needs. They must be declared in C++ as `FGameplayAttributeData` properties inside an `UAttributeSet` subclass.

We'll start with a minimal set: Health, MaxHealth, Stamina, MaxStamina, and a meta attribute called PendingDamage. The meta attribute is a temporary holding value used in the damage pipeline — it receives raw damage, gets processed (armor, resistances, shields), and the final result is subtracted from Health. It resets to zero after every application.

=== "Header (.h)"
    ```cpp
    // Source/YourProject/Public/YourProjectAttributeSet.h
    #pragma once

    #include "CoreMinimal.h"
    #include "AttributeSet.h"
    #include "AbilitySystemComponent.h"
    #include "YourProjectAttributeSet.generated.h"

    // This macro generates getter, setter, and init functions for each attribute.
    // Without it, you'd write 4 boilerplate functions per attribute by hand.
    #define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
        GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
        GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
        GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
        GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

    UCLASS()
    class YOURPROJECT_API UYourProjectAttributeSet : public UAttributeSet
    {
        GENERATED_BODY()

    public:
        UYourProjectAttributeSet();

        // --- Vital Attributes ---

        UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Vital")
        FGameplayAttributeData Health;
        ATTRIBUTE_ACCESSORS(UYourProjectAttributeSet, Health)

        UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealth, Category = "Vital")
        FGameplayAttributeData MaxHealth;
        ATTRIBUTE_ACCESSORS(UYourProjectAttributeSet, MaxHealth)

        UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Stamina, Category = "Vital")
        FGameplayAttributeData Stamina;
        ATTRIBUTE_ACCESSORS(UYourProjectAttributeSet, Stamina)

        UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxStamina, Category = "Vital")
        FGameplayAttributeData MaxStamina;
        ATTRIBUTE_ACCESSORS(UYourProjectAttributeSet, MaxStamina)

        // --- Meta Attributes (not replicated) ---

        /** Transient damage value used in the damage pipeline. Not replicated. */
        UPROPERTY(BlueprintReadOnly, Category = "Meta")
        FGameplayAttributeData PendingDamage;
        ATTRIBUTE_ACCESSORS(UYourProjectAttributeSet, PendingDamage)

        // --- Overrides ---

        virtual void GetLifetimeReplicatedProps(
            TArray<FLifetimeProperty>& OutLifetimeProps) const override;

        /**
         * Called before an attribute's value is changed.
         * Use this for clamping — the value hasn't been committed yet.
         */
        virtual void PreAttributeChange(
            const FGameplayAttribute& Attribute, float& NewValue) override;

        /**
         * Called after a Gameplay Effect executes.
         * This is your damage pipeline — process PendingDamage here.
         */
        virtual void PostGameplayEffectExecute(
            const FGameplayEffectModCallbackData& Data) override;

    protected:
        UFUNCTION()
        void OnRep_Health(const FGameplayAttributeData& OldHealth);

        UFUNCTION()
        void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);

        UFUNCTION()
        void OnRep_Stamina(const FGameplayAttributeData& OldStamina);

        UFUNCTION()
        void OnRep_MaxStamina(const FGameplayAttributeData& OldMaxStamina);
    };
    ```

=== "Source (.cpp)"
    ```cpp
    // Source/YourProject/Private/YourProjectAttributeSet.cpp
    #include "YourProjectAttributeSet.h"
    #include "Net/UnrealNetwork.h"
    #include "GameplayEffectExtension.h"

    UYourProjectAttributeSet::UYourProjectAttributeSet()
    {
        InitHealth(100.f);
        InitMaxHealth(100.f);
        InitStamina(100.f);
        InitMaxStamina(100.f);
        InitPendingDamage(0.f);
    }

    void UYourProjectAttributeSet::GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);

        DOREPLIFETIME_CONDITION_NOTIFY(
            UYourProjectAttributeSet, Health, COND_None, REPNOTIFY_Always);
        DOREPLIFETIME_CONDITION_NOTIFY(
            UYourProjectAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
        DOREPLIFETIME_CONDITION_NOTIFY(
            UYourProjectAttributeSet, Stamina, COND_None, REPNOTIFY_Always);
        DOREPLIFETIME_CONDITION_NOTIFY(
            UYourProjectAttributeSet, MaxStamina, COND_None, REPNOTIFY_Always);
    }

    void UYourProjectAttributeSet::PreAttributeChange(
        const FGameplayAttribute& Attribute, float& NewValue)
    {
        Super::PreAttributeChange(Attribute, NewValue);

        // Clamp current values to their maximums.
        // NOTE: This clamps the *modifier query* result, not the base value.
        // Always re-clamp in PostGameplayEffectExecute too.
        if (Attribute == GetHealthAttribute())
        {
            NewValue = FMath::Clamp(NewValue, 0.f, GetMaxHealth());
        }
        else if (Attribute == GetStaminaAttribute())
        {
            NewValue = FMath::Clamp(NewValue, 0.f, GetMaxStamina());
        }
    }

    void UYourProjectAttributeSet::PostGameplayEffectExecute(
        const FGameplayEffectModCallbackData& Data)
    {
        Super::PostGameplayEffectExecute(Data);

        if (Data.EvaluatedData.Attribute == GetPendingDamageAttribute())
        {
            // PendingDamage is a meta attribute — grab the value, then zero it out.
            const float DamageAmount = GetPendingDamage();
            SetPendingDamage(0.f);

            if (DamageAmount > 0.f)
            {
                // Apply damage to Health.
                // In a real project, you'd check armor, shields, etc. here.
                const float NewHealth = FMath::Clamp(
                    GetHealth() - DamageAmount, 0.f, GetMaxHealth());
                SetHealth(NewHealth);

                // Extension point: check for death (NewHealth <= 0) and broadcast a gameplay event or tag.
            }
        }

        // Final safety clamp — always keep Health and Stamina within bounds.
        if (Data.EvaluatedData.Attribute == GetHealthAttribute())
        {
            SetHealth(FMath::Clamp(GetHealth(), 0.f, GetMaxHealth()));
        }
        else if (Data.EvaluatedData.Attribute == GetStaminaAttribute())
        {
            SetStamina(FMath::Clamp(GetStamina(), 0.f, GetMaxStamina()));
        }
    }

    void UYourProjectAttributeSet::OnRep_Health(
        const FGameplayAttributeData& OldHealth)
    {
        GAMEPLAYATTRIBUTE_REPNOTIFY(
            UYourProjectAttributeSet, Health, OldHealth);
    }

    void UYourProjectAttributeSet::OnRep_MaxHealth(
        const FGameplayAttributeData& OldMaxHealth)
    {
        GAMEPLAYATTRIBUTE_REPNOTIFY(
            UYourProjectAttributeSet, MaxHealth, OldMaxHealth);
    }

    void UYourProjectAttributeSet::OnRep_Stamina(
        const FGameplayAttributeData& OldStamina)
    {
        GAMEPLAYATTRIBUTE_REPNOTIFY(
            UYourProjectAttributeSet, Stamina, OldStamina);
    }

    void UYourProjectAttributeSet::OnRep_MaxStamina(
        const FGameplayAttributeData& OldMaxStamina)
    {
        GAMEPLAYATTRIBUTE_REPNOTIFY(
            UYourProjectAttributeSet, MaxStamina, OldMaxStamina);
    }
    ```

### Why Two Clamp Locations?

You'll notice we clamp in both `PreAttributeChange` and `PostGameplayEffectExecute`. This isn't redundant — they serve different purposes:

- **PreAttributeChange** clamps the result of modifier queries (what `GetHealth()` returns via CurrentValue). It catches overflow from stacking modifiers.
- **PostGameplayEffectExecute** clamps after an instant effect modifies the BaseValue. This is where damage and healing actually land.

If you only clamp in one place, certain edge cases slip through. Clamp in both.

!!! info "Meta attributes don't replicate"
    Notice that `PendingDamage` has no `ReplicatedUsing` and no entry in `GetLifetimeReplicatedProps`. Meta attributes are server-only transient values — they exist to hold intermediate calculations during `PostGameplayEffectExecute` and are zeroed out immediately after. They never need to reach clients.

## 5. The Character Base Class

Your character needs an Ability System Component (the hub from [The Mental Model](mental-model.md)) and a reference to its Attribute Set. It also needs to implement `IAbilitySystemInterface` so that GAS can find the ASC from any actor.

=== "Header (.h)"
    ```cpp
    // Source/YourProject/Public/YourProjectCharacterBase.h
    #pragma once

    #include "CoreMinimal.h"
    #include "GameFramework/Character.h"
    #include "AbilitySystemInterface.h"
    #include "GameplayAbilitySpec.h"
    #include "YourProjectCharacterBase.generated.h"

    class UAbilitySystemComponent;
    class UYourProjectAttributeSet;
    class UGameplayAbility;
    class UGameplayEffect;

    UCLASS()
    class YOURPROJECT_API AYourProjectCharacterBase : public ACharacter,
                                                      public IAbilitySystemInterface
    {
        GENERATED_BODY()

    public:
        AYourProjectCharacterBase();

        // --- IAbilitySystemInterface ---

        virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

        // --- Startup Configuration ---

        /** Abilities granted to this character when it spawns. */
        UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "GAS|Startup")
        TArray<TSubclassOf<UGameplayAbility>> StartupAbilities;

        /** Effects applied to this character when it spawns (e.g., base stat initialization). */
        UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "GAS|Startup")
        TArray<TSubclassOf<UGameplayEffect>> StartupEffects;

    protected:
        UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "GAS")
        TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

        UPROPERTY()
        TObjectPtr<UYourProjectAttributeSet> AttributeSet;

        virtual void PossessedBy(AController* NewController) override;
        virtual void OnRep_PlayerState() override;

        /** Grants startup abilities and applies startup effects. Call once after ASC is ready. */
        void InitializeAbilities();

    private:
        bool bAbilitiesInitialized = false;
    };
    ```

=== "Source (.cpp)"
    ```cpp
    // Source/YourProject/Private/YourProjectCharacterBase.cpp
    #include "YourProjectCharacterBase.h"
    #include "AbilitySystemComponent.h"
    #include "YourProjectAttributeSet.h"

    AYourProjectCharacterBase::AYourProjectCharacterBase()
    {
        // Create the ASC and Attribute Set as default subobjects.
        // They'll appear in the Blueprint component list automatically.
        AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(
            TEXT("AbilitySystemComponent"));
        AbilitySystemComponent->SetIsReplicated(true);
        AbilitySystemComponent->SetReplicationMode(
            EGameplayEffectReplicationMode::Mixed);

        AttributeSet = CreateDefaultSubobject<UYourProjectAttributeSet>(
            TEXT("AttributeSet"));
    }

    UAbilitySystemComponent* AYourProjectCharacterBase::GetAbilitySystemComponent() const
    {
        return AbilitySystemComponent;
    }

    void AYourProjectCharacterBase::PossessedBy(AController* NewController)
    {
        Super::PossessedBy(NewController);

        // Server: initialize the ASC's actor info so it knows who owns it.
        if (AbilitySystemComponent)
        {
            AbilitySystemComponent->InitAbilityActorInfo(this, this);
            InitializeAbilities();
        }
    }

    void AYourProjectCharacterBase::OnRep_PlayerState()
    {
        Super::OnRep_PlayerState();

        // Client: re-initialize actor info when the PlayerState replicates.
        if (AbilitySystemComponent)
        {
            AbilitySystemComponent->InitAbilityActorInfo(this, this);
        }
    }

    void AYourProjectCharacterBase::InitializeAbilities()
    {
        // Guard against double-initialization.
        if (bAbilitiesInitialized || !AbilitySystemComponent)
        {
            return;
        }

        // Only the server (or standalone) should grant abilities.
        if (!HasAuthority())
        {
            return;
        }

        // Grant startup abilities.
        for (const TSubclassOf<UGameplayAbility>& AbilityClass : StartupAbilities)
        {
            if (AbilityClass)
            {
                FGameplayAbilitySpec Spec(AbilityClass, 1,
                    INDEX_NONE, this);
                AbilitySystemComponent->GiveAbility(Spec);
            }
        }

        // Apply startup effects (e.g., base stat initialization).
        for (const TSubclassOf<UGameplayEffect>& EffectClass : StartupEffects)
        {
            if (EffectClass)
            {
                FGameplayEffectContextHandle Context =
                    AbilitySystemComponent->MakeEffectContext();
                Context.AddSourceObject(this);

                FGameplayEffectSpecHandle Spec =
                    AbilitySystemComponent->MakeOutgoingSpec(
                        EffectClass, 1.f, Context);

                if (Spec.IsValid())
                {
                    AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(
                        *Spec.Data.Get());
                }
            }
        }

        bAbilitiesInitialized = true;
    }
    ```

### Key Decisions Explained

**Why `InitAbilityActorInfo` in two places?** The ASC needs to know two things: the *owner* (who owns this ASC for replication purposes) and the *avatar* (the physical actor in the world). `InitAbilityActorInfo` sets both. On the server, `PossessedBy` is the earliest reliable point. On clients, `OnRep_PlayerState` is when the controller<->pawn relationship is established.

!!! danger "Forgetting InitAbilityActorInfo is the #1 GAS setup bug"
    If you skip this call, abilities will appear to grant but never activate, effects will fail to apply, and you'll get vague errors about invalid actor info. If something "should work but doesn't," check this first.

**Why `Mixed` replication mode?** GAS offers three replication modes. `Mixed` is the best default for player-controlled characters — it replicates minimal data to simulated proxies and full data to the owning client. See [Replication Modes](../networking/replication-modes.md) for the full breakdown.

**Why `bAbilitiesInitialized`?** `PossessedBy` can fire more than once (e.g., when a player respawns and gets re-possessed). The guard prevents granting duplicate abilities.

!!! note "ASC on the PlayerState"
    Some architectures (notably Lyra) put the ASC on the `PlayerState` instead of the `Character`. This lets the ASC persist across respawns. It's a valid pattern, but it changes the `InitAbilityActorInfo` calls and adds complexity. We're starting with ASC-on-Character because it's simpler to learn. You can refactor to PlayerState-based ASC later if your game needs it — the [Ability System Component](../core-concepts/ability-system-component.md) page covers both approaches.

## 6. The Base Ability Class

Every gameplay ability in your project should inherit from a project-specific base class, not directly from `UGameplayAbility`. This gives you a single place to add shared properties and defaults as your project grows.

For now, we'll add one custom property — an input tag for binding abilities to Enhanced Input actions — and set the instancing policy.

=== "Header (.h)"
    ```cpp
    // Source/YourProject/Public/YourProjectGameplayAbility.h
    #pragma once

    #include "CoreMinimal.h"
    #include "Abilities/GameplayAbility.h"
    #include "GameplayTagContainer.h"
    #include "YourProjectGameplayAbility.generated.h"

    UCLASS(Abstract)
    class YOURPROJECT_API UYourProjectGameplayAbility : public UGameplayAbility
    {
        GENERATED_BODY()

    public:
        UYourProjectGameplayAbility();

        /**
         * The input tag that activates this ability.
         * Matched against InputAction tags in the input binding system.
         */
        UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
        FGameplayTag InputTag;
    };
    ```

=== "Source (.cpp)"
    ```cpp
    // Source/YourProject/Private/YourProjectGameplayAbility.cpp
    #include "YourProjectGameplayAbility.h"

    UYourProjectGameplayAbility::UYourProjectGameplayAbility()
    {
        // InstancedPerActor: one instance per character per ability.
        // Safe default — the ability can hold state without interfering
        // with other characters using the same ability class.
        InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    }
    ```

### Why InstancedPerActor?

GAS offers three instancing policies:

| Policy | Behavior | Use Case |
|---|---|---|
| `NonInstanced` | Shared CDO, no per-actor state | Simple, fire-and-forget abilities (Epic's built-in jump uses this) |
| `InstancedPerActor` | One instance per actor | Abilities that need state, tasks, or replication |
| `InstancedPerExecution` | New instance per activation | Abilities that can run multiple simultaneously |

We set `InstancedPerActor` on the base class because it supports the widest range of ability patterns — member variables, ability tasks, and replication all require instancing. Individual abilities that don't need these features can override to `NonInstanced` for lighter weight. See [Instancing Policy](../gameplay-abilities/instancing-policy.md) for the full comparison.

### Why a Custom InputTag Property?

The Enhanced Input system works with `FGameplayTag` to route input to abilities. By putting this tag directly on your base ability class, every Blueprint ability can set its input binding right in the class defaults — no extra mapping tables needed. We'll use this in [Your First Ability](your-first-ability.md).

## 7. Compile and Verify

Close the Unreal Editor (if open), then build from your IDE. You should get a clean compile with zero errors.

??? warning "Common compilation errors"

    **`Cannot open include file: 'AbilitySystemComponent.h'`**
    :   You forgot to add `GameplayAbilities` to your `Build.cs` PublicDependencyModuleNames.

    **`Cannot open include file: 'GameplayEffectExtension.h'`**
    :   Same — check your `Build.cs` for the `GameplayAbilities` module.

    **Linker errors mentioning `UAbilitySystemComponent`**
    :   The `GameplayAbilities` plugin isn't enabled in your `.uproject`.

    **`GENERATED_BODY` errors or "unrecognized token"**
    :   Make sure your `.generated.h` include is the **last** include in each header file. UHT (Unreal Header Tool) requires this.

    **`YOURPROJECT_API` not recognized**
    :   The API macro must match your module name exactly, in all caps. If your module is `Crucible`, the macro is `CRUCIBLE_API`.

### Verify in the Editor

Once it compiles:

1. Open the editor
2. Create a new Blueprint class with your character base as the parent
3. Open the Blueprint — you should see **AbilitySystemComponent** in the Components panel
4. In Class Defaults, you should see the **Startup Abilities** and **Startup Effects** arrays
5. Create a Blueprint child of your base ability — you should see the **Input Tag** property in Class Defaults

If all five check out, your GAS foundation is ready.

!!! success "Checkpoint"
    You now have:

    - [x] GAS modules in Build.cs
    - [x] GameplayAbilities plugin enabled
    - [x] Custom AbilitySystemGlobals subclass with ini config
    - [x] An Attribute Set with Health, MaxHealth, Stamina, MaxStamina, and a PendingDamage pipeline
    - [x] A Character base class with ASC, AttributeSet, and startup arrays
    - [x] A Base Ability class with InputTag and InstancedPerActor policy
    - [x] A clean compile

## What's Next

The foundation is in place. Time to build something with it. Head to [Your First Ability](your-first-ability.md) to create a melee attack ability that ties together everything you've just set up.
