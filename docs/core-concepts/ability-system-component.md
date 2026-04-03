---
icon: material/chip
---

# The Ability System Component

The **Ability System Component** (`UAbilitySystemComponent`, often shortened to ASC) is the hub of GAS. Every actor that participates in the ability system needs one. It's the manager, the bookkeeper, the switchboard — it owns and orchestrates everything else.

## What the ASC Owns

The ASC is responsible for:

- **Granted Abilities** — the set of `UGameplayAbility` instances this actor can activate
- **Active Gameplay Effects** — all currently active effects (buffs, debuffs, DoTs) modifying this actor
- **Attribute Sets** — the stat containers (Health, Mana, etc.) registered to this actor
- **Gameplay Tags** — the tag state for this actor, including tags granted by effects and "loose" tags added manually
- **Replication** — syncing all of the above across the network

Think of the ASC as the single point of contact for anything ability-related on an actor. When you want to activate an ability, you ask the ASC. When you want to apply an effect, you apply it *to* the ASC. When you want to know if an actor is stunned, you query the ASC's tags.

## Placement: Character vs PlayerState

The first design decision you'll face is: *where does the ASC live?*

There are two common placements, and the right choice depends on your game.

### On the Character (Simple)

```cpp
UCLASS()
class AMyCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    AMyCharacter();

    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

private:
    UPROPERTY(VisibleAnywhere)
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;
};
```

**When to use this:** Single-player games, prototyping, learning GAS, or any case where the character never needs to persist data across respawns.

**Advantages:**

- Simpler setup — everything lives in one place
- Easier to reason about — the ASC is on the same actor as the mesh, animations, and collision

**Disadvantage:** When the character is destroyed (respawn), the ASC and all its state (active effects, tag counts, ability cooldowns) are destroyed with it. If a player had a 30-second buff with 10 seconds remaining, it's gone on respawn.

### On the PlayerState (Multiplayer)

```cpp
UCLASS()
class AMyPlayerState : public APlayerState, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    AMyPlayerState();

    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

private:
    UPROPERTY(VisibleAnywhere)
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;
};
```

**When to use this:** Multiplayer games where players respawn. The `APlayerState` persists across pawn destruction/recreation — the ASC and all its state survive respawns.

**Advantages:**

- Buff timers, tag state, and cooldowns survive pawn death
- Consistent with how Fortnite and Lyra handle it

**Disadvantage:** Slightly more complex initialization — you need to tell the ASC about both the PlayerState (the owner) and the Pawn (the avatar), and keep that in sync when the pawn changes.

!!! tip "Rule of thumb"
    If your game has multiplayer respawns, put the ASC on the PlayerState. If it's single-player or you're learning, put it on the Character. You can always move it later — the rest of GAS doesn't care where the ASC lives, only that `GetAbilitySystemComponent()` returns it.

## IAbilitySystemInterface

You might have noticed both examples implement `IAbilitySystemInterface`. This is a C++ interface with a single method:

```cpp
virtual UAbilitySystemComponent* GetAbilitySystemComponent() const = 0;
```

GAS uses this interface everywhere to find the ASC for a given actor. When you call `UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(Actor)`, it casts the actor to `IAbilitySystemInterface` and calls `GetAbilitySystemComponent()`. If your actor doesn't implement this interface, GAS can't find the ASC.

!!! warning "This must be C++"
    `IAbilitySystemInterface` is a C++ interface — it cannot be implemented in Blueprint. This is one of the hard C++ requirements of GAS. Your character (or PlayerState) base class must implement it in C++.

If your ASC lives on the PlayerState but your abilities need to find it from the *Pawn*, have your Pawn class also implement `IAbilitySystemInterface` and return the PlayerState's ASC:

```cpp
UAbilitySystemComponent* AMyCharacter::GetAbilitySystemComponent() const
{
    const AMyPlayerState* PS = GetPlayerState<AMyPlayerState>();
    return PS ? PS->GetAbilitySystemComponent() : nullptr;
}
```

## Replication Modes

The ASC has three replication modes that control how ability and effect data is synced across the network. The mode is set once at construction:

```cpp
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
```

| Mode | What Replicates to... | Owner Client | Other Clients | Best For |
|---|---|---|---|---|
| **Full** | GEs, Tags, Cues | Everything | Everything | AI-controlled actors (NPCs, minions) |
| **Mixed** | GEs to owner, Tags + Cues to others | Full GE data | Tags + Cues only | Player-controlled characters |
| **Minimal** | Tags + Cues only | Tags + Cues | Tags + Cues | AI that only needs visual feedback |

**Quick decision guide:**

- Player-controlled characters → **Mixed** (the owner needs full GE data for prediction; other clients just need tags and cues for visuals)
- AI with abilities that matter (boss with interruptible cast bars) → **Full**
- AI where only visual state matters (minions that show a stun VFX) → **Minimal**

!!! info "More detail"
    Replication is a deep topic. See the [Networking deep dive](../networking/replication-modes.md) for bandwidth implications, the replication proxy, and optimization techniques.

## InitAbilityActorInfo

After creating the ASC, you **must** call `InitAbilityActorInfo` to tell it who owns the ASC and who the physical avatar is:

```cpp
AbilitySystemComponent->InitAbilityActorInfo(/*OwnerActor=*/ this, /*AvatarActor=*/ this);
```

If the ASC is on the PlayerState but the avatar is the Pawn:

```cpp
AbilitySystemComponent->InitAbilityActorInfo(/*OwnerActor=*/ this, /*AvatarActor=*/ GetPawn());
```

### When to Call It

The timing matters and is different on server vs client:

=== "ASC on Character"

    ```cpp
    // Server: called when a controller possesses this pawn
    void AMyCharacter::PossessedBy(AController* NewController)
    {
        Super::PossessedBy(NewController);
        AbilitySystemComponent->InitAbilityActorInfo(this, this);
    }

    // Client: called when PlayerState is replicated
    void AMyCharacter::OnRep_PlayerState()
    {
        Super::OnRep_PlayerState();
        AbilitySystemComponent->InitAbilityActorInfo(this, this);
    }
    ```

=== "ASC on PlayerState"

    ```cpp
    // Server: call in PossessedBy (the PlayerState exists and the Pawn just got possessed)
    void AMyCharacter::PossessedBy(AController* NewController)
    {
        Super::PossessedBy(NewController);

        AMyPlayerState* PS = GetPlayerState<AMyPlayerState>();
        if (PS)
        {
            PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
        }
    }

    // Client: call in OnRep_PlayerState (the PlayerState just replicated)
    void AMyCharacter::OnRep_PlayerState()
    {
        Super::OnRep_PlayerState();

        AMyPlayerState* PS = GetPlayerState<AMyPlayerState>();
        if (PS)
        {
            PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
        }
    }
    ```

!!! danger "What breaks if you skip it"
    If `InitAbilityActorInfo` is never called (or called too late), you'll see:

    - Abilities that fail to activate with no clear error
    - Effects that apply but don't modify the right attributes
    - Animation montages that don't play (the avatar actor is null)
    - Gameplay Cues that fire on the wrong actor or don't fire at all
    - Replication that silently does nothing

    If your GAS setup "isn't working" and you can't figure out why, check this first.

## AbilitySystemReplicationProxyInterface

In multiplayer projects, the `IAbilitySystemReplicationProxyInterface` provides an optimization path where a proxy actor can batch replication data for multiple ASCs. This is an advanced pattern used in large-scale games to reduce bandwidth.

For most projects, you don't need this. If you're building something with many AI actors that need ability state replicated efficiently, see [Replication Proxy](../networking/replication-proxy.md) for details.

## One ASC Per Actor vs Shared ASC

The standard pattern is one ASC per actor. But there are cases where you might want to share an ASC:

- **Vehicles** — the driver's ASC might serve the vehicle too
- **Summons/Pets** — a summoned entity might share the summoner's ASC
- **Weapons** — some designs have the weapon be a separate actor that uses the character's ASC

Shared ASC patterns work but add complexity around avatar switching and cleanup. If you're starting out, use one ASC per actor. Sharing is an optimization you can layer in later if you need it.

!!! tip "Key takeaway"
    The ASC is the entry point for everything in GAS. Get its placement and initialization right, and the rest of the system falls into place. Get it wrong, and you'll chase mysterious bugs for hours.
