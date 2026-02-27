---
name: setup-sentry-autofix
description: Set up Sentry auto-fix in the current repo. Creates the GitHub Actions workflow, configures required secrets (Anthropic API key, Sentry token), sets up Sentry alerts, and tests the full pipeline. Use when the user wants to connect Sentry error auto-healing to their repo.
---

# Setup Sentry Auto-Fix

You are setting up [sentry-autofix-action](https://github.com/sprint-mode/sentry-autofix-action) in the user's current repository. This action uses Claude Code + Sentry MCP to automatically analyze Sentry errors, diagnose root causes, implement fixes, and create PRs.

## How it works

```
Sentry Error → Sentry Alert creates GitHub Issue (label: sentry)
  → GitHub Action triggers → Claude Code + Sentry MCP
  → Analyzes error, reads stacktrace, navigates code
  → Creates branch + fix + PR → Human reviews and merges
```

Claude Code runs inside a GitHub Actions runner. It uses an **Anthropic API key** (paid, usage-based) to call the Claude API. Each auto-fix run costs approximately $0.50–$5 USD depending on bug complexity.

## Setup Steps

Follow these steps in order. For each step, actually perform the action (create files, run commands) — don't just describe what to do.

### Step 1: Detect repo context

Before anything, figure out:

```bash
# What's the repo name?
gh repo view --json nameWithOwner -q .nameWithOwner

# What's the default branch?
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main"

# Does .github/workflows/ exist?
ls .github/workflows/ 2>/dev/null || echo "No workflows directory yet"
```

Use the repo name and default branch in the workflow file below.

### Step 2: Create the workflow file

Create `.github/workflows/sentry-autofix.yml`:

````yaml
name: Sentry Auto-Fix

on:
  # Automatic: Sentry creates a GitHub Issue with label "sentry"
  issues:
    types: [opened]

  # Manual: dev triggers from Actions tab or CLI with a Sentry URL
  workflow_dispatch:
    inputs:
      sentry_issue_url:
        description: 'Sentry issue URL (e.g., https://your-org.sentry.io/issues/12345/)'
        required: true
        type: string

jobs:
  autofix:
    # Only run for Sentry-created issues (auto) or manual dispatch
    if: >
      github.event_name == 'workflow_dispatch' ||
      contains(github.event.issue.labels.*.name, 'sentry')
    uses: sprint-mode/sentry-autofix-action/.github/workflows/sentry-autofix.yml@main
    with:
      sentry_issue_url: ${{ inputs.sentry_issue_url || '' }}
      severity_filter: 'fatal,error'       # Only auto-fix errors and fatals (skip warnings/info)
      base_branch: 'main'                  # ← change if your default branch is different
      max_turns: 30                        # Increase for complex bugs (costs more)
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_AUTOFIX_WEBHOOK }}  # Optional
````

Adjust `base_branch` to match the default branch detected in Step 1.

### Step 3: Check and configure required secrets

Run this to see what secrets already exist:

```bash
gh secret list
```

The workflow needs these GitHub repo secrets:

#### ANTHROPIC_API_KEY (required)

This is how Claude Code authenticates with the Anthropic API. It's a **paid API key** — each auto-fix run uses Claude API credits.

**How to get it:**
1. Go to https://console.anthropic.com/
2. Sign up or log in
3. Go to Settings → API Keys → Create Key
4. Copy the key (starts with `sk-ant-...`)

**How to set it:**
```bash
# Option A: Set interactively (prompts for value)
gh secret set ANTHROPIC_API_KEY

# Option B: Set from clipboard (macOS)
pbpaste | gh secret set ANTHROPIC_API_KEY
```

**Cost:** ~$0.50–$5 USD per auto-fix run. Model `claude-sonnet-4-6` is the default (good balance of quality/cost). You can change to `claude-opus-4-6` for harder bugs (more expensive) or `claude-haiku-4-5` for cheaper runs (less capable).

#### SENTRY_AUTH_TOKEN (required)

This lets Claude read your Sentry issues, stacktraces, and breadcrumbs via the Sentry MCP server.

**How to get it:**
1. Go to https://sentry.io/settings/auth-tokens/
2. Click "Create New Token"
3. Required scopes: `project:read`, `event:read`, `issue:read`
4. Copy the token

**How to set it:**
```bash
gh secret set SENTRY_AUTH_TOKEN
```

#### SLACK_AUTOFIX_WEBHOOK (optional)

If set, the action sends a Slack notification when a fix PR is created or when auto-fix fails.

**How to get it:**
1. Go to your Slack App → Incoming Webhooks
2. Create a new webhook for your desired channel
3. Copy the webhook URL

**How to set it:**
```bash
gh secret set SLACK_AUTOFIX_WEBHOOK
```

After setting secrets, verify:
```bash
gh secret list
```

You should see at least `ANTHROPIC_API_KEY` and `SENTRY_AUTH_TOKEN` listed.

### Step 4: Verify Sentry-GitHub integration

The action relies on Sentry's native GitHub integration to create Issues automatically.

Tell the user to check:

1. **Sentry side:** Go to https://sentry.io/settings/integrations/github/
   - Their GitHub org must be connected
   - The current repo must be linked to a Sentry project

2. **GitHub side:** Go to repo Settings → Integrations
   - Sentry should appear as an installed integration

If the integration isn't set up yet, guide them:
- In Sentry: Settings → Integrations → GitHub → Add Installation → select their org
- Link the relevant Sentry project to this GitHub repo

### Step 5: Create Sentry Alert Rule

This is the trigger: when Sentry detects a new error, it creates a GitHub Issue automatically.

Tell the user to do this in the Sentry UI:

1. Go to **Sentry** → select the project → **Alerts** → **Create Alert**
2. Choose **Issues (Errors)** as the alert type
3. Configure:
   - **When:** A new issue is created
   - **Filter (recommended):** The issue's level is equal to `error` OR `fatal`
   - **Then:** Create a GitHub Issue
     - **Repository:** Select this repo (e.g., `sprint-mode/osma-api`)
     - **Assignee:** Leave empty
     - **Labels:** `sentry` ← **this is critical** — the workflow filters on this label
4. **Name:** "Auto-fix: Create GitHub Issue for new errors"
5. Click **Save Rule**

**Important:** The label must be exactly `sentry` (lowercase). The GitHub Action checks `contains(github.event.issue.labels.*.name, 'sentry')`.

### Step 6: Commit the workflow

```bash
git add .github/workflows/sentry-autofix.yml
git commit -m "ci: add Sentry auto-fix workflow

Uses sprint-mode/sentry-autofix-action to automatically analyze Sentry
errors with Claude Code and create fix PRs."
```

### Step 7: Test it

#### Manual test (recommended first)

Pick any existing Sentry issue URL and run:

```bash
# Push the branch first if not already pushed
git push

# Trigger the workflow manually
gh workflow run sentry-autofix.yml -f sentry_issue_url="<PASTE_SENTRY_ISSUE_URL_HERE>"

# Watch the run
gh run watch
```

#### What to expect

1. The Action checks out your code
2. Claude Code + Sentry MCP fetches the error details from Sentry
3. Claude analyzes the stacktrace, navigates your code, identifies root cause
4. Claude creates a new branch `autofix/sentry-<id>`, implements the fix
5. Claude runs relevant tests
6. Claude creates a PR with diagnosis, fix description, and risk assessment
7. (Optional) Slack notification sent

#### If it fails

Common issues:
- **"No Sentry issue URL found"** → The URL format wasn't recognized. Check it's a valid Sentry issue URL.
- **"Could not load prompt template"** → Network issue fetching from GitHub. Try again.
- **Claude can't access Sentry** → Check `SENTRY_AUTH_TOKEN` has correct scopes.
- **Timeout** → Increase `max_turns` (default 30). Complex bugs may need 50+.

## Configuration options

These go in the `with:` section of the consumer workflow:

| Input | Default | Description |
|-------|---------|-------------|
| `severity_filter` | `fatal,error` | Comma-separated. Which Sentry severity levels to auto-fix. |
| `max_turns` | `30` | Max Claude conversation turns. Higher = more complex bugs, more cost. |
| `base_branch` | `main` | Branch to create fix branch from. |
| `model` | `claude-sonnet-4-6-20250514` | Claude model. Options: `claude-opus-4-6` (best, expensive), `claude-sonnet-4-6` (balanced), `claude-haiku-4-5` (cheap, less capable). |
