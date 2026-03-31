---
name: implement
description: Use for /implement, structured implementation, multi-phase workflow. Triggers on /implement, "implement this", "build this feature", phased implementation, structured workflow. 3-phase process (Context → Plan → Execute) that prevents vibe coding by requiring understanding, pattern-matching, and approval before any code is written.
---

# /implement — Structured 3-Phase Implementation

Eliminate vibe coding. Every change must be understood, pattern-matched, documented, and approved before code is written.

**Invoked as:** `/implement $ARGUMENTS`

If `$ARGUMENTS` is empty, use `AskUserQuestion` to ask: "What do you want to implement, fix, or change?"

## Prerequisites

**Before starting Phase 1**, tell the user: "Switch to bypass mode so the workflow runs uninterrupted. The skill has its own approval gates between phases." Only say this once at the start.

## Triage — Fast-Path

Assess complexity before any setup:

- **Trivial** (≤2 files, clear requirements, no architectural impact): Derive slug, run `mkdir -p docs/plans/$(date +%Y-%m)/`, create a single `<slug>-plan.md` using this template, then execute and verify:
  ```markdown
  # <Task Title>
  ## Context
  <Brief requirements + affected files>
  ## Plan
  <Steps with verify criteria>
  ## Verification
  <How to confirm done>
  ```
  **Disqualifiers** — if ANY apply, use the full workflow:
  - Touches shared types, interfaces, or schemas
  - Changes API contracts or public surface area
  - Modifies database schemas or migrations
  - Affects more than one domain boundary
- **Non-trivial** (when in doubt, choose this): Full 3-phase workflow below.

## Setup (non-trivial only)

1. **Derive a task slug** — short, descriptive, filesystem-safe (e.g., `fix-player-crash-on-seek`, not `task-1`)
2. Run `mkdir -p docs/plans/$(date +%Y-%m)/`
3. One doc will be created: `docs/plans/YYYY-MM/<slug>.md` — all phases live in this single file as sections (`## Context`, `## Plan`, `## Implementation`, `## Post-Mortem`)
4. If `<slug>.md` already exists, ask: continuation or fresh start?
   - **Continuation**: Read the file. Detect last completed phase by which sections exist (`## Context` → resume at Phase 2; `## Plan` → resume at Phase 3). Pick up from there.
   - **Fresh start**: Append `-v2` to the slug.

## Phase Workflow

**CRITICAL: Only read the reference file for the current phase. Do NOT read ahead.**

| Phase | What to do | Reference file | Gate |
|-------|-----------|---------------|------|
| **1 — Context** | Gather requirements, discover patterns, map affected files | Read `phase-1-context.md` | User approves `## Context` in `<slug>.md` |
| **2 — Plan** | Design step-by-step implementation plan | Read `phase-2-plan.md` | User approves `## Plan` in `<slug>.md` via `ExitPlanMode` |
| **3 — Execute** | Implement the approved plan | Read `phase-3-execute.md` | Verification passes, user confirms |

### Phase transitions

- **Phase 1 → 2**: After user approves `## Context`, **run `/clear`** to free context, then call `EnterPlanMode` and read `phase-2-plan.md`
- **Phase 2 → 3**: Use `ExitPlanMode` for approval. Once approved, **run `/clear`** to free context, then read `phase-3-execute.md`
- **Never skip a phase. Never merge phases. Never code before Phase 2 is approved.**
- **Always `/clear` between phases** to keep context window lean. The `<slug>.md` file carries all needed state.

### Plan mode vs. plan doc — CRITICAL

`EnterPlanMode`/`ExitPlanMode` are **approval gates only**. The `<slug>.md` file in `docs/plans/` is the actual plan artifact.

- **NEVER present the plan as inline text in plan mode.** Always write the `## Plan` section to `docs/plans/YYYY-MM/<slug>.md` first, then tell the user to review that file.
- The plan mode UI may auto-generate its own name — **ignore it**. Always refer to the plan by its slug and `.md` file path (e.g., "Review the plan at `docs/plans/2026-03/collapsible-nav-bar.md`").
- When calling `ExitPlanMode`, reference the `.md` file so the user knows exactly what they're approving.
- The same applies to Phase 1: write `## Context` to `<slug>.md` first, then present it to the user for approval.

## Core Rules

1. **Reuse before creating** — always search for existing utilities, components, and patterns first
2. **Match conventions** — new code must follow naming, structure, and patterns discovered in Phase 1
3. **Small steps** — each plan step touches ≤3-4 files, project builds after each step
4. **Follow the plan** — if you need to deviate, STOP, explain via `AskUserQuestion`, get approval
5. **Verify everything** — "it compiles" is not done; run the full verification plan
6. **Clean rollback point** — commit/stash before starting Phase 3
7. **Docs are the source of truth** — Any change to specs, requirements, decisions, or answers to clarifying questions MUST be reflected in `<slug>.md`. Never keep decisions only in chat — if it matters, write it down. When the user answers a question or changes a requirement mid-phase, update the doc immediately before continuing.
8. **Resolve conflicts early** — if the request conflicts with project conventions (CLAUDE.md, linter rules, architectural patterns), raise via `AskUserQuestion` before proceeding. Document the resolution in the context doc.
9. **One at a time** — only one `/implement` session may be active. Complete or explicitly abandon the current session before starting another.

## Anti-Patterns

1. Writing code before Phase 2 is approved
2. Creating new code when reusable existing code was found
3. Proceeding to the next phase without explicit user approval
4. Making changes outside the approved plan scope
5. Writing docs from memory instead of actual file reads
6. Keeping decisions, requirement changes, or clarification answers only in chat without updating `<slug>.md`
