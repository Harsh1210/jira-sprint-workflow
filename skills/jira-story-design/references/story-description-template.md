# Parent Task description template

Write the parent Task's description in Jira using this markdown skeleton. Replace every bracketed section with real content from the brainstorm and plan.

```markdown
## Original Feedback

> [Quote the person or stakeholder who requested this — teammate, customer, internal user. If the Task was created from scratch, write a one-paragraph summary of the motivation.]

## Investigation

[What we found when we dug into this. Current-state code paths, why the existing behavior is wrong/missing, relevant files or modules.]

## Requirements

- [ ] [Acceptance criterion 1 — testable, specific]
- [ ] [Acceptance criterion 2]
- [ ] [Acceptance criterion 3]
- [ ] ...

## Design / Spec

[The full design from brainstorming. Architecture choices, data model, API surface, UI changes, trade-offs, rejected alternatives with reasons. If the brainstorm produced Mermaid diagrams or tables, include them here — Jira's markdown renders Mermaid.]

[Do not summarize. Paste the full design content. A teammate reads this top to bottom and knows what's being built.]

## Implementation Plan Summary

[A high-level overview of the plan's chunks. Each chunk gets 1-3 sentences describing what gets built in it. The full per-step plan with code lives in the subtasks 1 description — this section is navigation, not the source of truth.]

Plan source file: `${user_config.plans_directory}/YYYY-MM-DD-<feature>.md`

## Test Plan Summary

[A high-level overview of the test scenarios. The full automated + manual test plan lives in subtask 1b's description and the test plan file.]

Test plan source file: `${user_config.plans_directory}/YYYY-MM-DD-<feature>-test-plan.md`

## Dependencies

- **Blocked by:** [`<ISSUE-KEY>` if applicable, or "none"]
- **Related:** [`<ISSUE-KEY>` cross-references]
- **External:** [Third-party APIs, feature flags, access required, etc.]

## Out of Scope

- [Thing we considered and deferred, with one-line reason]
- [Another deferred thing]

## Open Questions

- [Anything left for the team member or Story owner to resolve before starting]
```

## Rules for filling this in

1. **"Design / Spec"** should be the full brainstorm output. If it's 2000 words, paste 2000 words. Teammate needs to understand the WHY, not just the WHAT.

2. **"Implementation Plan Summary" and "Test Plan Summary"** are intentionally lightweight in the parent Task. The FULL plan and test plan go into subtasks 1 and 1b respectively, where they're actually used. The parent Task gives the reader a table of contents.

3. **"Requirements"** is an acceptance checklist. Unchecked at design time. Team member checks boxes as they verify each criterion during execution.

4. **"Open Questions"** should be empty by the time you transition to Selected for Development. If there are genuine unknowns, resolve them during brainstorming, not after.

5. **Jira quirks**:
   - Markdown renders well in Atlassian (headings, lists, code blocks, tables, links)
   - Mermaid diagrams render in Atlassian Cloud as of late 2025
   - Checkboxes render as interactive checkboxes (team member can click them)
   - Code blocks with language tags render with syntax highlighting
