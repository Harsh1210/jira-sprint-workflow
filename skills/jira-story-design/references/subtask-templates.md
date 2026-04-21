# Subtask description templates

Each of the 11 subtasks has a specific role, instructions, and acceptance criteria. Every subtask description is **self-contained** — a team member should be able to execute a subtask using only the Jira issue.

## Universal skeleton

Every subtask description must contain:

```markdown
**Role:** [Title] — expert in [domain]

**Context:**

[2-3 paragraph summary of what the feature is and why it exists. Pull from the parent Task's Design/Spec section — you can condense but cover: what's being built, the motivation, and the key architectural choices. A teammate picking up THIS subtask gets enough context here to work without reading the parent.]

**Parent Task:** <PARENT-KEY> (full spec + dependencies live there; read if you need more depth)

**Worktree:** [path, documented in subtask 0's completion comment]
**Branch:** [name, documented in subtask 0's completion comment]

[Subtask-specific content goes here — plan chunks, test cases, review checklist, etc.]

**Acceptance Criteria:**
- [Bullet points of what "done" looks like for THIS subtask]
```

## Subtask 0 — Setup Branch & Worktree

**Assignee:** unassigned (Sub Agent)

```markdown
**Role:** Senior Staff DevOps Engineer — expert in git worktrees, branch management, local Docker environments

**Context:**

[Feature summary, 2-3 paragraphs from parent Task Design/Spec]

**Parent Task:** <PARENT-KEY>

**Uses skill:** `superpowers:using-git-worktrees`

**Instructions:**

1. Read parent Task <PARENT-KEY> for the feature name and which repo it affects (map the Component to the appropriate repo in your team's conventions)
2. Create a git worktree:
   - Path: `~/Documents/workspace/<repo>-worktrees/<PARENT-KEY>-<slug>/`
   - Branch: `feat/<PARENT-KEY>-<slug>` off `main`
3. Copy `.env` and any local-only config from the main workspace to the worktree (never debug config that already works — see `feedback_worktree_env`)
4. Run package installs inside the worktree (`npm install` or `bun install`)
5. Bring up the Docker stack from the worktree (`docker compose up -d` or equivalent). **NEVER run `docker compose down -v` — it wipes the local DB which has real production data.**
6. Verify the stack is accessible (http://localhost or whatever the repo uses)
7. Post a Jira comment on THIS subtask documenting:
   - Worktree path
   - Branch name
   - Docker stack confirmation

**Acceptance Criteria:**
- Worktree exists at documented path with clean branch from `main`
- Docker stack runs cleanly from worktree
- `.env` copied, nothing missing
- Jira comment posted with worktree + branch details (subsequent subtasks reference this)
```

## Subtask 1 — Implement Feature

**Assignee:** unassigned (Sub Agent)

```markdown
**Role:** Senior Staff Implementation Engineer — expert in [the specific tech stack for the repo this Component maps to — e.g., TypeScript + React + Node for a web app, Python + Django for a server, Go + Postgres, etc.]

**Context:**

[Full feature summary from parent Task, 2-3 paragraphs]

**Parent Task:** <PARENT-KEY>

**Worktree / Branch:** see subtask 0's completion comment

**Uses skill:** `superpowers:subagent-driven-development`

**Full Implementation Plan (embedded below):**

[EMBED THE PLAN CONTENT HERE. Target: as much as fits within Jira's ~32KB description limit. For small features, the full plan fits — paste everything. For large framework-level features, paste the chunk/task outline with file paths + key code snippets (schema, core interfaces) and point to the repo file for per-step code. The sub-agent executing this subtask reads BOTH the Jira description AND the plan file — the file is the authoritative source of truth for sub-agent execution.]

**Plan source file (repo):** `docs/superpowers/plans/YYYY-MM-DD-<feature>.md`

**Instructions:**

1. Read parent Task <PARENT-KEY> for the full spec if you need architectural context
2. Work inside the worktree from subtask 0
3. Use `superpowers:subagent-driven-development` — dispatch one sub-agent per chunk in the plan with the chunk's content as explicit instructions
4. Commit after each chunk with a descriptive message referencing <PARENT-KEY>
5. Push commits to the feature branch
6. Post a Jira comment on THIS subtask for each chunk completed

**Acceptance Criteria:**
- Every chunk in the plan is implemented
- All commits pushed to the feature branch
- `npm run build:local` (or equivalent) succeeds inside the worktree
- Database migrations apply cleanly (container applies on boot — never run them manually)
```

## Subtask 1b — Plan Functional Test Cases

**Assignee:** unassigned (Sub Agent) · **Runs parallel with subtask 1**

```markdown
**Role:** Senior Staff QA Architect — expert in functional test design, user journey mapping, Playwright, human-readable QA checklists

**Context:**

[Full feature summary from parent Task]

**Parent Task:** <PARENT-KEY>

**Instructions:**

1. Read the parent Task's Design/Spec and Requirements sections
2. Produce a test plan in `docs/superpowers/plans/YYYY-MM-DD-<feature>-test-plan.md` with TWO sections:

   ### Automated Test Cases
   - Each case: `Given / When / Then`, URL, selectors, expected assertions
   - Consumable by Playwright + a sub-agent (used in subtasks 4 and 10)
   - Cover happy path, edge cases, error states, regressions
   - Mark critical cases with `[CRITICAL]` — these are the subset for subtask 10's prod sanity

   ### Manual Test Cases
   - Each case: plain-English "do this, verify that" with URL + expected screenshot reference
   - Consumable by the team member in subtask 7
   - Include a checkbox per case
   - Focus on: visual polish, UX feel, keyboard/accessibility, cross-browser, "does this feel right"

3. Post a Jira comment on THIS subtask with the FULL test plan content embedded (so subtasks 4, 7, 10 can read it from Jira without opening the repo)
4. Commit the test plan file to the repo

**Acceptance Criteria:**
- Test plan file committed with both sections
- Jira comment on this subtask has the full test plan content (not just a link)
- At least 3 `[CRITICAL]` cases flagged for regression in subtask 10
- Manual cases cover UX scenarios automation won't catch
```

## Subtask 2 — Code Review

**Assignee:** unassigned (Sub Agent)

```markdown
**Role:** Enterprise Architect — expert in multi-system code review, security, API design, data flow, and the specific tech stack involved in this feature

**Context:**

[Full feature summary from parent Task]

**Parent Task:** <PARENT-KEY>

**Worktree / Branch:** see subtask 0

**Review Framework (use as your pass/fail checklist, categorize each finding):**

1. **Functional correctness** — does the code implement what the spec says? Any silent deviations?
2. **Cross-system integration** — for every API call, cookie, shared JWT, shared DB table:
   - If multiple services touch shared data (cross-service consistency), does this change keep them in sync?
   - Are authentication flows (login, SSO, tokens) intact?
   - Do new endpoints match existing auth-middleware patterns in the codebase?
   - Do database schema changes affect other services reading the same tables?
3. **Security** — SQLi, XSS, CSRF, auth bypass, IDOR, secret leakage, cookie flags, CORS origins
4. **Data integrity** — migration safety (reversible? NOT NULL columns? backfill?), FK constraints, transactions
5. **Performance** — N+1 queries, missing indexes, unbounded list queries, large payloads, unnecessary re-renders
6. **API design** — REST conventions, Zod schemas, error shapes, status codes
7. **Code quality** — TypeScript strictness, dead code, naming, obvious simplifications
8. **Tests** — do tests cover the behavior? Mocks vs real DB (per the team's testing conventions)

**Instructions:**

1. Check out the feature branch in the worktree
2. Review the diff end-to-end against the 8 categories above
3. Post a Jira comment on THIS subtask with findings grouped by category. Each finding must include:
   - `file:line` citation
   - Severity: `[BLOCKER]`, `[MAJOR]`, or `[MINOR]`
   - Clear description of the issue
   - Suggested fix
4. Report every blocker; up to 5 majors; up to 5 minors.

**Acceptance Criteria:**
- Comment posted with findings grouped by all 8 categories
- Every finding has file:line + severity + description + suggested fix
- Subtask 3 can read this comment and fix deterministically without further context
```

## Subtask 3 — Fix Code Review Issues

**Assignee:** unassigned (Sub Agent)

```markdown
**Role:** Senior Staff Implementation Engineer — expert in applying code review feedback

**Context:**

[Full feature summary]

**Parent Task:** <PARENT-KEY>
**Worktree / Branch:** see subtask 0

**Instructions:**

1. Read subtask 2's comment — the full findings list is there
2. Fix every `[BLOCKER]` finding. Fix all `[MAJOR]` findings unless you have a justified reason to disagree.
3. For `[MINOR]` findings, fix where easy; otherwise explain in a comment why you're deferring.
4. Commit per-finding with messages like `fix(review): <short description> (addresses <PARENT-KEY> finding #3)`.
5. Post a Jira comment on THIS subtask listing each finding and your resolution (fixed / deferred with reason / disagreed with rationale).

**Acceptance Criteria:**
- All `[BLOCKER]` findings resolved
- `[MAJOR]` findings mostly resolved with per-finding comment for any deferred
- Comment posted with per-finding resolution log
```

## Subtask 4 — UI Testing

**Assignee:** unassigned (Sub Agent)

```markdown
**Role:** Senior Staff Testing Engineer — expert in Playwright, browser automation, screenshot capture, visual QA

**Context:**

[Full feature summary]

**Parent Task:** <PARENT-KEY>
**Worktree / Branch:** see subtask 0

**Test cases to execute:** see the Automated Test Cases section from subtask 1b's comment (also available at `docs/superpowers/plans/YYYY-MM-DD-<feature>-test-plan.md`)

**Instructions:**

1. Ensure Docker stack is up from the worktree
2. Open Playwright, connect to http://localhost (or whatever URL the repo uses)
3. Execute EVERY case in the Automated Test Cases section
4. For each case, capture: URL, action, expected result, actual result, screenshot
5. Post a Jira comment on THIS subtask with this exact table format:

   | # | Case Name | Expected | Actual | Status | Screenshot |
   |---|-----------|----------|--------|--------|------------|
   | 1 | [Case name] | [Expected] | [Actual] | PASS or FAIL | [path or attachment] |

6. Upload all screenshots as Jira attachments on this subtask
7. Summary line at the bottom of the comment: "N/M passing. Blockers: [list of failed case names, or 'none']"

**Acceptance Criteria:**
- Every Automated Test Case appears in the comparison table with PASS or FAIL
- Screenshots attached for any FAIL (and ideally for passes too, for the record)
- Summary line present with counts and blocker list
```

## Subtask 5 — Fix UI Testing Issues

**Assignee:** unassigned (Sub Agent)

Same structure as subtask 3 but fixing issues flagged in subtask 4's comparison table. After fixes, loop back to subtask 4 to re-run failing cases until all pass.

## Subtask 6 — Deploy to Local Docker

**Assignee:** unassigned (Sub Agent)

```markdown
**Role:** Senior Staff DevOps Engineer — expert in Docker builds, nginx, migrations

**Context:**

[Full feature summary]

**Parent Task:** <PARENT-KEY>
**Worktree / Branch:** see subtask 0

**Instructions:**

1. From the worktree, run the local build command (whatever the team uses, e.g., `npm run build`, `make`, `cargo build`)
2. Restart the relevant local services (e.g., `docker compose restart`, a systemd unit, or a `tilt up` reload)
3. Verify database migrations have applied (check logs — migrations run on container boot, never manually)
4. Confirm the feature is accessible at http://localhost (or correct URL)
5. Post a Jira comment on THIS subtask with: build command run, container restart confirmation, migration status, URL where feature is live

**Acceptance Criteria:**
- Build succeeds
- Containers restart cleanly
- Migrations applied (no pending ones)
- Feature accessible at documented URL
```

## Subtask 7 — Manual Testing (human owner)

**Assignee:** the parent Task's assignee (the human-in-the-loop owner of this Story). Default: `${user_config.default_owner_account_id}`.

```markdown
**Role:** The human-in-the-loop owner of this Story (the parent Task's assignee) — the only human checkpoint in the 11-step chain

**Context:**

[Full feature summary]

**Parent Task:** <PARENT-KEY>

**Manual test checklist:** see the Manual Test Cases section from subtask 1b's comment

**Instructions:**

1. Open http://localhost (Docker stack from subtask 6)
2. Walk through EVERY case in the Manual Test Cases section
3. Check each box as you verify (Jira markdown renders interactive checkboxes)
4. For any issues, add a Jira comment with `Issue: [description]` and a screenshot
5. If all cases pass: comment `Manual testing complete — all N cases passed` and transition subtask to Done
6. If issues exist: DO NOT transition. Instead:
   - Comment the issues
   - Reassign subtask 5 (Fix UI Testing Issues) to loop back
   - Re-test once it returns

**Acceptance Criteria:**
- Every Manual Test Case checkbox is checked
- Any issues caught are fixed (looped through 5 → 6 → 7) before Done
```

## Subtask 8 — Senior Architect Review (Sub Agent)

**Assignee:** unassigned (Sub Agent)

**Approach:** Dispatch via `superpowers:code-reviewer` or a focused review subagent. Pass the full parent Task spec + implementation plan + test plan + git diff as context; frame the agent as a senior enterprise architect reviewing for shipping risk.

```markdown
**Role:** Enterprise Architect — expert in multi-system production readiness reviews across the team's infrastructure. Distinct from subtask 2's code review: that was correctness-focused; this is shipping-risk focused.

**Context:**

[Full feature summary]

**Parent Task:** <PARENT-KEY>
**Branch:** see subtask 0

**Review angles (use as a pass/fail checklist — comment one subsection per angle):**

1. **Correctness vs spec** — does this actually do what the spec says? Any silent deviations?
2. **Cross-system impact** — touches shared data, shared auth, multi-service dependencies? Are other systems safe?
3. **Security** — auth paths, secrets, CORS, cookie scope, RBAC, new public endpoints?
4. **Performance at prod scale** — will this slow the dashboard? unbounded queries? N+1?
5. **Data integrity** — migration safety on prod data (real YTD data), FK cascades, rollback plan
6. **Integration touchpoints** — AdvanceHQ, Azure, SharePoint, email, any third party
7. **Auth flow** — login, session, permission checks, SSO across subdomains?
8. **Rollback / kill switch** — if this breaks, can we revert quickly? Migration reversible?
9. **Observability** — log levels, error tracking, audit log entries for mutations
10. **UX polish** — anything that looks "off" that automation couldn't catch

**Instructions:**

1. Check out the feature branch locally
2. Review the diff end-to-end
3. Post a Jira comment with pass/fail per angle (all 10)
4. If any blocker → reassign sub-agent to fix, loop back through subtasks 4 → 5 → 6 → 7 → 8
5. If all clear → transition subtask to Done. Subtask 9 proceeds.

**Acceptance Criteria:**
- All 10 angles have a pass/fail in the comment
- Any blockers are fixed and re-reviewed
- Final comment ends with: "Architect review passed — cleared for prod"
```

## Subtask 9 — Push to Prod

**Assignee:** unassigned (Sub Agent)

> [!important] Open a PR. NEVER merge directly to main.
> Subtask 9 must **open a pull request** to `main` and **wait for a human to click merge**. Direct pushes to `main` bypass the final-human-sanity step and erase GitHub's diff-review surface. The sub-agent's job is to prepare the PR cleanly (push branch → open PR with architect-recommended description → comment PR URL on the Jira subtask), then stop. The orchestrator pauses until the human merges — GitHub Actions takes over from there.

```markdown
**Role:** Senior Staff Release Engineer — expert in git merges, production deployments, CI/CD, PR hygiene

**Context:**

[Full feature summary]

**Parent Task:** <PARENT-KEY>
**Branch:** see subtask 0

**Instructions:**

1. Push the feature branch to origin (if not already pushed).
2. Open a PR from the feature branch to `main` using `gh pr create` (or raw REST if `gh` is unavailable). The PR description MUST include:
   - **Summary** of what's being shipped (pull from parent Task's spec section).
   - **Bundled scope** subsection if the architect review (subtask 8) called out any out-of-scope commits bundled in this PR (e.g., infra/tech-debt fixes). Copy the architect's rationale verbatim.
   - **Test plan** link/summary (from subtask 1b).
   - **Rollback plan** — copy from architect review's "Rollback / kill switch" angle.
3. Post a Jira comment on THIS subtask with:
   - PR URL
   - Target branch (`main`)
   - Final line: `Waiting for human merge.`
4. Do NOT transition this subtask to Done yet. Leave at `In Progress` until the human merges and the orchestrator verifies the deploy succeeded.

**After the human merges (orchestrator — not this sub-agent):**

1. Detect merge via `gh pr view --json state,mergeCommit` or by checking the commit on `main`.
2. Monitor GitHub Actions deploy run.
3. Confirm deploy succeeded (API container restarted, migrations applied, feature accessible on prod URL).
4. Post a Jira comment with: merge commit hash, deploy log excerpt, timestamp, prod URL.
5. Transition the subtask to Done.

**Acceptance Criteria:**
- PR opened with correct description (Summary + Bundled scope + Test plan + Rollback)
- PR URL posted on Jira subtask
- Subtask stays `In Progress` until human merges and orchestrator confirms deploy
- After merge: deploy verified, comment posted with commit hash + prod URL, subtask transitioned to Done
```

**Why PR-not-direct-merge is non-negotiable:**

- Final human sanity check on the diff before prod
- GitHub's PR review UI surfaces changes differently than local `git diff` — reviewers catch things the chain missed
- The PR record is the audit trail — "who approved this prod change" is unambiguous
- If the architect review flagged bundled-scope commits, the PR description makes them visible to anyone not in the chain

## Subtask 10 — Prod Sanity Test

**Assignee:** unassigned (Sub Agent)

```markdown
**Role:** Senior Staff QA Engineer — expert in production smoke testing, regression detection

**Context:**

[Full feature summary]

**Parent Task:** <PARENT-KEY>

**Test cases to execute:** the `[CRITICAL]` subset from subtask 1b's Automated Test Cases (for prod regression smoke)

**Instructions:**

1. Run Playwright (or the team's E2E framework) against the production URL using the appropriate prod config
2. Execute only the `[CRITICAL]` cases from subtask 1b
3. Post a Jira comment on THIS subtask with a pass/fail table per case
4. If any FAIL → urgent Jira mention to the story owner + immediate rollback assessment
5. If all PASS → comment `Production sanity passed`. Transition parent Task <PARENT-KEY> to Done.

**Acceptance Criteria:**
- All `[CRITICAL]` cases executed against production
- Results table posted
- Parent Task transitioned to Done (only if all pass)
```
