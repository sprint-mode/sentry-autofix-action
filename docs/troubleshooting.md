# Troubleshooting

## Action doesn't trigger

**Issue:** GitHub Issue is created but the action doesn't run.

**Checks:**
1. Does the issue have the `sentry` label? The action filters on `contains(github.event.issue.labels.*.name, 'sentry')`.
2. Is the workflow file at `.github/workflows/sentry-autofix.yml`?
3. Is the workflow enabled? Go to Actions tab > check it's not disabled.

## "No Sentry issue URL found" error

**Issue:** The action can't find the Sentry URL in the GitHub issue body.

**Cause:** Sentry's GitHub issue creation might format the URL differently.

**Fix:** Check the GitHub issue body. The action looks for URLs matching `https://.*sentry.*issues/[0-9]+`. If your Sentry URL format is different, open an issue on this repo.

## Claude can't access Sentry

**Issue:** Claude reports it can't connect to Sentry MCP.

**Checks:**
1. Is `SENTRY_AUTH_TOKEN` set in your repo secrets?
2. Does the token have the required scopes: `project:read`, `event:read`, `issue:read`?
3. Is the token still valid (not expired/revoked)?

## Claude creates a PR but the fix is wrong

This is expected sometimes — Claude is powerful but not perfect. The design intentionally requires human review before merge.

**What to do:**
1. Review the PR and close it if the fix is wrong
2. Comment on the PR explaining what's wrong (this helps improve the prompt over time)
3. Fix the issue manually

## Slack notifications not working

**Checks:**
1. Is `SLACK_AUTOFIX_WEBHOOK` (or `SLACK_WEBHOOK_URL`) set in secrets?
2. Is the webhook URL still valid?
3. Check the action logs — the Slack step logs warnings if the notification fails.

## Action times out

**Cause:** Claude ran out of conversation turns before completing.

**Fix:** Increase `max_turns` in your consumer workflow (default: 30). Complex bugs may need 50+.
