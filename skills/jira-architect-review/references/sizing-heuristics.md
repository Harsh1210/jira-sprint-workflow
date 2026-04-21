# Sizing heuristics — examples and calibration

The tiers (minor / small / medium / large) map to review depth. Here are real-style examples so the reviewer knows where a given diff falls.

## Minor (3 angles: correctness, security, rollback)

Changes that are surface-level, obviously safe, and reversible in seconds.

**Examples:**
- Fix a typo in a UI label
- Update a log message or error string
- Add a missing null check guard
- Bump a dependency patch version (no API changes)
- Fix a broken link in documentation that ships with the app

**Signals:**
- 1–3 files changed
- ≤ 50 lines total
- No migration, no new endpoints, no new modules
- No changes under `auth`, `security`, or shared-data paths

**What to skip:**
- Performance review — nothing structural changed
- Cross-system — isolated change
- UX polish — only if the change is visual (typo in user-facing text) in which case verify it reads right; otherwise skip

**Typical review time:** 2–3 minutes.

---

## Small (5 angles: + data integrity, + UX polish)

A single bug fix or a small self-contained feature. One module, one scope.

**Examples:**
- Add a new filter option to an existing list page
- Fix a race condition on submit-button clicks (add disabled state)
- Add a new field to an existing form + DB column
- Modify an existing endpoint to accept a new optional query param
- Add a new status value to an enum column

**Signals:**
- 3–10 files changed
- 50–300 lines total
- Optional single migration (usually additive — add column, add index)
- Maybe 1 new endpoint handler or 1 new form
- Changes stay within a single module

**What to skip:**
- Cross-system — single module, no shared-resource changes
- Performance — no new query patterns
- Integrations — no third-party calls
- Auth — no changes to auth middleware
- Observability — no new async paths

**Typical review time:** 5–7 minutes.

---

## Medium (7 angles: + cross-system, + performance)

A typical new feature spanning multiple files, with some shared-resource impact.

**Examples:**
- New page with a new list view + detail view + form
- New table joining to an existing one, with queries joining both
- Refactor of a list endpoint to add sorting + filtering + pagination
- New background job (cron or queued) that runs periodically
- Add a new notification type using the existing notification framework

**Signals:**
- 10–20 files changed
- 300–1000 lines total
- Usually a migration (adds table OR adds multiple columns)
- 1–3 new endpoints
- Touches shared framework code (DataTable, filter-parser, notification, etc.)
- Maybe a new small module or submodule

**What to skip:**
- Integration touchpoints — no third-party dependency
- Auth — unless the feature is behind a new role
- Observability — unless it introduces background work

**Typical review time:** 10–15 minutes.

---

## Large (all 10 angles)

A framework change, a cross-cutting concern, a multi-module feature, or anything with significant production risk surface.

**Examples:**
- Entire new entity-agnostic framework (the WFR-5 Export Framework — 32 files, 3925 added lines)
- Auth refactor, session-management rewrite
- Migration from one ORM pattern to another
- Adding a third-party integration (AdvanceHQ sync, Stripe webhooks)
- New cross-service communication (e.g., backend A now calls backend B)
- Any change that touches `authenticate.ts` / `authorize.ts` at depth

**Signals:**
- 20+ files changed
- 500+ lines added
- New top-level module (`backend/src/modules/<name>/`, `frontend/src/components/<name>/`)
- Migration + multiple new endpoints
- Touches auth, secrets, or shared-data paths
- Introduces a new pattern the rest of the codebase will adopt

**What to skip:**
- Nothing — run all 10 angles
- If the spec explicitly defers something (e.g., "no integration with Stripe yet — just the schema"), note that in the corresponding angle as "deferred — see spec"

**Typical review time:** 15–30 minutes.

---

## Edge cases

### The change is mostly test / docs / tooling

Apply one level smaller than the metrics would suggest. 2000 lines of new tests is easier to review than 2000 lines of new prod code.

### The change is mostly generated

Look at the non-generated portion and size from that. Ignore schema dumps, compiled output, lockfile churn.

### The change is a data migration only (UPDATE-only, no DDL)

Size based on the scope of data touched:
- Single table, small set: small
- Multiple tables OR data-correctness-critical: medium
- Migration that could corrupt data if wrong: large

### The change is a revert

Always at least Small (run correctness + rollback + data integrity). A revert can re-introduce bugs the original fix closed.

### The change touches a security-sensitive file

Force Auth angle regardless of size. Minor change to `authenticate.ts` is NOT minor from a risk perspective.

---

## Calibration — "when in doubt, size up"

The cost of an extra review angle is a few minutes of architect time. The cost of missing an angle on a large change is a production incident. Err on the side of one tier larger if the metrics are ambiguous.

Do NOT size down from the heuristics to save time. Size up when a signal says "this is risky" even if the counts are low.
