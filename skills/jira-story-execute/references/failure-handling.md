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

## Skipping fix subtasks (3 and 5) when there's nothing to fix

If the main subtask returns cleanly with no findings/failures, the paired fix subtask has no work. Don't dispatch a sub-agent for it — that wastes tokens on a no-op.

- **Subtask 2 (Code Review) returns 0 BLOCKER + 0 MAJOR findings** → skip subtask 3. Orchestrator posts `Skipped: subtask 2 returned 0 blockers / 0 majors — no fixes required.` on subtask 3 and transitions it to Done directly.
- **Subtask 4 (UI Testing) returns all PASS** → skip subtask 5. Orchestrator posts `Skipped: subtask 4 returned 18/18 PASS — no UI fixes required.` on subtask 5 and transitions it to Done directly.

In both cases, the orchestrator's next action is to advance to the subtask after the fix (i.e., 4 after skipping 3, or 6 after skipping 5). This was validated on the first live run: WFR-20 was skipped when WFR-19 cycle 2 returned 18/18 PASS, and the chain proceeded directly to WFR-21.

## Pre-existing bugs discovered mid-chain

A sub-agent sometimes discovers a bug unrelated to the Story's scope but blocking the chain's execution (e.g., a schema migration can't apply because of drift in a shared library). Options, in preferred order:

1. **Fix inline + flag for architect review** (default for small, defensive fixes)
   - Orchestrator acknowledges the discovery, produces a minimal fix on the feature branch
   - The fix gets its own commit with a clear `chore:` prefix and message explaining it's tech-debt adjacent to the feature
   - Subtask 8 (Architect Review) explicitly evaluates the "bundled-scope commits" decision: bundle with the Story's PR (with callout in the PR description) vs cherry-pick to a separate PR
   - Validated on WFR-5: bootstrap-migrations.ts drift was fixed inline, bundled with the Story's PR per architect recommendation

2. **Escalate and open a separate Jira task** (for larger fixes or scope-creep risks)
   - If the fix is non-trivial (>1 file, >20 lines, or crosses module boundaries), escalate to the user
   - Open a new Jira task for the fix; link to the blocked Story as "is blocked by"
   - Pause the Story's chain until the fix ships via its own chain

3. **Work around** (last resort — only if fix is unsafe from the current environment)
   - Example: applying a migration manually from `docker exec` because bootstrap isn't fixed yet
   - MUST be accompanied by option 1 or 2 — workarounds leave the root cause in place

### What to avoid

- Don't iterate on the fix more than 2 cycles without escalating. The WFR-5 run hit 3 commits on bootstrap-migrations.ts before getting it right — that should have escalated sooner.
- Don't mix fix commits and Story commits on the same code change. Keep bundled commits separately-authored so they can be cherry-picked later if needed.

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
