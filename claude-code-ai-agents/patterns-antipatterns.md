# Patterns & Anti-Patterns -- Quick Reference

Each entry: DO or DON'T, **bold name**, 1-2 sentence explanation, brief code example where useful.

---

## CLAUDE.md Patterns (8)

DO **Keep under 100 lines** -- Leave room for user-level + skills. Every extra line competes for the 150-200 instruction compliance budget.

DON'T **Paste code snippets** -- Code goes stale. Use file:line refs: "Follow pattern in src/utils/api.ts:15-30."

DO **Use file:line references** -- "See src/auth/jwt.ts:42 for the token pattern." Claude reads the file directly.

DON'T **Duplicate hook rules** -- If a hook blocks `rm -rf`, don't also write it in CLAUDE.md. Wastes instruction budget.

DO **Review in sprint retros** -- Ask: "Would removing this cause mistakes?" If no, delete.

DON'T **Put secrets anywhere** -- Not CLAUDE.md, not comments, not commits. Always env vars.

DO **Commit project CLAUDE.md to git** -- Team knowledge compounds. PR review for changes.

DON'T **Write essays** -- Instructions not documentation. "Use ESM imports, not CommonJS" -- done.

---

## Skills Patterns (6)

DO **Load reference files lazily** -- SKILL.md tells Claude WHEN to read each reference. Don't load everything upfront.

DON'T **Duplicate CLAUDE.md content** -- Skills supplement, not replace.

DO **Use AskUserQuestion before acting** -- RULE #1. Clarify before assuming architecture, data sources, design.

DON'T **Create skills for one-off tasks** -- Skills = recurring patterns. One-offs are conversations.

DO **Keep triggers specific** -- "SwiftUI" good. "code" too broad. Generic triggers load the skill when irrelevant.

DO **Cross-reference other skills** -- "For iOS, read ios-26-app/engineering.md." Skills are composable.

---

## Hooks Patterns (8)

DO **Use exit 2 for safety-critical blocks** -- Rules that must NEVER be violated go in PreToolUse hooks.

```bash
echo "BLOCKED: destructive command" >&2; exit 2
```

DON'T **Make HTTP requests in hooks** -- Synchronous. Network calls add latency to EVERY tool call.

DO **Keep hooks under 200ms** -- Check file extension first, bail early on non-matching tools.

```bash
echo "$FILE" | grep -qE '\.(ts|tsx)$' || exit 0
```

DON'T **Run full test suites in PostToolUse** -- Targeted checks only (single file lint, type check).

DO **Log all tool calls via PostToolUse** -- JSONL audit trail. Cheap, invaluable for debugging.

DON'T **Forget stdin** -- Hook receives CLAUDE_TOOL_INPUT via env AND stdin. Parse with `jq`.

DO **Test hooks with simulated inputs** -- Export env vars, pipe JSON, check exit codes before real use.

DO **Use matcher patterns to scope** -- `"matcher": "mcp__stripe__*"` not `"*"` when possible.

---

## Memory Patterns (6)

DO **Save corrections immediately** -- User says "don't do X" -> save feedback memory now.

DON'T **Save ephemeral state** -- Current branch, WIP files, today's tasks don't belong in memory.

DO **Convert relative dates to absolute** -- "Thursday" -> "2026-04-03". Must be interpretable weeks later.

DON'T **Save what the code shows** -- Architecture, patterns, file structure -- Claude discovers by reading.

DO **Verify before recommending from memory** -- "Memory says auth.ts exists" -- Glob first. May be renamed.

DO **Save WHY not just WHAT** -- "Use JWT because mobile clients need stateless auth" > "Use JWT."

---

## Agent SDK Patterns (10)

DO **Orchestrator + workers for complex tasks** -- Orchestrator (opus) plans, workers (sonnet) execute. 90% perf improvement.

```typescript
const plan = await runAgent(planPrompt, dir, { model: "claude-opus-4-6" });
await Promise.all(tasks.map(t => runAgent(t, dir, { model: "claude-sonnet-4-6" })));
```

DON'T **Share context between parallel workers** -- Each query() gets its own context. Pass data via files.

DO **Set maxTurns on every agent** -- Without limits agents loop indefinitely.

```typescript
options: { maxTurns: 30 }
```

DON'T **Use bypassPermissions outside sandboxes** -- Disables ALL safety. Docker/VM only.

DO **File-based handoffs between agents** -- Agent A writes result.json, Agent B reads it. Typed contracts.

```typescript
writeFileSync("analysis.json", JSON.stringify(result));
const input = Schema.parse(JSON.parse(readFileSync("analysis.json", "utf-8")));
```

DON'T **Inline large context in prompts** -- Write to file, tell agent to read it. Keeps prompts clean.

DO **Validate handoffs with Zod/Pydantic** -- Every agent boundary is a contract boundary.

DON'T **Give unscoped Bash** -- `"Bash"` = unrestricted shell. `"Bash(npm test*)"` = scoped.

DO **AbortController for timeouts** -- Kill agents exceeding time budget.

```typescript
const ctrl = new AbortController();
setTimeout(() => ctrl.abort(), 5 * 60_000);
```

DO **Route by model cost** -- Haiku scripted, sonnet implementation, opus planning/review only.

---

## MCP Consuming Patterns (6)

DO **Always ${ENV_VAR} for credentials** -- Never hardcode tokens in mcp.json.

```json
{ "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" } }
```

DON'T **Use Stripe live keys with agents** -- Always `sk_test_*`. Add a hook to enforce.

DO **Pin MCP server versions** -- `@modelcontextprotocol/server-github@1.2.3` not `@latest`.

DON'T **Load 5+ MCP servers at once** -- Tool schemas consume context. Load only what you need.

DO **Test tools interactively first** -- `/mcp` to verify behavior before automating.

DO **Inline mcpServers in SDK for scoping** -- Override mcp.json when agent needs specific servers only.

---

## MCP Building Patterns (8)

DO **One tool, one thing** -- `get_user` + `update_user` not `manage_users`. Narrow = precise.

DON'T **Silently truncate** -- Return `{ results, hasMore: true, cursor: "abc" }` with pagination.

DO **Descriptions for LLMs not humans** -- Specific, actionable: "Search files by glob pattern. Returns paths, sizes, mod times."

DON'T **Pass LLM args to shell** -- `exec(args.cmd)` = injection. Use `execFile` with arg arrays.

```typescript
// DON'T: exec(userInput)
// DO:    execFile("grep", ["-r", userInput, "."])
```

DO **Validate all inputs with Zod/Pydantic** -- LLMs hallucinate arguments. Validate before executing.

DON'T **Return secrets in tool results** -- Filter TOKEN/KEY/SECRET/PASSWORD fields.

DO **Return actionable errors** -- "File not found: /src/auth.ts. Did you mean /src/auth/index.ts?" Not "Error: failed."

DO **Audit log every tool call** -- JSONL: timestamp, tool, sanitized input, success/failure.

---

## Agentic Workflow Patterns (8)

DO **Specs over ad hoc prompts** -- TaskSpec -> reproducible output -> measurable quality.

```json
{ "title": "Add JWT refresh", "acceptance_criteria": ["POST /auth/refresh returns new pair"] }
```

DON'T **Skip HitL for irreversible actions** -- Deleting data, sending emails, merging PRs -- always checkpoint.

DO **Checkpoints for long-running agents** -- Write progress to file after each step. Resume on failure.

```typescript
writeFileSync("checkpoint.json", JSON.stringify({ completed: ["step1"], pending: ["step2"] }));
```

DON'T **Restart from scratch after partial failure** -- Read checkpoint, continue.

DO **Critic/Reviewer pattern for quality** -- Worker produces, critic reviews, worker revises. Max 3 iterations.

DON'T **Use opus for simple tasks** -- Haiku for scripted, sonnet for implementation, opus for reasoning only.

DO **Define failure modes before building** -- Tool error? Model refusal? Timeout? Design retry + escalation upfront.

DO **Track cost per pipeline run** -- tokens x price. Alert on overruns. Without tracking, costs drift.

---

## Security Patterns (8)

DO **All inputs untrusted** -- Files, MCP responses, web scrapes -- all can contain injection payloads.

DON'T **Give more tools than needed** -- Read-only task: `["Read", "Glob", "Grep"]`. Never `["Bash"]` when Read suffices.

DO **Sandbox background agents** -- Docker: no network, read-only FS, memory limits.

DON'T **Log secrets** -- Audit logs must sanitize. Redact TOKEN/KEY/SECRET patterns.

DO **Rotate credentials regularly** -- Agent tokens at higher exposure risk. 90-day minimum.

DON'T **Execute model output directly** -- Never eval()/exec() on generated text. Validate first.

DO **PreToolUse hooks as security gates** -- Block dangerous commands, enforce file boundaries, validate MCP inputs.

DO **Incident response plan** -- Kill agent -> audit log -> git diff -> rotate creds if exposed.
