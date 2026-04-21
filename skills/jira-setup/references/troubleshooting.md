# Setup troubleshooting

Common failure modes during the `jira-setup` walkthrough and how to resolve them.

## Auth failures

### `401 Unauthorized` when calling `/rest/api/3/myself`

**Cause:** username and token don't match.

**Most common reasons:**
1. User pasted their Atlassian **password** instead of a freshly-generated API token. The password won't work for REST auth on Cloud.
2. User copied only part of the token (truncated at a character that looks like the end, e.g. `=` in the middle).
3. User generated the token on a different Atlassian account than the email they typed.
4. Token was revoked after generation (check https://id.atlassian.com/manage-profile/security/api-tokens — if it's not there, it was revoked or never created).

**Fix:**
- Re-open the token page, generate a new token, paste the full string (usually 180+ chars).
- Double-check the email — it must match the Atlassian account the token was generated on.

### `403 Forbidden` when calling `/rest/api/3/myself`

**Cause:** rare; usually Atlassian Access / enterprise SSO is rejecting API token auth.

**Fix:** the user's org admin needs to allow API tokens for the user (or the user needs to create a scoped OAuth app — outside this plugin's scope). Tell the user to escalate to their Jira admin.

### `401` specifically on `/rest/api/3/project/search` but not `/myself`

**Cause:** the token works but the user isn't a member of any visible project. Unlikely but possible for brand-new accounts.

**Fix:** ask the admin to add the user to at least one project.

## Connectivity failures

### `000` exit code or "Could not resolve host"

**Cause:** DNS doesn't see the Atlassian hostname.

**Checks:**
- Is the URL spelled right? (Common typo: `.atlassian.com` instead of `.atlassian.net`.)
- Are they on VPN that blocks it?
- Is there a corporate proxy? Add `--proxy` to curl or `HTTPS_PROXY` env var.

### SSL/TLS errors

**Cause:** corporate MITM proxy intercepting HTTPS.

**Fix:** user's IT team typically provides a CA cert bundle. Set `CURL_CA_BUNDLE=/path/to/ca.pem` or use `REQUESTS_CA_BUNDLE` for Node-based probes.

## Project / component issues

### User sees the project in the UI but not in `/rest/api/3/project/search`

**Cause:** team-managed projects with restricted visibility can hide from REST for unprivileged users.

**Fix:** user types the project key directly when prompted (skip the picker). The config still works — only discovery fails.

### Component creation returns 403

**Cause:** user isn't a project admin.

**Fix:** fall back to telling them:

> "You don't have permission to create Components in `${project_key}` via the API. Ask your project admin to add them, or create them yourself in the UI: `${jira_url}/plugins/servlet/project-config/${project_key}/components`"

Then let the setup proceed with whatever components already exist.

## Config persistence issues

### `Edit` tool fails to modify `~/.claude/settings.json`

**Cause:** file doesn't exist yet, or is a symlink to a location without write permission.

**Fix:** if absent, create it with a minimal structure:

```json
{
  "pluginConfigs": {
    "jira-sprint-workflow@jira-sprint-workflow": {
      "options": { ... }
    }
  }
}
```

If it's a symlink, follow it and write to the real target.

### Settings modification succeeds but config doesn't take effect

**Cause:** Claude Code caches plugin config at session start. Changes require a restart.

**Fix:** tell the user to restart Claude Code. `/plugin reload jira-sprint-workflow` also picks up changes in most versions.

## Keychain / sensitive-value edge cases

### User is on Linux without a keychain daemon

**Cause:** `gnome-keyring` or `kwallet` isn't running.

**Fix:** Claude Code falls back to `~/.claude/.credentials.json` (plaintext). Same effect as putting the token in `settings.json` directly. No action needed — just document it for the user if they ask.

### User wants to rotate the API token later

**Fix:** run `/plugin configure jira-sprint-workflow` and paste the new token, OR re-run this setup skill (it'll detect existing config and offer to update just the token).

## Jira Server / Data Center (not Cloud)

### The URL doesn't end in `.atlassian.net`

**Cause:** user is on a self-hosted Jira instance.

**Fix:** this plugin targets Atlassian Cloud exclusively. Server/DC uses different API paths (`/rest/api/2/` instead of `/rest/api/3/`), different auth (Personal Access Tokens, not API tokens), and different URL patterns. Tell the user:

> "This plugin is built for Jira Cloud. You're on Jira Server/Data Center based on your URL. We don't support that today — please open an issue at https://github.com/Harsh1210/jira-sprint-workflow/issues if you'd like us to add Server support."

Abort setup without writing config.
