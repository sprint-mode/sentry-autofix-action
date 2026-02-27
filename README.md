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

If you use [Claude Code](https://claude.ai/claude-code), you can set up everything automatically with a single command.

### Install the skill

```bash
# From your project's root directory:
mkdir -p .claude/skills
curl -o .claude/skills/setup-sentry-autofix.md \
  https://raw.githubusercontent.com/sprint-mode/sentry-autofix-action/main/skills/setup-sentry-autofix.md
```

### Run it

Open Claude Code in your repo and type:

```
/setup-sentry-autofix
```

Claude will:
1. Create the `.github/workflows/sentry-autofix.yml` workflow
2. Check which secrets you have and guide you to create the missing ones
3. Walk you through the Sentry Alert Rule configuration
4. Offer to run a manual test with a real Sentry issue

### What secrets does it need?

| Secret | Required | What it does | Cost |
|--------|----------|-------------|------|
| `ANTHROPIC_API_KEY` | Yes | Authenticates Claude Code with the Anthropic API | ~$0.50–$5 per auto-fix run |
| `SENTRY_AUTH_TOKEN` | Yes | Lets Claude read your Sentry issues and stacktraces | Free (Sentry API) |
| `SLACK_AUTOFIX_WEBHOOK` | No | Sends Slack notifications when PRs are created | Free |

Get your Anthropic API key at [console.anthropic.com](https://console.anthropic.com/) and your Sentry token at [sentry.io/settings/auth-tokens](https://sentry.io/settings/auth-tokens/) (scopes: `project:read`, `event:read`, `issue:read`).

## Documentation

- [Setup Guide](docs/setup-guide.md)
- [Sentry Alert Setup](docs/sentry-alert-setup.md)
- [Troubleshooting](docs/troubleshooting.md)

## License

MIT
