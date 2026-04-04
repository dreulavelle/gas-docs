---
icon: material/link-variant
---

# External Resources

Curated links to official documentation, community guides, video tutorials, and sample projects. Last verified against UE 5.7.

## Official (Epic Games)

| Resource | Description |
|---|---|
| [Gameplay Ability System Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-for-unreal-engine) | Epic's official GAS documentation hub |
| [Gameplay Attributes and Effects](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-attributes-and-gameplay-effects-for-the-gameplay-ability-system-in-unreal-engine) | Official guide to attributes and effects |
| [Using Gameplay Tags](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-gameplay-tags-in-unreal-engine) | Official Gameplay Tags documentation |

## Community Guides

| Resource | Description |
|---|---|
| [tranek/GASDocumentation](https://github.com/tranek/GASDocumentation) | The original comprehensive GAS guide -- covers architecture, patterns, and networking in depth. Community gold standard. |
| [Jerakin/GAS-CheatSheet](https://github.com/Jerakin/GAS-CheatSheet) | Compact cheat sheet for GAS classes and their relationships |
| [GameplayTags and FNames In-Depth](https://itsbaffled.github.io/posts/UE/GameplayTags-And-FNames-In-Depth) | Deep technical dive into FName internals, GameplayTag memory layout, replication bit-tuning, and performance characteristics. Excellent for understanding what's happening under the hood. |

## Video Tutorials

| Resource | Description |
|---|---|
| [Ali Elzoheiry's GAS Playlist](https://www.youtube.com/watch?v=1QgGyndH_u4&list=PLNwKK6OwH7eVaq19HBUEL3UnPAfbpcUSL) | 26-video series covering GAS from setup through advanced patterns. Practical, project-based approach. |

## Sample Projects

| Project | Description |
|---|---|
| **Lyra Starter Game** | Epic's official sample project demonstrating modern GAS usage — ability sets, GE components, input binding via Enhanced Input. Available from the Epic Games Launcher. |
| **Action RPG** | Epic's older but still instructive sample showing GAS in an RPG context — abilities, effects, inventory integration. Available from the Epic Games Launcher (Learning tab). |

## Source Code

GAS source lives inside your engine installation. The key directories:

| Path (relative to engine root) | Contents |
|---|---|
| `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/` | All public headers — start here for API reference |
| `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/` | Implementation files — read these when the headers aren't enough |
| `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/` | All built-in `UAbilityTask` headers |
| `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Async/` | All `UAbilityAsync` task headers |
| `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/` | All `UGameplayEffectComponent` subclass headers |

For a default Windows installation, the full path is:

```
C:\Program Files\Epic Games\UE_5.7\Engine\Plugins\Runtime\GameplayAbilities\
```

Replace `UE_5.7` with your engine version.

## Related Pages

- [Getting Started](../getting-started/index.md) -- the recommended starting point for newcomers following these resources
