# Jira API probes for setup validation

Copy-paste-ready commands for each step of the setup walkthrough. Variable names match the field names in `plugin.json`'s `userConfig`.

All of these use Basic auth with the `jira_username:jira_api_token` pair base64-encoded. Tokens are user-scoped, so these calls only see what the user has access to in Jira.

## Preferred transport

- **curl** — available on macOS and most Linux distributions
- **`bun -e`** with `fetch` — use when `curl` is absent (some sandboxes). Bun is pre-installed in Claude Code's runtime for most users

## Auth header builder

```bash
# Bash
auth=$(printf '%s' "${jira_username}:${jira_api_token}" | base64)
```

```js
// bun -e
const auth = Buffer.from(`${process.env.JIRA_USERNAME}:${process.env.JIRA_API_TOKEN}`).toString("base64");
```

## Probe 1 — URL reachability (no auth)

Purpose: confirm the URL the user typed actually resolves to an Atlassian Cloud site.

```bash
curl -s -o /dev/null -w '%{http_code}' "${jira_url}/status"
```

Expected: `200` (some sites return `401` here and that's fine — the URL is valid).

Fail states:
- `000` — DNS/network failure. The URL is wrong or there's a proxy issue.
- `404` — not an Atlassian site; ask the user to re-check.
- timeout — captive portal or firewall.

## Probe 2 — Auth check + owner discovery (`/myself`)

Purpose: single call that validates the username+token combo AND gives us the user's `accountId` / `displayName` / `emailAddress` for the "default owner" default.

```bash
curl -s -o /tmp/jira-myself.json -w '%{http_code}' \
  -H "Authorization: Basic ${auth}" \
  -H "Accept: application/json" \
  "${jira_url}/rest/api/3/myself"
```

Parse the JSON body on 200:

```bash
# requires jq
jq -r '.accountId,.displayName,.emailAddress' /tmp/jira-myself.json
```

Or via `bun -e`:

```bash
bun -e "
  const auth = Buffer.from(process.env.JIRA_USERNAME + ':' + process.env.JIRA_API_TOKEN).toString('base64');
  const r = await fetch(process.env.JIRA_URL + '/rest/api/3/myself', {
    headers: { 'Authorization': 'Basic ' + auth, 'Accept': 'application/json' }
  });
  if (!r.ok) { console.error(r.status, await r.text()); process.exit(1); }
  const me = await r.json();
  console.log(JSON.stringify({ accountId: me.accountId, displayName: me.displayName, emailAddress: me.emailAddress }));
"
```

Fail states:
- `401` — username+token don't match. Usually the user copied only part of the token, or pasted the Atlassian password by mistake. Re-ask.
- `403` — rare; might mean MFA enforcement rejected API token access. The user needs to regenerate the token.
- Network error — fall back to telling them to check VPN / firewall.

## Probe 3 — List projects the user can see

Purpose: let the user pick their `project_key` from a list rather than typing it.

```bash
curl -s \
  -H "Authorization: Basic ${auth}" \
  -H "Accept: application/json" \
  "${jira_url}/rest/api/3/project/search?maxResults=50&orderBy=key"
```

Response shape:

```json
{
  "values": [
    { "id": "10000", "key": "WFR", "name": "Wishlist - Features and Requests", ... },
    { "id": "10001", "key": "ENG", "name": "Engineering", ... }
  ],
  "total": 2
}
```

Show `key — name` to the user. If `total > 50`, add `&startAt=50` pagination.

## Probe 4 — Resolve a different owner by email

Only needed if the user wants someone other than themselves as the default owner.

```bash
email="jane@acme.com"
curl -s \
  -H "Authorization: Basic ${auth}" \
  -H "Accept: application/json" \
  "${jira_url}/rest/api/3/user/search?query=${email}"
```

Response: array of user objects. Match on `emailAddress` (exact) or fall back to the first hit. Extract `accountId`.

Empty array → the email doesn't exist in this Jira instance (user isn't on the team yet).

## Probe 5 — List existing components in the project

Purpose: pre-fill the `components` config with what's already in Jira.

```bash
curl -s \
  -H "Authorization: Basic ${auth}" \
  -H "Accept: application/json" \
  "${jira_url}/rest/api/3/project/${project_key}/components"
```

Returns an array of `{ id, name, description, lead?, assigneeType, ... }`. Show the names.

## Probe 6 — Create a new component

```bash
curl -s -X POST \
  -H "Authorization: Basic ${auth}" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"${component_name}\",\"project\":\"${project_key}\"}" \
  "${jira_url}/rest/api/3/component"
```

Requires `Administer Projects` permission. If 403, tell the user to create it via the Jira UI (Project Settings → Components).

## Probe 7 — List project statuses (for workflow-equivalent check)

```bash
curl -s \
  -H "Authorization: Basic ${auth}" \
  -H "Accept: application/json" \
  "${jira_url}/rest/api/3/project/${project_key}/statuses"
```

Surface status names so the user can confirm a "Selected for Development"-equivalent exists.

## Probe 8 — List issue types for the project

Purpose: confirm `Sub-task` (or `Subtask`) is available.

```bash
# Need project ID, not key — get it from Probe 3's response
project_id="10000"
curl -s \
  -H "Authorization: Basic ${auth}" \
  -H "Accept: application/json" \
  "${jira_url}/rest/api/3/issuetype/project?projectId=${project_id}"
```

Check for `name` matching `/sub.?task/i`. If absent, the user's project is team-managed and they'll need to enable subtasks via Project Settings → Issue Types → Add Sub-task.

## Probe 9 — Final smoke test after config save

Round-trips the full config to prove everything still works after persistence:

```bash
curl -s -o /dev/null -w '%{http_code}' \
  -H "Authorization: Basic ${auth}" \
  -H "Accept: application/json" \
  "${jira_url}/rest/api/3/project/${project_key}?expand=projectKeys,description"
```

Expected: 200. If anything other than 200, something changed between step 3 and now (unlikely in a 30-second walkthrough but possible).
