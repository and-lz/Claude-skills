# Phase 3: Implementation

You are in Phase 3. The user has approved both context and plan docs.

## Before Starting

- Ensure there is a clean rollback point (commit or stash uncommitted changes)
- Create a feature branch if not already on one

## Execution Rules

1. **Follow the plan exactly.** It was approved for a reason. Do not freelance.
2. **If you must deviate**: STOP. Explain via `AskUserQuestion`. Get approval. Record the deviation in `<slug>-plan.md` immediately (append to a `## Deviations` section) so the plan stays the source of truth.
3. **Before running a step's verification**, confirm the plan has a `**Verify**:` field for that step. If missing, treat it as a plan gap — add verification criteria before proceeding.
4. **After each step**: Run the verification for that step. Fix failures before moving on.
5. **After each step passes verification**, commit with message: `<plan-slug>: step N — <short description>`.
6. **If a step fails after 2 attempts**: If the fix is trivial and within plan scope (e.g., missing import), fix it directly. Otherwise, git reset to the last passing step's commit (enabled by step-level commits) and re-plan from that point. Do not push through broken state.
7. **After all steps pass**, make a final commit for the implementation doc. Push if remote exists.

## Output: `docs/plans/YYYY-MM/<slug>-implementation.md`

After all steps complete and verification passes:

```markdown
# Implementation: <Task Title>

**Context**: [<slug>-context.md](./<slug>-context.md)
**Plan**: [<slug>-plan.md](./<slug>-plan.md)
**Status**: Complete

## Deviations
- None (or: Step X — planned Y, did Z because <reason>, user-approved: yes/no)

## Verification Results
- Build: Pass/Fail
- Tests: Pass/Fail
- Manual: Pass/Fail — <notes>

## Acceptance Criteria
- [x] <Criterion> — verified by <how>
```

## Completion

After writing the implementation doc, use `AskUserQuestion`:
> "Phase 3 complete. Please review `docs/plans/YYYY-MM/<slug>-implementation.md` and the code diffs. Did the implementation meet expectations? Any issues or surprises?"

## Post-Mortem (if problems occurred)

If there were bugs, wrong assumptions, or unexpected side effects, create `docs/plans/YYYY-MM/<slug>-postmortem.md`:

```markdown
# Post-Mortem: <Task Title>

## What Went Wrong
<Specific issues>

## Root Cause
<Missing context question? Wrong assumption? Overlooked dependency?>

## What Was Missed
- Phase 1: <What context failed to capture>
- Phase 2: <What plan failed to account for>

## Lessons Learned
- <Actionable, specific lesson>
```

If a lesson is reusable (not task-specific), save it as a feedback memory.
