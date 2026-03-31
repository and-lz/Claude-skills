# Phase 3: Implementation

You are in Phase 3. The user has approved both `## Context` and `## Plan` sections in `<slug>.md`.

## Before Starting

- Ensure there is a clean rollback point (commit or stash uncommitted changes). This pre-Phase-3 commit is your **nuclear rollback point** if all else fails.
- Create a feature branch if not already on one
- Append the `## Implementation` section to `<slug>.md` immediately (see Output section) with status "In Progress"

## Execution Rules

1. **Follow the plan exactly.** It was approved for a reason. Do not freelance.
2. **If you must deviate**: STOP. Explain via `AskUserQuestion`. Get approval. Record the deviation in the `### Deviations` subsection under `## Plan` in `<slug>.md` so the plan stays the source of truth.
3. **Scope creep boundary**: Incidental fixes (missing import, typo, lint error) required for a step's verification to pass are acceptable. Any change to files not listed in the plan, or any new behavior not in acceptance criteria, is a deviation — follow rule 2.
4. **Before running a step's verification**, confirm the plan has a `**Verify**:` field for that step. If missing, treat it as a plan gap — add verification criteria before proceeding.
5. **After each step**: Run the verification for that step. Fix failures before moving on.
6. **After each step passes verification**: commit with message `<plan-slug>: step N — <short description>`, then update the implementation doc with the step's verification result.
7. **If a step fails after 2 attempts**: If the fix is trivial and within plan scope (e.g., missing import), fix it directly. Otherwise, **fix forward** — create a corrective step (e.g., step 5b) that addresses the root cause. Do not rebase or amend earlier step commits. If fix-forward is not viable, reset to the pre-Phase-3 rollback commit and re-plan from that point.
8. **After all steps pass**, update the implementation doc status to "Complete". Make a final commit. Push if remote exists.

## Output: `## Implementation` section in `docs/plans/YYYY-MM/<slug>.md`

Append this section at the **start** of Phase 3 with status "In Progress". Update it after each step. Mark "Complete" when all steps pass.

```markdown
## Implementation

**Status**: In Progress | Complete

### Step Results
- Step 1: <description> — Pass/Fail — <notes>
- Step 2: <description> — Pass/Fail — <notes>
(updated incrementally after each step)

### Final Verification
- Build: Pass/Fail
- Tests: Pass/Fail
- Manual: Pass/Fail — <notes>

### Acceptance Criteria
- [x] <Criterion> — verified by <how>
```

## Completion

After updating `## Implementation` to "Complete", use `AskUserQuestion`:
> "Phase 3 complete. Please review `## Implementation` in `docs/plans/YYYY-MM/<slug>.md` and the code diffs. Did the implementation meet expectations? Any issues or surprises?"

## Post-Mortem (mandatory when deviations exist)

Append a `## Post-Mortem` section to `<slug>.md` when **any** deviation was recorded, or when there were bugs, wrong assumptions, or unexpected side effects:

```markdown
## Post-Mortem

### What Went Wrong
<Specific issues>

### Root Cause
<Missing context question? Wrong assumption? Overlooked dependency?>

### What Was Missed
- Phase 1: <What context failed to capture>
- Phase 2: <What plan failed to account for>

### Lessons Learned
- <Actionable, specific lesson>
```

If a lesson is reusable (not task-specific), save it as a feedback memory.
