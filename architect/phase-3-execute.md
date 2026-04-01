# Phase 3: Implementation

You are in Phase 3. The user has approved both `## Context` and `## Plan` sections in `<slug>.md`.

## Before Starting

- **Staleness check**: If the plan was approved in a different session or >24 hours ago, run `git log --since="<plan-approval-date>" -- <affected-files>`. If affected files changed, warn the user and offer to re-validate Phase 1 + Phase 2 before executing.
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
7. **If a step fails after 2 attempts**, assess the failure scope:
   (a) **Trivial and within scope** (missing import, typo, lint error) — fix directly.
   (b) **Localized regression** (step N broke something step M built, but fix is clear) — create a corrective step (e.g., step 5b) that fixes both the current step and the regression. Do not amend earlier commits.
   (c) **Cascading failure** (step N fundamentally conflicts with earlier steps, or the fix would require rewriting multiple completed steps) — STOP. Mark implementation status as "Paused — cascading failure at step N". Use `AskUserQuestion` to present options: (i) reset to pre-Phase-3 rollback and re-plan, (ii) revise the plan from step N onward (partial re-plan), (iii) abandon the task.
   (d) **Environmental failure** (build infra broken, dependency unavailable, CI down) — mark as "Paused — blocked on \<reason\>", save state, inform user.
8. **After all steps pass**, update the implementation doc status to "Complete". Make a final commit. Push if remote exists.

## Pausing Mid-Implementation

If the user requests a pause, or if a blocking issue is discovered:
1. Complete the current step if possible (do not leave half-written code)
2. Commit all completed work with message `<slug>: paused after step N`
3. Update `## Implementation` in `<slug>.md`:
   - Set status to "Paused after step N"
   - Add `### Pause State` with:
     - Last completed step and its verification result
     - Next step to resume from
     - Any in-flight state or context needed for resumption
     - Reason for pause
4. Tell the user: "Paused at step N. To resume, invoke `/architect` — it will detect the existing doc and pick up from step N+1."

### Resuming from pause
When `/architect` is invoked and `<slug>.md` exists with status "Paused after step N":
1. Read the full `<slug>.md` (Context + Plan + Implementation so far)
2. Run staleness check on affected files
3. If no conflicts: resume from step N+1
4. If conflicts found: warn user and offer to re-validate from the conflicting step

## Scope Expansion During Phase 3

If during execution the user requests additional functionality or you discover the approved plan is insufficient:

1. **Do NOT modify existing approved steps** — they were approved for a reason
2. STOP current execution after completing the current step
3. Call `EnterPlanMode` with context: "Scope expansion needed"
4. Propose additive steps (e.g., "Step 7a, 7b") that extend the plan
5. Update `<slug>.md` with the new steps under a `### Scope Expansion` subsection
6. Get user approval via `ExitPlanMode`
7. Resume execution including the new steps

If the expansion fundamentally changes the architecture or invalidates completed steps, treat it as a new `/architect` task rather than a scope expansion.

## Abandoning a Task

If the user explicitly abandons the task mid-Phase-3:
1. Complete or revert the current in-progress step (no half-applied changes)
2. Commit completed steps: `<slug>: abandoned after step N`
3. Update `<slug>.md`:
   - Set status to "Abandoned"
   - Add `### Abandonment` section with reason and what was completed
4. If any completed steps are independently valuable (e.g., a utility function that's useful regardless), note them so they aren't accidentally reverted
5. Ask the user if they want to revert all Phase 3 changes to the pre-Phase-3 rollback point, or keep the partial work

## Output: `## Implementation` section in `docs/plans/YYYY-MM/<slug>.md`

Append this section at the **start** of Phase 3 with status "In Progress". Update it after each step. Mark "Complete" when all steps pass.

```markdown
## Implementation

**Status**: In Progress | Paused after step N | Complete | Abandoned

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

## PR Preparation (if on a feature branch)

After all steps pass and `## Implementation` is marked "Complete":

1. **Commit hygiene**: Review the commit history. Each step should be one commit. If there are fixup commits from corrective steps, ask the user if they want to squash.
2. **PR description**: Draft a PR body using this template and present to user:
   ```markdown
   ## Summary
   <1-2 sentence description of what this PR does>

   ## Changes
   <bulleted list of key changes, referencing the plan steps>

   ## Testing
   <what was verified and how>

   ## Cross-cutting concerns addressed
   <from the Phase 2 checklist — security, a11y, performance, etc.>

   ## Plan document
   `docs/plans/YYYY-MM/<slug>.md`
   ```
3. Present the draft PR to the user. Do NOT push or create the PR automatically — the user may want to adjust.

## Completion

After updating `## Implementation` to "Complete", use `AskUserQuestion`:
> "Phase 3 complete. Please review `## Implementation` in `docs/plans/YYYY-MM/<slug>.md` and the code diffs. Did the implementation meet expectations? Any issues or surprises?"

## Post-Mortem (always)

Append a `## Post-Mortem` section to `<slug>.md` after **every** completed task, not just when deviations exist.

```markdown
## Post-Mortem

### What Went Well
- <Specific things that worked — patterns found, clean steps, good estimates>

### What Went Wrong (if anything)
- <Specific issues, or "None — execution matched the plan">

### Root Cause (if issues)
<Missing context question? Wrong assumption? Overlooked dependency?>

### What Was Missed (if issues)
- Phase 1: <What context failed to capture>
- Phase 2: <What plan failed to account for>

### Lessons Learned
- <Actionable, specific lesson>

### Cross-Cutting Concerns Review
- Were the right concerns identified in Phase 2? Any that should have been caught?
```

If a lesson is reusable (not task-specific), save it as a feedback memory.
If a cross-cutting concern was missed (e.g., forgot a11y but it mattered), note it as a lesson for future tasks.
