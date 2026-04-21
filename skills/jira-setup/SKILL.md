---
name: jira-setup
description: First-time setup walkthrough for the jira-sprint-workflow plugin — interactively collects Jira URL, credentials, project key, owner account ID, and components; guides the user to Atlassian Cloud to generate an API token; validates each value via a live API call; and writes everything to Claude Code's pluginConfigs so the jira-story-design / jira-story-execute / jira-architect-review skills have what they need. Use whenever the user says "set up the jira plugin", "configure jira-sprint-workflow", "I just installed the plugin, what now?", "walk me through jira setup", "first-time setup", or when any of the other skills in this plugin detect missing config and hand off to you.
---

# Jira Sprint Workflow — Guided Setup

This skill is the on-ramp for a freshly-installed `jira-sprint-workflow` plugin. It asks one question at a time, validates each answer against the Jira API, and persists the result so the other three skills (`jira-story-design`, `jira-story-execute`, `jira-architect-review`) just work.

## When this skill runs

1. A user invokes it directly: "set up the jira plugin", "configure jira-sprint-workflow", etc.
2. Another skill in this plugin detects missing config and hands off here. In that case, finish the walkthrough, then return control to the invoking skill.

Skip this skill if every required config key already has a non-empty value — print a one-line summary of current config and stop.

## The eight values we collect

| # | Key | Source | Sensitive |
|---|-----|--------|-----------|
| 1 | `jira_url` | User types it | no |
| 2 | `jira_username` | User types it | no |
| 3 | `jira_api_token` | User generates at Atlassian Cloud (we open the page) | **yes** |
| 4 | `project_key` | Auto-discovered via `/rest/api/3/project/search`; user picks | no |
| 5 | `default_owner_account_id` | Auto-discovered via `/rest/api/3/myself` | no |
| 6 | `components` | Auto-discovered via `/rest/api/3/project/<key>/components`; offer to create missing | no |
| 7 | `plans_directory` | Default `docs/superpowers/plans`; confirm | no |
| 8 | `specs_directory` | Default `docs/superpowers/specs`; confirm | no |

Only keys 1, 2, 3 need typing — everything else we discover or default.

## Before starting — check where we are

Read `~/.claude/settings.json`. If `pluginConfigs["jira-sprint-workflow@jira-sprint-workflow"].options` already has values, tell the user:

> "I found an existing config for `jira-sprint-workflow`:
>
> ```
> jira_url:                 <value>
> jira_username:            <value>
> jira_api_token:           <masked: ATAT…A6AB> (or: not set)
> project_key:              <value>
> default_owner_account_id: <value>
> components:               <value>
> plans_directory:          <value>
> specs_directory:          <value>
> ```
>
> Do you want to (a) keep all of this and exit, (b) update specific fields, or (c) start over from scratch?"

Branch on the answer. Only walk through every step if they choose (c) or if the config is genuinely empty.

## The walkthrough (ask one question at a time)

### Step 1 — Jira Cloud URL

Ask:

> "What's your Jira Cloud URL? It looks like `https://<your-team>.atlassian.net` — you can grab it from the address bar when you're logged into Jira in a browser."

Validate format: must match `^https://[a-z0-9-]+\.atlassian\.net/?$`. Strip any trailing slash, path, or `/jira` suffix.

Verify it's reachable: `GET ${jira_url}/status` should return 200 with a JSON body containing `{"state":"RUNNING"}` or similar. If it 404s or times out, tell the user and re-ask.

Store as `jira_url`.

### Step 2 — Jira username

Ask:

> "What email do you sign into Jira with? (This is the email associated with your Atlassian account.)"

Validate it's a well-formed email. Store as `jira_username`. We can't verify it yet — that happens after step 3 when we try the API.

### Step 3 — API token (guided)

Explain and open the page:

> "Jira's REST API uses an API token as the password, not your Atlassian password. I'll open the token management page — create a new token labeled `jira-sprint-workflow`, copy it, and paste it back here.
>
> Opening: https://id.atlassian.com/manage-profile/security/api-tokens"

Use the `open` command (macOS), `xdg-open` (Linux), or `start` (Windows) via Bash:

```bash
open "https://id.atlassian.com/manage-profile/security/api-tokens" 2>/dev/null || \
  xdg-open "https://id.atlassian.com/manage-profile/security/api-tokens" 2>/dev/null || \
  echo "Please open this URL manually: https://id.atlassian.com/manage-profile/security/api-tokens"
```

Then instruct:

> "Steps on that page:
> 1. Click **Create API token**
> 2. Label it `jira-sprint-workflow` (or whatever name you want)
> 3. Expiry: 1 year is fine
> 4. Click **Create**, then **Copy** the token — Atlassian only shows it once
>
> Paste the token here:"

Wait for the paste. Validate format: Atlassian tokens are typically `ATATT3xFfGF0...` (long base64-like). Trim whitespace.

**Verify** by calling the Jira API with the first three values:

```bash
auth=$(printf '%s' "${jira_username}:${jira_api_token}" | base64)
curl -s -o /tmp/jira-myself.json -w '%{http_code}' \
  -H "Authorization: Basic ${auth}" \
  -H "Accept: application/json" \
  "${jira_url}/rest/api/3/myself"
```

(If `curl` isn't available, use `bun -e` with `fetch` — see `references/api-probes.md`.)

- `200` → auth works. Parse `accountId`, `emailAddress`, `displayName` — we'll use these in step 5.
- `401` → bad username or token. Tell user which and re-ask from step 2 or 3.
- `403` → token lacks scope (unlikely with user-scoped tokens). Surface the error.
- anything else → show the raw error body and let the user decide.

Store as `jira_api_token`.

### Step 4 — Project key

Fetch the list of projects the authenticated user can see:

```
GET ${jira_url}/rest/api/3/project/search?maxResults=50
```

Show a numbered list of `key — name`. Ask:

> "Which project will hold your Stories and Subtasks? I see these:
>
>   1. WFR — Wishlist - Features and Requests
>   2. ENG — Engineering
>   3. DOC — Documentation
>
> Pick a number, or type a project key directly."

Validate the chosen key exists in the result set. Store as `project_key`.

### Step 5 — Default owner account ID

We already have this from step 3's `/myself` call. Confirm:

> "I'll use **your** account (`${displayName}` — `${emailAddress}`) as the default owner for parent Tasks and Subtask 7 (Manual Testing). That's the human checkpoint in the chain — assigning it to you means Jira pings you when it's your turn.
>
> Keep this, or use a different account?"

If they want a different account, ask for an email and resolve it:

```
GET ${jira_url}/rest/api/3/user/search?query=<email>
```

Store the `accountId` as `default_owner_account_id`.

### Step 6 — Components

List the project's existing components:

```
GET ${jira_url}/rest/api/3/project/${project_key}/components
```

Show them:

> "Your project `${project_key}` has these Components:
>
>   - CRM
>   - Auth
>   - Infrastructure
>
> Components represent product areas — every subtask gets tagged with one so filtering by area works. Should I (a) use these as-is, (b) add more (I can create them now), or (c) replace with a new list?"

If (b) or (c), collect the new names and create each one that doesn't exist via:

```
POST ${jira_url}/rest/api/3/component
{"name":"<name>","project":"${project_key}"}
```

Store the final comma-separated list as `components`.

### Step 7 — Plans directory

Ask:

> "Where should implementation plans and test plans be saved in the repo?
>
> Default: `docs/superpowers/plans` (works great alongside the superpowers plugin's writing-plans skill)."

Accept the default if they press Enter or say "yes"/"default". Otherwise take their input. Store as `plans_directory`.

### Step 8 — Specs directory

Ask:

> "And where should design specs go?
>
> Default: `docs/superpowers/specs`"

Same pattern. Store as `specs_directory`.

## Persist the config

Read `~/.claude/settings.json`. Merge the collected values into `pluginConfigs["jira-sprint-workflow@jira-sprint-workflow"].options`:

```json
{
  "pluginConfigs": {
    "jira-sprint-workflow@jira-sprint-workflow": {
      "options": {
        "jira_url": "...",
        "jira_username": "...",
        "jira_api_token": "...",
        "project_key": "...",
        "default_owner_account_id": "...",
        "components": "...",
        "plans_directory": "...",
        "specs_directory": "..."
      }
    }
  }
}
```

Use the Edit tool (not Write — preserves other settings). If `pluginConfigs` already has other plugins' configs, don't touch them — insert/update only this plugin's entry.

**Sensitive-value note:** The `jira_api_token` field is marked `"sensitive": true` in `plugin.json`. Claude Code prefers the OS keychain for sensitive values. Writing directly to `settings.json` works (skills can still read it via `${user_config.jira_api_token}`), but the plaintext sits on disk. If the user would rather stash it in keychain, tell them to run `/plugin configure jira-sprint-workflow` and re-enter just the token — the CLI will mask the input and use keychain automatically. Offer this as an option; don't force it.

## Final validation

Make one more round-trip to prove everything works end-to-end:

```
GET ${jira_url}/rest/api/3/project/${project_key}?expand=projectKeys,description
```

- Confirms project exists
- Confirms auth still works
- Confirms user has read access

Then print a green-check summary:

> "✅ Setup complete.
>
> ```
> jira_url:                 https://myteam.atlassian.net
> jira_username:            jane@myteam.com
> jira_api_token:           ATAT…A6AB (stored)
> project_key:              ENG
> default_owner_account_id: 712020:xxxx (Jane Doe)
> components:               CRM, Auth, Infrastructure
> plans_directory:          docs/superpowers/plans
> specs_directory:          docs/superpowers/specs
> ```
>
> **Restart Claude Code** (or reload the plugin) to pick up the new config.
>
> After restart, try any of:
>   • `design WFR-42` — scope a feature end-to-end (invokes jira-story-design)
>   • `pick up WFR-42` — execute the 11-subtask chain (invokes jira-story-execute)
>   • `review this change for shipping` — run the architect review (invokes jira-architect-review)"

## One-time Jira project prep (offer at the end)

After config is saved, offer optional Jira project prep:

> "Two quick Jira-side prep items that make the workflow smooth — want me to check/set these?
>
> 1. Confirm your workflow has a status equivalent to 'Selected for Development' — this is what the skill transitions intake Tasks to after designing them. Common names: 'Selected for Development', 'Ready for Dev', 'To Do', 'Up Next'.
>
> 2. Confirm Sub-task is an available issue type in your project. Some team-managed projects call it 'Subtask' (one word), some 'Sub-task'. The skill detects this at runtime but it's nice to know now."

For (1): `GET ${jira_url}/rest/api/3/project/${project_key}/statuses` — surface the status list and let the user map it.

For (2): `GET ${jira_url}/rest/api/3/issuetype/project?projectId=<id>` — check that something matching `/sub.?task/i` exists.

Report findings. Don't modify the workflow — the user owns that.

## Edge cases

- **The user already set config via `/plugin configure`** — detect this on startup (see "Before starting"), skip the walkthrough, just confirm the values.
- **User pastes a token with surrounding quotes or whitespace** — trim.
- **User's `jira_url` contains `/jira` or a trailing path** — strip to the bare hostname.
- **Auth works but project_key returns 404** — the user typed a key they can't access, or the key doesn't exist. Re-show the list from step 4.
- **The plugin hasn't been installed yet** — if `~/.claude/plugins/cache/jira-sprint-workflow/` doesn't exist, say: "Install the plugin first with `/plugin install jira-sprint-workflow@jira-sprint-workflow` and then re-run me."
- **`curl` is unavailable (some sandboxes)** — use `bun -e` + `fetch`; see `references/api-probes.md`.
- **The user is on Jira Server / Data Center, not Cloud** — the API paths differ (`/rest/api/2/` vs `/rest/api/3/`) and auth uses PAT not token. Detect Server by the URL not ending in `.atlassian.net` and tell them this plugin targets Cloud; if they want Server support, open an issue.

## What NOT to do

- Don't persist partial config. If the user aborts midway, discard what you collected for this session.
- Don't echo the API token back in plaintext in any message. Mask to `ATAT…<last-4>`.
- Don't attempt to create the project, add the user, or touch Jira permissions. If the user lacks access, tell them to ask their Jira admin.
- Don't hardcode example values (Carma, WFR, etc.). All examples in prompts should be generic placeholders.

## References

- `references/api-probes.md` — exact curl / `bun -e` commands for each validation call (copy-paste ready)
- `references/troubleshooting.md` — common errors (401, 403, CORS) and their usual causes

## Related skills

- `jira-story-design` — invoke this after setup completes (or after user says "let's design <KEY>")
- `jira-story-execute` — invoke this after setup completes (or after user says "pick up <KEY>")
- `jira-architect-review` — standalone; uses the same config
