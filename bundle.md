---
bundle:
  name: pr-review
  version: 1.0.0
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

# PR Review specific agents
agents:
  pr-reviewer:
    path: pr-review:agents/pr-reviewer.md
---

# Amplifier PR Review Bundle

Comprehensive workflow for reviewing Pull Requests to amplifier-app-cli (and other Amplifier ecosystem repos).

## Quick Start

```
"Help me review the PR at https://github.com/microsoft/amplifier-app-cli/pull/65"
```

The PR reviewer agent will guide you through the entire process.

## What This Bundle Does

1. **Fetches PR information** from GitHub URL
2. **Clones the PR branch** and any needed Amplifier repos
3. **Runs automated analysis** in shadow environment:
   - Code quality review (zen-architect)
   - Security audit (security-guardian)
   - Python code checks (ruff, pyright)
   - Smoke tests with PR code
4. **Generates installation guide** for manual testing
5. **Saves review findings** for later posting
6. **Posts findings to PR** via GitHub CLI
7. **Provides cleanup instructions** to restore official install

## Available Recipes

| Recipe | Purpose |
|--------|---------|
| `pr-review-full.yaml` | Complete automated review workflow |
| `pr-review-shadow-test.yaml` | Shadow environment testing only |
| `pr-review-post-findings.yaml` | Post saved findings to GitHub PR |
| `pr-review-cleanup.yaml` | Restore official Amplifier install |

## Available Agents

| Agent | Purpose |
|-------|---------|
| `pr-review:pr-reviewer` | Interactive PR review guide |

## Context Files

@pr-review:context/pr-review-guide.md

---

@foundation:context/shared/common-system-base.md
