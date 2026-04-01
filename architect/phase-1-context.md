# Phase 1: Context Refinement

You are in Phase 1. Do NOT read phase-2-plan.md or phase-3-execute.md yet.

## Part A — Requirements Gathering

Use `AskUserQuestion` to clarify:
- **Goal**: What is the business/user goal? What problem does this solve?
- **Acceptance criteria**: How will you verify "done"? Be specific — no vague "it works."
- **Scope**: What is explicitly out of scope?
- **Edge cases**: Error states, empty states, unexpected inputs?
- **Constraints**: Performance? Backward compatibility? Platform targets?
- **Platform**: iOS, tvOS, cross-platform, web, or backend? (Determines which domain skills to consult.)

If requirements conflict with existing project conventions or architecture (CLAUDE.md, linter rules, established patterns), raise the conflict via `AskUserQuestion` immediately. Document the resolution in the Q&A Record.

Wait for answers. If answers reveal new ambiguities, ask follow-ups. Do NOT proceed to Part B until requirements are fully clear.

## Part B — Codebase Pattern Discovery

**This is the most critical part.** Discover how this project already does things before proposing anything new.

### Project rules and conventions
- Review CLAUDE.md, ~/.claude/CLAUDE.md, .claude/skills/, .claude/commands/
- Check linter/formatter/build configs

### Domain skill integration

If the task involves a specific platform, read the relevant domain skill's SKILL.md to identify which reference files apply:
- iOS/iPadOS/Swift/SwiftUI → Read `ios-26-app/SKILL.md`, note relevant reference files
- tvOS/Apple TV → Read `tvos-26-app/SKILL.md`, note relevant reference files
- Cross-platform → Read both, note shared vs platform-specific concerns

Record which domain skill references were consulted in the context doc under `### Domain References`. Do NOT read all reference files — only those relevant to the task (e.g., a navigation change reads layout and navigation refs, not media or camera refs).

Skip this step if the task has no platform-specific concerns.

### Existing patterns
- Use `Grep` and `Glob` to find similar code — use parallel Explore agents for independent areas (max 3 concurrent to limit memory pressure)
- Read 2–3 representative examples in full (not snippets)
- Identify: naming conventions, file organization, module boundaries, import/error/logging patterns
- Look for reusable utilities, helpers, protocols, components

### Affected files — map the blast radius
- Identify every file that will change and read each one. For >10 affected files, read 3–4 representative samples and list the rest with their expected change type.
- Identify directories and naming patterns for new files (exact paths are finalized in Phase 2)
- Trace callers/importers for ripple effects
- **Dependency chains**: For affected files, trace import/dependency relationships. Note which files must change first to maintain buildability (this ordering feeds Phase 2 step sequencing).

### Concurrent development check

If working in a team repository:
- Run `git log --oneline -20` to check recent commits touching affected files
- Run `git branch -r --list '*/*'` to check for in-flight branches that may conflict
- If conflicts are likely, note them in the Risks section of the context doc

Skip this check for solo projects or when the user confirms no concurrent work.

### Dependency discovery

If the task requires new packages, libraries, or frameworks not already in the project:
- List each dependency with: name, purpose, license, last release date, maintenance status
- Flag any with: restrictive licenses (GPL in proprietary projects), no releases in >12 months, known security advisories, large size impact
- Record findings under `### Dependencies` in the context doc
- If any dependency is flagged, raise via `AskUserQuestion` before proceeding

Skip this for standard platform frameworks (UIKit, SwiftUI, Foundation, etc.).

## Phase 1 Exit Ramp

If discovery reveals that the request is already implemented, unnecessary, or fundamentally better solved by a different approach, present findings via `AskUserQuestion`. The user may:
- (a) Proceed with modifications to the existing implementation
- (b) Pivot to a different task
- (c) Close with no changes

Do not force-fit a 3-phase workflow onto a task that doesn't need one.

## Size Check

If Phase 1 discovers **>15 affected files** or **>5 domain boundaries**, warn the user: the task is likely too large for a single `/architect` cycle. Suggest splitting into sequential sub-tasks and immediately invoke `/architect` for the first sub-task.

## Staleness Check

If this Phase 1 doc was written in a previous session (file modification date >24 hours ago), re-verify before proceeding to Phase 2:
- Run `git log --since="<context-doc-date>" -- <affected-files>` to detect changes
- If affected files have changed, update the Context section with new state
- If changes are significant (new code in affected areas, refactored interfaces), warn the user that context may be stale and offer to re-run Phase 1

## Completeness Self-Check

Before writing the context doc, verify:
- [ ] Every acceptance criterion is specific and testable
- [ ] 2–3 similar existing patterns found and read in full
- [ ] Every affected file identified; representative samples read for large sets (>10 files)
- [ ] Reusable code populated — or "none found" with search evidence
- [ ] Edge cases addressed for each requirement
- [ ] Domain skill references identified (if platform-specific task)
- [ ] New dependencies vetted (if any required)

If any check fails, investigate more before writing the doc.

## Output: `## Context` section in `docs/plans/YYYY-MM/<slug>.md`

Create the file with a top-level heading and the `## Context` section:

```markdown
# <Task Title>

## Context

### Goal

<What this achieves and why>

### Acceptance Criteria
- [ ] <Specific, testable criterion>

### Out of Scope
- <What this does NOT do>

### Edge Cases
- <Scenario> → <Expected behavior>

### Q&A Record
- Q: <Question> → A: <Answer>

### Decisions & Rationale
- <Decision> — chose X over Y because <reason>. (Captures rejected alternatives and reasoning that would be lost on `/clear`.)

### Codebase Analysis

#### Existing Patterns to Follow
- <Pattern> — see `path/to/example.ext:line` — <How to use it>

#### Reusable Code Found
- <Utility/component> at `file:line` — <How to reuse>

#### Affected Files
- `path` (modify) — <What changes>
- `path` (create) — <Purpose>

#### Risks
- <Risk> (Low/Med/High) — <Mitigation>

### Domain References (if platform-specific)
- Platform: <iOS/tvOS/cross-platform>
- Skill: `<skill-name>/SKILL.md`
- Reference files to consult in Phase 2: `<file1>`, `<file2>`

### Dependencies (if new deps required)
- `<package>` — <purpose> — license: <X> — last release: <date> — status: <OK/flagged>

### Concurrent Work (if team repo)
- <branch/PR> touching <files> — risk: <low/med/high>
```

## Gate

After writing the `## Context` section to `<slug>.md`, use `AskUserQuestion`:
> "Phase 1 complete. Please review `## Context` in `docs/plans/YYYY-MM/<slug>.md`. Does this accurately capture requirements and patterns? Anything wrong, missing, or unclear?"

**Do NOT proceed to Phase 2 until the user explicitly approves.**

## If Context Is Rejected

When the user rejects the `## Context`:
1. Ask what specifically is wrong or missing (via `AskUserQuestion`)
2. Incorporate feedback, update the `## Context` section in `<slug>.md`
3. Re-present for approval with a summary of what changed
4. Track iteration count in the doc: `**Context iteration**: 2/3`
5. After 3 rejections, escalate: "We've iterated 3 times on context. Should we (a) continue refining, (b) pair on this live, or (c) abandon and rethink the approach?"

When approved: **run `/clear`** to free context, then call `EnterPlanMode` and read `phase-2-plan.md`.
