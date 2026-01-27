---
bundle:
  name: pr-review
  version: 2.0.0
  description: Comprehensive PR review workflow for amplifier-app-cli submissions

includes:
  # Core foundation for agents and behaviors
  - bundle: git+https://github.com/microsoft/amplifier-foundation@main
  
  # Shadow environments for isolated testing
  - bundle: git+https://github.com/microsoft/amplifier-bundle-shadow@main
  
  # Smoke testing capabilities
  - bundle: git+https://github.com/samueljklee/amplifier-bundle-smoke-test@main
  
  # Recipe execution
  - bundle: git+https://github.com/microsoft/amplifier-bundle-recipes@main
  
  # Python development tools
  - bundle: git+https://github.com/microsoft/amplifier-bundle-python-dev@main

# Additional tools needed for PR operations
tools:
  - module: tool-bash
    source: git+https://github.com/microsoft/amplifier-module-tool-bash@main
  - module: tool-filesystem
    source: git+https://github.com/microsoft/amplifier-module-tool-filesystem@main
  - module: tool-search
    source: git+https://github.com/microsoft/amplifier-module-tool-search@main
  - module: tool-web
    source: git+https://github.com/microsoft/amplifier-module-tool-web@main
---

# Amplifier PR Review Bundle

Comprehensive workflow for reviewing Pull Requests to amplifier-app-cli (and other Amplifier ecosystem repos).

## Quick Start

```bash
mkdir pr-65 && cd pr-65
amplifier
```

Then in the session:
```
Run the pr-review-full recipe with pr_url=https://github.com/microsoft/amplifier-app-cli/pull/65
```

Or directly via CLI:
```bash
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-full.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65"}'
```

## What This Bundle Does

The `pr-review-full` recipe performs a complete, deterministic PR review:

1. **Fetches PR information** from GitHub URL
2. **Clones the PR branch** to local workspace
3. **Clones amplifier-core** for cross-repo contract checking
4. **Cross-repo analysis** - checks if PR breaks any contracts core depends on
5. **Creates shadow environment** with LOCAL PR code (not GitHub - avoids UV cache issues)
6. **Installs Amplifier from local clone** inside shadow
7. **Runs full smoke test suite** inside shadow
8. **Code quality review** (zen-architect)
9. **Security audit** (security-guardian)
10. **Python checks** (ruff, pyright)
11. **Generates review report** and optional manual install guide

## Why Recipe-Only (No Agent)

Previous versions had a `pr-reviewer` agent that was supposed to invoke the recipe.
The agent repeatedly went rogue and did manual ad-hoc work instead.

Now there's just the recipe - deterministic, same steps every time.

## Available Recipes

| Recipe | Purpose |
|--------|---------|
| `pr-review-full.yaml` | Complete automated review workflow |

## Context Options

| Option | Default | Description |
|--------|---------|-------------|
| `pr_url` | (required) | Full GitHub PR URL |
| `workspace_dir` | `./pr-review` | Where to clone repos |
| `skip_shadow_tests` | `false` | Skip shadow environment tests |
| `skip_llm_smoke` | `false` | Skip LLM smoke tests (providers that aren't configured will skip anyway) |

## Prerequisites

- **GitHub CLI (gh)**: Must be installed and authenticated
- **Docker/Podman**: Required for shadow environments
- **Provider credentials**: For LLM-based smoke tests (optional - unconfigured providers skip)

## Context Files

@pr-review:context/pr-review-guide.md

---

@foundation:context/shared/common-system-base.md
