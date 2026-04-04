# Gameplay Ability System Reference

The comprehensive guide to Unreal Engine's Gameplay Ability System. Every class, every concept, every pattern — for all skill levels.

**[Read the docs →](https://gas.spoked.dev)**

## What's Inside

- **Getting Started** — mental model, project setup, your first ability
- **Core Concepts** — ASC, Tags, Attributes, Effects, Abilities, Cues
- **Deep Dives** — Gameplay Effects, Abilities, Cues, and Networking in depth
- **Patterns & Recipes** — damage pipelines, buff systems, tag architecture, step-by-step procedures
- **Examples** — 7 complete abilities from beginner to advanced (Blueprint + C++)
- **Debug & Optimize** — showdebug, troubleshooting, replication optimization
- **Reference** — task catalog, GE component catalog, modifier formula, glossary

## Built From

- [Official UE Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-for-unreal-engine)
- [UE 5.7 engine source code](https://www.unrealengine.com/en-US/ue-on-github) (136 public headers in the GameplayAbilities plugin)
- [tranek/GASDocumentation](https://github.com/tranek/GASDocumentation) — the original community GAS guide
- Hands-on experience building action RPG combat systems

## Tech Stack

- [Zensical](https://zensical.org) — static site generator (Material for MkDocs compatible)
- [Cloudflare Pages](https://pages.cloudflare.com) — hosting and deployment
- [uv](https://github.com/astral-sh/uv) — Python package management

## Development

```bash
# Install dependencies
uv pip install zensical

# Serve locally with hot reload
zensical serve

# Build the static site
zensical build
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on submitting changes.

## License

This documentation is provided as a community resource. It is not affiliated with or endorsed by Epic Games.
