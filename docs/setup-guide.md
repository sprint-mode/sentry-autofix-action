# Setup Guide

Get Sentry auto-fix running in your repo in 5 minutes.

## Prerequisites

- A GitHub repo with a Sentry project tracking its errors
- [Sentry-GitHub integration](https://sentry.io/integrations/github/) installed
- An [Anthropic API key](https://console.anthropic.com/)
- A [Sentry Auth Token](https://sentry.io/settings/auth-tokens/) with scopes: `project:read`, `event:read`, `issue:read`

## Step 1: Add the workflow

Copy [`examples/consumer-workflow.yml`](../examples/consumer-workflow.yml) to `.github/workflows/sentry-autofix.yml` in your repo.

Or create it manually:

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

## Step 2: Add secrets

Go to your repo > Settings > Secrets and variables > Actions:

| Secret | Required | Source |
|--------|----------|--------|
| `ANTHROPIC_API_KEY` | Yes | [Anthropic Console](https://console.anthropic.com/) |
| `SENTRY_AUTH_TOKEN` | Yes | [Sentry Auth Tokens](https://sentry.io/settings/auth-tokens/) — scopes: `project:read`, `event:read`, `issue:read` |
| `SLACK_AUTOFIX_WEBHOOK` | No | Slack App > Incoming Webhooks |

## Step 3: Configure Sentry alerts

See [Sentry Alert Setup](sentry-alert-setup.md) for creating the alert rule that auto-creates GitHub Issues.

## Step 4: Test it

### Manual test
1. Go to your repo > Actions > "Sentry Auto-Fix"
2. Click "Run workflow"
3. Paste a Sentry issue URL
4. Watch the action run

### Automatic test
1. Trigger an error in your app that Sentry captures
2. The Sentry alert should create a GitHub Issue with label `sentry`
3. The action should trigger automatically

## Configuration options

| Input | Default | Description |
|-------|---------|-------------|
| `severity_filter` | `fatal,error` | Comma-separated severity levels to auto-fix |
| `max_turns` | `30` | Max Claude conversation turns |
| `base_branch` | `main` | Branch to create fix from |
| `model` | `claude-sonnet-4-6-20250514` | Claude model to use |
