---
name: setup-sentry-autofix
description: Set up Sentry auto-fix in the current repo. Creates the GitHub Actions workflow, checks for required secrets, and provides Sentry alert configuration instructions.
---

# Setup Sentry Auto-Fix

You are setting up the sentry-autofix-action in the user's current repository.

## Steps

### 1. Create the workflow file

Create `.github/workflows/sentry-autofix.yml` with this content:

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

### 2. Check for required secrets

Run:
```bash
gh secret list
```

Check if these secrets exist:
- `ANTHROPIC_API_KEY` — required
- `SENTRY_AUTH_TOKEN` — required
- `SLACK_AUTOFIX_WEBHOOK` — optional

For any missing required secrets, tell the user:
- **ANTHROPIC_API_KEY**: Get from https://console.anthropic.com/
- **SENTRY_AUTH_TOKEN**: Get from https://sentry.io/settings/auth-tokens/ with scopes: `project:read`, `event:read`, `issue:read`

### 3. Check Sentry-GitHub integration

Tell the user to verify:
1. Go to https://sentry.io/settings/integrations/github/
2. Ensure their GitHub org is connected
3. Ensure the current repo is linked to a Sentry project

### 4. Provide Sentry Alert setup instructions

Tell the user to create an Alert Rule:
1. Go to Sentry > project > Alerts > Create Alert
2. Type: Issues (Errors)
3. When: A new issue is created
4. Filter: Level is `error` or `fatal`
5. Action: Create a GitHub Issue in this repo with label `sentry`

### 5. Offer a test

Ask the user if they want to test with a known Sentry issue URL. If yes:
```bash
gh workflow run sentry-autofix.yml -f sentry_issue_url="<URL>"
```

### 6. Commit the workflow

```bash
git add .github/workflows/sentry-autofix.yml
git commit -m "ci: add Sentry auto-fix workflow"
```
