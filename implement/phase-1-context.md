# Phase 1: Context Refinement

You are in Phase 1. Do NOT read phase-2-plan.md or phase-3-execute.md yet.

## Part A — Requirements Gathering

Use `AskUserQuestion` to clarify:
- **Goal**: What is the business/user goal? What problem does this solve?
- **Acceptance criteria**: How will you verify "done"? Be specific — no vague "it works."
- **Scope**: What is explicitly out of scope?
- **Edge cases**: Error states, empty states, unexpected inputs?
- **Constraints**: Performance? Backward compatibility? Platform targets?

Wait for answers. If answers reveal new ambiguities, ask follow-ups. Do NOT proceed to Part B until requirements are fully clear.

## Part B — Codebase Pattern Discovery

**This is the most critical part.** Discover how this project already does things before proposing anything new.

### Project rules and conventions
- Review CLAUDE.md, ~/.claude/CLAUDE.md, .claude/skills/, .claude/commands/
- Check linter/formatter/build configs

### Existing patterns
- Use `Grep` and `Glob` to find similar code — use parallel Explore agents for independent areas (max 3 concurrent to limit memory pressure)
- Read 2–3 representative examples in full (not snippets)
- Identify: naming conventions, file organization, module boundaries, import/error/logging patterns
- Look for reusable utilities, helpers, protocols, components

### Affected files — map the blast radius
- Identify every file that will change and read each one. For >10 affected files, read 3–4 representative samples and list the rest with their expected change type.
- Identify directories and naming patterns for new files (exact paths are finalized in Phase 2)
- Trace callers/importers for ripple effects

## Completeness Self-Check

Before writing the context doc, verify:
- [ ] Every acceptance criterion is specific and testable
- [ ] 2–3 similar existing patterns found and read in full
- [ ] Every affected file identified; representative samples read for large sets (>10 files)
- [ ] Reusable code populated — or "none found" with search evidence
- [ ] Edge cases addressed for each requirement

If any check fails, investigate more before writing the doc.

## Output: `docs/plans/YYYY-MM/<slug>-context.md`

```markdown
# Context: <Task Title>

## Requirements

### Goal
<What this achieves and why>

### Acceptance Criteria
- [ ] <Specific, testable criterion>

### Out of Scope
- <What this does NOT do>

### Edge Cases
- <Scenario> → <Expected behavior>

## Q&A Record
- Q: <Question> → A: <Answer>

## Codebase Analysis

### Existing Patterns to Follow
- <Pattern> — see `path/to/example.ext:line` — <How to use it>

### Reusable Code Found
- <Utility/component> at `file:line` — <How to reuse>

### Affected Files
- `path` (modify) — <What changes>
- `path` (create) — <Purpose>

### Risks
- <Risk> (Low/Med/High) — <Mitigation>
```

## Gate

After writing the file, use `AskUserQuestion`:
> "Phase 1 complete. Please review `docs/plans/YYYY-MM/<slug>-context.md`. Does this accurately capture requirements and patterns? Anything wrong, missing, or unclear?"

**Do NOT proceed to Phase 2 until the user explicitly approves.**

When approved: **run `/clear`** to free context, then call `EnterPlanMode` and read `phase-2-plan.md`.
