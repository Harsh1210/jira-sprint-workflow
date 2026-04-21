---
name: jira-architect-review
description: Perform a right-sized enterprise architect review on a feature branch for a Jira Story — autodetects feature size from the diff and applies the matching review checklist (3 angles for minor fixes, 5 for small features, 7 for medium, all 10 for large / framework-level changes). Use whenever the user says "review this branch", "architect review", "prod readiness review", or whenever subtask 8 of the jira-sprint-workflow chain fires. The sub-dispatched reviewer scales effort to the change — a 3-line typo fix does not need a 10-angle audit, and a 3000-line framework change should not get a 2-angle rubber stamp. Output is a categorized Jira comment with pass/block per angle plus a bundled-scope decision if out-of-scope commits rode along.
---

# Jira Architect Review

Run a right-sized enterprise architect review on a feature branch. Sub-dispatched during subtask 8 of the `jira-sprint-workflow` chain, or manually when the user wants a production-readiness check.

The review scales effort to the change:

- **Minor changes** (typo, copy tweak, 1-line fix): 3 angles — correctness, security basics, rollback. ~2 min.
- **Small features** (typical bug fix or small new feature): 5 angles — adds data integrity + UX polish. ~5 min.
- **Medium features** (standard Story): 7 angles — adds cross-system + performance. ~10 min.
- **Large features** (framework-level, multi-module, schema migrations): all 10 angles. ~15+ min.

A 3-line typo fix should not get a 10-angle audit, and a 3000-line framework change should not get a rubber-stamp. The skill autodetects size and picks the matching angle set.

## Core principle: right-size the review to the risk

The 10-angle review was designed for framework-level changes. Applying it to every small bug fix wastes architect time and creates review fatigue (which leads to rubber-stamping). The 3 always-on angles (correctness, security, rollback) catch 80% of shipping risk on small changes; the remaining 7 are added as the surface area grows.

## When the skill activates

- Subtask 8 in the `jira-sprint-workflow` chain fires
- User explicitly invokes: "architect-review this branch", "prod-readiness on WFR-X", "review the diff for WFR-5"
- Someone finishing a feature wants a final check before opening a PR

If the user gives a Story key, fetch the parent Task + branch info + any prior review findings.

## Config gate (run before step 1)

This skill reads the same plugin user config as the other skills (`jira_url`, `jira_username`, `jira_api_token`, `project_key`, etc.). Before proceeding, verify `pluginConfigs["jira-sprint-workflow@jira-sprint-workflow"].options` in `~/.claude/settings.json` has all eight fields populated. If any are missing, invoke the `jira-setup` skill first, then resume.

## Step-by-step

### Step 1 — Read the parent Task + branch

Fetch from Jira:
- Parent Task description (spec, requirements, out-of-scope)
- Subtask 2 code review comment (if present — context for what's already been caught)
- Subtask 4 UI testing comment (if present)
- Subtask 7 manual-test comment (if present — tells you what the human verified)

Fetch from git:
- Full diff `git diff main...<feature-branch>` (or `HEAD` of the worktree vs `origin/main`)
- Commit list `git log main..HEAD --oneline`
- `git diff main...HEAD --stat` (files + lines changed summary)

### Step 2 — Size the feature

Compute these metrics from the diff:

| Signal | Small boundary | Medium boundary | Large boundary |
|---|---|---|---|
| Files changed | ≤ 3 | ≤ 15 | > 15 |
| Lines added + deleted | ≤ 100 | ≤ 500 | > 500 |
| Migration files present | — | Any | Multiple |
| New API endpoints | 0 | 1–3 | 4+ |
| New modules (`backend/src/modules/X/`, `frontend/src/components/X/`) | 0 | 1 | 2+ |
| Touches auth / security-sensitive paths (`authenticate`, `authorize`, `secrets`, `JWT`, cookies) | — | Any | Multiple |
| Touches shared data stores (cross-service DB tables, shared caches) | — | — | Any |

**Size decision:**
- **Minor** — all signals at the smallest end AND no migration AND no new endpoints
- **Small** — most signals in "small" range, maybe one in "medium"
- **Medium** — signals in "medium" range, maybe one in "large"
- **Large** — any "large" signal, OR new module, OR migration + new endpoints, OR touches auth

Write the chosen size to the Jira comment's first line so it's auditable:
```
Review size: LARGE — 32 files, +3925/-59 lines, 1 migration, 5 new endpoints, 1 new module. Using all 10 angles.
```

### Step 3 — Select the angle set

Use `references/angles-library.md` for the full definitions. Required subset by size:

| Angle | Minor | Small | Medium | Large |
|---|---|---|---|---|
| 1. Correctness vs spec | ✅ | ✅ | ✅ | ✅ |
| 2. Cross-system impact | — | — | ✅ | ✅ |
| 3. Security | ✅ | ✅ | ✅ | ✅ |
| 4. Performance at prod scale | — | — | ✅ | ✅ |
| 5. Data integrity | — | ✅ | ✅ | ✅ |
| 6. Integration touchpoints | — | — | — | ✅ |
| 7. Auth flow | — | — | — | ✅ |
| 8. Rollback / kill switch | ✅ | ✅ | ✅ | ✅ |
| 9. Observability | — | — | — | ✅ |
| 10. UX polish | — | ✅ | ✅ | ✅ |
| **Angles total** | **3** | **5** | **7** | **10** |

**Exception — always include an angle if:**
- The diff touches `authenticate.ts`, `authorize.ts`, or any auth/session file → force include Auth (7) regardless of size
- The diff includes a migration → force include Data integrity (5)
- The diff touches anything under `frontend/src/` and size is Small+ → force include UX polish (10)
- The diff adds new observability-worthy code paths (cron jobs, background workers) → force include Observability (9)

Document any forced inclusions in the "Review size:" line so they're explicit.

### Step 4 — Run the review

For each selected angle:
1. Open the relevant files in the diff
2. Against the angle's checklist (in `references/angles-library.md`), evaluate
3. Mark ✅ Pass (brief justification) or ❌ Block (what needs to change before prod)

If any angle is Block, the overall verdict is BLOCK even if all others pass.

### Step 5 — Bundled-scope decision

If the branch includes commits that are OUT-OF-SCOPE for the parent Story (e.g., pre-existing tech-debt fixes bundled because they unblocked the feature), make an explicit decision:

- **Bundle** — keep them in this PR. Required if the fix is necessary for the feature to work correctly in prod (e.g., migration-tracking repair that unblocks new migrations).
- **Cherry-pick** — extract to a separate PR. Recommended if the fix is independent and could ship on its own, OR if bundling would obscure the Story's diff.

Document the decision and rationale in the review comment.

### Step 6 — Post the review to Jira

Post ONE consolidated comment on the subtask (or wherever the review was requested). Structure:

```
Architect Review — <parent-key> <parent-summary>

Review size: <SIZE> — <signals summary>. Using <N> angles.

===== ANGLE REVIEW =====

1. <Angle name> — PASS | BLOCK
<brief justification or block details>

2. ...

===== BUNDLED SCOPE =====

<only if applicable — bundle/cherry-pick decision with rationale>

===== SUMMARY =====

- Angles Pass: <N>
- Angles Block: <N>
- Overall verdict: PASS | BLOCK
- Bundled scope: bundle / cherry-pick / N/A
- Minor non-blocking follow-ups: <list or "none">

<Final line: either "Architect review passed — cleared for prod" or "Architect review blocked: <count> blockers — see above">
```

Transition the subtask to Done if the review is complete (pass or block — the review itself is done). The orchestrator decides next action based on verdict.

## Return format

```
VERDICT: PASS | BLOCK
SIZE: minor | small | medium | large
ANGLES_RUN: <N>
ANGLES_PASS: <N>
ANGLES_BLOCK: <N>
BUNDLED_SCOPE_DECISION: bundle | cherry-pick | N/A
NEXT_ACTION: advance_to_prod | back_to_fix | escalate
```

## References

- `references/angles-library.md` — full checklist per angle, with specific things to look for
- `references/sizing-heuristics.md` — examples of real changes at each size tier to calibrate

Read these only when you reach the relevant step.

## Known environment notes

- MCP may disconnect mid-review — fall back to raw REST via creds in the user's auto-memory reference file
- Sandbox PATH may lack `cat`, `head`, `grep`, `git`, `docker` — use absolute paths (`/usr/bin/git`, `/opt/homebrew/bin/gh`) or Read/Grep tools
- `git diff` output can be huge on large reviews — pipe to a file under `/tmp/` rather than into your context if >500 lines
- If the diff exceeds what fits in review context, sample: read the most risky files first (migrations, auth, security, public endpoints)

## Related skills

- `jira-story-design` — creates the Story + subtasks; this skill is dispatched from subtask 8 of the resulting chain
- `jira-story-execute` — runs the 11-subtask chain; invokes this skill for subtask 8
- `superpowers:code-reviewer` — generic code-review skill. This one is focused on architecture + production readiness, at the size-adjusted depth.
