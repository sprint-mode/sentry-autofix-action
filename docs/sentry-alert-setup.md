# Sentry Alert Setup

Configure Sentry to automatically create GitHub Issues when new errors occur.

## Prerequisites

- [Sentry-GitHub integration](https://sentry.io/integrations/github/) installed and configured
- Your repo linked to a Sentry project

## Create the Alert Rule

1. Go to **Sentry** > your project > **Alerts** > **Create Alert**
2. Select **Issues (Errors)** as the alert type
3. Configure:

### When
- **When**: A new issue is created

### Filter (optional but recommended)
- **If**: The issue's level is equal to `error` or `fatal`
- This prevents warnings and info messages from triggering auto-fix

### Action
- **Then**: Create a GitHub Issue
  - **Repository**: Select your GitHub repo
  - **Assignee**: Leave empty (the action handles it)
  - **Labels**: `sentry` (this is how the action identifies Sentry-created issues)

### Alert name
- Name it something like: "Auto-fix: Create GitHub Issue for new errors"

4. Click **Save Rule**

## Verify it works

1. Use Sentry's "Test" feature to trigger a sample alert
2. Check that a GitHub Issue was created in your repo with:
   - Label: `sentry`
   - Body containing the Sentry issue URL
3. The GitHub Action should trigger within seconds

## Tips

- Sentry de-duplicates issues, so the same error won't create multiple GitHub Issues
- You can add more filters (e.g., only specific environments, only high-frequency issues)
- If you want to limit which errors trigger auto-fix, use Sentry's alert conditions
