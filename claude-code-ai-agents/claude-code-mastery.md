# Claude Code Mastery

Everything you need to know about customizing Claude Code itself — CLAUDE.md authoring, skills anatomy, hooks, settings.json, plan mode, slash commands, and the memory system. This file covers Claude Code configuration only. For Agent SDK, MCP, agentic workflows, or security, see the other reference files in this skill.

---

## 1. CLAUDE.md — the instruction hierarchy

### How the hierarchy works

Claude Code loads CLAUDE.md files in a layered cascade. Each layer adds to the previous without replacing it. The load order, from broadest to most specific:

1. **User-level** (`~/.claude/CLAUDE.md`) — personal instructions that apply to every project you work on
2. **Project-root** (`<project>/CLAUDE.md` or `<project>/.claude/CLAUDE.md`) — team conventions committed to git
3. **Subdirectory** (`<project>/src/CLAUDE.md`, `<project>/packages/api/CLAUDE.md`, etc.) — scoped conventions for specific parts of the codebase

When all three exist, Claude sees them all. User-level instructions are overridden by more-specific project instructions when they conflict. Subdirectory instructions are active only when Claude is working in or near that directory.

**Practical implications:**
- Put personal preferences (preferred response style, personal tooling shortcuts, how you like test files organized) in user-level CLAUDE.md
- Put team conventions (tech stack, code style, PR process, deployment procedures) in project-level CLAUDE.md and commit it
- Put package-specific rules (API package has different conventions than frontend package) in subdirectory CLAUDE.md files in a monorepo

### The 150-200 instruction compliance cliff

Claude Code follows CLAUDE.md instructions at approximately 80% compliance under normal conditions. This number drops significantly beyond 150-200 instructions (not lines — discrete actionable instructions). The mechanism is attention dilution: as more instructions compete for the model's attention, earlier instructions in a long list receive less weight in generation.

Practical consequences:
- Instructions 1-50: very high compliance, ~90-95%
- Instructions 51-100: high compliance, ~85%
- Instructions 101-150: good compliance, ~75-80%
- Instructions 150+: declining compliance, especially for instructions at the end of the list

This means instruction ordering matters: put your most important conventions near the top. And it means CLAUDE.md length is a strategic resource, not a dumping ground.

**The cliff is why hooks exist.** Any rule that must be followed 100% of the time cannot rely on CLAUDE.md alone. Use hooks for mandatory enforcement.

### What to include in CLAUDE.md

**Good candidates:**
- **Tech stack and version pins**: "This project uses TypeScript 5.4, Next.js 15, React 19, Tailwind CSS v4." Prevents Claude from suggesting outdated patterns.
- **Package manager**: "Always use `pnpm`. Never suggest `npm install` or `yarn`."
- **Code style decisions**: "Use `const` over `let` unless mutation is required. Avoid `var`." Concrete, checkable rules.
- **Naming conventions**: "React components use PascalCase. Utility functions use camelCase. Constants use SCREAMING_SNAKE_CASE."
- **File organization**: "API routes go in `src/app/api/`. Database models go in `src/lib/db/models/`. Shared types go in `src/types/`."
- **Recurring shortcuts**: "To run tests: `pnpm test`. To build: `pnpm build`. To start dev server: `pnpm dev`."
- **Things to avoid**: "Never use `any` in TypeScript. Never use inline styles. Never commit `.env` files."
- **Pointers to authoritative files**: "Auth implementation is in `src/lib/auth/index.ts:42`. Database schema is in `prisma/schema.prisma`."
- **PR process**: "Before committing, run `pnpm lint && pnpm test`. PR titles use conventional commit format."

**Bad candidates:**
- **Secrets and credentials**: CLAUDE.md is often committed to git. Never.
- **Code snippets**: "Here's how we do authentication: `[50 lines of code]`." Code goes stale. Refer to the actual file instead.
- **Long prose explanations**: Background context, architectural histories, "you should know that we originally built this as..." — this is documentation, not instruction. Put it in a README or a skill reference file.
- **One-off decisions**: "For this current refactor, use the new approach." Put session-specific decisions in plan files.
- **Complete API documentation**: Use `file:line` references to authoritative sources instead.

### File references: the right way to point at code

Instead of pasting code into CLAUDE.md (which goes stale and wastes budget), use `file:line` references to point Claude at the authoritative source:

```markdown
## Authentication

Auth implementation: `src/lib/auth/index.ts:1-80`
Session middleware: `src/middleware.ts:15-45`
Auth types: `src/types/auth.ts`
```

Claude will read these files on demand rather than having them loaded into context unconditionally.

### Advisory vs deterministic

CLAUDE.md instructions are **advisory** — they guide behavior without enforcing it. The model genuinely tries to follow them, but 100% compliance is not guaranteed, and compliance degrades with context length, conversation drift, and instruction count.

For any rule that must be enforced without exception, see Section 3 (Hooks). The pattern is:
- Advisory (CLAUDE.md): "Prefer functional components over class components."
- Deterministic (hook): "Never allow writes to files in `/etc/` or `~/.ssh/`."

The distinction is: what is the cost of a single violation? Advisory rules are fine for style and convention. Deterministic rules are required for security, safety, and compliance.

### Complete example: CLAUDE.md for a Next.js + TypeScript project

```markdown
# Project: Acme SaaS Platform

## Tech stack
- TypeScript 5.4 (strict mode, `"noUncheckedIndexedAccess": true`)
- Next.js 15 with App Router. Never use Pages Router patterns.
- React 19. Use Server Components by default. Client Components only when interactivity requires it.
- Tailwind CSS v4. Use `@theme` for design tokens. No inline styles. No arbitrary values unless unavoidable.
- Prisma 5.x for ORM. Schema at `prisma/schema.prisma`.
- Auth.js v5 for authentication. Config at `src/auth.ts`.
- Zod for all schema validation (API responses, form data, env vars).
- pnpm. Never suggest npm or yarn.

## Code conventions
- `const` over `let`. Never `var`.
- No `any`. Use `unknown` + narrowing or proper types.
- React components: PascalCase, one per file.
- Utility functions: camelCase in `src/lib/`.
- Shared types: `src/types/`. Never define types inline in component files.
- Server Actions in `src/actions/`. Always validate inputs with Zod before database operations.
- API routes in `src/app/api/`. Return typed responses using `NextResponse.json<T>()`.
- Database queries only in `src/lib/db/`. Never query Prisma directly in components or Server Actions.

## File organization
```
src/
  app/          ← Next.js App Router routes
  actions/      ← Server Actions
  components/   ← Shared React components
  lib/
    db/         ← All database access (Prisma queries)
    auth/       ← Auth utilities and session helpers
  types/        ← Shared TypeScript types
  hooks/        ← Client-side React hooks
```

## Key files (read before modifying related areas)
- Auth flow: `src/auth.ts` + `src/middleware.ts:1-40`
- DB client: `src/lib/db/client.ts`
- Env validation: `src/env.ts` (using `@t3-oss/env-nextjs`)
- Design tokens: `src/styles/tokens.css`

## Commands
- Dev: `pnpm dev`
- Test: `pnpm test` (Vitest)
- Test E2E: `pnpm test:e2e` (Playwright)
- Lint: `pnpm lint` (ESLint + Prettier)
- Build: `pnpm build`
- DB migrate: `pnpm db:migrate`
- DB studio: `pnpm db:studio`

## What to avoid
- Never use `export default` in non-component files. Use named exports.
- Never commit `.env` or `.env.local`.
- Never use `process.env` directly — always use the typed `env` object from `src/env.ts`.
- Never add `// @ts-ignore` without a team-approved comment explaining why.
- Never use `document` or `window` in Server Components.

## Before committing
1. `pnpm lint` — must pass with zero warnings
2. `pnpm test` — all tests must pass
3. PR title format: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`
```

---

## 2. Skills — reusable prompt fragments

### What a skill is

A skill is a markdown file that gets loaded into Claude's context when one or more trigger keywords appear in the conversation. Unlike CLAUDE.md (always loaded, always present), skills are loaded on demand — they add domain knowledge exactly when it's needed, without permanently consuming context budget.

The skill system implements a form of lazy context loading: Claude Code watches the conversation for trigger matches and injects the relevant skill file. This keeps base context slim and brings in specialized knowledge at the right moment.

### YAML frontmatter schema

Every skill starts with YAML frontmatter in the `---` block:

```yaml
---
name: my-skill-name
description: >
  This is the trigger text. Every keyword in this description is a
  potential match target. The more precise and comprehensive the
  keyword list, the more reliably the skill triggers.
  Examples: TypeScript, React, Next.js, App Router, Server Components,
  useActionState, Tailwind CSS v4, @theme directive.
---
```

**`name`** — the skill identifier. Used in the file path (`~/.claude/skills/<name>/SKILL.md`) and for cross-references.

**`description`** — this is the trigger source. Claude Code uses keyword matching against the description field to decide when to load the skill. Think of the description as a comprehensive keyword list that doubles as a human-readable summary. Include: the main technology names, version numbers, key API names, CLI flags, file names, and common task descriptions that would appear in conversation when this skill is relevant.

**Trigger matching in practice:**
- Conversational message "I need to build a Next.js Server Component with Tailwind v4" → triggers `web-frontend` skill
- Conversational message "Help me set up a PreToolUse hook" → triggers `claude-code-ai-agents` skill
- Conversational message "Build an iOS app with SwiftUI" → triggers `ios-26-app` skill

The match is fuzzy and keyword-based, not semantic. If a keyword appears in conversation and also appears in any skill's description field, that skill is a candidate for loading. Include the keywords your users will actually type.

### Skill file locations

**User-level skills** (personal, applies across all projects):
```
~/.claude/skills/<skill-name>/SKILL.md
~/.claude/skills/<skill-name>/reference-file-1.md
~/.claude/skills/<skill-name>/reference-file-2.md
```

**Project-level skills** (team-shared, committed to git):
```
<project>/.claude/skills/<skill-name>/SKILL.md
<project>/.claude/skills/<skill-name>/reference-file-1.md
```

Project-level skills take precedence over user-level skills when both exist with the same name. This allows teams to override personal workflow preferences for specific projects.

### Skills as fragments

Skills supplement — they do not replace — the system prompt and CLAUDE.md. Think of a skill as a domain expert joining the conversation: they bring specialized knowledge on demand without dominating every interaction.

A SKILL.md file typically contains:
- A reference table listing companion reference files and when to read each
- Core principles for the domain
- A decision framework (the main structured workflow)
- Anti-patterns (what not to do)
- Code generation guidelines

### Reference files: eager vs lazy loading

A SKILL.md often has companion reference files that contain deep domain knowledge. These should be loaded selectively, not all at once.

**Eager loading** (instruct Claude to read immediately when skill loads):
- Core reference files needed for almost every task in this domain
- Small files (<5KB) with universally applicable information
- Decision tables that affect how to interpret all other guidance

**Lazy loading** (instruct Claude to read when a specific trigger occurs):
- Deep reference files needed only for specific sub-tasks
- Large files (>10KB) that would significantly impact context budget
- Platform-specific or version-specific details

The SKILL.md's reference table communicates this intent via the "Read when" column:

```markdown
| File | Read when |
|------|-----------|
| `core-patterns.md` | Building any component in this domain — read immediately |
| `security.md` | Security configuration, auth, secrets management |
| `testing.md` | Writing tests or debugging test failures |
| `performance.md` | Optimization work, performance profiling, bundle analysis |
```

### Cross-references between skills

A skill can instruct Claude to read reference files from other skills when relevant. This enables composition without duplication:

```markdown
## Cross-skill integration

When building agent workflows that include iOS app integration:
- Read `../ios-26-app/foundation-models.md` for on-device inference patterns
- Read `../ios-26-app/swift-concurrency.md` for async agent coordination

When building agent workflows that include Next.js frontend:
- Read `../web-frontend/engineering.md` for Claude API integration patterns
- Read `../web-frontend/auth-security.md` for session and token handling
```

### AskUserQuestion usage in skills

Skills should enforce clarification before acting. RULE #1 — always ask first — applies with equal force inside skills as it does in the base system. Every skill's decision framework should begin with a set of clarifying questions surfaced via `AskUserQuestion`. Do not assume the user's context, constraints, or requirements. A single clarifying question asked upfront saves multiple rounds of wrong-direction implementation.

### The Agent Skills open standard

The SKILL.md format (`---` YAML frontmatter + markdown body + companion reference files in a named directory) is an open standard that works across multiple AI coding assistants in 2026:
- **Claude Code**: native support, keyword-based trigger matching
- **Cursor**: compatible via AI rules format with SKILL.md convention
- **Gemini CLI**: compatible via context injection

Skills you author in SKILL.md format are portable across these tools, though trigger matching behavior varies by implementation. Write skills as self-contained reference documents that are useful when read directly, not just when triggered by keyword matching.

### Minimal SKILL.md example

```markdown
---
name: prisma-patterns
description: >
  Use for Prisma ORM, database schema, migrations, Prisma Client, 
  Prisma queries, relations, transactions, schema.prisma, 
  prisma migrate, prisma generate, prisma studio, seeding,
  database performance, N+1 queries, Prisma middleware.
---

# Prisma Patterns

Before writing any Prisma code, read the relevant reference files below.

## Reference files

| File | Read when |
|------|-----------|
| `schema-patterns.md` | Designing models, relations, indexes, enums, migrations |
| `query-patterns.md` | Writing queries, filtering, pagination, aggregations, transactions |
| `performance.md` | Fixing N+1 queries, query optimization, connection pooling |
| `testing.md` | Mocking Prisma in tests, test database setup |

## Core principles

### 1. Schema is the source of truth
Never write TypeScript types for database entities by hand — derive them from the Prisma schema using `Prisma.UserGetPayload<...>` and generated types. Hand-written types drift from the actual schema and cause runtime errors.

### 2. Queries belong in the data layer
All Prisma queries belong in `src/lib/db/`. Never import the Prisma client directly in Server Actions, API routes, or components. Data access layer functions are testable in isolation; scattered queries are not.

## RULE #1 — Ask before designing schema

Use AskUserQuestion to clarify:
1. What entities does this feature need? What are their relationships?
2. What queries will be most frequent? (drives index decisions)
3. Is soft deletion required, or hard delete?
4. What's the expected data volume? (affects index strategy significantly)
```

---

## 3. Hooks — 100% compliance enforcement

### What hooks are

Hooks are shell scripts (or any executable) that Claude Code invokes synchronously at lifecycle points during tool execution. Unlike CLAUDE.md guidelines (advisory), hooks are code — they run deterministically at specified points and their exit codes control whether Claude proceeds.

Hooks are the enforcement layer of Claude Code customization. Use them for any rule that must be followed 100% of the time: security policies, code quality gates, mandatory side effects (formatting, logging), and prohibited actions.

### The two hook types

**PreToolUse** — runs before a tool executes. Can inspect the tool input and block execution by returning exit code 2. Use for:
- Security checks (block writes to sensitive paths)
- Validation (ensure required args are present)
- Confirmation gates (require explicit user approval for destructive commands)
- Audit logging (record what Claude is about to do)

**PostToolUse** — runs after a tool executes, receives the tool output. Cannot block (the tool already ran). Use for:
- Automatic formatting (`prettier`, `gofmt`, etc.) after file writes
- Testing (run tests after code changes)
- Notifications (alert when long-running tools complete)
- Audit logging (record what Claude did and what the result was)

### Exit code semantics

| Exit code | Meaning | Behavior |
|-----------|---------|----------|
| `0` | Success — proceed | Claude continues with the tool call normally |
| `2` | Blocking error | Claude sees the hook's stdout as an error message and does NOT execute the tool |
| Any other | Warning | Claude sees the hook's stdout as context but still executes the tool |

The stdout of a hook with exit code 2 is shown to Claude as the reason the tool call was blocked. Write clear, actionable error messages — Claude will use them to understand what happened and adjust its approach.

### Hook configuration location

**Project-level** (committed to git, applies to everyone in the repo):
```
<project>/.claude/settings.json
```

**User-level** (personal, applies to all projects):
```
~/.claude/settings.json
```

The hooks field in `settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/pre-bash-hook.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/post-edit-hook.sh"
          }
        ]
      }
    ]
  }
}
```

### The `matcher` field

The `matcher` field targets specific tools or groups of tools:

| Matcher value | Matches |
|--------------|---------|
| `"*"` | All tool calls |
| `"Bash"` | All Bash tool calls |
| `"Edit"` | All Edit tool calls |
| `"Write"` | All Write tool calls |
| `"Bash(rm *)"` | Bash calls where the command starts with `rm` |
| `"Bash(git push*)"` | Bash calls starting with `git push` |
| `"Edit(*.ts)"` | Edit calls on TypeScript files |
| `"Edit(/etc/*)"` | Edit calls targeting /etc/ |

Glob patterns in the matcher are supported. Multiple matcher entries can target the same tool type — all matching hooks run for a given tool call.

### Environment variables available to hook scripts

| Variable | Value | Available in |
|----------|-------|-------------|
| `CLAUDE_TOOL_NAME` | Name of the tool being called (e.g., `"Bash"`, `"Edit"`) | Pre and Post |
| `CLAUDE_TOOL_INPUT` | JSON-encoded tool input (command string, file path + content, etc.) | Pre and Post |
| `CLAUDE_TOOL_OUTPUT` | JSON-encoded tool output (stdout, return value, etc.) | Post only |
| `CLAUDE_SESSION_ID` | Unique identifier for the current Claude Code session | Pre and Post |
| `CLAUDE_PROJECT_DIR` | Absolute path to the project root | Pre and Post |

Parse `CLAUDE_TOOL_INPUT` with `jq` (shell) or `json.loads()` (Python) to extract specific fields.

### Example: PreToolUse blocking dangerous Bash commands

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/pre-bash-security.sh
# Blocks dangerous shell commands. Exit 2 to block, 0 to allow.

set -euo pipefail

# Parse the command from tool input
COMMAND=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.command // ""')

# Block recursive force-delete
if echo "$COMMAND" | grep -qE 'rm\s+-[^-]*r[^-]*f|rm\s+-[^-]*f[^-]*r'; then
  echo "BLOCKED: Recursive force-delete (rm -rf) is not permitted by project policy."
  echo "Use 'rm -r' with explicit path confirmation, or 'trash' for recoverable deletion."
  exit 2
fi

# Block piping curl output to shell
if echo "$COMMAND" | grep -qE 'curl.+\|\s*(ba)?sh|wget.+\|\s*(ba)?sh'; then
  echo "BLOCKED: Piping curl/wget output to shell is a security risk."
  echo "Download the file first, inspect it, then execute if appropriate."
  exit 2
fi

# Block writes to sensitive system paths
if echo "$COMMAND" | grep -qE '(>|>>|tee)\s+/etc/|sudo\s+tee\s+/etc/'; then
  echo "BLOCKED: Writes to /etc/ are not permitted."
  exit 2
fi

# Allow everything else
exit 0
```

Configuration:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "~/.claude/hooks/pre-bash-security.sh" }]
      }
    ]
  }
}
```

### Example: PostToolUse auto-formatting TypeScript files

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/post-edit-format.sh
# Runs prettier after any TypeScript/JavaScript file edit.

set -euo pipefail

# Parse the file path from tool input
FILE_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // ""')

# Only format TypeScript and JavaScript files
if echo "$FILE_PATH" | grep -qE '\.(ts|tsx|js|jsx|mjs|cjs)$'; then
  # Run prettier in background to avoid blocking (fire-and-forget)
  (cd "$CLAUDE_PROJECT_DIR" && npx prettier --write "$FILE_PATH" 2>/dev/null) &
fi

exit 0
```

For the complete hooks reference — 12+ examples, performance rules, Python hook examples, complex multi-condition logic, and debugging tips — see `hooks-automation.md`.

---

## 4. settings.json — full schema reference

The `settings.json` file is Claude Code's runtime configuration. It controls model selection, effort level, tool permissions, hook registration, and more.

**Project-level** (committed to git):
```
<project>/.claude/settings.json
```

**User-level** (personal, applies to all projects):
```
~/.claude/settings.json
```

Project-level settings are merged with user-level settings. Where both define the same field, project-level takes precedence. This allows user-level defaults that projects can override.

### Complete annotated schema

```json
{
  "model": "claude-opus-4-6",
  "effortLevel": "high",
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm *)",
      "Bash(pnpm *)",
      "Bash(npx prettier *)",
      "Bash(npx eslint *)",
      "Edit(*)",
      "Read(*)",
      "Write(*)",
      "Glob(*)",
      "Grep(*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl * | bash)",
      "Bash(wget * | sh)",
      "Bash(sudo *)",
      "Edit(/etc/*)",
      "Edit(~/.ssh/*)",
      "Edit(~/.aws/*)",
      "Write(/etc/*)"
    ],
    "defaultMode": "default"
  },
  "additionalDirectories": [
    "/Users/dev/shared-libs",
    "/Users/dev/design-tokens"
  ],
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/pre-bash-security.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/post-edit-format.sh"
          }
        ]
      }
    ]
  },
  "enabledPlugins": ["github-mcp", "stripe-mcp"],
  "spinnerTipsEnabled": false
}
```

### Field reference

**`model`** — Override the model used for this project. Valid values: `"claude-opus-4-6"`, `"claude-sonnet-4-6"`, `"claude-haiku-4-5"`. If omitted, Claude Code uses the default model. Project-level settings can pin a specific model; user-level settings can set a personal default.

**`effortLevel`** — Controls reasoning depth and compute:
- `"low"` — fast, uses haiku-class reasoning. Good for simple edits, quick searches, repetitive tasks.
- `"medium"` — balanced (default). Good for most development tasks.
- `"high"` — deep, uses extended thinking (opus-class). Good for architecture decisions, complex debugging, multi-file refactors. Significantly higher latency and cost.

**`permissions.allow`** — Explicit whitelist of permitted tool calls using glob patterns. Format: `"ToolName(argument-glob)"`. Examples:
- `"Bash(git *)"` — allows all git commands
- `"Edit(src/*)"` — allows edits only within `src/`
- `"Read(*)"` — allows reading any file
- `"Bash"` — allows any Bash command (use with caution)

**`permissions.deny`** — Explicit denylist. Evaluated after `allow`. Deny overrides allow — if a command matches both an allow rule and a deny rule, it is denied. Use deny rules for absolute prohibitions that should never be overridden.

**`permissions.defaultMode`** — What happens for tool calls that match neither allow nor deny:
- `"default"` — Claude Code prompts the user to approve or deny
- `"plan"` — Forces plan mode: read-only exploration until the user explicitly approves implementation
- `"acceptEdits"` — Auto-approves all edits without user prompt (use only when you trust the full toolset)

**`additionalDirectories`** — An array of absolute paths that Claude Code is allowed to read and edit, in addition to the project root. Useful for monorepos with shared packages outside the project directory, or for pointing at personal library folders.

**`hooks`** — Hook registration. See Section 3 for full hook documentation. The structure is:
```json
{
  "hooks": {
    "PreToolUse": [ { "matcher": "...", "hooks": [{ "type": "command", "command": "..." }] } ],
    "PostToolUse": [ { "matcher": "...", "hooks": [{ "type": "command", "command": "..." }] } ]
  }
}
```

**`enabledPlugins`** — Array of MCP server plugin names to activate for this project. Names must match the server identifiers in `mcp.json`. See `mcp-consuming.md` for full MCP configuration reference.

**`spinnerTipsEnabled`** — `false` to disable the loading spinner tips. Minor UX preference. Default: `true`.

### Precedence rules

When both user-level and project-level `settings.json` exist:
- `model` — project overrides user
- `effortLevel` — project overrides user
- `permissions.allow` — arrays are merged (union), project additions add to user-level allows
- `permissions.deny` — arrays are merged (union), project additions add to user-level denies
- `permissions.defaultMode` — project overrides user
- `additionalDirectories` — arrays are merged
- `hooks` — project hooks are added to user-level hooks; both sets run
- `enabledPlugins` — arrays are merged

This means you can set personal baseline permissions (tools you always allow, paths you always deny) at user level, and projects can extend those without replacing them.

---

## 5. Plan mode — structured decision gates

### What plan mode is

Plan mode is a read-only exploration and design phase that runs before implementation. In plan mode, Claude can read files, search code, browse the codebase, and write to plan files — but cannot modify existing code, run build commands, or make any changes to the project.

Plan mode serves as a human-in-the-loop checkpoint: it forces a design review before implementation begins, gives the human a concrete artifact (the plan document) to approve or modify, and ensures implementation decisions are made deliberately rather than discovered incrementally.

### Activating plan mode

**Via tool** — Claude Code exposes `EnterPlanMode` as a tool. Skills and CLAUDE.md can instruct Claude to always enter plan mode for specific tasks:
```
For any multi-file refactor, call EnterPlanMode before writing any code.
```

**Via settings** — Set `defaultMode: "plan"` in `settings.json` to make plan mode the default for all sessions in a project. Claude will automatically enter plan mode on session start and require explicit approval to begin implementation.

**Via conversation** — Users can type "plan first" or "enter plan mode" to trigger plan mode interactively.

**Via the `architect` skill** — The architect skill uses `EnterPlanMode` as a built-in approval gate between its three phases (Context, Plan, Execute). This is the recommended pattern for complex multi-file features.

### What's allowed in plan mode

**Allowed:**
- `Read` — read any file
- `Glob` — search file paths
- `Grep` — search file contents
- `Write` to plan files only (see naming convention below)
- `Agent` with `Explore` or `Plan` subagent types

**Not allowed:**
- `Edit` — cannot modify existing files
- `Write` to non-plan files
- `Bash` commands that modify state (git commits, file operations, build commands)
- `Bash` commands with side effects (npm install, db migrations)

This restriction is enforced by the Claude Code runtime, not by CLAUDE.md — it cannot be overridden by conversation.

### Plan file naming convention

Plan files are written during plan mode and serve as the persistent artifact of the design session:

```
docs/plans/YYYY-MM/<feature-slug>.md
```

Examples:
- `docs/plans/2026-04/user-profile-redesign.md`
- `docs/plans/2026-04/stripe-payment-integration.md`
- `docs/plans/2026-04/agent-pipeline-v2.md`

Commit plan files to git. They are permanent records of architectural decisions — valuable for onboarding, postmortems, and understanding why code is structured the way it is.

### `ExitPlanMode` — presenting the plan for approval

After completing exploration and writing a plan file, Claude calls `ExitPlanMode`. This surfaces the plan to the user with an explicit approve/reject/revise prompt. Implementation only begins after the user approves.

The `ExitPlanMode` call includes:
- A summary of the plan (key decisions, components to create/modify, approach)
- The path to the plan file
- An explicit request for approval

**Typical plan mode flow:**
1. User describes feature → Claude enters plan mode
2. Claude explores codebase (Read, Glob, Grep)
3. Claude writes plan to `docs/plans/YYYY-MM/<slug>.md`
4. Claude calls `ExitPlanMode` with summary
5. User reviews plan file and approves, rejects, or requests changes
6. If approved: Claude exits plan mode and begins implementation
7. If changes requested: Claude re-enters plan mode, revises plan, requests approval again

### The plan file as a durable artifact

The plan file's value extends beyond the current session. A plan file committed to git:
- Survives context resets (even if the conversation is lost, the plan persists)
- Provides onboarding context for new team members
- Creates an auditable record of why architectural decisions were made
- Enables "replay" of decisions when refactoring or debugging

Plan files are not just temporary scaffolding — they are design documents. Write them clearly enough that someone unfamiliar with the conversation history can understand the reasoning.

### Anti-pattern: using plan mode without writing a plan file

The most common plan mode mistake is using it purely conversationally: "Let me think about this..." without writing a plan file. When the context window fills or the session ends, all that thinking is lost. The plan file is the point — always write it.

---

## 6. Slash commands — custom workflow triggers

### What slash commands are

Slash commands are markdown files that define reusable workflows. When a user types `/<command-name>` in Claude Code, the corresponding markdown file is loaded as a prompt and Claude executes the workflow described in it.

Slash commands are the user-facing interface for automating common workflows — they are simpler than skills (no YAML frontmatter, no trigger matching) but more explicit (user intentionally invokes them).

### File locations

**User-level** (personal, available in all projects):
```
~/.claude/commands/<name>.md
```

**Project-level** (team-shared, committed to git):
```
<project>/.claude/commands/<name>.md
```

The command name is the filename without `.md`. A file at `~/.claude/commands/commit.md` is invoked as `/commit`.

### `$ARGUMENTS` substitution

The text after the command name is injected into the command file at the `$ARGUMENTS` placeholder:

```
User types: /review-pr 1234
$ARGUMENTS = "1234"
```

This enables commands that take parameters without requiring a full conversation to extract them.

### Example: `/commit` — automated conventional commit

File: `~/.claude/commands/commit.md`

```markdown
Create a git commit for the currently staged changes.

Follow this process:
1. Run `git diff --staged` to see what's staged
2. Run `git log --oneline -5` to understand recent commit style
3. Analyze the changes and determine the commit type:
   - `feat:` new feature or capability
   - `fix:` bug fix
   - `refactor:` code change with no behavior change
   - `chore:` tooling, dependencies, build changes
   - `docs:` documentation only
   - `test:` test additions or changes
   - `perf:` performance improvement
4. Write a commit message: `<type>(<optional-scope>): <imperative description>`
   - Subject line: 50 chars max, imperative mood ("add" not "added")
   - Body (if needed): wrapped at 72 chars, explains WHY not WHAT
5. Create the commit: `git commit -m "$(cat <<'EOF'\n<message>\nEOF\n)"`
6. Show the created commit with `git log -1 --stat`

Do not stage additional files unless the user asked. Do not push. Do not amend unless explicitly asked.
```

### Example: `/review-pr $ARGUMENTS` — pull request review

File: `~/.claude/commands/review-pr.md`

```markdown
Review GitHub pull request #$ARGUMENTS.

Use the GitHub MCP tool to fetch the PR details.

Follow this review process:
1. Fetch PR metadata: title, description, files changed, author, target branch
2. Read each changed file in full
3. For each file, check:
   - Does the implementation match the PR description?
   - Are there any obvious bugs or logic errors?
   - Are there security concerns (SQL injection, XSS, secret exposure, missing auth)?
   - Does the code follow project conventions from CLAUDE.md?
   - Are edge cases handled?
   - Is error handling appropriate?
4. Check for missing tests for new functionality
5. Write a structured review:

## Summary
[One paragraph: overall assessment, blocking issues if any]

## Blocking Issues
[List of things that must be fixed before merge — numbered]

## Suggestions
[Non-blocking improvements — numbered, labeled "Optional:"]

## Questions
[Anything unclear that needs clarification from the author]

Be specific: quote the relevant code, reference the file and line. A vague review helps no one.
```

### Combining slash commands with skills

A slash command can reference a skill's workflow:

```markdown
# /scaffold-feature

This command scaffolds a new feature using the web-frontend skill's component generation workflow.

$ARGUMENTS is the feature name.

1. Create the directory structure for feature "$ARGUMENTS":
   - `src/app/(app)/$ARGUMENTS/page.tsx` — page component
   - `src/app/(app)/$ARGUMENTS/loading.tsx` — loading state
   - `src/components/$ARGUMENTS/` — feature components directory
   - `src/actions/$ARGUMENTS.ts` — Server Actions
   - `src/lib/db/$ARGUMENTS.ts` — database access layer

2. Follow the web-frontend skill's Server Component patterns for page.tsx
3. Follow the web-frontend skill's Server Action patterns for actions
4. Follow the prisma-patterns skill's data layer patterns for db access
```

---

## 7. Memory system

### Auto-save behavior

Claude Code automatically saves memories when it observes patterns worth preserving: corrections you make to its behavior, preferences you express, project-specific facts it learns during the session, and validated approaches that produced good results. This happens without explicit prompting — Claude decides what's worth persisting based on relevance and recurrence.

The auto-save is not comprehensive: Claude doesn't save everything. It saves what it predicts will be useful in future sessions. If you want something saved explicitly, tell Claude directly: "Remember that I prefer…" or "Save the fact that this project uses…"

### Memory file locations

**Project memory** — facts relevant to a specific project:
```
~/.claude/projects/<project-hash>/memory/
```

The `<project-hash>` is a hash of the project's absolute path. Claude manages this mapping automatically — you don't need to know the hash.

**User (global) memory** — personal preferences and cross-project patterns:
```
~/.claude/projects/-global/memory/
```

The `-global` directory is the special directory for cross-project user memory.

Each memory directory contains individual memory files (`.md`) plus a `MEMORY.md` index file.

### MEMORY.md index format

The top-level `MEMORY.md` in a memory directory is an index of all memory entries:

```markdown
# Memory Index

## User memories
- [my-coding-preferences.md](my-coding-preferences.md) — preferred code style, language choices, response format
- [typescript-patterns.md](typescript-patterns.md) — validated TypeScript approaches this user prefers

## Project memories  
- [acme-architecture.md](acme-architecture.md) — current architecture state, key decisions
- [acme-team-context.md](acme-team-context.md) — team members, responsibilities, communication style
- [acme-ongoing-work.md](acme-ongoing-work.md) — in-progress features, blockers, next steps

## Reference memories
- [stripe-integration-notes.md](stripe-integration-notes.md) — Stripe API quirks and workarounds discovered
```

Claude reads the MEMORY.md index first, then loads specific memory files when their descriptions match the current context. The index keeps the cost of memory retrieval low — Claude doesn't load all memory files unconditionally.

**Index size limit**: MEMORY.md is truncated beyond approximately 200 lines. If your memory index grows beyond this, Claude will start missing memory entries. Periodically prune stale entries.

### Memory file format

Individual memory files use YAML frontmatter followed by markdown content:

```markdown
---
name: typescript-patterns
description: Validated TypeScript patterns and preferences for this user
type: feedback
created: 2026-03-15
updated: 2026-04-01
---

# TypeScript Patterns

## Validated approaches

- Use `satisfies` operator to validate object literal types without widening: `const config = { ... } satisfies Config`
- Prefer `unknown` over `any` — narrow with type guards rather than casting
- Use `Awaited<ReturnType<typeof fn>>` to extract async return types
- Zod `.parse()` at API boundaries, `.safeParse()` when you need to handle errors gracefully

## Anti-patterns to avoid (user has corrected these)

- Do not use `as Type` casts except as a last resort with an explanatory comment
- Do not use `@ts-ignore` — use `@ts-expect-error` with a comment explaining the expected error
- Do not repeat type definitions — derive from Zod schemas or Prisma generated types
```

### The 4 memory types

| Type | Description | What goes here |
|------|-------------|----------------|
| `user` | Who the user is and how they work | Communication style, expertise level, preferred response format, personal conventions |
| `feedback` | Corrections and validated approaches | Times Claude got something wrong and was corrected, approaches that worked well and were validated |
| `project` | Ongoing work context | Architecture state, current implementation progress, team decisions, key file locations |
| `reference` | Pointers to external systems | API endpoint URLs, service names, environment names, team contact info, external tool links |

### Triggering explicit saves

To save a memory explicitly in conversation:
- "Remember that I always prefer named exports over default exports."
- "Save the fact that we use `pnpm` for this project, never `npm`."
- "Remember this architectural decision: we chose Redis for session storage instead of database sessions because of scale requirements."

Claude will create or update the appropriate memory file and confirm the save.

### Memory freshness and verification

Memories can become stale. The architecture decision saved 6 months ago may no longer be accurate. The team contact saved 3 months ago may have left the company. Claude should verify memory-based recommendations when the stakes are high:

- For code suggestions based on memory: "I have a memory that this project uses X — is that still accurate?"
- For team/process information: "My notes say you preferred Y approach — still the case?"
- For external service configuration: "I recorded that the staging URL was Z — checking that's still valid before proceeding."

### Memory vs plan files vs todo lists

These three persistence mechanisms serve different purposes:

| Mechanism | Scope | Lifetime | Content |
|-----------|-------|----------|---------|
| **Memory files** | Cross-session | Persistent (months/years) | Long-lived facts, preferences, project state |
| **Plan files** | Per-feature | Persistent (committed to git) | Design decisions, implementation plan, approval record |
| **Todo lists** | Per-session | Session-only (not persisted) | In-progress task tracking, next steps |

Memory files are for facts you expect to be relevant in future sessions. Plan files are for the design artifact of a specific feature or decision — they persist to git, not to memory. Todo lists exist only for the duration of a conversation and are not saved.

A common mistake is using todo lists for information that should be in a plan file (lost on context reset) or using memory files for information that should be in a plan file (not committed to git, not shared with team). Match the persistence mechanism to the actual lifetime and sharing requirements of the information.
