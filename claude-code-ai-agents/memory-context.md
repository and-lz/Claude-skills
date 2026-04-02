# Memory System & Context Window Management

Comprehensive reference for Claude Code's file-based memory system, CLAUDE.md instruction hierarchy, context window budgeting, and cross-session continuity patterns.

---

## 1. The Four Memory Types

Claude Code persists knowledge across sessions using markdown files with YAML frontmatter, stored in `~/.claude/projects/<project-hash>/memory/`. Each file belongs to exactly one of four types.

### 1.1 `user` -- Who the User Is

Captures role, expertise level, preferences, and knowledge gaps. Saved when learning the user's background, tech stack familiarity, or working style.

```markdown
---
name: user-role
description: Senior fullstack engineer, strong iOS background, learning React
type: user
---
User is a senior engineer with 8 years of iOS/Swift experience, now ramping up on React 19 and Next.js 15. Comfortable with concurrency (GCD, async/await), less familiar with RSC data-fetching patterns.

**Why:** Calibrate explanations -- skip iOS basics, explain React idioms in terms of Swift equivalents when helpful.

**How to apply:** Default to Swift-style naming conventions in discussions. When introducing React patterns, briefly map to the iOS equivalent (e.g., "useEffect cleanup is like deinit for side effects").
```

### 1.2 `feedback` -- Corrections and Validated Approaches

Records what to do AND what NOT to do, confirmed by user action (correction or explicit approval). Critical rule: save from both failure and success to prevent negativity drift.

```markdown
---
name: feedback-testing-strategy
description: Always use real database in integration tests, never mock DB layer
type: feedback
---
Do not mock the database layer in integration tests for this project. Use the real Postgres instance via Docker Compose test profile.

**Why:** A mocked DB masked a production migration failure for 3 weeks. The mock returned the old column name while prod had been migrated. User explicitly corrected this on 2026-03-15.

**How to apply:** Integration test files (`*.integration.test.ts`) must import from `test/setup-db.ts`, never from `test/mocks/`. Unit tests may still mock external HTTP services.
```

### 1.3 `project` -- Ongoing Work Context

Captures who is doing what, why, and by when. Always convert relative dates to absolute dates at save time ("next Thursday" becomes "2026-04-03").

```markdown
---
name: project-merge-freeze
description: Code freeze window for mobile release cut starting 2026-04-05
type: project
---
Merge freeze begins 2026-04-05 (Saturday) through 2026-04-07 (Monday) for the v3.2 mobile release cut. Only P0 hotfixes may merge during this window. Release manager: @sarah.

**Why:** Last release had a broken build from a last-minute merge. Team agreed to enforce freeze windows.

**How to apply:** If user asks to merge a PR between 2026-04-05 and 2026-04-07, warn about the freeze. Only proceed if user confirms it is a P0 hotfix.
```

### 1.4 `reference` -- Pointers to External Systems

Records where to find information that lives outside the codebase: dashboards, trackers, wiki pages, Slack channels, runbooks.

```markdown
---
name: reference-linear-projects
description: Linear project mappings for team bug tracking and feature work
type: reference
---
- Pipeline/ingestion bugs: Linear project **INGEST** (linear.app/team/ingest)
- API feature work: Linear project **API** (linear.app/team/api)
- Mobile bugs: Linear project **MOBILE** (linear.app/team/mobile)
- On-call escalation channel: Slack #oncall-eng

**How to apply:** When creating or referencing issues, use the correct Linear project. Link PRs to Linear issues using the `LIN-XXXX` prefix in commit messages.
```

---

## 2. MEMORY.md Index Format

Every project's memory directory contains a `MEMORY.md` file that acts as the index. This file is **always loaded into the conversation context**, so it must stay compact.

### Format Rules

| Rule | Detail |
|------|--------|
| Location | `~/.claude/projects/<project-hash>/memory/MEMORY.md` |
| Entry format | `- [Title](file.md) -- one-line hook` |
| Max line length | Under 150 characters per entry |
| Max total lines | 200 lines before automatic truncation |
| Frontmatter | None (MEMORY.md has no YAML frontmatter) |
| Organization | Semantic grouping by topic, not chronological |
| Loading behavior | Always loaded into context at conversation start |

### Example MEMORY.md

```markdown
## User
- [User role](user_role.md) -- senior fullstack engineer, iOS + React expertise
- [Response style](user_style.md) -- prefers terse responses, no preamble

## Feedback
- [Testing strategy](feedback_testing.md) -- always use real DB in integration tests, never mock
- [PR workflow](feedback_pr_workflow.md) -- bundle refactors into single PRs, don't split
- [Import style](feedback_imports.md) -- use path aliases (@/), never relative imports beyond ../

## Project
- [Merge freeze](project_merge_freeze.md) -- freeze 2026-04-05 to 2026-04-07 for mobile release
- [Auth migration](project_auth_migration.md) -- migrating from session cookies to JWT, in progress
- [API v3](project_api_v3.md) -- new versioned API launching 2026-04-15

## Reference
- [Bug tracker](reference_linear.md) -- pipeline bugs in Linear project INGEST
- [Monitoring](reference_grafana.md) -- API latency and error dashboards on Grafana
- [Runbooks](reference_runbooks.md) -- incident response runbooks in Notion wiki
```

### Why Semantic Grouping Matters

MEMORY.md is loaded every conversation. If entries are chronological, the model must scan all 200 lines to find relevant context. Semantic grouping lets the model skip irrelevant sections. Group by: User, Feedback, Project, Reference -- or by domain area for large projects.

---

## 3. Auto-Save Triggers

Claude Code automatically creates memory files when it detects certain conversational signals. No explicit command is needed.

### Trigger Categories

**Explicit save requests:**
- "Remember that we use pnpm, not npm"
- "Save that the staging URL changed to staging-v2.example.com"
- "Note that @daniel owns the payments module"

**Behavioral corrections (feedback type):**
- "Don't do that -- always use the factory function instead"
- "Stop adding comments to obvious code"
- "No, we never deploy on Fridays"

**Confirmed non-obvious approaches (feedback type -- positive signal):**
- User: "Yes, exactly -- that's the pattern we want"
- User: "Perfect, keep doing it that way"
- User: "That's right, the env var overrides the config file"

**Background sharing (user type):**
- "I'm a data scientist, not a web developer"
- "We use Linear for tracking and Notion for docs"
- "The team is 4 engineers in UTC+1"

### The Save Process

1. Write the memory file with appropriate YAML frontmatter (name, description, type)
2. Add a pointer line to MEMORY.md under the correct semantic section
3. If MEMORY.md would exceed 200 lines, consider consolidating related entries

### What Does NOT Trigger Auto-Save

- Asking Claude to read a file (observation, not a fact to remember)
- Routine code generation (the code itself is the artifact)
- Debugging a transient issue (ephemeral, belongs in commit message)
- Restating something already in CLAUDE.md (redundant)

---

## 4. CLAUDE.md Hierarchy -- The Instruction Cascade

CLAUDE.md files form a layered instruction system. All matching files are concatenated into the system prompt. Later files in the cascade can override earlier ones.

### Load Order

```
~/.claude/CLAUDE.md                          # Layer 1: User-global
    |                                        #   Personal preferences, all projects
    v                                        #   NOT committed to any repo
<project>/CLAUDE.md                          # Layer 2: Project root
    |                                        #   Team-shared conventions
    v                                        #   Committed to git
<project>/.claude/CLAUDE.md                  # Layer 3: Alternative project location
    |                                        #   Same scope as Layer 2
    v                                        #   Committed to git
<project>/src/CLAUDE.md                      # Layer 4: Subdirectory-scoped
    |                                        #   Only active when working in src/
    v                                        #   Committed to git
<project>/src/components/CLAUDE.md           # Layer 5+: Deeper subdirectories
                                             #   Only active in src/components/
                                             #   Committed to git
```

### Override Semantics

- All loaded CLAUDE.md files are concatenated into a single system prompt block
- Later layers appear after earlier layers in the prompt
- When instructions conflict, the later (more specific) layer takes precedence
- Subdirectory CLAUDE.md files only load when Claude is actively working in that directory tree

### Practical Layer Strategy

| Layer | What belongs here | Example |
|-------|-------------------|---------|
| User-global (`~/.claude/CLAUDE.md`) | Personal response style, editor preferences, universal rules | "Be terse. No emojis. Always use plan mode for multi-file changes." |
| Project root (`<project>/CLAUDE.md`) | Team coding standards, tech stack, repo structure, CI rules | "Use pnpm. Run `pnpm check` before committing. API routes in src/app/api/." |
| Subdirectory (`src/api/CLAUDE.md`) | Module-specific conventions, local patterns | "All route handlers must validate with zod. Use the withAuth wrapper." |

### Critical Rules

- **Never put secrets in CLAUDE.md** -- these files are always in context and often committed to git
- **Project-level CLAUDE.md should be committed** -- it is how the team shares Claude conventions
- **User-level CLAUDE.md should NOT be committed** -- it lives in `~/.claude/`, outside any repo
- **Keep it concise** -- every line consumes context budget (see Section 6)

---

## 5. Context Window Budget

The context window is a fixed-size resource. Every byte loaded -- system prompt, CLAUDE.md content, tool schemas, conversation history, tool results, model reasoning -- competes for the same space.

### Budget Breakdown (Approximate)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  System prompt (Claude Code internals)                  ~15%    │
│    - Core behavioral instructions                               │
│    - Safety guidelines                                          │
│    - Tool usage protocols                                       │
│                                                                 │
│  CLAUDE.md files (all loaded levels)                    ~5-10%  │
│    - User-global instructions                                   │
│    - Project-level instructions                                 │
│    - Active subdirectory instructions                           │
│                                                                 │
│  Active skill content                                   ~5-15%  │
│    - Currently triggered skill(s)                               │
│    - Skill-specific instructions and examples                   │
│                                                                 │
│  Memory files (MEMORY.md always; others on relevance)   ~2-5%   │
│    - MEMORY.md index (always loaded)                            │
│    - Individual memory files (loaded when relevant)             │
│                                                                 │
│  Tool schemas (MCP servers + built-in tools)            ~5-10%  │
│    - Built-in tool definitions (Read, Edit, Bash, etc.)         │
│    - MCP server tool schemas (if connected)                     │
│    - Deferred tool stubs                                        │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  AVAILABLE for conversation + reasoning                 ~55-68% │
│    - User messages and conversation history                     │
│    - Tool call results (file contents, grep output, etc.)       │
│    - Model's internal reasoning and responses                   │
│    - Intermediate computation and planning                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### What Consumes the Most Context

In practice, the biggest consumers of the "available" budget are:

1. **Tool results** -- Reading large files, verbose grep output, or long command output can consume thousands of tokens per call. A single `Read` of a 500-line file can use 2-4% of the context.
2. **Conversation history** -- Long back-and-forth accumulates. Each message pair (user + assistant) stays in context for the entire session.
3. **Repeated file reads** -- Reading the same file multiple times in a session wastes budget. Read once, reference by line number.

### Budget Conservation Tactics

- **Read specific line ranges** instead of entire files: `Read(file, offset=100, limit=50)`
- **Use `files_with_matches` mode** in Grep before reading full content
- **Use `head_limit`** on Grep to avoid dumping 1000 matches into context
- **Avoid re-reading files** already in context -- reference by the line numbers shown earlier
- **Keep CLAUDE.md files lean** -- every word is loaded every turn
- **Use skills (on-demand)** instead of putting everything in CLAUDE.md
- **Use hooks (zero context cost)** for rules that must always be enforced

---

## 6. The 150-200 Instruction Compliance Cliff

Empirical observation: Claude reliably follows approximately 150-200 discrete instructions. Beyond this threshold, compliance degrades -- the model starts silently ignoring some rules.

### The Arithmetic

| Source | Approx. instruction slots consumed |
|--------|-----------------------------------|
| Claude Code system prompt | ~50 |
| User-global CLAUDE.md | Variable (yours to control) |
| Project CLAUDE.md | Variable (yours to control) |
| Active skill(s) | Variable (loaded on demand) |
| Memory files | Variable (loaded on relevance) |
| **Total budget** | **~150-200** |
| **Your budget** (after system prompt) | **~100-150** |

### Symptoms of Exceeding the Cliff

- Claude ignores specific CLAUDE.md rules while following others
- Inconsistent behavior: follows a rule in one turn, ignores it in the next
- Rules near the end of long CLAUDE.md files are ignored more often
- Complex conditional rules ("if X and Y but not Z, then...") fail first

### Strategies to Stay Under Budget

**1. The deletion test:** For every line in CLAUDE.md, ask: "If I remove this, will Claude make a concrete mistake?" If the answer is no, remove it.

**2. Move domain knowledge to skills:** Skills load on demand, only when triggered. A 2000-word skill about React patterns costs zero context when you are working on backend code.

**3. Move critical rules to hooks:** Hooks execute outside the context window with 100% compliance. A PreToolUse hook that blocks `rm -rf /` costs zero instruction slots and never fails.

**4. Use file:line references, not code pasting:**
```markdown
# BAD (wastes context budget):
When writing API routes, always use this pattern:
export async function GET(req: Request) {
  const session = await getSession(req);
  if (!session) return Response.json({ error: "Unauthorized" }, { status: 401 });
  // ... handler logic
}

# GOOD (nearly free):
API route pattern: see src/lib/api-helpers.ts:15-30. All new routes must follow this structure.
```

**5. Merge related rules into single statements:**
```markdown
# BAD (3 instruction slots):
- Always use pnpm, never npm or yarn.
- Always run pnpm check before committing.
- Always run pnpm test before pushing.

# GOOD (1 instruction slot):
Use pnpm exclusively. Run `pnpm check && pnpm test` before any commit/push.
```

**6. Use hierarchical CLAUDE.md to scope rules:** A rule about React component patterns in `src/components/CLAUDE.md` only loads when working in that directory, freeing budget for other directories.

---

## 7. Stale Memory Handling

Memories are snapshots. Codebases change. A memory saved last week may reference files that have been renamed, functions that were refactored, or decisions that were reversed.

### Verification Protocol

Before acting on a memory, validate it against the current state:

| Memory claims... | Verification action |
|------------------|-------------------|
| A file path exists | `Glob` or `Read` the path |
| A function/class/flag exists | `Grep` for the identifier |
| A repo structure pattern | `ls` the relevant directory |
| A git branch or tag | `git branch -a` or `git tag` |
| Repo-level state (dependencies, config) | Read the actual config file |
| A team decision | Ask the user if still current (if high-stakes) |

### The Trust Hierarchy

```
Current observation (tool output)  >  Memory file  >  Assumption
```

If a memory says "the auth module is in `src/auth/`" but `Glob("**/auth/**")` returns nothing, trust the glob. The directory was likely moved.

### When to Update vs. Delete

- **Update** if the core fact is still true but details changed (file moved, function renamed)
- **Delete** if the fact is no longer true at all (project cancelled, decision reversed)
- **Keep** if you cannot verify either way -- but add a note: "Unverified as of 2026-04-01"

### Staleness Indicators

- Memory references a date in the past ("freeze starts 2026-03-01") -- may be expired
- Memory references a feature branch -- branch may be merged or abandoned
- Memory references a specific dependency version -- version may have changed
- Memory was saved more than 30 days ago without re-confirmation

---

## 8. Memory vs. Plan Files vs. Task Lists

Claude Code has multiple persistence mechanisms. Choosing the wrong one leads to either lost context (too ephemeral) or cluttered memory (too permanent).

### Comparison Matrix

| Mechanism | Scope | Survives session? | Survives reboot? | Best for |
|-----------|-------|-------------------|-------------------|----------|
| **Memory files** | Cross-session, cross-conversation | Yes | Yes | User preferences, team conventions, feedback corrections, external system pointers |
| **Plan files** | Current implementation | Yes (saved to docs/) | Yes | Architecture decisions, implementation designs, system diagrams |
| **Task lists** (TodoWrite) | Current session | No | No | Step-by-step execution tracking, progress checkpoints within a single task |
| **Context window** | Current turn | No | No | Active reasoning, tool results, intermediate computation |

### Decision Flowchart

```
Is this fact useful in FUTURE conversations?
  YES → Will it change within days?
    YES → Don't save (ephemeral)
    NO  → Save as memory (appropriate type)
  NO  → Is it needed across multiple turns in THIS session?
    YES → Is it a step-by-step plan?
      YES → TodoWrite (task list)
      NO  → Keep in conversation (context window)
    NO  → Don't persist at all
```

### Anti-Patterns

| Anti-pattern | Why it's wrong | What to do instead |
|-------------|----------------|-------------------|
| Saving current branch name to memory | Changes every task | Let git tell you (`git branch --show-current`) |
| Saving WIP file list to memory | Changes every commit | Use `git status` |
| Saving debugging steps to memory | Fix belongs in code, context in commit | Write a clear commit message |
| Saving PR activity summaries | Stale within hours | Use `gh pr list` for current state |
| Saving code patterns to memory | Derivable from reading the codebase | Reference the file and line number |
| Saving git history to memory | `git log` is always current | Use `git log --oneline -20` |

---

## 9. What NOT to Save in Memory

Memory is a high-value, limited-capacity system. Every memory file adds to the MEMORY.md index, which is loaded every conversation. Saving low-value information degrades the signal-to-noise ratio.

### The Hard "No" List

**Code patterns derivable from reading code:**
The codebase IS the source of truth for its own patterns. If you can discover it with Grep or Read, do not save it.

**Git history or state:**
Git is a perfect record of its own history. `git log`, `git blame`, `git diff` are always available and always current. Memory would be a stale copy.

**Debugging session details:**
The fix is in the code. The context is in the commit message. The debugging journey has no future value.

**Anything already in CLAUDE.md:**
CLAUDE.md is already loaded every conversation. Duplicating its content into memory wastes two slots instead of one.

**Ephemeral task details:**
Current branch, files being edited, WIP status, uncommitted changes -- all of this changes constantly and is queryable from git.

**PR lists or activity summaries:**
"User merged 3 PRs today" has no future value. If something surprising happened in those PRs, save THAT specific insight.

**File paths or architecture maps:**
Directory structure can be discovered with `ls` and `Glob`. Saving it creates a stale snapshot that will diverge from reality.

### The "Ask Yourself" Test

Before saving anything to memory, pass it through these filters:

1. **Can I discover this from the codebase or tools?** If yes, don't save.
2. **Will this be true in 30 days?** If probably not, don't save.
3. **Would a new conversation without this memory make a mistake?** If no, don't save.
4. **Is this about a specific person's preference or decision?** If yes, probably save.
5. **Is this about an external system I can't query?** If yes, probably save.

---

## 10. Cross-Session Continuity Patterns

For projects with long-lived context, these patterns maximize the value of the memory system across sessions.

### Pattern 1: Decision Log Memory

Save the **WHY** behind non-obvious architectural or design decisions. The decision itself may be visible in code, but the reasoning is invisible.

```markdown
---
name: auth-architecture-decision
description: Why we chose JWT over session cookies for the API
type: project
---
Chose JWT over session cookies for the REST API.

**Why:** Mobile clients (iOS + Android) need stateless auth; cookies complicate cross-origin requests from the React app. Session storage would require Redis, adding operational complexity the team cannot support with current headcount.

**How to apply:** All new API endpoints must use Bearer token auth, not cookie-based sessions. The token format is defined in `src/lib/auth/token.ts`. Refresh token rotation is handled by the `/api/auth/refresh` endpoint.
```

### Pattern 2: User Workflow Preferences

Save **HOW** the user likes to work -- their process preferences, not just technical preferences.

```markdown
---
name: pr-workflow-preference
description: User prefers single bundled PRs for refactors, not many small ones
type: feedback
---
For refactors in this codebase, user prefers one bundled PR over many small ones.

**Why:** Splitting refactors creates review overhead without adding safety -- the team reviews the full diff anyway. Confirmed when I suggested splitting the type migration into 5 PRs and user said "no, keep it in one PR."

**How to apply:** Default to single PRs for refactors unless the user explicitly requests splitting. For feature work (not refactors), ask about PR strategy.
```

### Pattern 3: External System Pointers

Save **WHERE** to find things that live outside the codebase and cannot be discovered by reading code.

```markdown
---
name: monitoring-dashboards
description: Grafana dashboards for API latency, error rates, and deploy status
type: reference
---
- API latency: grafana.internal/d/api-latency (oncall watches this)
- Error rates: grafana.internal/d/api-errors (alert threshold: >1% 5xx)
- Deployment status: grafana.internal/d/deploys (shows canary vs. stable)
- Feature flags: LaunchDarkly project "backend-api" (ask @platform for access)

**How to apply:** Check latency and error dashboards when editing request-path code. Check deploy dashboard after merging to main. Feature flag changes require LaunchDarkly, not code changes.
```

### Pattern 4: Failure Post-Mortems

Save lessons from incidents or failures that should change future behavior.

```markdown
---
name: feedback-migration-testing
description: Always run migrations against a prod-size dataset before merging
type: feedback
---
Never merge a database migration without testing it against a production-sized dataset.

**Why:** The `add_index_users_email` migration passed CI (1000 rows) but locked the users table for 47 minutes in production (12M rows). Incident on 2026-03-20.

**How to apply:** Before approving any migration PR, ask: "Has this been tested against a prod-size dataset?" If the answer is no or unknown, flag it as a blocker. For index additions on large tables, use `CREATE INDEX CONCURRENTLY`.
```

### Pattern 5: Team Context

Save information about team members, roles, and ownership that cannot be discovered from code.

```markdown
---
name: team-ownership
description: Module ownership and escalation contacts
type: reference
---
- Payments module: @daniel (on parental leave until 2026-05-01, escalate to @maria)
- Auth/identity: @sarah
- Data pipeline: @alex (also on-call rotation lead)
- iOS app: @user (the person I'm working with)
- Design system: @jessica (reviews all component PRs)

**How to apply:** Tag the correct owner on PRs touching their module. During Daniel's leave, payments PRs go to Maria.
```

---

## 11. Advanced: Context Window Optimization Techniques

### Technique 1: Progressive Disclosure Reads

Instead of reading an entire large file, use a two-step approach:

1. **Grep** for the specific function/class/pattern you need
2. **Read** only the relevant line range (offset + limit)

This can reduce context consumption by 80-95% compared to reading the full file.

### Technique 2: Strategic File Ordering

When you need to read multiple files, read the most uncertain file first. If it answers your question, you avoid reading the others entirely.

### Technique 3: CLAUDE.md as a Router

Instead of putting all instructions in CLAUDE.md, use it as a routing table:

```markdown
# Project CLAUDE.md (lean router)

- API patterns: see src/app/api/CLAUDE.md
- Component conventions: see src/components/CLAUDE.md
- Testing strategy: see tests/CLAUDE.md
- Deploy process: see .github/CLAUDE.md
```

Each subdirectory CLAUDE.md only loads when working in that directory, keeping the base context minimal.

### Technique 4: Memory Consolidation

When MEMORY.md grows beyond ~50 entries, consolidate:

- Merge related feedback memories into a single file with multiple rules
- Archive project memories for completed projects (move to a `memory/archive/` directory)
- Verify and prune stale entries (see Section 7)

### Technique 5: Skill vs. CLAUDE.md Partitioning

| Put in CLAUDE.md | Put in a skill |
|-------------------|----------------|
| Rules that apply to ALL tasks | Rules that apply to a specific domain |
| Under 3 lines | Over 3 lines of domain knowledge |
| Must never be forgotten | Can be loaded on demand |
| Process rules (commit style, PR format) | Technical patterns (React, iOS, API design) |

---

## 12. Debugging Context Issues

### Symptom: Claude Ignores a CLAUDE.md Rule

**Possible causes:**
1. Instruction count exceeded the compliance cliff (~150-200)
2. Conflicting instruction elsewhere in the cascade
3. Rule is ambiguously worded (Claude interprets it differently)
4. Rule is near the end of a very long CLAUDE.md

**Fixes:**
1. Audit total instruction count across all loaded CLAUDE.md files
2. Search for contradictions with `Grep` across all CLAUDE.md files
3. Rewrite the rule to be unambiguous and action-oriented
4. Move critical rules to the top of the file
5. Move the rule to a hook for 100% enforcement

### Symptom: Claude Uses Outdated Information

**Possible causes:**
1. Stale memory file with old file paths or function names
2. CLAUDE.md references a pattern that was refactored
3. Memory conflicts with current codebase state

**Fixes:**
1. Verify memory against current state (Section 7)
2. Update or delete the stale memory
3. Add a verification step to the memory's "How to apply"

### Symptom: Context Window Runs Out Mid-Task

**Possible causes:**
1. Reading too many full files into context
2. Long conversation with many tool calls
3. Too many MCP tool schemas loaded
4. Verbose Grep output without head_limit

**Fixes:**
1. Read specific line ranges, not full files
2. Start a new conversation (context resets, memory persists)
3. Disconnect unused MCP servers
4. Always use head_limit on Grep (default is 250, lower it for broad searches)

---

## Summary: The Memory System Mental Model

```
                    ┌─────────────────────┐
                    │   Context Window     │
                    │   (finite, shared)   │
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
    ┌─────────▼──────┐ ┌─────▼─────┐ ┌───────▼──────┐
    │  System Prompt  │ │ CLAUDE.md │ │  Conversation │
    │  (fixed cost)   │ │ (cascade) │ │   (growing)   │
    └────────────────┘ └───────────┘ └──────────────┘
                              │
                    ┌─────────▼───────────┐
                    │     Memory System    │
                    │                     │
                    │  MEMORY.md (always) │
                    │  + files (on demand)│
                    │                     │
                    │  user | feedback |  │
                    │  project | reference│
                    └─────────────────────┘
                              │
                    ┌─────────▼───────────┐
                    │   Skills (on demand) │
                    │   Hooks (zero cost)  │
                    │   MCP (tool schemas) │
                    └─────────────────────┘
```

**Core principles:**
1. Memory files survive across sessions; context window does not
2. MEMORY.md is always loaded; individual memory files load on relevance
3. CLAUDE.md instructions cascade from global to specific; later overrides earlier
4. Every byte in context competes with reasoning capacity
5. Hooks enforce rules at zero context cost with 100% compliance
6. Skills load domain knowledge on demand, not permanently
7. Verify memories before acting on them -- trust current observation over stored state
8. Save the WHY, not the WHAT -- the what is in the code
