# Phase 2: Architectural Plan

You are in Phase 2 (plan mode). The user has approved the context doc. Do NOT read phase-3-execute.md yet.

## Planning Rules

- **Reuse before creating** — the context found existing utilities/patterns; the plan MUST use them
- **Match conventions** — new code follows naming, structure, and patterns from Phase 1
- **Ground everything** — every file path, function name, and approach should come from the `## Context` section. If planning reveals new information not captured in Phase 1, update `## Context` in `<slug>.md` before using it in the plan
- **Small steps** — each step = one logical change, ≤3-4 files
- **Buildable after each step** — no broken intermediate states
- **Order by dependency** — steps must follow the dependency chain discovered in Phase 1. Files that others import/depend on change first. If step 3 touches a shared interface, steps 4+ that consume it come after.
- **Consult domain references** — if Phase 1 identified domain skill references (e.g., `ios-26-app/engineering.md`, `tvos-26-app/references/focus-system.md`), read them NOW to inform step design. Platform-specific patterns (e.g., actor isolation for Swift concurrency, focus flow for tvOS) must be reflected in the plan steps, not discovered during Phase 3.

## Cross-Cutting Concerns Checklist

Review each concern below. Mark as "N/A" or address it in the plan. Only concerns relevant to THIS task need action — do not force-fit items that don't apply. If a concern applies, add a dedicated step, a verification criterion in an existing step, or a note in Risks.

| Concern | When it typically applies |
|---------|--------------------------|
| **Security** — input validation, auth checks, data exposure, injection risks | Input handling, auth, data storage, network calls |
| **Performance** — memory, CPU, network, battery impact; benchmarks needed? | Hot paths, large data sets, battery/memory sensitive code |
| **Accessibility** — VoiceOver, Dynamic Type, contrast, focus order | Any UI change (see domain skill refs for platform specifics) |
| **Observability** — logging, metrics, crash reporting for new code paths | New code paths, error handling, user-facing features |
| **Testing** — unit vs integration vs UI tests; mocking strategy; coverage | Almost always — specify what kind of tests and why |
| **Concurrency** — race conditions, actor isolation, main thread safety, deadlocks | Async code, shared state, actors, multi-threaded access |
| **Memory** — retain cycles, leak potential (closures, delegates, async captures) | Closures, delegation, long-lived objects, caches |
| **API contracts** — versioning, backward compat, deprecation for changed interfaces | Changed interfaces, public surface area, shared protocols |
| **CI/CD** — new build steps, pipeline changes, build time impact | New targets, dependencies, build config changes |
| **Documentation** — API docs, changelog entries, migration guide for breaking changes | Public API changes, breaking changes, new features |
| **Cross-platform** — does this change affect shared modules between platforms? | Code shared between iOS/tvOS or other platform boundaries |
| **i18n** — new user-facing strings, pluralization, RTL layout impact | New UI text, format strings, layout changes |

### How to use this checklist
- Read each row. If irrelevant (e.g., "Security" for a pure UI spacing fix), mark N/A.
- If relevant, add either: (a) a dedicated plan step, (b) a verification criterion in an existing step, or (c) a note in Risks.
- Record the completed checklist in `<slug>.md` under `## Plan > ### Cross-Cutting Concerns`.
- For platform-specific concerns (a11y, memory, concurrency), consult the domain skill reference files identified in Phase 1.

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

### Cross-Cutting Concerns
| Concern | Applies? | Action |
|---------|----------|--------|
| Security | N/A / Yes | <step ref or mitigation> |
| Performance | N/A / Yes | <step ref or mitigation> |
| Accessibility | N/A / Yes | <step ref or mitigation> |
| Observability | N/A / Yes | <step ref or mitigation> |
| Testing | N/A / Yes | <step ref or mitigation> |
| Concurrency | N/A / Yes | <step ref or mitigation> |
| Memory | N/A / Yes | <step ref or mitigation> |
| API contracts | N/A / Yes | <step ref or mitigation> |
| CI/CD | N/A / Yes | <step ref or mitigation> |
| Documentation | N/A / Yes | <step ref or mitigation> |
| Cross-platform | N/A / Yes | <step ref or mitigation> |
| i18n | N/A / Yes | <step ref or mitigation> |

### Verification Plan
- Build: `<command from project's CLAUDE.md or build config>` → succeeds
- Tests: `<command from project's CLAUDE.md or build config>` → all pass
- Manual: <What to check> → <Expected outcome>

### PR Preparation (if applicable)
- Branch name: `<branch-name>`
- PR title: `<concise title>`
- Key reviewers: `<who should review and why>`
- Review notes: `<what reviewers should focus on>`

### Risks
- <Risk> (Low/Med/High) — <Mitigation>
```

## Gate

1. **Append** the `## Plan` section to `docs/plans/YYYY-MM/<slug>.md` using the Edit tool.
2. **Present** the plan to the user by telling them: "Review `## Plan` in `docs/plans/YYYY-MM/<slug>.md`." Do NOT paste the plan as inline text — the `.md` file IS the plan.
3. **Call `ExitPlanMode`** to request approval, referencing the `.md` file path.

**Do NOT start writing code until the user approves the plan.**

## If Plan Is Rejected

When the user rejects the `## Plan`:
1. Ask what specifically needs to change (via `AskUserQuestion`)
2. Determine if the change is:
   (a) **Plan-level**: modify steps, reorder, add/remove steps → update `## Plan`
   (b) **Context-level**: wrong assumptions, missing requirements → go back to Phase 1, update `## Context`, then re-plan
3. Update `<slug>.md` with changes, re-present for approval
4. Track: `**Plan iteration**: 2/3`
5. After 3 rejections: same escalation as Phase 1 rejection

When approved: **run `/clear`** to free context, then read `phase-3-execute.md`.
