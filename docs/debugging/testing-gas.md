---
title: Testing GAS
icon: material/test-tube
description: Writing automated tests for GAS abilities, effects, and attributes using UE's testing framework and the built-in test utilities.
---

# Testing GAS

Gameplay Abilities are among the most complex systems you'll build. Each ability is effectively a state machine that interacts with tags, attributes, effects, networking, prediction, and gameplay cues -- all simultaneously. Manual testing catches the obvious stuff, but it misses the combinatorial explosions: what happens when a player activates an ability while a stacking debuff is being removed during a cooldown reset triggered by another ability ending? You can't click through every permutation by hand.

Automated tests are the answer. The engine ships with test utilities specifically for GAS, and Epic uses them internally to validate core functionality. This page covers how to use those utilities, how to structure your own tests, and what patterns to follow.

## Why Test GAS

The case for automated GAS testing is stronger than for most gameplay systems:

- **State machine complexity.** An ability can be in many states (inactive, activated, committed, executing, ending, cancelled), and transitions between them are gated by tags, costs, cooldowns, and network conditions. Each transition is a potential bug.

- **Modifier interactions.** Five effects modifying the same attribute through different operations (additive, multiplicative, override) can produce unexpected results. A test that applies a known set of effects and checks the final value catches aggregation bugs instantly.

- **Tag combinatorics.** Blocking tags, required tags, immunity tags, cancel tags -- the interaction matrix grows fast. A test can verify that "ability X is blocked when tag Y is present" without you manually granting and revoking tags in PIE.

- **Regression safety.** GAS configurations change constantly during development. A designer tweaks a cooldown, a programmer refactors an ExecCalc, someone adds a new tag requirement. Tests catch the downstream breakage before it ships.

- **Networking edge cases.** Predicted abilities, replicated effects, and server corrections are nearly impossible to test manually with confidence. Even basic effect application tests run in a controlled environment catch issues that would only surface in multiplayer.

## The Built-in Test Utilities

Epic provides two classes specifically for testing GAS. They live in the `GameplayAbilities` plugin's public headers, so you can use them directly in your test modules.

### AAbilitySystemTestPawn

**Header:** `AbilitySystemTestPawn.h`

A minimal pawn that comes pre-wired with an Ability System Component. It inherits from `ADefaultPawn`, implements `IAbilitySystemInterface` and `IGameplayCueInterface`, and creates an ASC in its constructor:

```cpp
UCLASS(Blueprintable, BlueprintType, notplaceable, MinimalAPI)
class AAbilitySystemTestPawn : public ADefaultPawn,
    public IGameplayCueInterface,
    public IAbilitySystemInterface
{
    GENERATED_UCLASS_BODY()

    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

private:
    UPROPERTY()
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

public:
    static FName AbilitySystemComponentName;
};
```

In its `PostInitializeComponents`, it calls `AbilitySystemComponent->InitStats(UAbilitySystemTestAttributeSet::StaticClass(), NULL)`, which means the test pawn automatically gets the test attribute set registered on its ASC. You spawn one of these, and you immediately have a fully functional ASC with attributes ready to go.

**What it provides:**

- A pawn you can spawn in a test world without needing a game mode, player controller, or any project-specific setup
- An ASC with replication enabled (`SetIsReplicated(true)`)
- The test attribute set auto-registered via `InitStats`
- `IGameplayCueInterface` implementation so cue events route correctly

### UAbilitySystemTestAttributeSet

**Header:** `AbilitySystemTestAttributeSet.h`

A test attribute set with a rich selection of attributes covering common gameplay scenarios:

| Attribute | Type | Purpose |
|:---|:---|:---|
| `Health` | `float` | Core health value (HideFromModifiers -- use Damage meta attribute instead) |
| `MaxHealth` | `float` | Health cap (HideFromModifiers) |
| `Mana` | `FGameplayAttributeData` | Resource attribute using the struct-based format |
| `MaxMana` | `float` | Mana cap |
| `Damage` | `float` | Meta attribute for incoming damage (non-replicated) |
| `SpellDamage` | `float` | Powers spell-based effects |
| `PhysicalDamage` | `float` | Powers physical-based effects |
| `CritChance` | `float` | Critical hit chance |
| `CritMultiplier` | `float` | Critical hit damage multiplier |
| `ArmorDamageReduction` | `float` | Flat damage reduction percentage |
| `DodgeChance` | `float` | Chance to completely avoid damage |
| `LifeSteal` | `float` | Health recovery on damage dealt |
| `Strength` | `float` | Generic stat attribute |
| `StackingAttribute1` | `float` | For testing stacking behavior |
| `StackingAttribute2` | `float` | For testing stacking behavior |
| `NoStackAttribute` | `float` | For testing non-stacking behavior |

The set also overrides `PreGameplayEffectExecute` and `PostGameplayEffectExecute`. In the shipped source, `PostGameplayEffectExecute` implements a basic damage-to-health pipeline: when the `Damage` attribute is modified, it subtracts that value from `Health` and resets `Damage` to 0. This is exactly the meta attribute pattern described in [Damage Pipeline](../patterns/damage-pipeline.md).

!!! note "Why `mutable`?"
    You'll notice all attributes are marked `mutable`. The header includes a comment explaining this: it's done **only** so that tests can directly set attribute values without `const_cast` in hundreds of test lines. This is a testing convenience. Never use `mutable` on attributes in your real attribute sets.

## Setting Up a GAS Test Environment

The engine's GAS tests follow a consistent pattern. Here's how Epic sets up the test world internally, and how you should too.

### Creating a Test World

GAS tests need a `UWorld` because the ASC depends on actor lifecycle, ticking, and the gameplay tag system. The pattern is:

```cpp
// Create gameplay tags (required before any GAS operations)
UDataTable* TagTable = CreateYourGameplayTagDataTable();
UGameplayTagsManager::Get().PopulateTreeFromDataTable(TagTable);

// Create a test world
UWorld* World = UWorld::CreateWorld(EWorldType::Game, false);
FWorldContext& WorldContext = GEngine->CreateNewWorldContext(EWorldType::Game);
WorldContext.SetCurrentWorld(World);

FURL URL;
World->InitializeActorsForPlay(URL);
World->BeginPlay();
```

And tear it down after:

```cpp
World->EndPlay(EEndPlayReason::Quit);
GEngine->DestroyWorldContext(World);
World->DestroyWorld(false);
```

### Spawning Test Pawns with ASCs

Once you have a world, spawn test pawns and set initial attribute values:

```cpp
// Spawn a source (attacker) and destination (target)
AAbilitySystemTestPawn* SourceActor = World->SpawnActor<AAbilitySystemTestPawn>();
UAbilitySystemComponent* SourceASC = SourceActor->GetAbilitySystemComponent();

AAbilitySystemTestPawn* TargetActor = World->SpawnActor<AAbilitySystemTestPawn>();
UAbilitySystemComponent* TargetASC = TargetActor->GetAbilitySystemComponent();

// Set initial values
const float StartingHealth = 100.f;
TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health = StartingHealth;
TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->MaxHealth = StartingHealth;
```

### Constructing Effects in Code

The engine's tests create `UGameplayEffect` objects entirely in code using `NewObject`, without any asset loading:

```cpp
// The CONSTRUCT_CLASS macro used in Epic's tests
#define CONSTRUCT_CLASS(Class, Name) \
    Class* Name = NewObject<Class>(GetTransientPackage(), FName(TEXT(#Name)))
```

Then modifiers are added with a helper:

```cpp
template<typename MODIFIER_T>
FGameplayModifierInfo& AddModifier(
    UGameplayEffect* Effect,
    FProperty* Property,
    EGameplayModOp::Type Op,
    const MODIFIER_T& Magnitude)
{
    int32 Idx = Effect->Modifiers.Num();
    Effect->Modifiers.SetNum(Idx + 1);
    FGameplayModifierInfo& Info = Effect->Modifiers[Idx];
    Info.ModifierMagnitude = Magnitude;
    Info.ModifierOp = Op;
    Info.Attribute.SetUProperty(Property);
    return Info;
}
```

The `GET_FIELD_CHECKED` macro resolves an attribute's `FProperty`:

```cpp
#define GET_FIELD_CHECKED(Class, Field) \
    FindFieldChecked<FProperty>(Class::StaticClass(), GET_MEMBER_NAME_CHECKED(Class, Field))
```

### Ticking the World

For duration-based or periodic effects, you need to advance time. The engine uses this pattern:

```cpp
void TickWorld(float Time)
{
    const float Step = 0.1f;
    while (Time > 0.f)
    {
        World->Tick(ELevelTick::LEVELTICK_All, FMath::Min(Time, Step));
        Time -= Step;
        GFrameCounter++;
    }
}
```

The `GFrameCounter++` is necessary because the engine uses frame counters for various internal bookkeeping, and without incrementing it, certain time-dependent checks won't work correctly.

## Testing Abilities

### Activation Success and Failure

The engine's `AbilitySystemComponentTests.cpp` demonstrates the core pattern for testing ability lifecycle:

```cpp
void Test_ActivateAbilityFlow()
{
    // Grant the ability
    FGameplayAbilitySpec AbilitySpec{ UGameplayAbility::StaticClass(), 1 };
    FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(AbilitySpec);

    // Verify it was granted
    TArray<FGameplayAbilitySpecHandle> AllAbilities;
    ASC->GetAllAbilities(AllAbilities);
    Test->TestTrue(TEXT("Ability granted"), AllAbilities.Num() > 0);

    // Activate it
    const bool bActivated = ASC->TryActivateAbility(Handle, false);
    Test->TestTrue(TEXT("Activation succeeded"), bActivated);

    // Verify the spec is active
    FGameplayAbilitySpec* Spec = ASC->FindAbilitySpecFromHandle(Handle);
    Test->TestTrue(TEXT("Spec reports active"), Spec->IsActive());

    // Cancel and verify it ended
    ASC->CancelAbilityHandle(Handle);
    Test->TestFalse(TEXT("Spec inactive after cancel"), Spec->IsActive());
}
```

### Testing Tag-Based Blocking

To test that an ability is blocked by tags:

```cpp
void Test_AbilityBlockedByTag()
{
    // Create an ability CDO with ActivationBlockedTags
    UGameplayAbility* AbilityCDO = NewObject<UMyFireballAbility>();
    // ... configure its blocked tags ...

    FGameplayAbilitySpec Spec{ AbilityCDO, 1 };
    FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(Spec);

    // Apply an effect that grants the blocking tag
    CONSTRUCT_CLASS(UGameplayEffect, BlockingEffect);
    // ... add the blocking tag to GrantedTags ...
    BlockingEffect->DurationPolicy = EGameplayEffectDurationType::Infinite;
    FActiveGameplayEffectHandle BlockHandle =
        ASC->ApplyGameplayEffectToSelf(BlockingEffect, 1.f, ASC->MakeEffectContext());

    // Try to activate -- should fail
    const bool bActivated = ASC->TryActivateAbility(Handle, false);
    Test->TestFalse(TEXT("Blocked by tag"), bActivated);

    // Remove the blocking effect, try again -- should succeed
    ASC->RemoveActiveGameplayEffect(BlockHandle);
    const bool bActivatedAfter = ASC->TryActivateAbility(Handle, false);
    Test->TestTrue(TEXT("Activates after block removed"), bActivatedAfter);
}
```

### Testing Cost Deduction

```cpp
void Test_AbilityCostDeducted()
{
    const float StartingMana = 100.f;
    ASC->GetSet<UAbilitySystemTestAttributeSet>()->Mana =
        FGameplayAttributeData(StartingMana);

    // Grant an ability with a cost
    // (Your ability's CostGameplayEffect should subtract Mana)
    FGameplayAbilitySpec Spec{ UMyAbilityWithCost::StaticClass(), 1 };
    FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(Spec);

    // Activate and commit
    ASC->TryActivateAbility(Handle, false);

    // Check mana was reduced
    float CurrentMana = ASC->GetSet<UAbilitySystemTestAttributeSet>()->Mana.GetCurrentValue();
    Test->TestTrue(TEXT("Mana reduced after commit"), CurrentMana < StartingMana);
}
```

### Testing Cooldowns

```cpp
void Test_CooldownPreventsReactivation()
{
    FGameplayAbilitySpec Spec{ UMyAbilityWithCooldown::StaticClass(), 1 };
    FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(Spec);

    // First activation should succeed
    bool bFirst = ASC->TryActivateAbility(Handle, false);
    Test->TestTrue(TEXT("First activation succeeds"), bFirst);

    // End the ability (cooldown GE is now active)
    ASC->CancelAbilityHandle(Handle);

    // Second activation should fail (on cooldown)
    bool bSecond = ASC->TryActivateAbility(Handle, false);
    Test->TestFalse(TEXT("Blocked by cooldown"), bSecond);

    // Tick past cooldown duration
    TickWorld(CooldownDuration + 0.1f);

    // Third activation should succeed
    bool bThird = ASC->TryActivateAbility(Handle, false);
    Test->TestTrue(TEXT("Succeeds after cooldown expires"), bThird);
}
```

### Listening to ASC Callbacks

The engine's test suite demonstrates how to monitor the full ability lifecycle via callbacks:

```cpp
// Register for all ASC ability callbacks
ASC->AbilityActivatedCallbacks.AddLambda([](UGameplayAbility* Ability) {
    // Fired when any ability activates
});

ASC->AbilityCommittedCallbacks.AddLambda([](UGameplayAbility* Ability) {
    // Fired when an ability commits (cost and cooldown applied)
});

ASC->AbilityFailedCallbacks.AddLambda(
    [](const UGameplayAbility* Ability, const FGameplayTagContainer& Tags) {
    // Fired when activation fails, Tags contains failure reasons
});

ASC->AbilityEndedCallbacks.AddLambda([](UGameplayAbility* Ability) {
    // Fired when an ability ends (naturally or via cancel)
});
```

This pattern lets you assert that the correct callbacks fire in the correct order.

## Testing Effects

### Instant Damage Modifies Health

This is the simplest test and the one to start with. Straight from the engine's test suite:

```cpp
void Test_InstantDamage()
{
    const float DamageValue = 5.f;
    const float StartingHealth =
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health;

    // Create a simple damage effect
    CONSTRUCT_CLASS(UGameplayEffect, DamageEffect);
    AddModifier(DamageEffect,
        GET_FIELD_CHECKED(UAbilitySystemTestAttributeSet, Health),
        EGameplayModOp::Additive,
        FScalableFloat(-DamageValue));
    DamageEffect->DurationPolicy = EGameplayEffectDurationType::Instant;

    // Apply it
    SourceASC->ApplyGameplayEffectToTarget(DamageEffect, TargetASC, 1.f);

    // Verify
    Test->TestEqual(TEXT("Health reduced"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health,
        StartingHealth - DamageValue);
}
```

### Damage Through Meta Attributes

The engine also tests the meta attribute remap pattern, where `Damage` is modified and `PostGameplayEffectExecute` translates it to `-Health`:

```cpp
void Test_InstantDamageRemap()
{
    const float DamageValue = 5.f;
    const float StartingHealth =
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health;

    CONSTRUCT_CLASS(UGameplayEffect, DamageEffect);
    AddModifier(DamageEffect,
        GET_FIELD_CHECKED(UAbilitySystemTestAttributeSet, Damage),
        EGameplayModOp::Additive,
        FScalableFloat(DamageValue));  // positive Damage, not negative Health
    DamageEffect->DurationPolicy = EGameplayEffectDurationType::Instant;

    SourceASC->ApplyGameplayEffectToTarget(DamageEffect, TargetASC, 1.f);

    // Health should have decreased
    Test->TestEqual(TEXT("Health reduced via Damage remap"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health,
        StartingHealth - DamageValue);

    // Damage meta attribute should have been reset to 0
    Test->TestEqual(TEXT("Damage reset to 0"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Damage, 0.f);
}
```

### Duration Buff Application and Removal

```cpp
void Test_DurationBuff()
{
    const float BuffValue = 30.f;
    const float StartingMana =
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Mana.GetCurrentValue();

    // Apply an infinite duration buff
    CONSTRUCT_CLASS(UGameplayEffect, ManaBuffEffect);
    AddModifier(ManaBuffEffect,
        GET_FIELD_CHECKED(UAbilitySystemTestAttributeSet, Mana),
        EGameplayModOp::Additive,
        FScalableFloat(BuffValue));
    ManaBuffEffect->DurationPolicy = EGameplayEffectDurationType::Infinite;

    FActiveGameplayEffectHandle Handle =
        SourceASC->ApplyGameplayEffectToTarget(ManaBuffEffect, TargetASC, 1.f);

    // Mana should be buffed
    Test->TestEqual(TEXT("Mana buffed"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Mana.GetCurrentValue(),
        StartingMana + BuffValue);

    // Remove the effect
    TargetASC->RemoveActiveGameplayEffect(Handle);

    // Mana should return to original value
    Test->TestEqual(TEXT("Mana restored"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Mana.GetCurrentValue(),
        StartingMana);
}
```

### Stacking Behavior

```cpp
void Test_StackLimit()
{
    const float ChangePerGE = 5.f;
    const uint32 StackLimit = 2;
    const float StartingValue =
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->StackingAttribute1;

    CONSTRUCT_CLASS(UGameplayEffect, StackingEffect);
    AddModifier(StackingEffect,
        GET_FIELD_CHECKED(UAbilitySystemTestAttributeSet, StackingAttribute1),
        EGameplayModOp::Additive,
        FScalableFloat(ChangePerGE));
    StackingEffect->DurationPolicy = EGameplayEffectDurationType::HasDuration;
    StackingEffect->DurationMagnitude =
        FGameplayEffectModifierMagnitude(FScalableFloat(10.f));
    StackingEffect->StackLimitCount = StackLimit;
    StackingEffect->SetStackingType(EGameplayEffectStackingType::AggregateByTarget);
    StackingEffect->StackDurationRefreshPolicy =
        EGameplayEffectStackingDurationPolicy::NeverRefresh;
    StackingEffect->StackExpirationPolicy =
        EGameplayEffectStackingExpirationPolicy::ClearEntireStack;

    // Apply one more than the limit
    for (uint32 i = 0; i <= StackLimit; ++i)
    {
        SourceASC->ApplyGameplayEffectToTarget(StackingEffect, TargetASC, 1.f);
    }

    // Value should be capped at StackLimit * ChangePerGE, not (StackLimit+1)
    Test->TestEqual(TEXT("Stack capped at limit"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->StackingAttribute1,
        StartingValue + (StackLimit * ChangePerGE));
}
```

### Periodic Effects

The engine tests periodic effects by ticking the world forward:

```cpp
void Test_PeriodicDamage()
{
    const int32 NumPeriods = 10;
    const float PeriodSecs = 1.0f;
    const float DamagePerPeriod = 5.f;
    const float StartingHealth =
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health;

    CONSTRUCT_CLASS(UGameplayEffect, PeriodicDmgEffect);
    AddModifier(PeriodicDmgEffect,
        GET_FIELD_CHECKED(UAbilitySystemTestAttributeSet, Health),
        EGameplayModOp::Additive,
        FScalableFloat(-DamagePerPeriod));
    PeriodicDmgEffect->DurationPolicy = EGameplayEffectDurationType::HasDuration;
    PeriodicDmgEffect->DurationMagnitude =
        FGameplayEffectModifierMagnitude(FScalableFloat(NumPeriods * PeriodSecs));
    PeriodicDmgEffect->Period.Value = PeriodSecs;

    SourceASC->ApplyGameplayEffectToTarget(PeriodicDmgEffect, TargetASC, 1.f);

    // First tick applies immediately
    TickWorld(SMALL_NUMBER);
    int32 NumApplications = 1;

    Test->TestEqual(TEXT("First tick applied"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health,
        StartingHealth - (DamagePerPeriod * NumApplications));

    // Tick through all periods
    TickWorld(PeriodSecs * 0.1f); // small buffer for float precision
    for (int32 i = 0; i < NumPeriods; ++i)
    {
        TickWorld(PeriodSecs);
        ++NumApplications;
    }

    // Tick past the end -- no more damage
    TickWorld(PeriodSecs);
    Test->TestEqual(TEXT("No damage after expiry"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health,
        StartingHealth - (DamagePerPeriod * NumApplications));
}
```

### Immunity

```cpp
void Test_EffectImmunity()
{
    const float StartingHealth =
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health;

    // Apply an immunity effect that grants immunity to "Damage" tag
    CONSTRUCT_CLASS(UGameplayEffect, ImmunityEffect);
    ImmunityEffect->DurationPolicy = EGameplayEffectDurationType::Infinite;
    // ... configure immunity via GE component or GrantedApplicationImmunityTags ...

    TargetASC->ApplyGameplayEffectToSelf(
        ImmunityEffect, 1.f, TargetASC->MakeEffectContext());

    // Now apply damage -- it should be blocked
    CONSTRUCT_CLASS(UGameplayEffect, DamageEffect);
    AddModifier(DamageEffect,
        GET_FIELD_CHECKED(UAbilitySystemTestAttributeSet, Health),
        EGameplayModOp::Additive,
        FScalableFloat(-50.f));
    DamageEffect->DurationPolicy = EGameplayEffectDurationType::Instant;
    // ... tag the damage effect with "Damage" ...

    SourceASC->ApplyGameplayEffectToTarget(DamageEffect, TargetASC, 1.f);

    // Health should be unchanged
    Test->TestEqual(TEXT("Immune to damage"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health,
        StartingHealth);
}
```

## Testing Attributes

### Clamping in PreAttributeChange

`PreAttributeChange` is called whenever the current value of an attribute is about to change. You can test that your clamping logic works:

```cpp
void Test_HealthClampedToMax()
{
    const float MaxHealth = 100.f;
    TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health = MaxHealth;
    TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->MaxHealth = MaxHealth;

    // Apply a heal that would exceed max
    CONSTRUCT_CLASS(UGameplayEffect, OverhealEffect);
    AddModifier(OverhealEffect,
        GET_FIELD_CHECKED(UAbilitySystemTestAttributeSet, Health),
        EGameplayModOp::Additive,
        FScalableFloat(50.f));
    OverhealEffect->DurationPolicy = EGameplayEffectDurationType::Instant;

    SourceASC->ApplyGameplayEffectToTarget(OverhealEffect, TargetASC, 1.f);

    // If your PreAttributeChange clamps properly, health stays at max
    Test->TestTrue(TEXT("Health clamped to max"),
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health <= MaxHealth);
}
```

### The Damage Pipeline in PostGameplayEffectExecute

As shown earlier, the test attribute set implements a `Damage -> -Health` remap in `PostGameplayEffectExecute`. Testing this pipeline verifies that:

1. The Damage meta attribute receives the incoming value
2. Health is reduced by that amount
3. Damage is reset to 0 after processing

This is exactly what `Test_InstantDamageRemap` does. For your own attribute sets, write similar tests for every remap, clamp, or side effect in your `PostGameplayEffectExecute`.

### Aggregator Equation Verification

The engine's `Test_AttributeAggregators` is particularly thorough -- it applies modifiers of every operation type and verifies the final aggregated value matches the expected formula:

```cpp
// The aggregation equation from the engine:
// ExpectedResult = ((BaseValue + Additive) * Multiplicative / Division
//                   * CompoundMultiply) + FinalAdd
//
// Where:
// Multiplicative compounds as: 1.0 + ForEach(Value - 1.0)
// Division compounds as:       1.0 + ForEach(Value - 1.0)
// CompoundMultiply compounds:  1.0 *= ForEach(Value)

constexpr float X = BuffValue;
const float ExpectedResult =
    ((ManaBaseValue + X + X)
     * (1.0f + (X - 1.0f) + (X - 1.0f))
     / (1.0f + (X - 1.0f))
     * (X * X))
    + X + X;

const float CurrentValue =
    TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Mana.GetCurrentValue();
Test->TestEqual(TEXT("Aggregation equation correct"), CurrentValue, ExpectedResult);
```

This kind of test is invaluable for verifying that your understanding of the aggregation formula matches the engine's actual behavior.

## Testing Prediction

Prediction is the hardest part of GAS to test because it inherently involves client-server interaction. The engine's tests do explore prediction to some degree -- for example, setting an actor's role to `ROLE_AutonomousProxy` and verifying that predicted effects behave correctly:

```cpp
// Simulate client-side prediction
SourceActor->SetRole(ENetRole::ROLE_AutonomousProxy);
SourceComponent->CacheIsNetSimulated();
DestActor->SetRole(ENetRole::ROLE_AutonomousProxy);
DestComponent->CacheIsNetSimulated();
```

However, full prediction testing requires a multi-process setup that's beyond what simple automation tests provide. Some practical approaches:

- **Test predicted vs. authoritative behavior separately.** Verify that the same effect application produces correct results under both `ROLE_Authority` and `ROLE_AutonomousProxy`.

- **Test prediction key generation.** Verify that `FPredictionKey::CreateNewPredictionKey` produces valid, unique keys.

- **Use Gauntlet or network automation.** For serious prediction testing, UE's Gauntlet framework can spin up dedicated server and client instances in automated tests. This is how Epic tests networked gameplay at scale.

- **Manual multiplayer testing remains essential.** Some prediction edge cases (rollback during high latency, duplicate prediction keys) are best caught in PIE with multiple clients and artificial lag (`NetEmulationProfile`).

## Practical Example: Complete Test Class

Here's a complete, self-contained test class following the same pattern Epic uses internally. It creates a test world, spawns pawns, grants an ability, activates it, and verifies that the attribute changed.

```cpp
#include "CoreMinimal.h"
#include "Misc/AutomationTest.h"
#include "Engine/Engine.h"
#include "AbilitySystemComponent.h"
#include "AbilitySystemTestPawn.h"
#include "AbilitySystemTestAttributeSet.h"
#include "GameplayAbilitiesModule.h"
#include "AbilitySystemGlobals.h"

// Your custom ability that applies a damage effect on activation
#include "MyTestFireballAbility.h"

#if WITH_EDITOR

#define CONSTRUCT_CLASS(Class, Name) \
    Class* Name = NewObject<Class>(GetTransientPackage(), FName(TEXT(#Name)))

#define GET_FIELD_CHECKED(Class, Field) \
    FindFieldChecked<FProperty>(Class::StaticClass(), \
    GET_MEMBER_NAME_CHECKED(Class, Field))

class FMyGASTestSuite
{
public:
    FMyGASTestSuite(UWorld* InWorld, FAutomationTestBase* InTest)
        : World(InWorld), Test(InTest)
    {
        SourceActor = World->SpawnActor<AAbilitySystemTestPawn>();
        SourceASC = SourceActor->GetAbilitySystemComponent();
        SourceASC->GetSet<UAbilitySystemTestAttributeSet>()->Health = 100.f;
        SourceASC->GetSet<UAbilitySystemTestAttributeSet>()->MaxHealth = 100.f;

        TargetActor = World->SpawnActor<AAbilitySystemTestPawn>();
        TargetASC = TargetActor->GetAbilitySystemComponent();
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health = 100.f;
        TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->MaxHealth = 100.f;
    }

    ~FMyGASTestSuite()
    {
        if (SourceActor) World->EditorDestroyActor(SourceActor, false);
        if (TargetActor) World->EditorDestroyActor(TargetActor, false);
    }

    void Test_GrantAndActivateAbility()
    {
        // Grant
        FGameplayAbilitySpec Spec{ UMyTestFireballAbility::StaticClass(), 1 };
        FGameplayAbilitySpecHandle Handle = SourceASC->GiveAbility(Spec);

        TArray<FGameplayAbilitySpecHandle> Abilities;
        SourceASC->GetAllAbilities(Abilities);
        Test->TestTrue(TEXT("Ability granted"), Abilities.Num() == 1);

        // Activate
        bool bActivated = SourceASC->TryActivateAbility(Handle, false);
        Test->TestTrue(TEXT("Ability activated"), bActivated);

        // Check the ability's effect changed the target
        // (assuming your ability applies damage to a target)
        // This depends on your ability implementation
    }

    void Test_SimpleInstantDamage()
    {
        const float Damage = 25.f;
        const float StartHealth =
            TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health;

        CONSTRUCT_CLASS(UGameplayEffect, DmgEffect);
        AddModifier(DmgEffect,
            GET_FIELD_CHECKED(UAbilitySystemTestAttributeSet, Health),
            EGameplayModOp::Additive, FScalableFloat(-Damage));
        DmgEffect->DurationPolicy = EGameplayEffectDurationType::Instant;

        SourceASC->ApplyGameplayEffectToTarget(DmgEffect, TargetASC, 1.f);

        Test->TestEqual(TEXT("Damage applied"),
            TargetASC->GetSet<UAbilitySystemTestAttributeSet>()->Health,
            StartHealth - Damage);
    }

private:
    template<typename T>
    FGameplayModifierInfo& AddModifier(
        UGameplayEffect* Effect, FProperty* Prop,
        EGameplayModOp::Type Op, const T& Mag)
    {
        int32 Idx = Effect->Modifiers.Num();
        Effect->Modifiers.SetNum(Idx + 1);
        FGameplayModifierInfo& Info = Effect->Modifiers[Idx];
        Info.ModifierMagnitude = Mag;
        Info.ModifierOp = Op;
        Info.Attribute.SetUProperty(Prop);
        return Info;
    }

    UWorld* World;
    FAutomationTestBase* Test;
    AAbilitySystemTestPawn* SourceActor;
    UAbilitySystemComponent* SourceASC;
    AAbilitySystemTestPawn* TargetActor;
    UAbilitySystemComponent* TargetASC;
};

// --- Automation Test Registration ---

#define ADD_GAS_TEST(Name) \
    TestFunctions.Add(&FMyGASTestSuite::Name); \
    TestFunctionNames.Add(TEXT(#Name))

class FMyGASTest : public FAutomationTestBase
{
public:
    typedef void (FMyGASTestSuite::*TestFunc)();

    FMyGASTest(const FString& InName) : FAutomationTestBase(InName, false)
    {
        ADD_GAS_TEST(Test_GrantAndActivateAbility);
        ADD_GAS_TEST(Test_SimpleInstantDamage);
    }

    virtual EAutomationTestFlags GetTestFlags() const override
    {
        return EAutomationTestFlags::EditorContext
             | EAutomationTestFlags::ProductFilter;
    }
    virtual uint32 GetRequiredDeviceNum() const override { return 1; }

protected:
    virtual FString GetBeautifiedTestName() const override
    {
        return "Game.AbilitySystem.MyGASTests";
    }

    virtual void GetTests(
        TArray<FString>& OutNames, TArray<FString>& OutCommands) const override
    {
        for (const FString& Name : TestFunctionNames)
        {
            OutNames.Add(Name);
            OutCommands.Add(Name);
        }
    }

    bool RunTest(const FString& Parameters) override
    {
        TestFunc Func = nullptr;
        for (int32 i = 0; i < TestFunctionNames.Num(); ++i)
        {
            if (TestFunctionNames[i] == Parameters)
            {
                Func = TestFunctions[i];
                break;
            }
        }
        if (!Func) return false;

        UWorld* World = UWorld::CreateWorld(EWorldType::Game, false);
        FWorldContext& Ctx = GEngine->CreateNewWorldContext(EWorldType::Game);
        Ctx.SetCurrentWorld(World);

        FURL URL;
        World->InitializeActorsForPlay(URL);
        World->BeginPlay();

        uint64 InitialFrameCounter = GFrameCounter;
        {
            FMyGASTestSuite Suite(World, this);
            (Suite.*Func)();
        }
        GFrameCounter = InitialFrameCounter;

        World->EndPlay(EEndPlayReason::Quit);
        GEngine->DestroyWorldContext(World);
        World->DestroyWorld(false);
        return true;
    }

    TArray<TestFunc> TestFunctions;
    TArray<FString> TestFunctionNames;
};

namespace
{
    FMyGASTest FMyGASTestInstance(TEXT("FMyGASTest"));
}

#endif // WITH_EDITOR
```

### Running the Tests

1. **Session Frontend.** Open `Window > Developer Tools > Session Frontend`, switch to the `Automation` tab, and search for your test name (e.g. "MyGASTests"). Check the tests and click Run.

2. **Command Line.** Run the editor with:
   ```
   UnrealEditor-Cmd.exe YourProject.uproject -ExecCmds="Automation RunTests Game.AbilitySystem.MyGASTests" -NullRHI -NoSound -Unattended
   ```

3. **CI/CD.** The command-line approach works in headless CI. Use `-NullRHI` and `-NoSound` to skip rendering. Return codes indicate pass/fail.

## The "Debug Ability" Approach

Not every team has the infrastructure for automated tests from day one. For non-automated testing, the "debug ability" pattern is useful:

1. **Create a debug/cheat ability** that you can activate via console command or debug input.
2. **Have it execute the scenario you want to test** -- apply effects, grant abilities, set attribute values.
3. **Use `ShowDebug AbilitySystem`** to visually verify the results.

```cpp
UCLASS()
class UGA_DebugDamageTest : public UGameplayAbility
{
    GENERATED_BODY()

    virtual void ActivateAbility(...) override
    {
        UAbilitySystemComponent* ASC = GetAbilitySystemComponentFromActorInfo();

        // Apply 50 damage to self
        UGameplayEffect* DamageGE = NewObject<UGameplayEffect>();
        DamageGE->DurationPolicy = EGameplayEffectDurationType::Instant;
        // ... add Damage modifier ...

        ASC->ApplyGameplayEffectToSelf(DamageGE, 1.f, MakeEffectContext(...));

        // Log results
        UE_LOG(LogTemp, Display, TEXT("Health after damage: %f"),
            ASC->GetNumericAttribute(
                UMyAttributeSet::GetHealthAttribute()));

        EndAbility(...);
    }
};
```

Grant this through a cheat manager or console command. It's quick, doesn't require test infrastructure, and gives you immediate feedback. The downside is it doesn't run automatically and doesn't catch regressions -- so treat it as a complement to real tests, not a replacement.

## What to Test (Checklist)

| Area | What to Verify |
|:---|:---|
| **Ability activation** | Succeeds when conditions met, fails when blocked/on cooldown/can't afford |
| **Ability cancellation** | Cancel-by-tag works, ability ends cleanly |
| **Cost deduction** | Attribute reduced by expected amount on commit |
| **Cooldown** | Applied on commit, blocks reactivation, expires correctly |
| **Instant effects** | Base value changes permanently |
| **Duration effects** | Current value changes while active, reverts on removal |
| **Periodic effects** | Fires expected number of times, stops at expiry |
| **Stacking** | Respects stack limit, refresh/reset policies work |
| **Modifier aggregation** | Formula produces expected result with multiple modifier types |
| **Meta attributes** | Damage->Health remap, attribute reset |
| **Clamping** | Values stay within expected bounds |
| **Immunity** | Blocked effects don't apply |
| **Tag interactions** | Granted tags appear, removed tags disappear, tag-gated logic works |

## Related Pages

- [ShowDebug AbilitySystem](showdebug-abilitysystem.md) -- visual debugging in PIE
- [Troubleshooting](troubleshooting.md) -- symptom-based diagnosis
- [Modifiers](../gameplay-effects/modifiers.md) -- the aggregation formula your tests should verify
- [Cooldowns and Costs](../gameplay-effects/cooldowns-and-costs.md) -- what cost/cooldown tests should check
