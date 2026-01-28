# Amplifier PR Review Bundle

Comprehensive workflow for reviewing Pull Requests to amplifier-app-cli (and other Amplifier ecosystem repos).

## Installation

```bash
amplifier bundle add git+https://github.com/robotdad/amplifier-bundle-pr-review --app
```

## Quick Start

Run the PR review recipe directly:

```bash
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-full.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65"}'
```

Or from within an Amplifier session:

```
> Run the pr-review-full recipe with pr_url=https://github.com/microsoft/amplifier-app-cli/pull/65
```

## What This Bundle Does

The `pr-review-full` recipe performs a complete, deterministic PR review:

### Phase 1: Fetch & Clone
- Parses PR URL and fetches metadata from GitHub
- Clones PR branch to local workspace at the exact commit
- Clones `amplifier-core` for cross-repo contract checking

### Phase 2: Code Analysis
- **Cross-repo contract check** - Verifies PR doesn't break interfaces that `amplifier-core` depends on
- **Code review** - Architecture, patterns, philosophy alignment (zen-architect)
- **Security audit** - OWASP Top 10, injection, secrets (security-guardian)
- **Python checks** - Linting and type checking (ruff, pyright)

### Phase 3: Shadow Environment Testing
- Creates isolated container environment
- **Injects PR code directly** (no git cloning - copies exact local files)
- Installs smoke-test bundle first (may pull dependencies)
- **Installs PR code with --force** (ensures PR version is what runs)
- Verifies correct commit is installed
- Runs comprehensive smoke tests
- Verifies version stability (no drift during tests)

### Phase 4: Final Report
- Synthesizes all findings into GitHub PR comment format
- Provides verdict: APPROVE / REQUEST_CHANGES / COMMENT
- Saves reports to workspace

## Prerequisites

| Requirement | Purpose |
|-------------|---------|
| GitHub CLI (`gh`) | Fetch PR info, post comments |
| `gh auth login` | Authenticate with GitHub |
| Docker or Podman | Shadow environments |
| Provider credentials | LLM-based smoke tests (optional - skipped if not configured) |

## Context Options

| Option | Default | Description |
|--------|---------|-------------|
| `pr_url` | (required) | Full GitHub PR URL |
| `workspace_base` | `./pr-review` | Where to clone repos |
| `skip_shadow_tests` | `false` | Skip shadow environment tests |
| `skip_llm_smoke` | `false` | Skip LLM smoke tests (faster) |

## Available Recipes

| Recipe | Description |
|--------|-------------|
| `pr-review-full.yaml` | Complete automated review workflow |
| `pr-review-post-findings.yaml` | Post saved findings to GitHub PR |
| `pr-review-cleanup.yaml` | Restore official Amplifier CLI (if needed) |

### Examples

```bash
# Full review
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-full.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65"}'

# Skip LLM tests (faster, CI-friendly)
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-full.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65", "skip_llm_smoke": true}'

# Post findings to PR
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-post-findings.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65", "findings_file": "./pr-review/pr-65-review.md"}'
```

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: FETCH & CLONE                                         │
│  - Parse PR URL, fetch metadata via gh CLI                      │
│  - Clone PR branch at exact commit                              │
│  - Clone amplifier-core for contract checking                   │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2: CODE ANALYSIS                                         │
│  - Cross-repo contract check (PR vs amplifier-core)             │
│  - Code review (zen-architect)                                  │
│  - Security audit (security-guardian)                           │
│  - Python quality checks (ruff, pyright)                        │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3: SHADOW ENVIRONMENT TESTING                            │
│  - Create isolated container                                    │
│  - INJECT PR code directly (no git clone - exact file copy)     │
│  - Install smoke-test bundle (pulls dependencies)               │
│  - Install PR code with --force (overwrites bundle's version)   │
│  - Verify correct commit installed                              │
│  - Run smoke tests                                              │
│  - Verify no version drift                                      │
│  *** Host Amplifier stays running and unchanged ***             │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 4: FINAL REPORT                                          │
│  - Synthesize findings from all phases                          │
│  - Generate verdict (APPROVE / REQUEST_CHANGES / COMMENT)       │
│  - Save reports to workspace                                    │
└─────────────────────────────────────────────────────────────────┘
```

## Output Files

The `pr-review-full` recipe generates:

| File | Contents |
|------|----------|
| `pr-review/pr-{number}-analysis.md` | Code analysis before shadow tests |
| `pr-review/pr-{number}-review.md` | Final comprehensive review |

## Key Design Decisions

### Why INJECT Instead of Git Clone?

The shadow tool's `local_sources` mechanism has a bug where it fetches from remote origin instead of using local content. We use `inject` to copy the exact local files directly into the container - no git, no Gitea, no remote fetching.

### Why Bundle First, Then PR Code?

The smoke-test bundle includes `amplifier-foundation@main` which can pull updated dependencies that overwrite the installed amplifier version. By installing the bundle first, then the PR code with `--force`, we ensure the PR code is what actually runs during tests.

### Why Cross-Repo Contract Checking?

PRs to `amplifier-app-cli` can break contracts that `amplifier-core` depends on. The recipe clones core and checks for interface mismatches that would cause runtime failures - exactly the kind of bug that's hard to catch without this check.

## Dependencies

This bundle includes:

- `amplifier-foundation` - Core agents (zen-architect, security-guardian, explorer)
- `amplifier-bundle-shadow` - Shadow environment support
- `amplifier-bundle-recipes` - Recipe execution
- `amplifier-bundle-python-dev` - Python code quality tools

The smoke-test bundle is installed inside the shadow environment during testing.

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

### Review not posting to GitHub

Check gh authentication and repo permissions:
```bash
gh auth status
gh repo view microsoft/amplifier-app-cli
```

## License

MIT
