---
title: Ranged Attack
description: Complete ranged attack ability with projectile spawning, GE spec passing for correct damage attribution, mana cost, and silence CC blocking.
---

# Example: Ranged Attack

<div class="example-badges" markdown>
  <span class="badge badge--intermediate">Intermediate</span>
</div>

## Overview

A ranged projectile ability that demonstrates spawning actors from within an ability, passing a pre-built damage spec to the projectile, and having the projectile -- not the ability -- apply the damage on hit. This is the pattern for fireballs, arrows, energy blasts, and anything else that travels through space before dealing damage. The critical lesson here is how `FGameplayEffectSpecHandle` carries the caster's ASC context to the point of impact, ensuring correct damage attribution even though the projectile (not the caster) triggers the application.

## What We're Building

- **Mana cost** of 20 per shot
- **Cooldown** of 1.5 seconds
- **Projectile spawning** aimed from the camera, not the character facing direction
- **GE spec passing** -- the ability creates the damage spec, the projectile applies it on hit
- **SetByCaller damage** of 30, data-driven via the spec
- **Silence blocking** -- can't cast while silenced (`CrowdControl.Soft.Silence`), distinct from melee which only blocks on hard CC
- **PendingDamage meta attribute** -- same damage pipeline as melee

## Prerequisites

!!! tip "What you need before starting"
    This example assumes you have completed [Project Setup](../getting-started/project-setup.md) and have:

    - A character with an **Ability System Component**
    - An **AttributeSet** with Health, Mana (or Magic), and PendingDamage attributes
    - A **base ability class** (`YourProjectGameplayAbility`) with an `InputTag` property
    - An **input binding system** that routes Enhanced Input actions to abilities by tag
    - A **static mesh or Niagara system** for the projectile visual (a sphere will do for prototyping)

    If any of that is missing, start with [Project Setup](../getting-started/project-setup.md).

## Step 1: Create the Effects

### GE_Cost_RangedAttack

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] -- Attribute** | `YourProjectAttributeSet.Mana` |
| **Modifiers[0] -- Modifier Op** | Add |
| **Modifiers[0] -- Magnitude** | Scalable Float: `-20.0` |

### GE_Cooldown_RangedAttack

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `1.5` (1.5 seconds) |
| **Granted Tags** | `Cooldown.Ability.RangedAttack` (via TargetTagsGameplayEffectComponent) |

### GE_Damage_Ranged

Same pattern as the melee damage effect -- SetByCaller magnitude targeting `PendingDamage`. You can reuse a single generic `GE_Damage` effect for both melee and ranged if you prefer.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] -- Attribute** | `YourProjectAttributeSet.PendingDamage` |
| **Modifiers[0] -- Modifier Op** | Add |
| **Modifiers[0] -- Magnitude Type** | Set By Caller |
| **Modifiers[0] -- Set By Caller Tag** | `SetByCaller.Damage` |

??? question "Why not reuse GE_Damage_Melee?"
    You absolutely can -- and in many projects, you should. A single `GE_Damage` with SetByCaller works for any damage source. The ability sets the magnitude, the effect applies it. We use separate names here for clarity, but sharing a damage effect across abilities is a common and recommended pattern. See [SetByCaller](../gameplay-effects/set-by-caller.md) for more on this.

## Step 2: Create the Ability

Create `GA_RangedAttack` with **YourProjectGameplayAbility** as the parent.

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Input Tag** | `InputTag.Combat.Secondary` | Maps to your ranged attack input |
| **Ability Tags** | `Ability.Combat.RangedAttack` | Identifies this ability for queries and blocking |
| **Activation Blocked Tags** | `State.Dead`, `CrowdControl.Hard`, `CrowdControl.Soft.Silence` | Can't cast while dead, stunned, or silenced |
| **Instancing Policy** | `InstancedPerActor` | Required when using Ability Tasks |
| **Net Execution Policy** | `LocalPredicted` | Feels responsive on the client |
| **Cost Gameplay Effect Class** | `GE_Cost_RangedAttack` | Mana check and deduction |
| **Cooldown Gameplay Effect Class** | `GE_Cooldown_RangedAttack` | 1.5s re-activation delay |

!!! info "Silence vs Root vs Stun"
    Notice we block on `CrowdControl.Soft.Silence` but not `CrowdControl.Soft.Root`. A rooted character can't move but *can* still cast ranged abilities -- that's the design distinction between root and silence. Compare with [Melee Attack](melee-attack.md) which blocks only on `CrowdControl.Hard` (stun) -- melee doesn't require "casting" so silence doesn't affect it. This is a deliberate design choice: silence shuts down casters, stun shuts down everyone.

### Event Graph

=== "Blueprint"

    !!! abstract "Event Graph"

        1. **ActivateAbility** fires
        2. **CommitAbility** checks mana cost and cooldown, applies both if successful. If it fails, **EndAbility** immediately
        3. **MakeOutgoingGESpec** creates the damage spec from `GE_Damage_Ranged` with the caster's ASC context baked in -- the spec remembers *who* created it
        4. **AssignSetByCallerMagnitude** sets `SetByCaller.Damage` to `30.0` on the spec
        5. **Get Aim Direction** from the camera's forward vector (not the character's facing)
        6. **SpawnActor** creates `BP_Projectile` at the character's position, aimed in the fire direction
        7. **Set DamageEffectSpecHandle** on the spawned projectile -- passes the pre-built spec carrying the caster's context
        8. **EndAbility** -- the ability's job is done. The projectile handles damage on its own schedule

    ```mermaid
    flowchart LR
        A["ActivateAbility"]:::event --> B["CommitAbility"]:::func
        B -->|Failed| C["EndAbility\n(cancelled)"]:::endpoint
        B -->|Success| D["MakeOutgoingGESpec\nGE_Damage_Ranged"]:::func
        D --> E["SetByCaller\nDamage = 30"]:::func
        E --> F["Get Aim\nDirection"]:::func
        F --> G["SpawnActor\nBP_Projectile"]:::func
        G --> H["Set DamageSpec\non Projectile"]:::func
        H --> I["EndAbility"]:::endpoint

        classDef event fill:#5c1a1a,stroke:#ff6666,color:#fff
        classDef func fill:#2a2a4a,stroke:#9b89f5,color:#fff
        classDef task fill:#1a3a5c,stroke:#4a9eff,color:#fff
        classDef endpoint fill:#1a4a2d,stroke:#6bcb3a,color:#fff
    ```

=== "C++"
    ```cpp
    void UGA_RangedAttack::ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData)
    {
        if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }

        AActor* AvatarActor = GetAvatarActorFromActorInfo();
        if (!AvatarActor)
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
            return;
        }

        // --- Create the damage spec with caster context ---
        FGameplayEffectSpecHandle DamageSpec =
            MakeOutgoingGameplayEffectSpec(
                DamageEffectClass,  // UPROPERTY: TSubclassOf<UGameplayEffect>
                GetAbilityLevel());

        DamageSpec.Data->SetSetByCallerMagnitude(
            FGameplayTag::RequestGameplayTag(
                FName("SetByCaller.Damage")),
            DamageAmount);  // UPROPERTY: float, default 30.0

        // --- Get aim direction from the player's camera ---
        FVector SpawnLocation;
        FRotator SpawnRotation;
        GetAimDirectionAndLocation(SpawnLocation, SpawnRotation);

        // --- Spawn the projectile ---
        FActorSpawnParameters SpawnParams;
        SpawnParams.Owner = AvatarActor;
        SpawnParams.Instigator = Cast<APawn>(AvatarActor);
        SpawnParams.SpawnCollisionHandlingOverride =
            ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

        AProjectileBase* Projectile =
            GetWorld()->SpawnActor<AProjectileBase>(
                ProjectileClass,  // UPROPERTY: TSubclassOf<AProjectileBase>
                SpawnLocation,
                SpawnRotation,
                SpawnParams);

        if (Projectile)
        {
            // Pass the pre-built spec to the projectile
            Projectile->DamageEffectSpecHandle = DamageSpec;
        }

        EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
    }

    void UGA_RangedAttack::GetAimDirectionAndLocation(
        FVector& OutLocation,
        FRotator& OutRotation) const
    {
        AActor* AvatarActor = GetAvatarActorFromActorInfo();
        const FGameplayAbilityActorInfo* ActorInfo =
            GetCurrentActorInfo();

        if (APlayerController* PC = Cast<APlayerController>(
                ActorInfo->PlayerController.Get()))
        {
            FVector CameraLocation;
            FRotator CameraRotation;
            PC->GetPlayerViewPoint(CameraLocation, CameraRotation);

            OutRotation = CameraRotation;
            // Offset to avoid self-hit with character capsule
            OutLocation = AvatarActor->GetActorLocation()
                + (CameraRotation.Vector() * SpawnOffset);
                // UPROPERTY: float, default 100.0
        }
        else
        {
            // Fallback for AI: use actor's forward vector
            OutLocation = AvatarActor->GetActorLocation()
                + (AvatarActor->GetActorForwardVector() * SpawnOffset);
            OutRotation = AvatarActor->GetActorRotation();
        }
    }
    ```

### Getting Aim Direction

For a third-person game, the projectile should fire toward where the camera is looking, not where the character is facing. The ability queries the PlayerController's view point and spawns the projectile in that direction.

=== "Blueprint"
    ```
    Get Player Camera Manager --> Get Actor Forward Vector --> (Aim Direction)
    Get Avatar Actor --> Get Actor Location + (Aim Direction * 100) --> (Spawn Location)
    ```

=== "C++"
    See the `GetAimDirectionAndLocation` method in the C++ tab above.

!!! tip "Spawn offset"
    The `* 100.f` offset prevents the projectile from spawning inside the character and immediately hitting the collision capsule. Adjust this based on your character's size. For a production setup, you'd typically use a socket on the character's mesh (weapon muzzle, hand, etc.) as the spawn point.

## Step 3: Create the Projectile Actor

The projectile is a standard Unreal actor -- it's not a GAS class, but it needs to know how to apply a GAS damage effect. Create a new Blueprint or C++ actor: `BP_Projectile` (or `AProjectileBase` in C++).

### Components

| Component | Type | Purpose |
|:---|:---|:---|
| **CollisionSphere** | Sphere Collision | Hit detection. Set radius to 10-20 units. Set collision profile to `Projectile` or similar |
| **ProjectileMovement** | UProjectileMovementComponent | Handles velocity, gravity, homing. Set Initial Speed and Max Speed (e.g., 3000) |
| **Mesh** | Static Mesh or Niagara | Visual representation. A sphere is fine for prototyping |

### Properties

The projectile needs one critical property: the damage spec from the ability.

=== "Blueprint"
    Add a variable to `BP_Projectile`:

    | Variable | Type | Purpose |
    |:---|:---|:---|
    | **DamageEffectSpecHandle** | `Gameplay Effect Spec Handle` | The fully configured damage spec, created by the ability |

=== "C++"
    ```cpp
    UCLASS()
    class AProjectileBase : public AActor
    {
        GENERATED_BODY()

    public:
        AProjectileBase();

        /** Damage spec created by the spawning ability.
          * Carries the caster's ASC context for correct attribution. */
        UPROPERTY(BlueprintReadWrite, Meta = (ExposeOnSpawn = true))
        FGameplayEffectSpecHandle DamageEffectSpecHandle;

    protected:
        UPROPERTY(VisibleAnywhere)
        TObjectPtr<USphereComponent> CollisionSphere;

        UPROPERTY(VisibleAnywhere)
        TObjectPtr<UProjectileMovementComponent> ProjectileMovement;

        UFUNCTION()
        void OnHit(UPrimitiveComponent* HitComp,
                    AActor* OtherActor,
                    UPrimitiveComponent* OtherComp,
                    FVector NormalImpulse,
                    const FHitResult& Hit);

        virtual void BeginPlay() override;
    };
    ```

### OnHit: Applying Damage

This is the critical part. When the projectile hits something, it applies the damage GE spec to the target's ASC. The spec already contains the correct instigator, source, and damage magnitude -- the ability configured all of that before passing it to the projectile.

=== "Blueprint"
    ```
    Event OnComponentHit (CollisionSphere)
        |
        v
    Other Actor --> Get Ability System Component (via interface or cast)
        |
        v [Valid ASC?]
    Target ASC --> Apply Gameplay Effect Spec to Self
        |-- Spec Handle: DamageEffectSpecHandle (the stored spec)
        |
        v
    Destroy Actor (self)
    ```

    Important: also add a branch at the top to check `Other Actor != Get Instigator()` -- this prevents the projectile from damaging the caster if it spawns slightly inside them.

=== "C++"
    ```cpp
    void AProjectileBase::BeginPlay()
    {
        Super::BeginPlay();

        CollisionSphere->OnComponentHit.AddDynamic(
            this, &AProjectileBase::OnHit);

        // Destroy after 5 seconds if nothing is hit
        SetLifeSpan(5.0f);
    }

    void AProjectileBase::OnHit(
        UPrimitiveComponent* HitComp,
        AActor* OtherActor,
        UPrimitiveComponent* OtherComp,
        FVector NormalImpulse,
        const FHitResult& Hit)
    {
        // Don't damage the caster
        if (!OtherActor || OtherActor == GetInstigator())
        {
            Destroy();
            return;
        }

        // Get the target's ASC
        UAbilitySystemComponent* TargetASC =
            UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(
                OtherActor);

        if (TargetASC && DamageEffectSpecHandle.IsValid())
        {
            TargetASC->ApplyGameplayEffectSpecToSelf(
                *DamageEffectSpecHandle.Data.Get());
        }

        Destroy();
    }
    ```

!!! danger "Don't create a new GE spec in the projectile"
    A common mistake is having the projectile create its own `MakeOutgoingSpec` using its own context. The problem: the projectile isn't a GAS actor. It has no ASC, so `MakeEffectContext()` would have the wrong instigator and source. The damage would look like it came from nobody. Always pass the pre-built spec from the ability -- it carries the correct `FGameplayEffectContext` with the caster's ASC as the instigator.

### Why Passing the Spec Handle Works

When the ability creates the GE spec via `MakeOutgoingGameplayEffectSpec`, the spec's `FGameplayEffectContext` is automatically populated with:

| Context Field | Value | Used For |
|:---|:---|:---|
| **Instigator** | The caster's actor (from `GetAvatarActorFromActorInfo()`) | Kill credit, damage logs |
| **EffectCauser** | The caster's actor (by default) | Source identification |
| **InstigatorAbilitySystemComponent** | The caster's ASC | Attribute lookups, source-side modifiers |
| **AbilityInstance** | The ability that created the spec | Ability-specific damage scaling |

This context travels with the spec handle as a shared pointer. When the projectile applies the spec to the target seconds later and potentially far from the caster, all attribution is correct. Damage logs, kill credit, and any `PostGameplayEffectExecute` logic that checks the source will work properly.

??? tip "Setting the projectile as EffectCauser"
    If your damage pipeline needs to know that the damage came from a projectile (for knockback direction, hit effects, etc.), you can modify the context after creating the spec:

    ```cpp
    FGameplayEffectContextHandle Context =
        DamageSpec.Data->GetEffectContext();
    Context.AddActors({Projectile});
    ```

    Or in Blueprint, use **Get Effect Context** on the spec handle, then set the source object. The instigator (caster) stays correct while the causer becomes the projectile.

## Step 3: Wire Input

### 1. InputAction Asset

Create `IA_RangedAttack` (right-click > **Input > Input Action**). Set the Value Type to `Digital (Bool)`.

### 2. InputMappingContext

In your `IMC_Default`, add:

- **Input Action:** `IA_RangedAttack`
- **Key:** Right Mouse Button (or your preferred ranged attack key)

### 3. Route Input to the ASC

=== "Blueprint"
    In your Character Blueprint's Event Graph:

    1. Add an **Enhanced Input Action** event node for `IA_RangedAttack`
    2. From the exec pin, call **Get Ability System Component** on Self
    3. Iterate activatable abilities and try to activate any whose Input Tag matches `InputTag.Combat.Secondary`

=== "C++"
    ```cpp
    // Same pattern as melee/dodge input binding,
    // but matching InputTag.Combat.Secondary
    EnhancedInput->BindAction(
        RangedAttackAction,  // UPROPERTY: TObjectPtr<UInputAction>
        ETriggerEvent::Started,
        this, &AYourCharacter::OnRangedAttackInput);
    ```

See [Input Binding](../gameplay-abilities/input-binding.md) for the full production input routing setup.

### 4. Grant the Ability

Add `GA_RangedAttack` to your character's **Startup Abilities** array in Class Defaults.

## Step 4: Test

### Basic Test

1. Hit Play
2. Right-click -- a projectile should spawn and fly toward where the camera is looking
3. Mana drops by 20 (check with `showdebug abilitysystem`)
4. Spam right-click -- only fires every 1.5 seconds
5. Aim at a target with an ASC -- confirm their Health drops by 30
6. Apply `CrowdControl.Soft.Silence` to your character -- confirm the ability is blocked

### ShowDebug Checklist

| Scenario | Expected Result |
|:---|:---|
| Fire at a target | Target's PendingDamage receives 30, Health drops after processing |
| Fire with insufficient mana | Nothing happens -- cost check fails |
| Fire during cooldown | Nothing happens -- cooldown tag blocks |
| Fire while silenced | Nothing happens -- `CrowdControl.Soft.Silence` in Activation Blocked Tags |
| Fire while rooted | Works -- rooted doesn't block casting |
| Projectile hits wall | Projectile destroys, no damage applied |
| Projectile hits caster | Projectile destroys (instigator check), no self-damage |
| Projectile misses everything | Destroys after lifespan expires (5 seconds) |

### Edge Cases

- **Caster dies after firing** -- the projectile is already in flight with a valid spec. It will still deal damage and attribute it to the (now dead) caster. This is correct behavior
- **Target gains `State.Invulnerable` after projectile fires** -- the damage is checked at application time, so invulnerability at the moment of impact blocks it. This is correct
- **Multiple projectiles from rapid fire** -- each carries its own spec handle (shared pointer), so they all attribute correctly and independently

!!! warning "Common issues"
    1. **Projectile hits self** -- spawn offset is too small, or the instigator check in `OnHit` is missing
    2. **No damage on target** -- verify the target has an ASC and that `DamageEffectSpecHandle` is set on the projectile *after* spawning
    3. **Wrong damage amount** -- check that `SetSetByCallerMagnitude` is called *before* passing the spec to the projectile
    4. **Damage attributed to nobody** -- you're creating a new spec in the projectile instead of using the one from the ability. This is the most common ranged ability bug
    5. **Projectile doesn't move** -- check that `ProjectileMovementComponent` has `InitialSpeed` and `MaxSpeed` set to non-zero values

## The Full Flow

Here's the complete sequence from button press to damage application:

1. **Input** fires `IA_RangedAttack`
2. Input handler finds abilities with `InputTag.Combat.Secondary` and calls `TryActivateAbility`
3. The ASC checks activation requirements:
    - Blocked tags: Does the owner have `State.Dead`, `CrowdControl.Hard`, or `CrowdControl.Soft.Silence`?
4. If checks pass, the ability **activates** and `CommitAbility` runs:
    - Cost: Do we have 20+ Mana? If yes, apply `GE_Cost_RangedAttack` (Mana -20)
    - Cooldown: Apply `GE_Cooldown_RangedAttack` (tag granted for 1.5 seconds)
5. The ability creates a damage spec via `MakeOutgoingGameplayEffectSpec(GE_Damage_Ranged)`. The spec's `FGameplayEffectContext` is populated with the caster's actor, ASC, and ability reference
6. `SetSetByCallerMagnitude` sets the damage value (30.0) on the spec
7. The ability gets the camera aim direction and spawns `BP_Projectile` at an offset from the character
8. The ability passes `DamageEffectSpecHandle` to the projectile. The spec (and its context) now lives on the projectile
9. The ability calls **End Ability**. The ability is done -- it does not wait for the projectile
10. The projectile flies through space via `UProjectileMovementComponent`
11. On hit, the projectile gets the target's ASC and calls `ApplyGameplayEffectSpecToSelf` with the stored spec
12. The target's `PostGameplayEffectExecute` processes `PendingDamage` -- armor, shields, invulnerability checks -- then applies the final result to Health
13. The projectile destroys itself

The key insight: the ability and the damage application are temporally and spatially separated. The spec handle bridges that gap, carrying the caster's full context from the moment of casting to the moment of impact.

## Variations

### Charge-Up

Hold the input to charge, release to fire. Use the `WaitInputRelease` ability task to detect when the player lets go. Scale the `DamageAmount` and projectile speed based on charge time. Apply a `State.Charging` tag via Activation Owned Tags to block other abilities during charge. The ability stays active until the player releases, which means End Ability is deferred -- you need to handle interruption (stun during charge) cleanly.

### Homing Projectile

Set the ProjectileMovementComponent's `bIsHomingProjectile` to true and assign a `HomingTargetComponent` on the target actor. Populate the target via a line trace from the camera before spawning, or use GAS [Target Actors](../gameplay-abilities/targeting/target-actors.md) for more sophisticated targeting.

### Multi-Projectile (Shotgun)

Spawn multiple projectiles in a cone. Loop 5-8 times, each with a slightly randomized rotation offset from the aim direction. Use the same GE spec handle for all of them -- each projectile carries the same damage, and the spec handle is a shared pointer under the hood, so passing it to multiple actors is safe and efficient.

## Related Pages

- [Melee Attack](melee-attack.md) -- same damage pipeline, but applied directly instead of via projectile
- [Dodge Roll](dodge-roll.md) -- stamina cost, i-frames, root motion montage
- [SetByCaller](../gameplay-effects/set-by-caller.md) -- how the damage magnitude system works
- [Targeting](../gameplay-abilities/targeting/index.md) -- more sophisticated aim and target selection
- [Damage Pipeline](../patterns/damage-pipeline.md) -- how PendingDamage flows through to Health
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) -- WaitInputRelease and other tasks for variations
- [Immunity](../gameplay-effects/immunity.md) -- how State.Invulnerable blocks damage at application time
