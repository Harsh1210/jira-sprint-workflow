# Failure handling & retry patterns

How `jira-story-execute` reacts when things go wrong. The goal is clear escalation — never silently swallow a failure, never infinite-loop, always leave Jira in a state the next run can pick up.

## Classification

Every subtask's outcome falls into one of five buckets:

| Outcome | Trigger | Orchestrator action |
|---|---|---|
| PASS | Sub-agent returns `VERDICT: PASS` + subtask is `Done` in Jira | Advance to next subtask in chain |
| FAIL (expected) | Sub-agent returns `VERDICT: FAIL` AND subtask is a main subtask paired with a fix sibling (2 or 4) | Advance to the fix sibling (3 or 5) |
| FIX-DONE | Sub-agent returns `VERDICT: PASS` on a fix subtask (3 or 5) | Re-dispatch the PAIRED main subtask (2 or 4) — not advance |
| BLOCKED | Sub-agent returns `VERDICT: BLOCKED` | Escalate to user immediately |
| ERROR | Sub-agent crashes, times out, or returns malformed output | Retry once with fresh context; if still errors, escalate |

## Fix-loop state machine

```
    ┌──────────────────┐
    │ 2 Code Review    │
    │ (dispatch main)  │
    └────────┬─────────┘
             │
    ┌────────┴──────────┐
    │ verdict?          │
    └────────┬──────────┘
             │
     ┌───────┴────────┐
     │ PASS           │ FAIL
     ↓                ↓
    Advance to 4   Dispatch 3 Fix
                     │
                   (3 completes)
                     │
                   Re-dispatch 2 ◄── (loop back)
                     │
                   cycle count++
                     │
                   if cycle > 3 ─► Escalate
```

Same shape for 4 ↔ 5.

**Cycle counter:** track per-pair (2↔3 and 4↔5 independently). Reset at start of chain, not per invocation. If the skill is invoked multiple times on the same Story, read existing cycle counts from recent comments (search for `Cycle N` markers) to continue the count.

**Comment format for cycles:** when dispatching a fix, the orchestrator adds a comment: `Cycle N: fix dispatched in response to M findings/failures in <main-subtask-key>`. When re-dispatching the main, it adds: `Cycle N: re-review / re-test after <fix-subtask-key> completed`.

## Escalation triggers (skill exits immediately)

- Sub-agent returns `VERDICT: BLOCKED` with a blocker description
- Fix loop exceeds 3 cycles on the same pair
- Atlassian MCP or Jira API errors for more than 2 consecutive attempts
- Subtask 7 reached (human checkpoint — this is "normal" escalation, not a failure)
- Subtask 8 (Architect Review) returns any `Block` verdict on its 10 angles
- Git operations fail (merge conflicts, push rejected, missing remote)

## Resumption after escalation

When the user re-invokes the skill after an escalation:

1. Skill re-reads all subtask states + recent comments
2. If the last escalated subtask is now `Done` (user fixed it manually): advance past it
3. If still `In Progress` with unresolved blocker: re-read the latest comments to see if the user resolved the blocker since the last session; if so, resume; if not, escalate again with an updated summary

Never assume a blocker is resolved without evidence in Jira. The skill's "memory" is Jira state + comments, not session context.

## Dispatching with resumption context

When the skill re-dispatches a subtask that was previously attempted (either via fix-loop or escalation-then-resume), the dispatch prompt includes:

```
PRIOR ATTEMPTS: <N>
LAST ATTEMPT STATUS: <what the previous run posted as its final comment>
FIX COMMITS SINCE LAST ATTEMPT: <commit hashes from any fix subtask that ran between prior attempt and now>
LOOP CYCLE: <N of 3>

Your job this cycle: re-run the subtask against the current branch state. Post a fresh comment documenting what you find this cycle.
```

This prevents the sub-agent from duplicating prior work or missing that a fix was made.

## Idempotency in side effects

Every operation the skill (or sub-agents) perform should be safe to retry:

| Operation | Idempotency check |
|---|---|
| Transition subtask status | Before transitioning, check current status — skip if already at target |
| Post Jira comment | Duplicate comments are harmless but noisy; include a cycle/attempt marker so a reader can distinguish them |
| Create git branch | `git branch -M main` is idempotent; `git checkout -b <branch>` will fail if the branch exists — use `git checkout <branch> 2>/dev/null || git checkout -b <branch>` |
| `npm install` / `bun install` | Idempotent by design |
| Docker restart | Idempotent |
| Merge to main (subtask 9) | Use `git merge --no-ff`; if already merged, the command is a no-op |

## When in doubt, escalate

The bias of this system is to escalate rather than guess. The user's time is worth more than one unnecessary escalation. If a sub-agent is uncertain about how to proceed, or if the orchestrator sees inconsistent Jira state (e.g., a subtask marked Done but no Completed comment), pause and ask.

---

## Monitoring background agents — don't trust context alone

**The autonomy-breaking bug to avoid:** when a background agent completes, the orchestrator receives a `<task-notification>` as a tool result. That tool result can roll out of the conversation context window before the orchestrator acts on it — especially during long runs where many other tool calls stack up. The orchestrator then incorrectly reports "still running" when the agent has actually failed, escalated, or completed minutes ago.

**The fix:** on every tick (cron poll, user status query, next-step decision), the orchestrator MUST explicitly poll `TaskOutput(task_id, block: false)` for every known running task_id. Do NOT rely on notifications being in-context. This adds a cheap check that catches silently-completed runs.

### Required polling pattern

Maintain a list of active task IDs (set when you dispatch each background Agent). At every tick:

```
for each active_task_id:
  result = TaskOutput(task_id: active_task_id, block: false, timeout: 5000)
  if result.status == "completed":
    parse the VERDICT/JIRA_STATUS from result.output
    advance / loop / escalate per failure-handling rules
    remove from active list
  elif result.status == "failed" or "killed":
    post Failed: comment on the subtask
    escalate
    remove from active list
  else:  # still running
    check Jira for latest Progress: comment and relay to the user
```

If the orchestrator skips this check, it's making decisions based on stale in-context notifications — and a notification that's been pushed out of context simply doesn't exist from the orchestrator's perspective. Explicit polling is the only reliable state source for long-running sessions.

### Why Jira comments alone aren't sufficient

Jira's `Failed:` / `Completed:` comments reflect what the sub-agent posted. They DON'T reflect whether the Agent tool task itself crashed (process died before posting), timed out, or was `TaskStop`'d externally. `TaskOutput` reports the task-level state, which is the ground truth. Jira is the workstream state; TaskOutput is the runtime state. Both must be checked.
