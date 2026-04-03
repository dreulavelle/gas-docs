---
title: Net Execution Policies
description: The four ability net execution policies — LocalPredicted, LocalOnly, ServerInitiated, ServerOnly — with flow diagrams and decision guidance.
---

# Net Execution Policies

Every `UGameplayAbility` has a `NetExecutionPolicy` that determines where the ability's code runs in a networked game. This is set per-ability class and controls the interaction between client, server, prediction, and replication.

```cpp
// In your ability's constructor or Class Defaults
NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
```

## The Four Policies

### LocalPredicted

```cpp
EGameplayAbilityNetExecutionPolicy::LocalPredicted
```

**The default for player-activated abilities.** The ability activates on the owning client immediately (predicted) and on the server. The client doesn't wait for server confirmation before running `ActivateAbility`.

```
Client presses button
  → Client: ActivateAbility() runs immediately (with prediction key)
  → Client sends ServerTryActivateAbility RPC to server
  → Server: validates, runs ActivateAbility()
  → Server sends confirm/reject to client
  → Client reconciles
```

**Use for:**

- Any ability the player triggers that should feel instant (attacks, dodges, jumps, casts)
- Abilities that modify attributes predictively (cost deduction, self-buffs)

**Prediction:** Full prediction support. Side effects in the initial callstack are predicted.

**Key behavior:** If the server rejects the activation, the client rolls back all predicted side effects.

### LocalOnly

```cpp
EGameplayAbilityNetExecutionPolicy::LocalOnly
```

**Runs only on the owning client. Never runs on the server. Never replicated.**

```
Client presses button
  → Client: ActivateAbility() runs
  → (nothing sent to server)
  → (no server-side execution)
```

**Use for:**

- Cosmetic-only abilities (camera shake, UI effects, local particles)
- Client-side feedback that doesn't affect gameplay state
- Abilities that are purely informational (showing a preview, highlighting targets)

**Prediction:** No prediction needed -- there's nothing to reconcile since the server never knows about it.

!!! warning "No gameplay state changes"
    LocalOnly abilities should not modify replicated attributes, grant replicated tags, or do anything that the server needs to know about. If they do, you'll have desync.

### ServerInitiated

```cpp
EGameplayAbilityNetExecutionPolicy::ServerInitiated
```

**Activates on the server first, then replicates to the owning client.** The client does not predict -- it waits for the server to tell it to activate.

```
Client (or server) requests activation
  → Server: ActivateAbility() runs
  → Server replicates activation to owning client
  → Client: ActivateAbility() runs (after replication)
```

**Use for:**

- AI abilities (AI always runs on server)
- Abilities triggered by game events rather than player input
- Abilities where prediction would be incorrect or misleading (complex server-only calculations)
- Abilities granted by gameplay effects or environmental triggers

**Prediction:** None. The client waits for server confirmation before anything happens locally.

**Trade-off:** Adds a round-trip of latency before the client sees the ability activate. For player-facing abilities, this can feel sluggish.

### ServerOnly

```cpp
EGameplayAbilityNetExecutionPolicy::ServerOnly
```

**Runs exclusively on the server. Never runs on any client.**

```
Activation triggered
  → Server: ActivateAbility() runs
  → (nothing sent to client directly)
  → (effects/tags replicate normally through GE replication)
```

**Use for:**

- Backend game logic (applying periodic world effects, game phase transitions)
- Admin/cheat abilities
- Abilities that only need server-side state changes (the visual feedback comes from replicated GEs/cues)
- AI abilities where you don't even need the ability to run on simulated proxies

**Prediction:** None. Clients see the results through replicated effects, tags, and cues.

## Flow Comparison

| Step | LocalPredicted | LocalOnly | ServerInitiated | ServerOnly |
|:---|:---:|:---:|:---:|:---:|
| Client activates locally | Immediately | Immediately | After server confirms | Never |
| Server activates | Yes | Never | First | Yes (only) |
| Prediction key generated | Yes | No | No | No |
| RPC to server | Yes | No | Yes (request) | N/A |
| RPC to client | Confirm/Reject | None | Activate | None |
| Rollback on reject | Yes | N/A | N/A | N/A |

## Combining with Replication Modes

The execution policy interacts with the [replication mode](replication-modes.md):

| Policy | Full Mode | Mixed Mode | Minimal Mode |
|:---|:---|:---|:---|
| **LocalPredicted** | Full effect replication to all | Full to owner, tags/cues to others | Extra work needed |
| **LocalOnly** | No server interaction | No server interaction | No server interaction |
| **ServerInitiated** | Full replication of results | Owner gets full, others get tags/cues | Tags/cues only |
| **ServerOnly** | Results replicate to all | Results replicate per mode | Minimal results |

## NetworkSyncPoint Task

The `UAbilityTask_NetworkSyncPoint` is a special ability task that creates a synchronization barrier between client and server. Both sides must reach the sync point before either proceeds.

```cpp
UAbilityTask_NetworkSyncPoint* Task =
    UAbilityTask_NetworkSyncPoint::WaitNetSync(
        this, EAbilityTaskNetSyncType::BothWait);
Task->OnSync.AddDynamic(this, &UMyAbility::OnSyncComplete);
Task->ReadyForActivation();
```

Sync types:

| Type | Behavior |
|:---|:---|
| `OnlyServerWait` | Server waits for client to reach this point |
| `OnlyClientWait` | Client waits for server to reach this point |
| `BothWait` | Both wait for each other |

Use this when your ability has phases that must be synchronized -- for example, a targeting phase where the client picks a target and the server needs to validate before the ability proceeds.

## Decision Guide

| Ability Type | Recommended Policy |
|:---|:---|
| Player attack / combat ability | **LocalPredicted** |
| Dodge / movement ability | **LocalPredicted** |
| Self-buff activation | **LocalPredicted** |
| Item use | **LocalPredicted** |
| UI-only preview / highlight | **LocalOnly** |
| Camera shake / screen effect | **LocalOnly** |
| AI ability | **ServerInitiated** or **ServerOnly** |
| Environment effect (trap, zone) | **ServerOnly** |
| Game mode transition | **ServerOnly** |
| Ability triggered by GE | **ServerInitiated** |

## Related Pages

- [Prediction](prediction.md) -- how LocalPredicted abilities handle prediction
- [Replication Modes](replication-modes.md) -- what data flows to whom
- [Batching](batching.md) -- reducing RPC overhead for predicted abilities
- [Lifecycle and Activation](../gameplay-abilities/lifecycle-and-activation.md) -- the full ability activation flow
