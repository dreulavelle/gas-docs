# Contributing

Thanks for your interest in improving the GAS Bible! Here's how to contribute.

## Types of Contributions

### Content fixes
Typos, inaccurate code examples, broken links, outdated information — these are always welcome. Open a PR with the fix.

### New content
New examples, patterns, or deep-dive sections — open an issue first to discuss scope and placement before writing. This helps avoid duplicate work and ensures the content fits the site's structure.

### Design and UX
CSS improvements, navigation changes, accessibility fixes — open an issue describing the problem and proposed solution.

## How to Submit Changes

1. **Fork** the repository
2. **Create a branch** from `master` (`feat/your-change`, `fix/your-fix`, `docs/your-update`)
3. **Make your changes** in `docs/` (all content is Markdown)
4. **Test locally** — run `zensical serve` and verify your changes look correct
5. **Submit a PR** — describe what you changed and why

## Local Development

### Prerequisites

- **Python 3.12+** — [python.org](https://www.python.org/downloads/)
- **uv** (recommended) — fast Python package manager: [docs.astral.sh/uv](https://docs.astral.sh/uv/)
  ```bash
  # Install uv (macOS/Linux)
  curl -LsSf https://astral.sh/uv/install.sh | sh

  # Install uv (Windows)
  powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
  ```

### Setup

```bash
# Clone your fork
git clone https://github.com/YOUR_USERNAME/gas-docs.git
cd gas-docs

# Install Zensical (the site generator)
uv pip install zensical
```

### Running Locally

```bash
# Serve with hot reload — opens at http://127.0.0.1:8000
# Changes to any .md file rebuild automatically
zensical serve

# Or just build once (output goes to site/)
zensical build
```

!!! tip
    `zensical serve` watches for file changes and rebuilds automatically. Just save your markdown file and refresh the browser — no restart needed.

## Content Guidelines

### Writing style
- **Conversational but precise** — explain things clearly without being academic
- **Non-prescriptive** — teach how to think about GAS, not just what to type
- **All skill levels** — beginners should be able to follow, advanced users should find depth

### Formatting
- Use [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) features: admonitions, tabs, code blocks, tables
- Code examples should show **both Blueprint and C++** where applicable
- Use Mermaid flowcharts for ability event graphs (see existing examples for the color convention)
- Internal links use relative paths: `[link text](../section/page.md)`

### Code accuracy
- C++ code should compile against UE 5.7
- Reference the engine source at `Engine/Plugins/Runtime/GameplayAbilities/` for class names, function signatures, and enum values
- If you're unsure about an API, note it in the PR and we'll verify

### Structure
Pages should follow the conventions established by existing content:
- Frontmatter with `title:` and `description:`
- Clear heading hierarchy
- Cross-links to related pages
- "Related Pages" section at the bottom where appropriate

## Branch Naming

| Prefix | Use for |
|---|---|
| `feat/` | New content or features |
| `fix/` | Bug fixes, broken links, inaccurate content |
| `docs/` | Meta-documentation (README, CONTRIBUTING) |
| `style/` | CSS, theme, layout changes |
| `chore/` | Config, CI, tooling |

## Questions?

Open an issue — we're happy to help.
