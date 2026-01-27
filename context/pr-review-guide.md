# Amplifier CLI PR Review Guide

A step-by-step guide for reviewing and verifying Pull Requests to [amplifier-app-cli](https://github.com/microsoft/amplifier-app-cli).

---

## Overview

PR verification follows this workflow:

1. **Fetch & Analyze** - Get PR info, clone code, run automated analysis
2. **Shadow Testing** - Test in isolated environment (no host changes)
3. **Manual Installation** - Install PR version on host for hands-on testing
4. **Verify Installation** - Confirm correct version is installed
5. **Run Smoke Tests** - Comprehensive sanity check
6. **Post Findings** - Share review results on GitHub PR
7. **Clean Up** - Restore official release

---

## Automated Analysis Phase

### What Gets Analyzed

| Analysis | Agent | What It Checks |
|----------|-------|----------------|
| Code Quality | `zen-architect` | Architecture, patterns, complexity, philosophy alignment |
| Security | `security-guardian` | OWASP Top 10, secrets, injection, auth issues |
| Python Quality | `python-dev` | ruff (format/lint), pyright (types) |
| Smoke Tests | `smoke-tester` | CLI commands, providers, sessions, tools |

### Shadow Environment Benefits

- **No host changes** - Your installed Amplifier stays untouched
- **Isolated testing** - Test PR code without risk
- **Reproducible** - Fresh environment every time
- **Automated** - Can run full test suite hands-off

---

## Manual Installation Phase

After automated analysis, you may want to test the PR in your actual environment.

### Install the PR

**From a Remote Fork (GitHub PR):**
```bash
uv tool install --force git+https://github.com/<username>/amplifier-app-cli@<branch>
```

**From a Local Checkout:**
```bash
cd /path/to/amplifier-app-cli
uv tool uninstall amplifier-app-cli || true
uv tool install -e .
```

**Clean Installation (Recommended):**
```bash
# Remove both to ensure clean slate
uv tool uninstall amplifier || true
uv tool uninstall amplifier-app-cli || true

# Install from PR source
uv tool install git+https://github.com/<username>/amplifier-app-cli@<branch>
```

### Verify Installation

```bash
amplifier --version
```

The version includes the commit hash:
```
amplifier, version 2026.01.22-879a005
                             ^^^^^^^ commit from the PR
```

**Cross-check with PR:**
```bash
# Get expected commit from PR
gh pr view <pr-number> --json headRefOid --jq '.headRefOid[:7]'

# Compare with installed version
amplifier --version | grep -oE '[a-f0-9]{7}$'
```

---

## Module Source Overrides

If the PR depends on changes in other modules, configure source overrides.

**Via Settings File (`.amplifier/settings.yaml`):**
```yaml
bundle:
  active: amplifier-dev

sources:
  modules:
    tool-filesystem: /path/to/amplifier-module-tool-filesystem
```

**Via CLI:**
```bash
amplifier source add --project tool-filesystem /path/to/module
```

**Verify:**
```bash
amplifier source list
amplifier source show <module-name>
```

---

## Running Smoke Tests

**Full suite (includes LLM tests):**
```bash
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@smoke-test:recipes/smoke-test.yaml
```

**CLI-only tests (faster, no API calls):**
```bash
amplifier tool invoke recipes \
  operation=execute \
  recipe_path=@smoke-test:recipes/smoke-test.yaml \
  context='{"skip_llm": true}'
```

---

## Posting Findings to GitHub

After review, post findings to the PR:

```bash
gh pr comment <pr-number> --body-file review-findings.md
```

Or use the `pr-review-post-findings` recipe which formats and posts automatically.

---

## Cleanup / Restore Official Release

**Restore official CLI:**
```bash
uv tool install --force git+https://github.com/microsoft/amplifier-app-cli

# Verify restoration
amplifier --version
```

**Remove module overrides:**
```bash
amplifier source remove <module-name>
# Or manually edit .amplifier/settings.yaml
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Install from fork | `uv tool install --force git+https://github.com/<user>/amplifier-app-cli@<branch>` |
| Install from local | `uv tool install -e /path/to/amplifier-app-cli` |
| Verify version | `amplifier --version` |
| Get PR commit | `gh pr view <pr> --json headRefOid --jq '.headRefOid[:7]'` |
| Add module override | `amplifier source add --project <module> /path/to/module` |
| List overrides | `amplifier source list` |
| Run smoke tests | `amplifier tool invoke recipes operation=execute recipe_path=@smoke-test:recipes/smoke-test.yaml` |
| Post PR comment | `gh pr comment <pr> --body-file findings.md` |
| Restore official | `uv tool install --force git+https://github.com/microsoft/amplifier-app-cli` |

---

## Troubleshooting

### Wrong Version After Install

The `amplifier` umbrella may cache `amplifier-app-cli`. Uninstall both:

```bash
uv tool uninstall amplifier
uv tool uninstall amplifier-app-cli
uv tool install git+https://github.com/<user>/amplifier-app-cli@<branch>
```

### Module Override Not Taking Effect

1. Verify path is correct: `amplifier source show <module>`
2. Check settings file syntax: `cat .amplifier/settings.yaml`
3. Ensure you're in the right project directory

### Smoke Tests Fail

- **LLM tests skipped**: Check provider credentials (`ANTHROPIC_API_KEY`, etc.)
- **Session CRUD fails**: Try `amplifier session cleanup --force`
- **Agent delegation fails**: Verify `amplifier agents list` shows `foundation:explorer`

---

## PR Review Checklist

- [ ] Automated code review completed
- [ ] Automated security audit completed
- [ ] Shadow environment smoke tests pass
- [ ] PR branch installs cleanly (if manual testing)
- [ ] Version shows correct commit hash
- [ ] Dependent module overrides configured (if applicable)
- [ ] PR-specific functionality works as described
- [ ] No regressions in existing functionality
- [ ] Findings posted to PR
- [ ] Restored to official release after testing
