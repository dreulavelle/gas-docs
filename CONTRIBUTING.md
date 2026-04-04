# Contributing

Thanks for your interest in improving the GAS Bible! This guide covers everything you need to get started.

## Quick Start

1. **Fork** the repository and **clone** your fork
2. **Create a branch** from `master` (see [branch naming](#branch-naming) below)
3. **Make your changes** — all content lives in `docs/` as Markdown files
4. **Test locally** — run `zensical serve` and verify in your browser
5. **Submit a PR** — describe what you changed and why

## What We're Looking For

| Contribution | Process |
|---|---|
| **Typos, broken links, inaccurate code** | Just open a PR — no issue needed |
| **New examples, patterns, or pages** | Open an issue first to discuss scope and placement |
| **CSS, theme, or layout changes** | Open an issue describing the problem and proposed solution |

## Local Development

### Prerequisites

You'll need **Python 3.12+** and **uv** (a fast Python package manager).

- Python: [python.org/downloads](https://www.python.org/downloads/)
- uv: [docs.astral.sh/uv](https://docs.astral.sh/uv/)

```bash
# Install uv
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### Setup and Serve

```bash
git clone https://github.com/YOUR_USERNAME/gas-docs.git
cd gas-docs

# Install Zensical — https://zensical.org/docs/get-started/
uv pip install zensical

# Serve locally with hot reload (http://127.0.0.1:8000)
zensical serve
```

Save any `.md` file and the site rebuilds automatically — just refresh your browser.

To build without serving: `zensical build` (output goes to `site/`).

## Content Guidelines

### Writing Style

- **Conversational but precise** — explain things clearly without being academic
- **Non-prescriptive** — teach how to think about GAS, not just what to type
- **All skill levels** — beginners should be able to follow, advanced users should find depth
- **Show both Blueprint and C++** where applicable — use tabbed code blocks

### Code Accuracy

- C++ code should compile against **UE 5.7**
- Reference the engine source at `Engine/Plugins/Runtime/GameplayAbilities/` for class names, function signatures, and enum values
- If you're unsure about an API, note it in the PR and we'll verify

### Page Structure

Follow the conventions established by existing pages:

- **Frontmatter** with `title:` and `description:`
- **Clear heading hierarchy** — don't skip levels
- **Cross-links** to related pages using relative paths: `[link text](../section/page.md)`
- **Mermaid flowcharts** for ability event graphs (see existing examples for the color convention)
- **Admonitions** for tips, warnings, and important callouts
- **"Related Pages"** section at the bottom where appropriate

## Commits and Branches

### Commit Messages

We use [Conventional Commits](https://www.conventionalcommits.org/). Keep them short and descriptive:

```
feat: add passive aura example
fix: broken link to dodge-roll in glossary
docs: expand local dev setup in CONTRIBUTING
style: adjust Mermaid node colors for light mode
chore: remove stale branch protection config
```

### Branch Naming

| Prefix | Use for |
|---|---|
| `feat/` | New content or features |
| `fix/` | Bug fixes, broken links, inaccurate content |
| `docs/` | Meta-documentation (README, CONTRIBUTING) |
| `style/` | CSS, theme, layout changes |
| `chore/` | Config, CI, tooling |

## Questions?

Open an [issue](https://github.com/dreulavelle/gas-docs/issues) — happy to help.
