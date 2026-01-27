# Amplifier PR Review Bundle

Comprehensive workflow for reviewing Pull Requests to amplifier-app-cli (and other Amplifier ecosystem repos).

## Installation

```bash
amplifier bundle add /path/to/amplifier-bundle-pr-review --app
```

Or from a git repository (once published):
```bash
amplifier bundle add git+https://github.com/yourorg/amplifier-bundle-pr-review --app
```

## Quick Start

Start Amplifier and ask to review a PR:

```
amplifier

> Help me review https://github.com/microsoft/amplifier-app-cli/pull/65
```

The PR reviewer agent will guide you through the entire process.

## What This Bundle Does

### Automated Analysis (Shadow Environment)

1. **Fetches PR information** from GitHub URL using `gh` CLI
2. **Clones the PR branch** to a local workspace
3. **Creates a shadow environment** with the PR code
4. **Installs amplifier** inside the shadow (uses PR code)
5. **Verifies the installed version** matches the PR commit
6. **Runs smoke tests** inside the shadow
7. **Performs code review** using `zen-architect` agent
8. **Performs security audit** using `security-guardian` agent
9. **Runs Python checks** (ruff, pyright) on changed files
10. **Generates comprehensive review report**

### Manual Testing Guidance

After automated analysis, the bundle provides:

- Installation commands for testing on your host
- Version verification steps
- Specific tests to run based on PR changes
- Smoke test execution guidance

### Post-Review Actions

- **Post findings** to GitHub PR as a comment
- **Cleanup** to restore official Amplifier installation

## Prerequisites

| Requirement | Purpose |
|-------------|---------|
| GitHub CLI (`gh`) | Fetch PR info, post comments |
| `gh auth login` | Authenticate with GitHub |
| Docker or Podman | Shadow environments |
| Provider credentials | LLM-based smoke tests (optional) |

## Available Recipes

| Recipe | Description |
|--------|-------------|
| `pr-review-full.yaml` | Complete automated review workflow |
| `pr-review-post-findings.yaml` | Post saved findings to GitHub PR |
| `pr-review-cleanup.yaml` | Restore official Amplifier CLI |

### Running Recipes Directly

```bash
# Full review
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-full.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65"}'

# Post findings
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-post-findings.yaml \
  context='{"pr_url": "https://github.com/microsoft/amplifier-app-cli/pull/65", "findings_file": "./pr-review/pr-65-review.md"}'

# Cleanup
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@pr-review:recipes/pr-review-cleanup.yaml
```

## Available Agents

| Agent | Description |
|-------|-------------|
| `pr-review:pr-reviewer` | Interactive PR review guide |

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1: FETCH & ANALYZE                                   │
│  - Parse PR URL, fetch metadata                             │
│  - Clone PR branch                                          │
│  - Summarize changes                                        │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 2: SHADOW TESTING (automated, no host changes)       │
│  - Create shadow with PR code                               │
│  - Install & verify amplifier                               │
│  - Run smoke tests                                          │
│  - Code review (zen-architect)                              │
│  - Security audit (security-guardian)                       │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 3: MANUAL TESTING (optional, requires exit)          │
│  - Exit current Amplifier session                           │
│  - Install PR version on host                               │
│  - Run smoke tests in new session                           │
│  - Manual verification                                      │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 4: POST & CLEANUP                                    │
│  - Post findings to GitHub PR                               │
│  - Restore official Amplifier                               │
└─────────────────────────────────────────────────────────────┘
```

## Output Files

The `pr-review-full` recipe generates:

| File | Contents |
|------|----------|
| `pr-review/pr-{number}-review.md` | Comprehensive review findings |
| `pr-review/pr-{number}-install-guide.md` | Manual installation steps |

## Important: Self-Modification Boundary

**Amplifier cannot reinstall itself while running.** 

For manual testing on your host (beyond shadow testing), you must:

1. Exit the current Amplifier session
2. Run the provided installation commands
3. Start a new Amplifier session
4. Run smoke tests
5. Return to cleanup/post findings

The shadow environment testing avoids this limitation by testing in an isolated container.

## Dependencies

This bundle includes:

- `amplifier-foundation` - Core agents (zen-architect, security-guardian, explorer)
- `amplifier-bundle-shadow` - Shadow environment support
- `amplifier-bundle-smoke-test` - Smoke testing capabilities
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

### Commit mismatch warning

The installed version may differ from PR commit if:
- UV cached a previous version
- The PR was updated after cloning

Solution: Clean install in shadow or use `--force` flag.

### Review not posting to GitHub

Check gh authentication and repo permissions:
```bash
gh auth status
gh repo view microsoft/amplifier-app-cli
```

## License

MIT
