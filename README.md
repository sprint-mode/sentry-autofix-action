# Sentry Auto-Fix Action

Automatically fix Sentry bugs with Claude Code. When a new error appears in Sentry, this action analyzes the stacktrace, diagnoses the root cause, implements a fix, and creates a PR — ready for human review.

## How it works

```
Sentry Error → GitHub Issue (auto) → Claude Code analyzes → PR created → Human reviews & merges
```

1. **Sentry** detects a new error and creates a GitHub Issue (via native integration)
2. **GitHub Actions** triggers this workflow
3. **Claude Code** + **Sentry MCP** fetches error details, navigates your codebase, and implements a fix
4. A **PR** is created with full diagnosis, fix description, and risk assessment
5. A **human** reviews and merges

## Quick Start

### 1. Add the workflow to your repo

Create `.github/workflows/sentry-autofix.yml`:

````yaml
name: Sentry Auto-Fix

on:
  issues:
    types: [opened]
  workflow_dispatch:
    inputs:
      sentry_issue_url:
        description: 'Sentry issue URL'
        required: true
        type: string

jobs:
  autofix:
    if: >
      github.event_name == 'workflow_dispatch' ||
      contains(github.event.issue.labels.*.name, 'sentry')
    uses: sprint-mode/sentry-autofix-action/.github/workflows/sentry-autofix.yml@main
    with:
      sentry_issue_url: ${{ inputs.sentry_issue_url || '' }}
      severity_filter: 'fatal,error'
      base_branch: 'main'
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_AUTOFIX_WEBHOOK }}
````

### 2. Add secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes | Your Anthropic API key |
| `SENTRY_AUTH_TOKEN` | Yes | Sentry token with `project:read`, `event:read`, `issue:read` |
| `SLACK_AUTOFIX_WEBHOOK` | No | Slack webhook for notifications |

### 3. Configure Sentry

Create a Sentry Alert Rule to auto-create GitHub Issues with label `sentry` for new errors. See [Sentry Alert Setup](docs/sentry-alert-setup.md).

## Configuration

| Input | Default | Description |
|-------|---------|-------------|
| `severity_filter` | `fatal,error` | Severity levels to auto-fix |
| `max_turns` | `30` | Max Claude conversation turns |
| `base_branch` | `main` | Branch to create fix from |
| `model` | `claude-sonnet-4-6-20250514` | Claude model |

## Manual trigger

You can also trigger manually from the Actions tab or via CLI:

```bash
gh workflow run sentry-autofix.yml \
  -f sentry_issue_url="https://your-org.sentry.io/issues/12345/"
```

## Auto-Setup with Claude Code

If you use [Claude Code](https://claude.ai/claude-code), copy the `skills/setup-sentry-autofix.md` skill to your Claude Code skills directory. Then run:

```
/setup-sentry-autofix
```

Claude will create the workflow, check secrets, and guide you through Sentry configuration.

## Documentation

- [Setup Guide](docs/setup-guide.md)
- [Sentry Alert Setup](docs/sentry-alert-setup.md)
- [Troubleshooting](docs/troubleshooting.md)

## License

MIT
