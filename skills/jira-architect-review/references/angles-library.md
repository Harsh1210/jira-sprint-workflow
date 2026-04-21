# Angles library — the 10 architect review angles

Full checklist for each angle. Pick the subset appropriate to the review size (see `SKILL.md`).

Each angle is ≤ 10 things to check — not every bullet needs an answer. Use them as a prompt; skip bullets that don't apply to the diff.

---

## 1. Correctness vs spec — always run

**Goal:** does the implementation match the parent Story's Design/Spec section?

- [ ] Every bullet in "Requirements" has a corresponding change in the diff
- [ ] Edge cases called out in the spec are handled (error states, empty states, unusual inputs)
- [ ] No silent deviations — the architect review should NOT be the first place a deviation is documented
- [ ] If there are documented deviations (subtask 1 or 3 flagged them), they are justified and consistent with the spec's intent
- [ ] Acceptance criteria checkboxes in the parent Task can now be ticked

**Red flag:** code does something reasonable but different from what the spec asked for. File a finding either way — either the code is wrong or the spec needs an update.

---

## 2. Cross-system impact — Medium+

**Goal:** does anything outside this Story's component break?

- [ ] For each API call, cookie, shared JWT, shared DB table, or shared cache the diff touches: verify other systems that use it still work
- [ ] New endpoints use the same auth middleware pattern as existing endpoints in the same module
- [ ] Shared DB tables (users, audit_logs, etc.) have the same contract for all consumers
- [ ] If this Story is part of a framework used elsewhere: the framework's public interface stays backward-compatible
- [ ] If the diff touches `sharedtype`/`shared-enum` files: verify all consumers match

**Red flag:** new code treats a shared resource as if it owned it exclusively.

---

## 3. Security — always run

**Goal:** no new attack surface.

- [ ] SQL injection: every dynamic part is parameterized / uses the query builder — no string concatenation
- [ ] XSS: user-controllable strings rendered in HTML are escaped
- [ ] Auth bypass: `authenticate` runs before any state-changing handler; `authorize` runs where the resource is permissioned
- [ ] IDOR: endpoints that serve user-scoped data filter by user_id and return 404 (not 403) on mismatch, to avoid leaking existence
- [ ] Secrets: no hardcoded tokens, passwords, or keys in the diff
- [ ] CORS: new endpoints inherit the existing CORS policy (or justify the difference)
- [ ] Cookies: `httpOnly`, `secure`, `sameSite` set appropriately; new session handling matches existing patterns
- [ ] File downloads/uploads: validate type, size, path traversal
- [ ] CSV / spreadsheet: formula injection (leading `=`, `+`, `-`, `@`) neutralized if values are user-controlled
- [ ] Logged data: request bodies logged don't include secrets or PII beyond what's already logged

**Red flag:** anything involving raw SQL, file paths, or user input rendered back to the browser.

---

## 4. Performance at prod scale — Medium+

**Goal:** will this survive production load?

- [ ] N+1 queries: every list-endpoint query either joins or uses batched fetches
- [ ] Missing indexes: `WHERE`/`ORDER BY`/`JOIN` columns have indexes (check migration + existing schema)
- [ ] Unbounded list queries: every list endpoint has `LIMIT` and pagination (or is justified with a size cap)
- [ ] Large payloads: responses that could be >1MB have pagination or streaming
- [ ] Unnecessary re-renders: for React changes, `useMemo`/`useCallback`/proper `queryKey` on significant renders
- [ ] Background jobs: concurrency bounded, not spawning unbounded workers
- [ ] Long-running requests: streams properly; client has timeout expectations
- [ ] Scale assumptions: if the spec says "supports 10k rows", quick sanity check that the implementation can actually handle it

**Red flag:** fine on 10 rows, catastrophic on 10,000.

---

## 5. Data integrity — Small+

**Goal:** the DB stays consistent and recoverable.

- [ ] Migration is additive or clearly reversible (document the reverse in the migration comment)
- [ ] `NOT NULL` on new columns: handled with either a default OR a two-step migration (add column with default, backfill, then optionally tighten)
- [ ] Foreign keys: `ON DELETE` behavior is intentional (CASCADE, SET NULL, RESTRICT) and matches the business rule
- [ ] Transactions: multi-step state mutations wrapped in a transaction where correctness depends on atomicity
- [ ] Race conditions: concurrent requests for the same resource handled (unique constraints, atomic UPDATE..WHERE, row locks)
- [ ] Check constraints: enum-ish columns use CHECK or enum types, not strings
- [ ] Timestamps: created_at, updated_at defaulting to NOW() at DB level (not app level) where possible

**Red flag:** migration that drops or renames a column without explicit justification.

---

## 6. Integration touchpoints — Large

**Goal:** external systems keep working.

- [ ] Each third-party API call (Stripe, AdvanceHQ, Azure, SendGrid, etc.) has timeout + retry + error handling
- [ ] Secrets for third-party APIs are in env vars, not code
- [ ] Webhook handlers validate signatures
- [ ] Rate limits respected (either local throttling or backoff on 429s)
- [ ] Feature flags / gradual rollout possible if the integration is new
- [ ] If the integration fails, the rest of the app still works (graceful degradation)

**Red flag:** a failed third-party call takes down the request.

---

## 7. Auth flow — Large, or forced when auth code touched

**Goal:** authentication + authorization intact.

- [ ] New endpoints run `authenticate` before any state access
- [ ] `authorize(scope, level)` calls use the right scope names (matches existing convention)
- [ ] Session / JWT logic unchanged (or the change is intentional and scoped)
- [ ] SSO flow intact (if relevant): `me` endpoint returns expected shape; cookie domain correct for cross-subdomain
- [ ] Permission denials return 403 (not 401) once authenticated
- [ ] Rate limiting on auth endpoints intact
- [ ] No endpoint accidentally bypasses auth (check every new route registration)

**Red flag:** changes to middleware, session stores, or cookie handling.

---

## 8. Rollback / kill switch — always run

**Goal:** if this breaks in prod, how fast can we undo?

- [ ] Code rollback: `git revert <merge>` safely works. Document any follow-up needed (re-migrations, cache flushes)
- [ ] DB rollback: migration is reversible OR explicitly documented as one-way (with risk assessment)
- [ ] Feature flag / UI toggle: can the feature be disabled without a deploy (e.g., env var, config flag, or removing a single component)?
- [ ] Data implications: if users started using the feature, does rollback leave orphan data? (Usually fine if additive, dangerous if deletive.)
- [ ] Deploy speed: can we deploy a fix in <15 min via the normal CI pipeline?

**Red flag:** "we cannot roll this back cleanly" — usually a sign the migration or data model should be reconsidered.

---

## 9. Observability — Large, or forced when new async code paths added

**Goal:** if this misbehaves in prod, can we debug it?

- [ ] Errors logged with enough context (user_id, resource_id, operation) to reproduce
- [ ] Audit events written for mutations (`logAudit(...)` per the codebase's convention)
- [ ] Metrics / counters for anything high-frequency (request latency, queue depth, job duration)
- [ ] Log levels appropriate: warn for business-expected failures, error for bugs, info for normal operations
- [ ] Cron jobs / background workers log start + end + outcome
- [ ] Traces / request IDs propagate through the call chain (if the codebase uses them)

**Red flag:** new background worker with `console.log`-only logging.

---

## 10. UX polish — Small+, or forced when frontend touched

**Goal:** visual + interaction quality.

- [ ] Loading states: every async operation has a spinner / skeleton / disabled state
- [ ] Error states: failed operations show a clear message (not a generic "Something went wrong")
- [ ] Empty states: first-use / zero-results paths aren't confusing
- [ ] Disabled states: buttons that can't be clicked reflect that (opacity, cursor, tooltip)
- [ ] Keyboard: tab order sane; Enter/Esc work on forms/dialogs
- [ ] Dark mode: colors and contrast work in both themes
- [ ] Text: no lorem ipsum or debug strings left in
- [ ] Alignment: new controls inherit spacing/typography from adjacent components

**Red flag:** a new component that looks OK in the happy path but has no error or empty state.

---

## How to report findings

Each finding in a Jira review comment should have:
- Severity: `[BLOCKER]` (must fix before prod) / `[MAJOR]` (should fix before prod) / `[MINOR]` (nice to fix, non-blocking)
- Location: `file:line` or `module path`
- Description: what's wrong in 1-2 sentences
- Suggested fix: 1 sentence, or "see comment thread"

Keep findings terse. The value of the review is the categorized pass/block judgment — not an essay per finding.
