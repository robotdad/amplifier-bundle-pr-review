# Amplifier PR Review Bundle

Comprehensive workflow for reviewing Pull Requests to Amplifier ecosystem repos, with support for single PRs and multi-PR interdependent changes.

## Installation

```bash
amplifier bundle add git+https://github.com/robotdad/amplifier-bundle-pr-review
```

## Quick Start

### Single PR Review

```bash
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-full.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65"}'
```

### Multi-PR Review (Related PRs Together)

When PRs across repos depend on each other (e.g., a bundle PR that requires changes in another bundle):

```bash
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-multi.yaml \
  context='{
    "pr_urls": [
      "https://github.com/microsoft/amplifier-bundle-modes/pull/3",
      "https://github.com/microsoft/amplifier-bundle-recipes/pull/19"
    ]
  }'
```

Or from within an Amplifier session:

```
> Run the pr-review-multi recipe with these PRs: modes#3 and recipes#19
```

## Available Recipes

| Recipe | Use Case |
|--------|----------|
| `pr-review-full.yaml` | Single PR to amplifier-app-cli or similar |
| `pr-review-multi.yaml` | Multiple related PRs that depend on each other |
| `pr-review-post-findings.yaml` | Post saved findings to GitHub PR |
| `pr-review-cleanup.yaml` | Restore official Amplifier CLI (if needed) |

## Single PR Review (`pr-review-full`)

Reviews a single PR with full code analysis and shadow environment testing.

### What It Does

| Phase | Description |
|-------|-------------|
| **Fetch & Clone** | Parse PR URL, fetch metadata, clone at exact commit |
| **Code Analysis** | Cross-repo contract check, code review, security audit, Python checks |
| **Shadow Testing** | Isolated container with PR code injected, smoke tests |
| **Final Report** | Synthesized findings with verdict |

### Context Options

| Option | Default | Description |
|--------|---------|-------------|
| `pr_url` | (required) | Full GitHub PR URL |
| `workspace_base` | `./pr-review` | Where to clone repos |
| `skip_shadow_tests` | `false` | Skip shadow environment tests |
| `skip_llm_smoke` | `false` | Skip LLM smoke tests (faster) |

### Why INJECT for CLI PRs?

The shadow tool's `local_sources` mechanism uses git URL rewriting, but `uv tool install` has a caching layer that can bypass these rewrites. For CLI PRs, we use `inject` to copy exact local files directly into the container.

## Multi-PR Review (`pr-review-multi`)

Reviews multiple related PRs together, testing them as an integrated set.

### When to Use

- Bundle A depends on changes in Bundle B
- Coordinated changes across multiple repos
- Testing that PRs work together before merging

### What It Does

| Phase | Description |
|-------|-------------|
| **Parse All PRs** | Validate and fetch metadata for all PR URLs |
| **Clone All Repos** | Clone each PR at its exact commit |
| **Generate Mount Plan** | Analyze bundle dependencies, determine install order |
| **Cross-Repo Analysis** | Check PRs work together, identify integration points |
| **Shadow Testing** | All PRs loaded via `local_sources`, bundles installed in dependency order |
| **Per-Repo Smoke Tests** | Run repo-specific tests if defined (forward-compatible) |
| **Final Report** | Combined review with merge order recommendation |

### Context Options

| Option | Default | Description |
|--------|---------|-------------|
| `pr_urls` | (required) | Array of GitHub PR URLs |
| `dependency_order` | (auto-detected) | Manual override for install order |
| `workspace_base` | `./pr-review-multi` | Where to clone repos |
| `skip_shadow_tests` | `false` | Skip shadow environment tests |
| `skip_llm_smoke` | `false` | Skip LLM smoke tests |

### The Mount Plan

The recipe analyzes `bundle.md` files to determine:
- Which PRs depend on which (via `includes:` references)
- What order to install bundles in the shadow environment
- Which git URLs need rewriting

### Why `local_sources` Works for Bundles

Unlike CLI installation (`uv tool install`), bundle installation (`amplifier bundle add`) uses `git clone` which **does respect** the URL rewrites that shadow's `local_sources` sets up. This allows testing multiple bundle PRs together.

### Per-Repo Smoke Tests (Future Pattern)

The recipe looks for repo-specific smoke tests at:

```
repo/
└── smoke-tests/
    └── smoke-test.yaml
```

If found, these are executed after standard smoke tests. This allows repos to define their own validation. Currently gracefully skipped if not present.

## Prerequisites

| Requirement | Purpose |
|-------------|---------|
| GitHub CLI (`gh`) | Fetch PR info, post comments |
| `gh auth login` | Authenticate with GitHub |
| Docker or Podman | Shadow environments |
| Provider credentials | LLM-based smoke tests (optional) |

## Workflow Diagrams

### Single PR Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  FETCH & CLONE                                                  │
│  - Parse PR URL, fetch metadata via gh CLI                      │
│  - Clone PR branch at exact commit                              │
│  - Clone amplifier-core for contract checking                   │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  CODE ANALYSIS                                                  │
│  - Cross-repo contract check (PR vs amplifier-core)             │
│  - Code review (zen-architect)                                  │
│  - Security audit (security-guardian)                           │
│  - Python quality checks (ruff, pyright)                        │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  SHADOW ENVIRONMENT TESTING                                     │
│  - Create isolated container                                    │
│  - INJECT PR code directly (exact file copy)                    │
│  - Install smoke-test bundle, then PR code with --force         │
│  - Run smoke tests, verify no version drift                     │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  FINAL REPORT                                                   │
│  - Synthesize findings, generate verdict                        │
│  - Save reports to workspace                                    │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-PR Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  PARSE & CLONE ALL PRs                                          │
│  - Validate all PR URLs                                         │
│  - Fetch metadata for each PR                                   │
│  - Clone all repos at exact commits                             │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  GENERATE MOUNT PLAN                                            │
│  - Analyze bundle.md dependencies                               │
│  - Determine installation order                                 │
│  - Identify cross-references between PRs                        │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  CROSS-REPO ANALYSIS                                            │
│  - Check PRs work together                                      │
│  - Identify integration points and potential conflicts          │
│  - Code review for each PR                                      │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  SHADOW ENVIRONMENT TESTING                                     │
│  - Create shadow with ALL repos via local_sources               │
│  - Install bundles in dependency order (git rewrites active)    │
│  - Run standard smoke tests                                     │
│  - Run per-repo smoke tests (if defined)                        │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  COMBINED REPORT                                                │
│  - Individual PR assessments                                    │
│  - Integration analysis                                         │
│  - Merge order recommendation                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Output Files

| Recipe | Output Files |
|--------|--------------|
| `pr-review-full` | `pr-review/pr-{number}-analysis.md`, `pr-review/pr-{number}-review.md` |
| `pr-review-multi` | `pr-review-multi/multi-pr-{numbers}-review.md` |

## Examples

```bash
# Single PR - full review
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-full.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65"}'

# Single PR - skip LLM tests (faster, CI-friendly)
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-full.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65", "skip_llm_smoke": true}'

# Multi-PR - test related bundle changes together
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-multi.yaml \
  context='{
    "pr_urls": [
      "https://github.com/microsoft/amplifier-bundle-modes/pull/3",
      "https://github.com/microsoft/amplifier-bundle-recipes/pull/19"
    ]
  }'

# Multi-PR - manual dependency order override
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-multi.yaml \
  context='{
    "pr_urls": ["...", "..."],
    "dependency_order": ["amplifier-bundle-modes", "amplifier-bundle-recipes"]
  }'

# Post findings to GitHub PR
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-post-findings.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65", "findings_file": "./pr-review/pr-65-review.md"}'
```

## Dependencies

This bundle includes:

- `amplifier-foundation` - Core agents (zen-architect, security-guardian, explorer)
- `amplifier-bundle-shadow` - Shadow environment support
- `amplifier-bundle-recipes` - Recipe execution
- `amplifier-bundle-python-dev` - Python code quality tools

## Troubleshooting

### "gh: command not found"

Install GitHub CLI: https://cli.github.com/

```bash
# macOS
brew install gh

# Ubuntu/Debian
sudo apt install gh

# Then authenticate
gh auth login
```

### Shadow environment fails

Check Docker/Podman is running:

```bash
docker ps  # or: podman ps
```

### LLM smoke tests skipped

Provider credentials aren't configured. Set environment variables:
- `ANTHROPIC_API_KEY` for Anthropic
- `OPENAI_API_KEY` for OpenAI

Or use `skip_llm_smoke: true` to skip these tests intentionally.

### Multi-PR dependency detection fails

If automatic dependency detection doesn't work correctly, use the `dependency_order` context option to manually specify the installation order.

### Review not posting to GitHub

Check gh authentication and repo permissions:

```bash
gh auth status
gh repo view microsoft/amplifier-app-cli
```

## License

MIT
