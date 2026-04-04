---
title: "Example: Ranged Attack"
icon: material/bow-arrow
description: Complete ranged attack ability with projectile spawning, travel-time damage, mana cost, and proper GE spec passing.
---

# Example: Ranged Attack

A ranged projectile ability that demonstrates spawning actors from within an ability, passing damage data to a projectile, and having the projectile — not the ability — apply the damage effect on hit. This is the pattern you'll use for fireballs, arrows, energy blasts, and anything else that travels through space before dealing damage.

!!! tip "Prerequisites"
    This example assumes you've completed [Project Setup](../getting-started/project-setup.md) and have a working character with an ASC, AttributeSet (Health, Mana, PendingDamage), and a base ability class. If you need Stamina instead of Mana for your project, just swap the attribute — the pattern is identical.

## What We're Building

A ranged attack that:

- **Costs mana** (10 per shot)
- **Has a cooldown** (0.8 seconds)
- **Spawns a projectile** in the camera's aim direction
- **Projectile applies damage on hit** using a GE spec created by the ability
- **Blocked by CC** — can't fire while dead, stunned, or silenced
- **Uses SetByCaller** for the damage amount (data-driven)

The key difference from melee: the ability doesn't apply damage directly. It creates a damage GE spec, hands it to the projectile, and the projectile applies it when it hits something. This means damage attribution (who dealt the damage, what ability caused it) is preserved correctly through the `FGameplayEffectContext`.

## Step 1: Create the Effects

### GE_Cost_RangedAttack

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.Mana` |
| **Modifiers[0] — Modifier Op** | Add |
| **Modifiers[0] — Magnitude** | Scalable Float: `-10.0` |

### GE_Cooldown_RangedAttack

| Setting | Value |
|:---|:---|
| **Duration Policy** | Has Duration |
| **Duration Magnitude** | Scalable Float: `0.8` |
| **GrantedTags** | `Cooldown.Ability.RangedAttack` |

### GE_Damage_Ranged

Same pattern as the melee damage effect — SetByCaller magnitude targeting `PendingDamage`. You can reuse a single generic `GE_Damage` effect for both melee and ranged if you prefer.

| Setting | Value |
|:---|:---|
| **Duration Policy** | Instant |
| **Modifiers[0] — Attribute** | `YourProjectAttributeSet.PendingDamage` |
| **Modifiers[0] — Modifier Op** | Add |
| **Modifiers[0] — Magnitude Type** | Set By Caller |
| **Modifiers[0] — Set By Caller Tag** | `SetByCaller.Damage` |

??? question "Why not reuse GE_Damage_Melee?"
    You absolutely can — and in many projects, you should. A single `GE_Damage` with SetByCaller works for any damage source. The ability sets the magnitude, the effect applies it. We use a separate name here for clarity, but sharing a damage effect across abilities is a common and recommended pattern. See [SetByCaller](../gameplay-effects/set-by-caller.md) for more on this.

## Step 2: Create the Projectile Actor

Create a new Blueprint actor: `BP_Projectile`. This is a standard Unreal actor — it's not a GAS class, but it needs to know how to apply a GAS damage effect.

### Components

| Component | Type | Purpose |
|:---|:---|:---|
| **CollisionSphere** | Sphere Collision | Hit detection. Set radius to ~10-20 units |
| **ProjectileMovement** | Projectile Movement Component | Handles velocity, gravity, homing. Set Initial Speed and Max Speed (e.g., 3000) |
| **Mesh** | Static Mesh (or Niagara) | Visual representation |

### Properties

The projectile needs to store the information required to apply damage on hit:

=== "Blueprint"
    Add these variables to `BP_Projectile`:

    | Variable | Type | Purpose |
    |:---|:---|:---|
    | **DamageEffectSpecHandle** | `Gameplay Effect Spec Handle` | The fully configured damage spec, created by the ability |

=== "C++"
    ```cpp
    UCLASS()
    class ABP_Projectile : public AActor
    {
        GENERATED_BODY()

    public:
        ABP_Projectile();

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

This is the critical part. When the projectile hits something, it needs to apply the damage GE spec to the target's ASC. The spec already contains the correct instigator, source, and damage magnitude — the ability configured all of that before passing it to the projectile.

=== "Blueprint"
    ```
    Event OnComponentHit (CollisionSphere)
        │
        ▼
    Other Actor ──► Get Ability System Component (via interface or cast)
        │
        ▼ [Valid ASC?]
    Target ASC ──► Apply Gameplay Effect Spec to Self
        ├── Spec Handle: DamageEffectSpecHandle (the stored spec)
        │
        ▼
    Destroy Actor (self)
    ```

=== "C++"
    ```cpp
    void ABP_Projectile::OnHit(
        UPrimitiveComponent* HitComp,
        AActor* OtherActor,
        UPrimitiveComponent* OtherComp,
        FVector NormalImpulse,
        const FHitResult& Hit)
    {
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
    A common mistake is having the projectile create its own `MakeOutgoingSpec` using its own context. The problem: the projectile isn't a GAS actor. It has no ASC, so `MakeEffectContext()` would have the wrong instigator and source. The damage would look like it came from nobody. Always pass the pre-built spec from the ability — it carries the correct `FGameplayEffectContext` with the caster's ASC as the instigator.

### Why This Works

When the ability creates the GE spec via `MakeOutgoingGameplayEffectSpec`, the spec's `FGameplayEffectContext` is automatically populated with:

- **Instigator** — the caster's actor (from `GetAvatarActorFromActorInfo()`)
- **EffectCauser** — the caster's actor (by default; you can override to be the projectile)
- **InstigatorAbilitySystemComponent** — the caster's ASC

This context travels with the spec handle. When the projectile applies it to the target, all attribution is correct — damage logs, kill credit, and any `PostGameplayEffectExecute` logic that checks the source will work properly.

??? tip "Setting the projectile as EffectCauser"
    If your damage pipeline needs to know that the damage came from a projectile (for knockback direction, hit effects, etc.), you can modify the context after creating the spec:

    ```cpp
    FGameplayEffectContextHandle Context =
        SpecHandle.Data->GetEffectContext();
    Context.AddActors({Projectile});
    ```

    Or in Blueprint, use **Get Effect Context** on the spec handle, then set the source object. The instigator (caster) stays correct while the causer becomes the projectile.

## Step 3: Create the Ability Blueprint

Create `GA_RangedAttack` with **YourProjectGameplayAbility** as the parent.

### Class Defaults

| Property | Value | Why |
|:---|:---|:---|
| **Input Tag** | `InputTag.Combat.Secondary` | Maps to ranged attack input |
| **Ability Tags** | `Ability.Combat.RangedAttack` | Identifies this ability |
| **Activation Blocked Tags** | `State.Dead`, `CrowdControl.Hard`, `CrowdControl.Soft.Silence` | Can't cast while dead, stunned, or silenced |
| **Cost Gameplay Effect Class** | `GE_Cost_RangedAttack` | Mana check and deduction |
| **Cooldown Gameplay Effect Class** | `GE_Cooldown_RangedAttack` | 0.8s re-activation delay |
| **Instancing Policy** | `InstancedPerActor` | Needed if using ability tasks |
| **Net Execution Policy** | `LocalPredicted` | Feels responsive on client |

!!! info "Silence vs Root"
    Notice we block on `CrowdControl.Soft.Silence` but not `CrowdControl.Soft.Root`. A rooted character can't move but *can* still cast ranged abilities — that's the design distinction between root and silence. Compare with [Jump](jump.md) which blocks on Root but not Silence.

### Event Graph

=== "Blueprint"
    ```
    Event ActivateAbility
        │
        ▼
    Commit Ability ──► [Failed] ──► End Ability (Cancelled = true)
        │
        ▼ [Success]
    Make Outgoing Gameplay Effect Spec (GE_Damage_Ranged)
        │
        ▼
    Assign Set By Caller Magnitude
        ├── Spec Handle: (from above)
        ├── Data Tag: SetByCaller.Damage
        └── Magnitude: 20.0
        │
        ▼
    Get Aim Direction (see below)
        │
        ▼
    Spawn Actor from Class
        ├── Class: BP_Projectile
        ├── Location: Character location + forward offset
        ├── Rotation: Aim direction rotation
        │
        ▼
    Set DamageEffectSpecHandle on the spawned projectile
        │
        ▼
    End Ability (Cancelled = false)
    ```

=== "Step-by-Step"
    1. **Commit Ability** — checks cost (mana) and cooldown, applies both
    2. **Make Outgoing GE Spec** — creates the damage spec with the caster's context baked in
    3. **Assign SetByCaller** — sets the damage value (20.0) on the spec
    4. **Get Aim Direction** — get the camera's forward vector for aiming
    5. **Spawn Actor** — creates `BP_Projectile` at the character's position, aimed in the fire direction
    6. **Set DamageEffectSpecHandle** — passes the pre-built spec to the projectile
    7. **End Ability** — the ability's job is done. The projectile handles damage on its own schedule

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

        // Create the damage spec with correct context
        FGameplayEffectSpecHandle DamageSpec =
            MakeOutgoingGameplayEffectSpec(
                DamageEffectClass, // UPROPERTY: TSubclassOf<UGameplayEffect>
                GetAbilityLevel());

        DamageSpec.Data->SetSetByCallerMagnitude(
            FGameplayTag::RequestGameplayTag(
                FName("SetByCaller.Damage")),
            DamageAmount); // UPROPERTY: float, default 20.0

        // Get aim direction from controller
        FVector SpawnLocation;
        FRotator SpawnRotation;
        GetAimDirectionAndLocation(SpawnLocation, SpawnRotation);

        // Spawn the projectile
        FActorSpawnParameters SpawnParams;
        SpawnParams.Owner = AvatarActor;
        SpawnParams.Instigator = Cast<APawn>(AvatarActor);
        SpawnParams.SpawnCollisionHandlingOverride =
            ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

        ABP_Projectile* Projectile = GetWorld()->SpawnActor<ABP_Projectile>(
            ProjectileClass,
            SpawnLocation,
            SpawnRotation,
            SpawnParams);

        if (Projectile)
        {
            Projectile->DamageEffectSpecHandle = DamageSpec;
        }

        EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
    }
    ```

### Getting Aim Direction

For a third-person game, the projectile should fire toward where the camera is looking, not where the character is facing. A simple approach:

=== "Blueprint"
    ```
    Get Player Camera Manager ──► Get Actor Forward Vector ──► (Aim Direction)
    Get Avatar Actor ──► Get Actor Location + (Aim Direction * 100) ──► (Spawn Location)
    ```

=== "C++"
    ```cpp
    void UGA_RangedAttack::GetAimDirectionAndLocation(
        FVector& OutLocation,
        FRotator& OutRotation)
    {
        AActor* AvatarActor = GetAvatarActorFromActorInfo();
        const FGameplayAbilityActorInfo* ActorInfo = GetCurrentActorInfo();

        if (APlayerController* PC = Cast<APlayerController>(
                ActorInfo->PlayerController.Get()))
        {
            FVector CameraLocation;
            FRotator CameraRotation;
            PC->GetPlayerViewPoint(CameraLocation, CameraRotation);

            OutRotation = CameraRotation;
            OutLocation = AvatarActor->GetActorLocation()
                + (CameraRotation.Vector() * 100.f); // offset to avoid self-hit
        }
        else
        {
            // Fallback for AI: use actor's forward vector
            OutLocation = AvatarActor->GetActorLocation()
                + (AvatarActor->GetActorForwardVector() * 100.f);
            OutRotation = AvatarActor->GetActorRotation();
        }
    }
    ```

!!! tip "Spawn offset"
    The `* 100.f` offset prevents the projectile from spawning inside the character and immediately hitting the collision capsule. Adjust this based on your character's size. For a production setup, you'd typically use a socket on the character's mesh (weapon muzzle, hand, etc.) as the spawn point.

## Step 4: Wire Input

1. **Input Action:** Create `IA_RangedAttack` (Digital/Bool)
2. **Input Mapping Context:** Map `IA_RangedAttack` to **Right Mouse Button** in `IMC_Default`
3. **Input-to-Tag mapping:** Route `IA_RangedAttack` to `InputTag.Combat.Secondary`
4. **Grant the ability:** Add `GA_RangedAttack` to your character's Startup Abilities array

## Step 5: Test

1. **Play** — right-click, a projectile should spawn and fly forward
2. **Mana** drops by 10 (check with `showdebug abilitysystem`)
3. **Cooldown** — spam right-click, only fires every 0.8s
4. **Damage** — aim at a target with an ASC, confirm their Health drops by 20
5. **CC** — apply `CrowdControl.Soft.Silence` to your character and confirm the ability is blocked

??? example "ShowDebug checklist"
    | Scenario | Expected |
    |:---|:---|
    | Fire at a target | Target's PendingDamage receives 20, Health drops after processing |
    | Fire with insufficient mana | Nothing happens — cost check fails |
    | Fire during cooldown | Nothing happens — cooldown tag blocks |
    | Fire while silenced | Nothing happens — Activation Blocked Tags |
    | Fire while rooted | Works — rooted doesn't block casting |
    | Projectile hits wall | Projectile destroys, no damage applied |

!!! warning "Common issues"
    1. **Projectile hits self** — make sure the spawn location is offset far enough, and add an ignore-instigator check in `OnHit`
    2. **No damage on target** — verify the target has an ASC and that `DamageEffectSpecHandle` is being set on the projectile *after* spawning
    3. **Wrong damage amount** — check that `SetSetByCallerMagnitude` is called *before* passing the spec to the projectile
    4. **Damage attributed to nobody** — you're probably creating a new spec in the projectile instead of using the one from the ability

## Variations

### Charge-Up

Hold the input to charge, release to fire. Use the `WaitInputRelease` ability task to detect when the player lets go. Scale the damage magnitude and projectile speed based on charge time. Apply a `State.Charging` tag via Activation Owned Tags to block other abilities during charge.

### Homing Projectile

Set the Projectile Movement Component's `bIsHomingProjectile` to true and assign a `HomingTargetComponent` on the target actor. You can populate the target via a simple line trace from the camera before spawning, or use GAS [Target Actors](../gameplay-abilities/targeting/target-actors.md) for more sophisticated targeting.

### Multi-Projectile (Shotgun)

Spawn multiple projectiles in a cone. Loop 5-8 times, each with a slightly randomized rotation offset from the aim direction. Use the same GE spec for all of them — each projectile carries the same damage, and the spec handle can be shared (it's a shared pointer under the hood).

### Cast Animation

Add a `PlayMontageAndWait` before the spawn step. The projectile spawns from an AnimNotify at the right frame in the cast animation (same pattern as [Melee Attack](melee-attack.md)). The ability stays active until the montage completes or is interrupted.

## Related

- [Melee Attack](melee-attack.md) — same damage pipeline, but applied directly instead of via projectile
- [SetByCaller](../gameplay-effects/set-by-caller.md) — how the damage magnitude system works
- [Targeting](../gameplay-abilities/targeting/index.md) — more sophisticated aim and target selection
- [Damage Pipeline](../patterns/damage-pipeline.md) — how PendingDamage flows through to Health
- [Ability Tasks](../gameplay-abilities/ability-tasks.md) — WaitInputRelease and other tasks for variations
