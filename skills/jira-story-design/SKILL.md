---
name: jira-story-design
description: Design a feature end-to-end in Jira — brainstorm, write spec, write implementation plan, and create a fully-loaded set of self-contained Jira subtasks that a teammate or sub-agent can execute without any local files. Use whenever the user wants to design, plan, brainstorm, break down, flesh out, turn a Jira Task into subtasks, or prepare any Jira intake ticket for execution. Also use when the user says "let's plan this", "let's design this feature", "create the subtasks", "break this down for the team", or asks for implementation planning on a Jira task. Do not skip this skill and improvise — the workflow here enforces brainstorming, plan writing, and self-contained subtasks that the team relies on.
---

# Jira Story Design

This skill orchestrates end-to-end Phase 1 of a sprint workflow: taking a one-line intake Task in Jira and turning it into a fully-designed set of 11 execution subtasks that any teammate or sub-agent can pick up and run.

## Core principle: Jira self-containment

A teammate must be able to pick up ANY subtask and execute it using ONLY the Jira issue content. They should never need to open a docs folder, grep the repo, or message the designer for context.

Concretely:
- Every subtask description includes the role (expert framing), relevant context (spec excerpt + requirements), subtask-specific content (plan chunks, test cases, review checklist), and acceptance criteria.
- Plan and test-plan files in the repo (at `${user_config.plans_directory}`) are the source of truth for sub-agent execution; Jira content mirrors them as much as fits within size limits.
- A subtask description that only says "see plans/..." without embedded content is wrong and must be fixed.

This is the single most important rule. Everything else supports it.

## Configuration (set once per team)

This skill reads values from the plugin's user config — values you set when you installed the plugin. If any are missing, prompt the user to run `/plugin configure jira-sprint-workflow`.

| Config key | Purpose |
|---|---|
| `jira_url` | Your Jira Cloud base URL (e.g. `https://myteam.atlassian.net`) |
| `jira_username` | Your Jira account email |
| `jira_api_token` | API token for REST fallback (keychain-stored) |
| `project_key` | Project where Tasks live (e.g. `WFR`, `ENG`) |
| `default_owner_account_id` | Default assignee for parent Tasks + Subtask 7 |
| `components` | Comma-separated product areas |
| `plans_directory` | Where plans are saved in the repo |
| `specs_directory` | Where specs are saved in the repo |

Placeholders in this skill reference them as `${user_config.<key>}`.

## When the user invokes this skill

Typical prompts that trigger this skill:
- "Let's design `<TASK-KEY>`"
- "Brainstorm this feature and create the tasks"
- "Plan out the work for adding X"
- "Break `<TASK-KEY>` into subtasks"
- "Turn this into a story for the team"

If the user hasn't told you which Jira task, ask. You need a specific parent Task key before proceeding.

## The orchestration flow (non-negotiable order)

```
1. Confirm intake Task exists (<TASK-KEY>) ────► if not, create it first
2. Invoke superpowers:brainstorming ──────► produces a shared design understanding
3. Write the spec into Jira parent Task ──► full markdown in the Task description
4. Invoke superpowers:writing-plans ──────► produces plans and test-plan files
5. Run plan-document-reviewer loop ───────► fix issues per chunk, re-review until clean
6. Confirm assignee ──────────────────────► default from user_config; confirm if different
7. Create 11 self-contained subtasks ─────► 0, 1, 1b, 2, 3, 4, 5, 6, 7, 8, 9, 10
8. Wire "is blocked by" chain links ──────► enforce execution order in Jira
9. Transition parent Task to "Selected for Development" (or your team's equivalent)
10. Verify the board looks right ─────────► every subtask has content, links, assignee
```

Each step has enforcement rules in the next sections.

## Step-by-step enforcement

### Step 1 — Verify parent Task

Use `mcp__claude_ai_Atlassian_Rovo__getJiraIssue` against `${user_config.jira_url}`. Confirm it's in `${user_config.project_key}` and is a Task (or Story — either works; see "Issue type" note below). If missing, ask the user what they want to design and create the Task first.

### Step 2 — Brainstorming is mandatory

If the user says "skip brainstorming, just make the tasks" or "I already know what I want, just write it up", push back. Brainstorming surfaces constraints, trade-offs, and edge cases that you cannot recover later without expensive rework. The five minutes saved upfront costs two hours later.

Invoke the `superpowers:brainstorming` skill. Let it run fully — don't truncate it. It will produce a design narrative that you'll use in step 3.

### Step 3 — Write the spec into Jira

Once brainstorming is done, edit the parent Task's description using `mcp__claude_ai_Atlassian_Rovo__editJiraIssue` with `contentFormat: "markdown"`. Use the template in `references/story-description-template.md`.

The description must contain full content, not summaries. If the brainstorm produced a 2000-word design, the Jira description has 2000 words.

### Step 4 — Write the implementation plan

Invoke the `superpowers:writing-plans` skill. It produces two files:
- `${user_config.plans_directory}/YYYY-MM-DD-<feature>.md` — full implementation plan, chunk-by-chunk with actual code
- `${user_config.plans_directory}/YYYY-MM-DD-<feature>-test-plan.md` — functional test cases (automated + manual)

These are the source of truth for sub-agent execution in subtask 1 and test execution in subtasks 4, 7, 10.

### Step 5 — Review the plan

Dispatch `plan-document-reviewer` agents in parallel per chunk. Fix issues they flag. Re-review until all chunks are clean. Commit the plan files.

### Step 6 — Confirm ownership

The parent Task stays assigned to the human-in-the-loop owner. Subtask 7 (Manual Testing) gets the SAME assignee as the parent (the human who owns the Story walks the manual test). Everything else is unassigned (Sub Agent).

**Default:** both parent Task and subtask 7 assigned to `${user_config.default_owner_account_id}`.

Only ask the user if the owner is different this time (e.g., a teammate is taking this Story).

### Step 7 — Create the 11 subtasks

Read `references/subtask-templates.md` for the role description, instructions, and acceptance criteria for each of the 11 subtasks. For each subtask:
- Create via `mcp__claude_ai_Atlassian_Rovo__createJiraIssue` with `parent: "<TASK-KEY>"` and `issueTypeName: "Sub-task"` (or `Subtask` — verify with `getJiraProjectIssueTypesMetadata`)
- Description must be fully self-contained (include role + context summary + subtask-specific content + acceptance criteria). Use `references/subtask-templates.md`.
- Set assignee per the rules in "Assignee convention" below
- Set Component + Priority explicitly (don't rely on parent inheritance — Jira is inconsistent on Sub-tasks)

> [!warning] Jira inherits the parent's assignee on subtask creation
> If you don't pass `assignee_account_id` explicitly, the new subtask will be assigned to whoever owns the parent. Since our convention has only ONE human-assigned subtask (#7), this is a trap — all other subtasks end up incorrectly owned. Either:
> 1. Pass `assignee_account_id: null` on every Sub Agent subtask's create call, OR
> 2. After creation, PUT `/rest/api/3/issue/<key>/assignee` with `{accountId: null}` for each Sub Agent subtask.
>
> After all creations, **always re-fetch every subtask** to verify `assignee` matches your intent. Do not trust that the create call obeyed you.

### Step 8 — Wire the blocking chain

Use `mcp__claude_ai_Atlassian_Rovo__createIssueLink` (or raw REST `POST /rest/api/3/issueLink`) to create "is blocked by" relationships:

```
0 → (nothing; first subtask)
1 → blocked by 0
1b → blocked by 0 (parallel with 1)
2 → blocked by 1
3 → blocked by 2
4 → blocked by 3 AND 1b
5 → blocked by 4
6 → blocked by 5
7 → blocked by 6
8 → blocked by 7
9 → blocked by 8
10 → blocked by 9
```

If `createIssueLink` rejects a link type name, check with `getIssueLinkTypes`.

> [!warning] createIssueLink direction is counterintuitive
> The REST v3 API for `issueLink` with type `Blocks` accepts `{type, inwardIssue, outwardIssue}`. In the behavior we've verified:
>
> - `inwardIssue` = the issue that **does the blocking** (the predecessor)
> - `outwardIssue` = the issue **being blocked** (the follower)
>
> To express "subtask 0 blocks subtask 1": `inwardIssue: <subtask-0-key>, outwardIssue: <subtask-1-key>`.
>
> **Verify on the first link you create** by fetching a follower subtask:
> ```
> GET /rest/api/3/issue/<follower-key>?fields=issuelinks
> ```
> The response's `issuelinks[]` should contain an entry with `inwardIssue` pointing at the predecessor and `type.inward === "is blocked by"`. If the direction looks flipped, swap inwardIssue/outwardIssue for all remaining calls (delete the wrong ones via `DELETE /rest/api/3/issueLink/:id`).
>
> Jira instances may behave differently — verify once per project.

### Step 9 — Transition parent Task

Use `mcp__claude_ai_Atlassian_Rovo__getTransitionsForJiraIssue` to find the transition ID for "Selected for Development" (or your team's equivalent — e.g., "Ready for Dev", "To Do", "Up Next") and apply it with `transitionJiraIssue`.

### Step 10 — Verify the board

Before reporting success:
- Re-fetch the parent Task and all 11 subtasks via `getJiraIssue`
- Confirm each has: description, assignee (per rules), component, priority, and the right blocking links (12 total)
- Report back to the user with: parent Task URL, subtask count, assignee assignments, any fields still needing manual attention

## Assignee convention

One human checkpoint in the chain. Everything else is Sub Agent work.

| Issue | Assignee |
|---|---|
| Parent Task | The human-in-the-loop owner of the Story (default: `${user_config.default_owner_account_id}`) |
| Subtask 7 (Manual Testing) | Same as the parent Task's assignee |
| All other subtasks (0, 1, 1b, 2, 3, 4, 5, 6, 8, 9, 10) | Unassigned (= Sub Agent) |

Unassigned means "Sub Agent picks it up autonomously." It's not a bug, it's the signal.

**Subtask 8 (Architect Review) is Sub Agent-driven** via `superpowers:code-reviewer` or a focused review subagent with the full spec + plan + diff as context. The sub-agent plays a senior enterprise architect reviewing for shipping risk. The human owner already signed off on UX in subtask 7; the architect review covers everything else (security, perf, cross-system, rollback, data integrity, etc.).

## Components and priority

**Components** (use instead of labels for product area). Set to one of your team's configured values from `${user_config.components}`. Create components in Jira via Project Settings → Components if they don't exist yet.

**Priority**: Highest (critical/blocker), High (important), Medium (nice to have), Low (someday). Never put `[P1]` in the summary — set the field.

## Issue type note

The parent ticket can be `Task` or `Story` — Jira lets both types have subtasks. Our convention favors leaving intake as `Task` (what most users file tickets as) and skipping a Task→Story conversion. The issue type is just a label; the subtask structure + content is what matters.

If your team's workflow requires Stories specifically, you can convert via `editJiraIssue` with `issueTypeName: "Story"` at Step 3, but it's not required.

## What NOT to do

- Don't skip brainstorming because the user "already has a design in mind". Run it anyway — it catches gaps.
- Don't put `see plans/...` in a subtask description without also embedding the content. Jira content must stand alone within size limits.
- Don't create subtasks without the "is blocked by" links. The chain ordering is what makes sub-agent execution safe.
- Don't leave Component, Priority, or Assignee blank. Prompt the user if you don't know.
- Don't transition the parent Task until all 11 subtasks exist with content, assignees, and links.
- Don't merge subtasks 1 and 1b into one. They run in parallel by design — the test plan must be written by a different brain than the implementation.
- Don't trust `createJiraIssue` to respect your intent — Jira silently inherits parent fields (assignee especially). Always re-fetch and verify every field.
- Don't batch `createIssueLink` calls without verifying direction on the first one.

## Size strategy — Jira description limits

Jira Cloud enforces a practical description limit of ~32KB. A full implementation plan for a framework-level feature can exceed this. Pragmatic strategy:

- **Parent Task description** — summary-level: Original Feedback, Investigation, Requirements checklist, Design/Spec (full), Implementation Plan Summary (chunk titles only), Test Plan Summary (counts), Dependencies, Out of Scope, Open Questions. Reference the plan/test-plan file paths.
- **Subtask 1 (Implement)** — plan outline (chunk titles + task-level file paths) + key code snippets (schema, core interfaces) + pointer to the plan file for full per-step code. The sub-agent executing this subtask reads BOTH Jira + the file.
- **Subtask 1b (Test Cases)** — the full test plan usually fits (~15KB). Embed the complete Automated + Manual sections directly. Also mirror as a Jira comment.
- **Other subtasks (2, 3, 4, 5, 6, 7, 8, 9, 10)** — fit comfortably (~2-12KB each). Embed role, context, checklist/framework, acceptance criteria in full.

If a description exceeds the limit, the API returns 400 with a body-size error. When that happens, trim the plan outline (drop code snippets) rather than sacrificing structure.

## Known tool quirks

- **MCP `createJiraIssue` occasional Cloudflare 403** during heavy batch creates. Fall back to raw REST with the bearer token from `${user_config.jira_api_token}` (keychain).
- **Markdown → ADF conversion via MCP** can double-escape brackets (e.g., `\[ \]` instead of `[ ]` in checkboxes). Review the created issue's rendered description and edit if needed.
- **Jira REST v3 raw API requires ADF format** — if bypassing the MCP tool, convert markdown paragraphs to `{type: "doc", version: 1, content: [...]}` ADF objects before POSTing.

## References

- `references/story-description-template.md` — parent Task description markdown template
- `references/subtask-templates.md` — role, instructions, and acceptance criteria for each of the 11 subtasks
- `references/jira-api-patterns.md` — example MCP tool calls for create, edit, link, transition, and REST fallback patterns

Read these when you reach the relevant step. Don't load them up-front.

## Related skills

- `superpowers:brainstorming` — invoke in step 2
- `superpowers:writing-plans` — invoke in step 4
- `plan-document-reviewer` — dispatch in step 5
- `superpowers:using-git-worktrees` — referenced by subtask 0's execution (runs in Phase 2, not here)
- `superpowers:subagent-driven-development` — referenced by subtask 1's execution
- `superpowers:code-reviewer` — referenced by subtask 8's execution

This skill handles Phase 1 (design). Phase 2 (execute) and per-subtask execution are separate skills (see the plugin's other skills if added).
