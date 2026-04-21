# Jira API patterns — MCP and REST fallback

Use the Atlassian Rovo MCP tools by default. Fall back to raw REST (via `bun -e` + fetch) only when the MCP tool doesn't support what you need.

## Common MCP calls

All MCP calls accept `cloudId: "${user_config.jira_url}"  # e.g. <your-jira>.atlassian.net` (or the UUID form `8c15a106-50a2-4874-95b8-36ce82d87b90`).

### Fetch a Task with full description

```
mcp__claude_ai_Atlassian_Rovo__getJiraIssue
  cloudId: "${user_config.jira_url}"  # e.g. <your-jira>.atlassian.net
  issueIdOrKey: "<TASK-KEY>"
  responseContentFormat: "markdown"
```

### Update parent Task description (embed spec)

```
mcp__claude_ai_Atlassian_Rovo__editJiraIssue
  cloudId: "${user_config.jira_url}"  # e.g. <your-jira>.atlassian.net
  issueIdOrKey: "<TASK-KEY>"
  contentFormat: "markdown"
  description: "<full markdown content from story-description-template.md>"
  additional_fields:
    priority: { name: "High" }
    components: [{ name: "<ComponentName>" }]  // from ${user_config.components}
  assignee_account_id: "${user_config.default_owner_account_id}"  // or a specific team member's account ID
```

### Create a subtask

```
mcp__claude_ai_Atlassian_Rovo__createJiraIssue
  cloudId: "${user_config.jira_url}"  # e.g. <your-jira>.atlassian.net
  projectKey: "${user_config.project_key}"
  issueTypeName: "Sub-task"     # confirm exact name via getJiraProjectIssueTypesMetadata
  parent: "<TASK-KEY>"
  summary: "0. Setup Branch & Worktree"
  contentFormat: "markdown"
  description: "<full subtask description from subtask-templates.md>"
  additional_fields:
    components: [{ name: "<ComponentName>" }]  // from ${user_config.components}
    priority: { name: "High" }
  assignee_account_id: null     # unassigned (Sub Agent); for subtask 7, use ${user_config.default_owner_account_id} or the parent Task's assignee
```

### Link two subtasks with "is blocked by"

```
mcp__claude_ai_Atlassian_Rovo__createIssueLink
  cloudId: "${user_config.jira_url}"  # e.g. <your-jira>.atlassian.net
  type: "Blocks"                   # confirm via getIssueLinkTypes
  inwardIssue: "<BLOCKER-KEY>"            # the blocker (subtask 0)
  outwardIssue: "<BLOCKED-KEY>"           # the blocked (subtask 1)
```

> [!warning] **Direction is counterintuitive — verify on the first link before batching.**
> 
> The verified behavior on `<your-jira>.atlassian.net`:
> - `inwardIssue` = the issue doing the blocking (predecessor)
> - `outwardIssue` = the issue being blocked (follower)
> 
> **Verification pattern after creating the first link:**
> ```
> GET /rest/api/3/issue/<follower-key>?fields=issuelinks
> ```
> 
> The response's `fields.issuelinks[]` should contain an entry where:
> - `inwardIssue` references the predecessor
> - `type.inward === "is blocked by"`
> 
> If the relationship appears reversed (e.g., the follower shows `outwardIssue` pointing at the predecessor with relation `blocks`, meaning the follower thinks it BLOCKS the predecessor), your direction is inverted. Swap `inwardIssue` ↔ `outwardIssue` for all subsequent calls, and delete the wrong ones via `DELETE /rest/api/3/issueLink/<link-id>`.

### Fix reversed links (recovery pattern)

```js
// 1. Collect all link IDs from affected subtasks
const keys = ["<SUBTASK-KEY-1>", "<SUBTASK-KEY-2>", ...];
const linkIds = new Set();
for (const k of keys) {
  const r = await fetch(`https://<your-jira>.atlassian.net/rest/api/3/issue/${k}?fields=issuelinks`, { headers });
  const d = await r.json();
  for (const l of (d.fields.issuelinks || [])) linkIds.add(l.id);
}

// 2. Delete all
for (const id of linkIds) {
  await fetch(`https://<your-jira>.atlassian.net/rest/api/3/issueLink/${id}`, { method: "DELETE", headers });
}

// 3. Re-create with swapped direction
```

### List available link types

```
mcp__claude_ai_Atlassian_Rovo__getIssueLinkTypes
  cloudId: "${user_config.jira_url}"  # e.g. <your-jira>.atlassian.net
```

### Find transition to "Selected for Development"

```
mcp__claude_ai_Atlassian_Rovo__getTransitionsForJiraIssue
  cloudId: "${user_config.jira_url}"  # e.g. <your-jira>.atlassian.net
  issueIdOrKey: "<TASK-KEY>"
```

Returns a list of transitions with IDs. Pick the one whose `name` or target status is "Selected for Development".

### Apply the transition

```
mcp__claude_ai_Atlassian_Rovo__transitionJiraIssue
  cloudId: "${user_config.jira_url}"  # e.g. <your-jira>.atlassian.net
  issueIdOrKey: "<TASK-KEY>"
  transition: { id: "<transition-id>" }
```

### Resolve a user by email or display name

```
mcp__claude_ai_Atlassian_Rovo__lookupJiraAccountId
  cloudId: "${user_config.jira_url}"  # e.g. <your-jira>.atlassian.net
  query: "user@example.com"     # or display name
```

## REST fallback (bun)

Fall back to raw REST when:
- The MCP tool is missing a capability (e.g., bulk link deletion)
- MCP calls fail with Cloudflare 403/429 (rare but happens during batched creates)
- You need to set fields the MCP tool doesn't expose

Credentials in `reference_jira.md` (auto-memory). Pattern:

```javascript
const auth = Buffer.from(`${process.env.JIRA_USERNAME}:${process.env.JIRA_API_TOKEN}`).toString("base64");
const headers = {
  "Authorization": "Basic " + auth,
  "Accept": "application/json",
  "Content-Type": "application/json"
};

const res = await fetch("https://<your-jira>.atlassian.net/rest/api/3/...", {
  method: "POST",
  headers,
  body: JSON.stringify({ /* ... */ })
});
```

### ADF requirement for raw REST

Unlike the MCP tool (which accepts markdown via `contentFormat: "markdown"`), the raw REST v3 API requires **ADF** (Atlassian Document Format) for any rich-text field. Minimal paragraph-level ADF:

```js
const adf = {
  type: "doc",
  version: 1,
  content: markdownText.split("\n\n").map(p => ({
    type: "paragraph",
    content: [{ type: "text", text: p }]
  }))
};
```

This loses headings/lists/code-blocks but is sufficient for descriptions. For full fidelity, prefer the MCP tool with `contentFormat: "markdown"`.

### Create subtask with explicit null assignee (REST pattern)

```js
const body = {
  fields: {
    project: { key: "${user_config.project_key}" },
    parent: { key: "<TASK-KEY>" },
    summary: "2. Code Review",
    issuetype: { name: "Sub-task" },
    description: adf,
    priority: { name: "High" },
    components: [{ name: "<ComponentName>" }]  // from ${user_config.components}
    // Note: omitting "assignee" does NOT unset it — Jira inherits from parent.
    // Either pass assignee: null OR unassign in a separate PUT call.
  }
};
await fetch("https://<your-jira>.atlassian.net/rest/api/3/issue", { method: "POST", headers, body: JSON.stringify(body) });
```

Then explicitly unassign:
```js
await fetch(`https://<your-jira>.atlassian.net/rest/api/3/issue/${key}/assignee`, {
  method: "PUT", headers, body: JSON.stringify({ accountId: null }),
});
```

## Error handling patterns

### "Issue type 'Sub-task' not found"

Check project config: `getJiraProjectIssueTypesMetadata(projectIdOrKey: "${user_config.project_key}")`. The issue type might be named `Subtask` (one word) or `Sub-task` depending on the project. Adjust `issueTypeName` accordingly.

### "Link type 'is blocked by' not found"

Check `getIssueLinkTypes`. The type is usually `Blocks` with inward name "is blocked by" and outward name "blocks". Use the type's `name` field (e.g., `"Blocks"`) in createIssueLink, then set inward/outward correctly.

### Description too long

Jira has a description size limit (~32 KB). If the spec + plan is larger, split into the parent Task description (summary + spec) and the subtask descriptions (plan chunks for subtask 1, test cases for 1b). The parent Task never needs to hold the full plan — that's what subtasks are for.

### Assignee not found

If `lookupJiraAccountId` returns nothing, the user may not have been added to the project. Ask the user to add them via Jira settings, then retry.

## Verification checklist before handoff

After creating everything, re-fetch every subtask and confirm:

```js
const keys = ["<TASK-KEY>", /* all subtask keys */];
for (const k of keys) {
  const r = await fetch(`https://<your-jira>.atlassian.net/rest/api/3/issue/${k}?fields=summary,status,priority,components,assignee,issuelinks`, { headers });
  const d = await r.json();
  // Inspect d.fields to confirm expected values
}
```

Confirm for each issue:
- [ ] Parent Task status = "Selected for Development"
- [ ] Parent Task assignee = the human-in-the-loop owner (${user_config.default_owner_account_id})
- [ ] Parent Task has Component (CRM etc.) and Priority (Highest/High/Medium/Low) set
- [ ] Parent Task description covers all 9 sections from `story-description-template.md`
- [ ] 11 subtasks exist (summaries: `0. Setup Branch & Worktree`, `1. Implement Feature`, `1b. Plan Functional Test Cases (parallel with #1)`, `2. Code Review`, `3. Fix Code Review Issues`, `4. UI Testing`, `5. Fix UI Testing Issues`, `6. Deploy to Local Docker`, `7. Manual Testing`, `8. Senior Architect Review`, `9. Push to Prod`, `10. Prod Sanity Test`)
- [ ] Subtask 7 assigned to the parent Task's assignee; ALL other subtasks unassigned (explicit `null`)
- [ ] Every subtask has role + context + subtask-specific instructions + acceptance criteria
- [ ] Every subtask has Component and Priority set (don't rely on parent inheritance — Jira is unreliable here for Sub-tasks)
- [ ] "Is blocked by" chain correct — verify direction on one link before trusting the rest
- [ ] 12 total links exist: 0→1, 0→1b, 1→2, 2→3, 3→4, 1b→4, 4→5, 5→6, 6→7, 7→8, 8→9, 9→10

If anything is missing or wrong, fix before reporting complete.
