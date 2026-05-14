# Jira Sprint Workflow Plugin for Claude Code

A Claude Code plugin that turns a one-line Jira intake ticket into a fully-designed, self-contained set of 15 subtasks that any teammate or sub-agent can execute.

**The core discipline:** every Jira subtask is self-contained. A teammate picks up a subtask, reads the description, and executes it — no docs folder, no slack thread, no prerequisite context required.

## What it does

Four skills covering setup + the full end-to-end flow:

### `jira-setup` — guided first-time configuration

Walks you through Jira Cloud URL, email, API token (opens the Atlassian token page in your browser), project picker, auto-discovered account ID, components, and directories. Validates live against `/rest/api/3/myself`. Writes config to `~/.claude/settings.json`. The other three skills hand off to this automatically if any required config is missing.

Activates on prompts like `set up the jira plugin`, `reconfigure jira plugin`, `connect jira`.

### `jira-story-design` — Phase 1 (design)

```
Brainstorm → Embed spec in Jira → Implementation plan → Test plan → 15 self-contained subtasks → "is blocked by" chain → Selected for Development
```

Activates on prompts like `let's design WFR-42`, `brainstorm this and create the tasks`, `break this into subtasks`.

### `jira-story-execute` — Phase 2 (execution)

```
Pick up Story → Autonomously walk 15-subtask chain → Pause at subtask 10 (sandbox manual test) → Resume → Pause at subtask 12 (merge approval) → Resume → Done
```

Activates on prompts like `pick up WFR-42`, `execute WFR-42`, `continue WFR-42`.

Sub-agents dispatched by Phase 2 transition their subtask to `In Progress` and post incremental `Progress:` comments as they work, so any crashed or paused run resumes cleanly from Jira state alone.

### The 15-subtask chain

```
  0. Branch+Worktree
       │
       ├─→ 1. Implement ───┐
       │                    ↓
       └─→ 1b. Test Cases → 2. Code Review → 3. Fix → 4. UI Test → 5. Fix → 6. Local Smoke → 7. Architect Review → 8. Fix → 9. Deploy to Sandbox → 10. Sandbox Manual Test [human ✋] → 11. Fix Sandbox → 12. Merge to Sandbox Branch [human ✋] → 13. Push to Prod (sandbox → main PR) → 14. Prod Sanity
```

**Two human checkpoints**:
- Step 10 — Sandbox Manual Test (validate the feature on a real-deployed sandbox env)
- Step 12 — Merge to Sandbox Branch (explicit `Approved — merge to sandbox branch` to join the release train)

Everything else runs via sub-agents. Fix loops (2↔3, 4↔5, 7↔8, and 9→10→11) cap at 3 cycles each before escalating.

### Release train (sandbox as the integration branch)

```
feature branch ──(subtask 12, with approval)──→ sandbox ──(subtask 13 PR)──→ main (prod)
```

The `sandbox` branch is the integration buffer — multiple features may merge into `sandbox` before any one of them ships to prod. Subtask 13 opens a `sandbox → main` PR with a release-train manifest listing every feature included.

### Fields set automatically per subtask

- **Priority** — inherited from parent Task, set explicitly (Jira inheritance for Sub-tasks is unreliable)
- **Component** — product area from your team's configured components
- **Assignee** — Sub Agent subtasks are `unassigned` (the signal for autonomous execution); subtasks 10 (Sandbox Manual Test) and 12 (Merge to Sandbox Branch) = the Story owner
- **"is blocked by" links** — wired per the chain diagram, direction verified on creation

## Installation

### Step 1 — Add the marketplace

From a Claude Code session:

```
/plugin marketplace add Harsh1210/jira-sprint-workflow
```

(Or for local development: `/plugin marketplace add ./path/to/this/directory`)

### Step 2 — Install the plugin

```
/plugin install jira-sprint-workflow
```

### Step 3 — Run the guided setup

After install, in any Claude Code session say:

```
set up the jira plugin
```

This triggers the `jira-setup` skill, which walks you through the whole thing in ~2 minutes:

1. Asks for your Jira Cloud URL and email
2. Opens `https://id.atlassian.com/manage-profile/security/api-tokens` in your browser so you can generate an API token, then validates it with a live call
3. Auto-discovers your `accountId` via `/rest/api/3/myself` (no hunting for it in Jira)
4. Lists the projects you have access to and lets you pick the target project
5. Lists the project's existing Components, offers to add missing ones (creates them via API if you have permission)
6. Confirms defaults for `plans_directory` (`docs/superpowers/plans`) and `specs_directory` (`docs/superpowers/specs`)
7. Validates the full config with a final API round-trip
8. Writes everything to `~/.claude/settings.json` under `pluginConfigs` so the three workflow skills pick it up

The other skills in this plugin (`jira-story-design`, `jira-story-execute`, `jira-architect-review`) all run a config gate on invocation — if they detect missing or empty fields, they hand off to `jira-setup` automatically before proceeding. You never end up midway through designing a Story with a broken auth token.

Restart Claude Code (or `/plugin reload`) after setup so the new config takes effect.

### Config values collected

The setup skill collects these (stored in `~/.claude/settings.json` under `pluginConfigs.jira-sprint-workflow@jira-sprint-workflow.options`; `jira_api_token` can live in the OS keychain instead if you prefer — run `/plugin configure jira-sprint-workflow` and re-enter just the token):

| Config key | Example value | Purpose |
|---|---|---|
| `jira_url` | `https://acme.atlassian.net` | Your Jira Cloud base URL |
| `jira_username` | `jane@acme.com` | Your Jira account email |
| `jira_api_token` | `ATATT3xFfGF...` | API token (https://id.atlassian.com/manage-profile/security/api-tokens) |
| `project_key` | `ENG` | Jira project where Tasks + Subtasks live |
| `default_owner_account_id` | `712020:959044f3-fca7-4abf-8021-c8f2a85aaa74` | Default assignee for parent Tasks + Subtask 10 (Sandbox Manual Test) + Subtask 12 (Merge to Sandbox Branch) — find yours at `https://<your-jira>.atlassian.net/rest/api/3/myself` |
| `components` | `Backend,Frontend,Infrastructure` | Comma-separated Component names (must already exist in your Jira project) |
| `plans_directory` | `docs/plans` | Where implementation plans + test plans are saved in the repo |
| `specs_directory` | `docs/specs` | Where design specs are saved |

You can reconfigure later by either (a) running `/plugin configure jira-sprint-workflow` (CLI-driven, masks sensitive input) or (b) saying "reconfigure jira plugin" to re-trigger the `jira-setup` skill (interactive, validates live against Jira).

### Step 4 — Jira project prep (one-time)

Before your first story:

1. **Create your Components** in Jira (Project Settings → Components). These represent product areas that features can touch — e.g., `Backend`, `Frontend`, `Infrastructure`, `Auth`. Add them to the `components` user config as a comma-separated list.
2. **Confirm your project supports Sub-tasks.** Some team-managed projects call them `Subtask` (one word), others `Sub-task`. The skill detects this automatically but a missing issue type will surface as an error.
3. **Note your workflow's "Selected for Development" equivalent status.** The skill transitions parent Tasks to this after creating subtasks. If your workflow uses a different name (e.g. `Ready for Dev`, `To Do`, `Up Next`), the skill will offer the available transitions and you pick.

## Dependencies

This plugin assumes the `superpowers` plugin is installed (provides `brainstorming`, `writing-plans`, `subagent-driven-development`, `code-reviewer`, `using-git-worktrees`, `plan-document-reviewer`).

```
/plugin marketplace add anthropics/claude-plugins-official
/plugin install superpowers
```

It also assumes the **Atlassian MCP server** is configured — either:

- **Official Atlassian MCP** (OAuth — recommended for most teams):
  ```json
  {
    "mcpServers": {
      "atlassian": {
        "command": "npx",
        "args": ["-y", "mcp-remote@latest", "https://mcp.atlassian.com/v1/mcp"]
      }
    }
  }
  ```
- **Or** the sooperset/mcp-atlassian server (API-token based, more tools)

See https://github.com/atlassian/atlassian-mcp-server for setup.

## Example usage

Imagine your team is Acme Corp. Jane (tech lead) has intake ticket `ENG-42` that reads:

> **ENG-42** — Customer export needs a CSV download

From a Claude Code session in her repo:

```
jane: let's design ENG-42
```

The skill kicks in automatically (matches the trigger phrase), and:

1. Fetches ENG-42 from Jira
2. Invokes `superpowers:brainstorming` — Jane answers clarifying questions about scope, users, data model
3. Produces a design spec and writes it to `docs/specs/2026-04-21-customer-export-csv-design.md`
4. Runs the `spec-document-reviewer` loop, fixes any issues
5. Jane reviews and approves the spec
6. Invokes `superpowers:writing-plans` — produces the implementation plan + test plan
7. Runs `plan-document-reviewer` per chunk, fixes any issues
8. Updates ENG-42's description with the full spec (markdown)
9. Creates 15 subtasks (ENG-43 through ENG-57), each self-contained, each correctly assigned (Jane on parent Task + subtasks 10 and 12; unassigned on the rest)
10. Wires 16 "is blocked by" links per the chain (0→1, 0→1b, 1→2, ..., 12→13, 13→14)
11. Transitions ENG-42 to Selected for Development
12. Reports back with a table of every subtask and its state

A teammate (or a sub-agent) can now pick up ENG-43 ("0. Setup Branch & Worktree") and execute — everything they need is in the Jira description.

## Key design decisions

### Why two human checkpoints?

The chain has many sub-agent steps because AI is good at: implementation, code review, UI testing with Playwright, Docker builds, sandbox deploys, prod deploys. Two things AI can't reliably judge:

1. **"Does this feel right on a real-deployed instance?"** — that's subtask 10 (Sandbox Manual Test). The human walks the feature on a real sandbox environment using structured test cases (What / Preconditions / How / Expected / Pass criteria) that subtask 1b produced. Catches issues that local Docker doesn't surface (infra interactions, third-party integrations, real DNS, real auth flows).
2. **"Is this feature ready to join the release train?"** — that's subtask 12 (Merge to Sandbox Branch). The human's explicit `Approved — merge to sandbox branch` comment is the audit-trail signal that this feature passed both sandbox manual test AND architect review and should be promoted.

Architect review (subtask 7) runs BEFORE sandbox deploy — gates whether the diff is worth deploying. Local Docker (subtask 6) is automated smoke only, no human walkthrough.

### Why self-contained subtask descriptions?

Because distributed teams and sub-agents both benefit from the same thing: **context locality**. A teammate picks up subtask 4 (UI Testing) a week after it was designed — if the description says "see docs/…", they have to leave Jira, clone the repo, navigate to a directory, and read the file. If the description embeds the test cases inline, they start executing in 30 seconds.

The tradeoff is that specs/plans live in two places (the repo AND Jira). The repo is the source of truth for sub-agent execution (subagents read files); Jira is the source of truth for humans and context. The two stay in sync because the design phase writes both together.

### Why keep intake as `Task`, not promote to `Story`?

In Jira, both `Task` and `Story` can have subtasks. The issue type is a label, not a structural difference. Most intake lands as `Task` (from users, stakeholders, shift-left triage), and forcing a Task→Story conversion costs UI clicks with no structural benefit. We leave it as `Task` unless the team's workflow specifically requires `Story`.

## Known tool quirks (verified in practice)

The skill documents these so future sessions don't re-discover them:

- **Jira subtask assignee inheritance** — new subtasks silently inherit the parent's assignee if you don't pass `assignee_account_id` explicitly. Always re-fetch to verify.
- **`createIssueLink` direction is counterintuitive** — in our verified behavior, `inwardIssue` = the blocker and `outwardIssue` = the blocked. Docs say otherwise; verify by fetching one link after creation before batching.
- **~32KB description limit** — long implementation plans don't fit in a single subtask description. Strategy: outline + key code in Jira, full plan in repo file (sub-agents read both).
- **MCP tool occasional Cloudflare 403** during heavy batch operations. Fall back to raw REST with the `jira_api_token` via `bun -e`/`fetch`.
- **Markdown → ADF** via the MCP tool can double-escape brackets in checkboxes. Review rendered output and fix if needed.

## Contributing

PRs welcome. The plugin is designed to be team-agnostic via the `userConfig` — if you find yourself hardcoding a value that should be configurable, please file an issue.

## Author

Harsh Agarwal · harsh12101998@gmail.com · https://github.com/Harsh1210

## License

[MIT](./LICENSE)
