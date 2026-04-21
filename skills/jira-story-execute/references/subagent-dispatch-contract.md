# Sub-agent dispatch contract

This is the exact contract the orchestrating skill (`jira-story-execute`) imposes on every sub-agent it spawns. It keeps Jira state accurate during execution so crashes or timeouts don't leave orphaned work.

## Sandbox environment — known gotchas

**The orchestrator MUST include these notes in every sub-agent dispatch prompt**, because dispatched sub-agents run in restricted shells and will waste tool calls rediscovering them:

1. **Jira MCP token may expire mid-run.** If `mcp__claude_ai_Atlassian_Rovo__*` returns `"re-authorization needed"` or `"not connected"` errors, immediately fall back to raw REST. Credentials live at `~/.claude/projects/<project>/memory/reference_jira.md` (or equivalent per the user's auto-memory path). Pattern:
   ```js
   const auth = Buffer.from(`${JIRA_USERNAME}:${JIRA_API_TOKEN}`).toString("base64");
   await fetch(`${JIRA_URL}/rest/api/3/...`, {
     method: ..., headers: { "Authorization": "Basic " + auth, "Content-Type": "application/json", "Accept": "application/json" },
     body: JSON.stringify(...)
   });
   ```
   Every sub-agent that touches Jira should know this fallback — don't make them rediscover it.

2. **Common shell tools are NOT in the sub-agent's PATH.** At minimum these have been observed missing: `cat`, `head`, `grep`, `mkdir`, `git`, `docker`, `python3`. Use absolute paths (`/usr/bin/git`, `/usr/local/bin/docker`, `/opt/homebrew/bin/gh`, etc.) or the orchestrator's Read/Grep/Glob tools. `bun` is in PATH and can substitute for file/directory operations:
   ```js
   // make a dir (mkdir unavailable)
   require("fs").mkdirSync("/tmp/work", { recursive: true })
   ```

3. **Side-effects against shared state need explicit authorization.** If the sub-agent needs to `docker exec` against a local Postgres, modify shared config, or write to anything outside its worktree, the harness's permission hooks may block. The sub-agent should escalate cleanly (`VERDICT: BLOCKED` + clear request) rather than retry or try to work around.

4. **Bun is first-class.** Most sub-agents can accomplish everything they need with `bun -e '<script>'` + `fetch()` for HTTP and Node's `fs`/`child_process` for local work. When in doubt, script in bun instead of relying on shell composition.



## Required prelude (sub-agent's first actions)

Before doing any real work, every dispatched sub-agent must:

1. **Transition its subtask to `In Progress`**
   ```
   mcp__claude_ai_Atlassian_Rovo__transitionJiraIssue
     cloudId: ${user_config.jira_url}
     issueIdOrKey: <SUBTASK-KEY>
     transition: { id: <transition-id-for-"In Progress"> }
   ```

2. **Check for existing progress comments (resumption)**

   Read the subtask's recent comments via `getJiraIssue` with `fields=comment`. Look for comments whose bodies start with `Started:`, `Progress:`, or `Resuming:`. If any exist:
   - This is a resumption — a prior sub-agent started but didn't finish
   - Summarize the last checkpoint you can infer
   - Post a comment: `Resuming: previous run reached <checkpoint>. Continuing from there.`

   If none exist (fresh start):
   - Post a comment: `Started: <1-line description of what you're about to do>`

3. **Chdir to the worktree** specified in the dispatch payload (from subtask 0's completion comment). All subsequent work happens inside this directory. Never work from the original repo root.

## Incremental progress comments

Post progress comments at natural checkpoint boundaries:

| Subtask | Checkpoint triggers |
|---|---|
| 0 (Setup Branch) | After worktree created; after packages installed; after Docker up |
| 1 (Implement) | After each chunk completed + committed |
| 1b (Test Cases) | After each test-plan section drafted (Automated / Manual) |
| 2 (Code Review) | After each category reviewed (functional, security, performance, etc.) |
| 3 (Fix Code Review) | After each finding batch resolved |
| 4 (UI Testing) | After each test case executed (pass/fail row added to table) |
| 5 (Fix UI Testing) | After each failure resolved |
| 6 (Local Deploy) | After build; after containers restarted; after smoke check |
| 7 (Manual Testing) | Human-driven — no sub-agent |
| 8 (Architect Review) | After each review angle (correctness, cross-system, security, etc.) |
| 9 (Push to Prod) | After merge; after deploy started; after deploy confirmed |
| 10 (Prod Sanity) | After each critical case executed |

**Format:**
```
Progress: <what just finished> — <optional metric or detail>
```

Example for subtask 1:
```
Progress: Chunk 2 complete — 4 files created, 2 modified, 1 commit pushed (abc1234).
```

## Required return message (end of sub-agent execution)

When the sub-agent finishes (successfully or not), it must:

1. **Post a final Jira comment** with verdict:
   - Success: `Completed: <1-3 sentence summary>. Acceptance criteria met: <bullet list>. Commits: <list if any>.`
   - Partial: `Partial: <summary>. Completed: <list>. Remaining: <list with reason>. Next sub-agent should start from: <checkpoint>.`
   - Failure: `Failed: <summary of what failed>. Last checkpoint: <where it got to>. Blocker: <description if blocked, else "none — retry may succeed">.`

2. **Transition the subtask status** appropriately:
   - Success → `Done`
   - Partial or Failure → leave at `In Progress` (do NOT transition to Done) so the orchestrator knows a retry/resume is needed

3. **Return a plain-text summary** to the orchestrator's tool-use result, formatted as:

   ```
   VERDICT: PASS | FAIL | BLOCKED
   JIRA_STATUS: Done | In Progress | <other>
   SUMMARY: <one or two sentences describing what happened>
   LOOP_BACK: <subtask-key or "none"> — only set if this is a fix subtask (3 or 5) and the main subtask (2 or 4) needs re-run
   NEXT_ACTION: <what the orchestrator should do next, e.g. "advance", "re-dispatch <key>", "escalate to human">
   ```

   The orchestrator parses these fields to decide whether to advance, loop, or escalate.

## Dispatch payload from the orchestrator

When the orchestrator spawns a sub-agent, the prompt includes:

```
You are executing subtask <SUBTASK-KEY> of Jira Story <STORY-KEY> on behalf of the jira-story-execute orchestrator.

**Jira config:**
  - Cloud URL: ${user_config.jira_url}
  - Cloud ID: <auto-resolved or from config>

**Your subtask:**
  <full Jira description of <SUBTASK-KEY>, fetched via getJiraIssue at dispatch time>

**Worktree:** <absolute path from subtask 0's completion comment>
**Branch:** <branch name from subtask 0's completion comment>
**Parent Story:** <STORY-KEY> (read its description for spec context if needed)

**Contract (non-negotiable):**
  1. Before any work: transition <SUBTASK-KEY> to "In Progress" via Atlassian MCP.
  2. Read recent comments on <SUBTASK-KEY> — if there's evidence of a prior incomplete run, post a "Resuming:" comment and continue from the last checkpoint.
  3. Otherwise post a "Started:" comment.
  4. Work inside the worktree. Post "Progress:" comments at natural checkpoint boundaries (see contract doc).
  5. At end:
     - Post "Completed:" / "Partial:" / "Failed:" comment.
     - Transition status to Done (on success) or leave In Progress (otherwise).
  6. Return a final message with fields VERDICT / JIRA_STATUS / SUMMARY / LOOP_BACK / NEXT_ACTION.

**Tools available:** all — including Atlassian MCP, Bash, Read/Write/Edit, Grep, Glob, and other skills.

Execute now.
```

## Handling sub-skill invocations from within the dispatched sub-agent

Some subtasks explicitly call out sub-skills the sub-agent should invoke:

- Subtask 1 (Implement) → use `superpowers:subagent-driven-development` to run chunk-by-chunk
- Subtask 2 (Code Review) → use `superpowers:code-reviewer` with the cross-system checklist
- Subtask 8 (Architect Review) → use `superpowers:code-reviewer` framed as enterprise architect, 10 angles

These are nested sub-agent dispatches. The outer sub-agent (spawned by this orchestrator) treats the inner sub-skill's output as one more progress checkpoint.

## What the sub-agent MUST NOT do

- Do NOT advance to the next subtask. That's the orchestrator's job.
- Do NOT transition sibling or parent Jira issues. Scope: this one subtask.
- Do NOT skip the prelude (transition + comment). Resumption depends on it.
- Do NOT return vague "done" messages. The orchestrator parses VERDICT / NEXT_ACTION fields — missing them breaks the chain.
- Do NOT work outside the assigned worktree. Mixing workspaces corrupts other work.
- Do NOT commit to `main` directly (that's subtask 9's job only).
