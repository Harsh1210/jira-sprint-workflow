---
name: jira-story-execute
description: Execute a Jira Story that's been designed with jira-story-design — autonomously walks the 11-subtask chain (Setup → Implement ∥ Test Cases → Code Review → Fix → UI Test → Fix → Local Deploy → Manual Test [human] → Architect Review → Push to Prod → Prod Sanity), pausing only at the human checkpoint. Use whenever the user says "pick up <STORY-KEY>", "execute <STORY-KEY>", "run this Story", "continue <STORY-KEY>", or wants to start or resume development on a Jira ticket that already has its subtasks created. Sub-agents update Jira status + post incremental comments as they work so a crashed or paused run can be resumed from the next invocation.
---

# Jira Story Execute

This skill autonomously walks a designed Jira Story through its 11-subtask chain to Done. It pauses only at the one human checkpoint (subtask 7, Manual Testing) and escalates if any sub-agent reports a blocker.

Companion to `jira-story-design`, which creates the Story + subtasks. This skill consumes them.

## Core principle: sub-agents update Jira as they work

Every sub-agent this skill dispatches MUST:
1. Transition its subtask to `In Progress` when it starts
2. Post a `Started` comment immediately with a short "here's what I'm about to do" note
3. Post incremental progress comments at key milestones (per chunk, per test case batch, per finding, etc.)
4. Post a final `Completed` or `Failed` comment with a PASS/FAIL/BLOCKED verdict
5. Transition the subtask to `Done` (on PASS) or leave `In Progress` (on FAIL/BLOCKED)

**Why:** if a sub-agent crashes, times out, or the session ends mid-chain, the next invocation of this skill reads the Jira state + comments and resumes from where the last sub-agent left off. Without these updates, resumption is impossible.

This rule is non-negotiable. The sub-agent dispatch contract (in `references/subagent-dispatch-contract.md`) enforces it.

## Config gate (run before anything else)

Read `~/.claude/settings.json` and verify `pluginConfigs["jira-sprint-workflow@jira-sprint-workflow"].options` has non-empty values for all eight fields: `jira_url`, `jira_username`, `jira_api_token`, `project_key`, `default_owner_account_id`, `components`, `plans_directory`, `specs_directory`. If any are missing:

1. Tell the user: "Plugin config is incomplete — I'll run the setup walkthrough first (~2 min), then come back and execute."
2. Invoke the `jira-setup` skill.
3. When it returns, resume from the user's "pick up/execute/continue `<KEY>`" prompt.

## Environment assumption

**Single-user, single-worktree.** The user who invokes this skill is the only person working in the worktree that subtask 0 created. All sub-agents this skill dispatches inherit that worktree — they chdir into it before doing anything.

If a second person needs to work on the same Story in parallel (e.g., to unblock a stuck run), they create their own worktree from the same branch. Do not race two orchestrators on one worktree.

## Orchestration flow

```
On invocation:
  1. Determine Story key (from user prompt or ask)
  2. Fetch Story + all 11 subtasks + their statuses + recent comments
  3. Read subtask 0's completion comment for worktree path + branch name
  4. Determine next eligible subtask:
       - First in chain where status IS NOT Done
       - All its "is blocked by" predecessors ARE Done
  5. Dispatch sub-agent for that subtask (see dispatch contract below)
  6. Wait for sub-agent completion
  7. Read subtask's latest Jira state + sub-agent's return message
  8. Advance, loop, or escalate per the rules below
  9. Go to step 4 until Story reaches Done or a pause condition fires
```

### Pause conditions (stop the chain and return control to the user)

| Condition | Action |
|---|---|
| Next subtask is #7 (Manual Testing) | Transition #7 to `In Progress`, ensure it's assigned to the parent Story's assignee, tell the user: "Subtask #7 is ready. Walk through the Manual Test Cases in its description. When done, transition it to Done in Jira (or say 'continue') and I'll resume." Exit. |
| Sub-agent reports BLOCKED | Post comment on the subtask describing the block, tell the user with enough context to triage. Exit. |
| A fix loop exceeds 3 cycles | Post comment "Fix loop exhausted after 3 cycles; escalating to human" on the affected subtasks. Exit. |
| Atlassian MCP / Jira API errors for >2 consecutive attempts | Report the error to the user. Exit. |
| Story reaches Done (subtask 10 complete) | Transition parent Story to Done. Post summary comment on parent. Exit with success message. |

## Sub-agent dispatch contract

For each subtask, dispatch a **general-purpose sub-agent** (via the Agent tool) with:

- **The full subtask description** from Jira (already self-contained per Phase 1's design discipline)
- **The worktree path** from subtask 0's comment
- **The parent Story key** for cross-reference
- **Explicit instructions** to update Jira status + comments as they work (see `references/subagent-dispatch-contract.md`)
- **A required return format** that signals PASS / FAIL / BLOCKED + the subtask's current Jira state

The dispatch uses `superpowers:subagent-driven-development` conventions but with Jira state tracking layered on.

Full contract: `references/subagent-dispatch-contract.md`.

## Fix-loop rules

Two subtasks can loop with their "fix" siblings:

| Main subtask | Fix subtask | Loop trigger |
|---|---|---|
| 2 (Code Review) | 3 (Fix Code Review) | #2's comment has unresolved BLOCKER or MAJOR findings after #3 completes → rerun #2 on the updated diff |
| 4 (UI Testing) | 5 (Fix UI Testing) | #4's comparison table has any FAIL row after #5 completes → rerun #4 |

**Loop cap:** 3 cycles. After the 3rd failed cycle, escalate to the user. This prevents infinite loops if a sub-agent can't actually fix what the review/test found.

When looping, the skill re-dispatches the main subtask (not the fix) — the fix subtask's Done state is preserved as a historical record. The re-dispatched main subtask picks up the new diff, finds what (if anything) still fails, and posts a new comment.

## Human checkpoint (subtask 7)

When the chain reaches subtask 7:
1. Skill transitions #7 to `In Progress`
2. Verifies the assignee is set (should be the parent Story's owner)
3. Posts a comment on #7: `Ready for manual testing. @<owner>, please walk through the Manual Test Cases and transition to Done when passing, or comment with issues to trigger a fix cycle.`
4. Tells the user (the session): "Subtask 7 (WFR-XX) is ready for you. Once you've done the manual test and transitioned it to Done in Jira — or come back here and say 'continue WFR-5' — I'll resume with subtask 8."
5. Exits the skill. Does NOT keep polling.

When the user resumes:
- Re-invoke the skill with the same Story key
- Skill reads state, sees #7 is Done (or sees comments indicating issues), and advances to #8 OR loops back through #5 → #6 → #7

If the user flags issues (comments on #7 indicating "fails"): skill re-dispatches subtask 5 Fix UI Testing with the user's feedback, and re-runs the 5 → 6 → 7 cycle (also capped at 3 cycles before escalation).

## Resume semantics

Any time this skill is invoked, it's idempotent-ish:
1. Fetch the current Jira state
2. Find the first subtask that isn't Done
3. Dispatch from there

Comments left by prior runs are not re-written. Sub-agents use their incremental comments as context ("I can see the previous run got to chunk 3; I'll continue from chunk 4").

If a subtask is currently `In Progress` with a `Started` comment but no `Completed` comment:
- The previous sub-agent likely crashed or was killed
- New sub-agent reads the existing progress comments and picks up from the last checkpoint
- New sub-agent's first action: post a comment `Resuming after previous run. Last checkpoint: <summary>.`

## Parent Story transitions

- **When subtask 0 starts:** transition parent Story from `Selected for Development` to `In Progress` (find transition via `getTransitionsForJiraIssue`).
- **When subtask 10 completes successfully:** transition parent Story to `Done`. Post a summary comment on the parent with commit range + production URL + link to prod sanity test results.

If either transition fails (permission, invalid status), skip it and warn the user — execution continues regardless.

## Return format for the user (end-of-session summary)

When the skill exits (for any reason), it produces a concise summary:

```
Story: WFR-5 "Agent list for Shae (Export Framework + Agents export)"
Exit reason: <paused-at-human-checkpoint | completed | blocked | error>

Progress:
  ✅ Subtask 0  (Setup Branch & Worktree)       — Done
  ✅ Subtask 1  (Implement Feature)             — Done
  ✅ Subtask 1b (Plan Functional Test Cases)    — Done
  ✅ Subtask 2  (Code Review)                   — Done
  ✅ Subtask 3  (Fix Code Review Issues)        — Done
  ✅ Subtask 4  (UI Testing)                    — Done (2 cycles)
  ✅ Subtask 5  (Fix UI Testing Issues)         — Done
  ✅ Subtask 6  (Deploy to Local Docker)        — Done
  🚶 Subtask 7  (Manual Testing)                — Waiting for Harsh
  ⏳ Subtask 8  (Senior Architect Review)       — Blocked by 7
  ⏳ Subtask 9  (Push to Prod)                  — Blocked by 8
  ⏳ Subtask 10 (Prod Sanity Test)              — Blocked by 9

Next action: [what the user needs to do]
Reinvoke: "continue WFR-5"
```

## Configuration (inherits from plugin)

Reads the same user config as `jira-story-design`:
- `jira_url`, `jira_username`, `jira_api_token`
- `project_key`
- `default_owner_account_id` (used for comment @-mentions)
- `components` (for validation only; execution doesn't create new subtasks)
- `plans_directory` (sub-agents may need to read plan/test-plan files; path reference)

## What NOT to do

- Don't advance past a subtask whose status is still `In Progress` — that means the sub-agent didn't complete. Re-dispatch or escalate instead.
- Don't skip a subtask's "is blocked by" predecessors. The chain's gating is what makes sub-agent execution safe.
- Don't dispatch sub-agents in parallel for dependent subtasks. Only #1 and #1b are allowed to run concurrently.
- Don't continue past subtask 7 without the human's explicit approval (either Jira transition to Done, or the user saying "continue" in the session).
- Don't retry a subtask more than 3 times in one chain execution. Escalate after 3.
- Don't silently swallow sub-agent failures. Post a `Failed` comment on the subtask with the sub-agent's error output, and escalate.

## Related skills

- `jira-story-design` — creates the Story + 11 subtasks (prerequisite for this skill to have anything to execute)
- `jira-architect-review` — used by subtask 8; right-sizes the review to 3 / 5 / 7 / 10 angles based on diff size
- `superpowers:subagent-driven-development` — pattern this skill uses to dispatch per-subtask sub-agents
- `superpowers:using-git-worktrees` — referenced by subtask 0 (already done by the time this skill runs)
- `superpowers:code-reviewer` — used by subtask 2 (Code Review) and as a fallback for subtask 8

## References

- `references/subagent-dispatch-contract.md` — exact instructions and return format for sub-agents
- `references/failure-handling.md` — retry patterns, resumption rules, escalation triggers

Read these when you reach the relevant step. Don't load up-front.
