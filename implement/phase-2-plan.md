# Phase 2: Implementation Plan

You are in Phase 2 (plan mode). The user has approved the context doc. Do NOT read phase-3-execute.md yet.

## Planning Rules

- **Reuse before creating** — the context found existing utilities/patterns; the plan MUST use them
- **Match conventions** — new code follows naming, structure, and patterns from Phase 1
- **Ground everything** — every file path, function name, and approach should come from the `## Context` section. If planning reveals new information not captured in Phase 1, update `## Context` in `<slug>.md` before using it in the plan
- **Small steps** — each step = one logical change, ≤3-4 files
- **Buildable after each step** — no broken intermediate states
- **Order by dependency** — steps must follow the dependency chain discovered in Phase 1. Files that others import/depend on change first. If step 3 touches a shared interface, steps 4+ that consume it come after.

## Output: `## Plan` section appended to `docs/plans/YYYY-MM/<slug>.md`

Append the `## Plan` section to the existing `<slug>.md` file (which already has `## Context`):

```markdown
## Plan

### Steps

#### Step 1: <Short description>
**Files**: `path/to/file.ext` (modify), `path/to/new.ext` (create)
**Pattern**: Following `path/to/example.ext`
**Changes**:
- <Specific change>
- <Specific change>
**Verify**: <How to verify — build, test, manual check>

#### Step 2: ...

### New Files
- `path` — <Purpose> — pattern from `existing-file`

### Verification Plan
- Build: `<command from project's CLAUDE.md or build config>` → succeeds
- Tests: `<command from project's CLAUDE.md or build config>` → all pass
- Manual: <What to check> → <Expected outcome>

### Risks
- <Risk> (Low/Med/High) — <Mitigation>
```

## Gate

1. **Append** the `## Plan` section to `docs/plans/YYYY-MM/<slug>.md` using the Edit tool.
2. **Present** the plan to the user by telling them: "Review `## Plan` in `docs/plans/YYYY-MM/<slug>.md`." Do NOT paste the plan as inline text — the `.md` file IS the plan.
3. **Call `ExitPlanMode`** to request approval, referencing the `.md` file path.

**Do NOT start writing code until the user approves the plan.**

When approved: read `phase-3-execute.md`.
