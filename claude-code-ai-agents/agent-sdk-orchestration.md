# Agent SDK & Multi-Agent Orchestration

A dense reference for senior engineers building production multi-agent systems with Claude Code in 2026. Covers the TypeScript and Python SDKs, orchestrator/worker patterns, context isolation, background agents, parallel execution, error handling, MCP integration, and model selection strategy.

---

## 1. Claude Code SDK Overview

### Installation

```bash
npm install @anthropic-ai/claude-code  # Node 18+ required
```

The package ships full TypeScript types. No separate `@types/` package needed.

### The `query()` Function

`query()` is an async generator. It streams `SDKMessage` objects as the agent works — each turn's assistant response, every tool call, every tool result, and lifecycle events all come through this stream. You consume them with `for await ... of`.

```typescript
import { query, type SDKMessage, type SDKOptions } from "@anthropic-ai/claude-code";
```

Full TypeScript signature:

```typescript
interface SDKOptions {
  maxTurns?: number;            // max agent turns (default: unlimited — always set this)
  systemPrompt?: string;        // completely replaces the default system prompt
  appendSystemPrompt?: string;  // appends to the existing system prompt (preferred for customization)
  permissionMode?: "default" | "acceptEdits" | "bypassPermissions";
  cwd?: string;                 // working directory for all file operations
  allowedTools?: string[];      // whitelist: only these tools are available
  disallowedTools?: string[];   // blacklist: all tools except these
  mcpServers?: MCPServerConfig[]; // inline MCP server config (no mcp.json needed)
  model?: string;               // override default (default: claude-opus-4-6)
  customApiKeyEnvVar?: string;  // name of env var containing the Anthropic API key
}

function query(params: {
  prompt: string;
  abortController?: AbortController;
  options?: SDKOptions;
}): AsyncGenerator<SDKMessage>;
```

`allowedTools` and `disallowedTools` are mutually exclusive — use one or the other. `allowedTools` is preferred in production because it enforces least-privilege.

### Full Working Example: Single-Agent Task Runner

```typescript
import { query, type SDKMessage } from "@anthropic-ai/claude-code";

async function runAgent(prompt: string, projectDir: string): Promise<string> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 5 * 60 * 1000); // 5-minute hard stop

  const output: string[] = [];

  try {
    for await (const msg of query({
      prompt,
      abortController: controller,
      options: {
        maxTurns: 20,
        cwd: projectDir,
        permissionMode: "default",
        allowedTools: ["Read", "Glob", "Grep", "Write", "Edit"],
        model: "claude-sonnet-4-6",
      },
    })) {
      if (msg.type === "assistant") {
        const text = msg.message.content
          .filter((b: any) => b.type === "text")
          .map((b: any) => b.text)
          .join("");
        if (text) output.push(text);
      }

      if (msg.type === "result") {
        if (msg.subtype === "error_max_turns") {
          console.warn("Agent hit max turns limit");
        } else if (msg.subtype === "error_during_turn") {
          console.error("Agent encountered an error:", msg);
        }
      }
    }
  } finally {
    clearTimeout(timeout);
  }

  return output.join("\n");
}
```

### SDKMessage Types

Every message yielded by `query()` has a `type` discriminant. The complete set:

**`{ type: "assistant", message: { role: "assistant", content: ContentBlock[] } }`**
The model's response. `content` is an array — iterate it, filter by `b.type === "text"` to get text, `b.type === "tool_use"` to see tool calls the model emitted.

**`{ type: "tool_use", name: string, input: object, id: string }`**
Emitted before a tool executes. `name` is the tool name (e.g., `"Read"`, `"Bash"`). `input` is the parsed JSON arguments. `id` correlates to the subsequent `tool_result`.

**`{ type: "tool_result", tool_use_id: string, content: string | ContentBlock[] }`**
The output from a tool execution. `tool_use_id` matches the `id` from the corresponding `tool_use` message. Inspect this to audit what tools returned.

**`{ type: "system", subtype: "init", ... }`**
Emitted once at agent startup. Contains session metadata: model, session ID, tools loaded, MCP servers attached.

**`{ type: "system", subtype: "result", ... }`**
Emitted at completion. Contains the final result summary, turn count, and cost metadata.

**`{ type: "result", subtype: "success" | "error_max_turns" | "error_during_turn", ... }`**
Terminal message. Always check `subtype` — `"error_max_turns"` means the task was not completed; increase `maxTurns` or decompose the task. `"error_during_turn"` indicates an unrecoverable tool failure or API error mid-stream.

### Message Consumption Patterns

**Collect final text only:**
```typescript
const texts: string[] = [];
for await (const msg of query({ prompt, options })) {
  if (msg.type === "assistant") {
    texts.push(
      ...msg.message.content
        .filter((b: any) => b.type === "text")
        .map((b: any) => b.text as string)
    );
  }
}
const result = texts.join("\n");
```

**Audit all tool calls:**
```typescript
const toolCalls: Array<{ name: string; input: object }> = [];
for await (const msg of query({ prompt, options })) {
  if (msg.type === "tool_use") {
    toolCalls.push({ name: msg.name, input: msg.input });
    console.log(`Tool: ${msg.name}`, JSON.stringify(msg.input).slice(0, 200));
  }
}
```

**Stream text in real time:**
```typescript
for await (const msg of query({ prompt, options })) {
  if (msg.type === "assistant") {
    msg.message.content
      .filter((b: any) => b.type === "text")
      .forEach((b: any) => process.stdout.write(b.text));
  }
}
```

---

## 2. Python SDK Equivalent

### Installation

```bash
pip install claude-agent-sdk      # pip
uv add claude-agent-sdk           # uv (recommended for fast installs)
```

Requires Python 3.10+. The package ships a full type stub.

### Basic Usage

```python
import asyncio
from claude_agent_sdk import query, ClaudeCodeOptions

async def run_agent(prompt: str, cwd: str) -> list:
    options = ClaudeCodeOptions(
        max_turns=20,
        cwd=cwd,
        permission_mode="default",
        allowed_tools=["Read", "Glob", "Grep", "Write", "Edit"],
        model="claude-sonnet-4-6",
    )

    messages = []
    async for message in query(prompt=prompt, options=options):
        messages.append(message)
        if hasattr(message, "content"):
            for block in message.content:
                if block.type == "text":
                    print(block.text, end="", flush=True)

    return messages

asyncio.run(run_agent("Add input validation to auth.py", "/path/to/project"))
```

### `ClaudeCodeProcess` Context Manager

For long-running sessions with multiple sequential tasks, use `ClaudeCodeProcess` to reuse the agent process. Each call to `agent.query()` is a new conversation but runs inside the same persistent process — eliminates startup overhead.

```python
from claude_agent_sdk import ClaudeCodeProcess, ClaudeCodeOptions

async def long_running_agent():
    options = ClaudeCodeOptions(
        max_turns=100,
        cwd="/project",
        permission_mode="acceptEdits",
        allowed_tools=["Read", "Write", "Edit", "Glob", "Grep"],
    )

    async with ClaudeCodeProcess(options) as agent:
        # First task — agent has no memory of previous conversations
        async for msg in agent.query("Analyze the codebase structure and write a summary to /project/.agent-work/analysis.md"):
            process_message(msg)

        # Second task — same process, fresh conversation context
        # The agent can read analysis.md but has no conversational memory
        async for msg in agent.query("Implement the feature described in /project/.agent-work/analysis.md"):
            process_message(msg)

        # Third task — build on results written to disk
        async for msg in agent.query("Write comprehensive tests for all functions modified in the previous step"):
            process_message(msg)


def process_message(msg):
    if hasattr(msg, "content"):
        for block in msg.content:
            if block.type == "text":
                print(block.text, end="", flush=True)
```

`ClaudeCodeProcess` is the right primitive when you have a pipeline of sequential agent tasks that share a project working directory but should not share conversational context. Each `agent.query()` call is isolated — context bleed between tasks is prevented by design.

### Python Error Handling

```python
from claude_agent_sdk import (
    query,
    ClaudeCodeOptions,
    MaxTurnsError,
    AgentError,
    APIError,
)
import asyncio

async def run_with_retry(prompt: str, options: ClaudeCodeOptions, max_attempts: int = 3) -> list:
    last_error = None
    for attempt in range(1, max_attempts + 1):
        try:
            messages = []
            async for msg in query(prompt=prompt, options=options):
                messages.append(msg)
            return messages
        except MaxTurnsError:
            # Double the turn budget on retry
            options = ClaudeCodeOptions(
                **{**options.__dict__, "max_turns": options.max_turns * 2}
            )
            last_error = MaxTurnsError(f"Max turns exceeded on attempt {attempt}")
        except APIError as e:
            if e.status_code == 529:  # overloaded
                delay = min(1000 * (2 ** attempt), 30_000) / 1000
                await asyncio.sleep(delay)
                last_error = e
            else:
                raise
        except AgentError as e:
            raise  # non-retryable

    raise last_error
```

---

## 3. Permission Modes

Permission modes determine which operations the agent can execute without prompting for approval. Wrong mode selection is a primary production security risk.

### `"default"` — Use This Unless You Have a Specific Reason

Behavior:
- Reads all files the process can access (no prompt needed)
- File writes and edits: respects `settings.json` allow/deny lists; prompts user in interactive mode
- Shell commands: prompts user unless explicitly allowed in settings
- In headless/non-TTY mode: blocks any action not on the explicit allow list

When to use: all interactive development, CI pipelines where you want guardrails, any environment with sensitive files.

`settings.json` configuration for default mode with explicit shell command allow list:

```json
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff*)",
      "Bash(git add*)",
      "Bash(git commit*)",
      "Bash(npm test*)",
      "Bash(npm run lint*)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(curl*)",
      "Bash(wget*)"
    ]
  }
}
```

### `"acceptEdits"` — CI/CD Pipelines with Reviewed Plans

Behavior:
- Auto-approves all file read operations
- Auto-approves all file write and edit operations (no prompt)
- Still blocks shell commands unless explicitly in the allow list
- No user prompt at any point

When to use: CI pipelines where a human has reviewed the task spec before the agent runs, automated code generation workflows, batch file processing where you accept the file change risk.

Risk: the agent can make arbitrary changes to any file the process can write. Scope the `cwd` tightly and run in a throw-away branch.

`settings.json` for a CI pipeline using `acceptEdits`:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm install --frozen-lockfile)",
      "Bash(npm test -- --ci)",
      "Bash(npm run build)",
      "Bash(git add -A)",
      "Bash(git commit -m*)"
    ],
    "deny": [
      "Bash(git push*)",
      "Bash(npm publish*)"
    ]
  }
}
```

Set `permissionMode: "acceptEdits"` in `SDKOptions` — do not set it globally in `settings.json` unless every project on this machine should have auto-approved edits.

### `"bypassPermissions"` — Isolated Sandbox Only

Behavior:
- Disables ALL permission checks
- No allow/deny lists consulted
- No approval prompts for anything
- Agent can execute any shell command, modify any file, make any network request

When to use: ONLY inside a Docker container or VM that:
1. Contains no real credentials or sensitive data
2. Has no network access to production systems
3. Is destroyed after the run completes
4. Is part of an automated test runner validating agent behavior

Never use:
- In `~/.claude/settings.json` (applies globally to all projects)
- On a developer machine with real API keys, SSH keys, or cloud credentials
- In any environment where a bad shell command could cause damage

Docker-sandboxed usage pattern:

```dockerfile
FROM node:20-alpine
WORKDIR /sandbox
COPY . .
RUN npm install
# Agent runs here with bypassPermissions — no real secrets in this image
CMD ["node", "run-agent.js"]
```

```typescript
// run-agent.js — only run inside the Docker container
for await (const msg of query({
  prompt: "Run the full test suite and fix any failing tests",
  options: {
    permissionMode: "bypassPermissions",
    maxTurns: 100,
    cwd: "/sandbox",
  },
})) { ... }
```

---

## 4. Orchestrator / Worker Pattern

### Architecture

The fundamental insight: a single powerful model doing everything is less effective than a coordinator reasoning about the whole task and multiple focused executors implementing pieces.

```
┌─────────────────────────────────────────────────────┐
│                   ORCHESTRATOR                       │
│  (claude-opus-4-6, high effort, full context)        │
│                                                      │
│  1. Read task spec                                   │
│  2. Decompose into subtasks                          │
│  3. Assign to workers                                │
│  4. Aggregate results                                │
│  5. Validate output                                  │
└──────┬────────────┬──────────────┬───────────────────┘
       │            │              │
       ▼            ▼              ▼
┌──────────┐ ┌──────────┐  ┌──────────┐
│ WORKER 1 │ │ WORKER 2 │  │ WORKER 3 │
│ (sonnet) │ │ (sonnet) │  │ (sonnet) │
│          │ │          │  │          │
│ auth     │ │ api      │  │ tests    │
│ layer    │ │ endpoints│  │ suite    │
└──────────┘ └──────────┘  └──────────┘
```

Internal benchmarks show 90.2% performance improvement using claude-opus-4-6 as orchestrator with claude-sonnet-4-6 workers versus a single claude-opus-4-6 agent on complex multi-domain tasks. The orchestrator's job is reasoning about decomposition and integration; workers handle bounded execution within a single domain.

### Complete TypeScript Orchestrator

```typescript
import { query } from "@anthropic-ai/claude-code";
import { writeFileSync, mkdirSync, readFileSync, appendFileSync } from "fs";
import { join } from "path";

interface TaskSpec {
  title: string;
  description: string;
  acceptance_criteria: string[];
  subtasks: SubTask[];
}

interface SubTask {
  id: string;
  domain: string;
  prompt: string;
  allowed_tools: string[];
}

interface WorkerResult {
  id: string;
  success: boolean;
  output: string;
  error?: string;
}

async function extractText(msg: any): Promise<string> {
  if (msg.type !== "assistant") return "";
  return msg.message.content
    .filter((b: any) => b.type === "text")
    .map((b: any) => b.text as string)
    .join("");
}

async function runWorker(subtask: SubTask, workDir: string, projectDir: string): Promise<WorkerResult> {
  const subtaskDir = join(workDir, subtask.id);
  mkdirSync(subtaskDir, { recursive: true });

  // Write isolated context — worker reads this, not a bloated inline prompt
  const contextFile = join(subtaskDir, "context.json");
  writeFileSync(contextFile, JSON.stringify(subtask, null, 2));

  const logFile = join(subtaskDir, "agent.log");
  const outputParts: string[] = [];

  try {
    for await (const msg of query({
      prompt: `Read ${contextFile} for your task context. Your domain: ${subtask.domain}. Task: ${subtask.prompt}. Write your output summary to ${join(subtaskDir, "output.md")}.`,
      options: {
        maxTurns: 30,
        cwd: projectDir,
        model: "claude-sonnet-4-6",
        allowedTools: subtask.allowed_tools,
        permissionMode: "acceptEdits",
      },
    })) {
      const text = await extractText(msg);
      if (text) outputParts.push(text);

      // Audit log for every turn
      if (msg.type === "tool_use") {
        appendFileSync(logFile, `TOOL: ${msg.name} ${JSON.stringify(msg.input).slice(0, 200)}\n`);
      }
    }

    const output = outputParts.join("\n");
    writeFileSync(join(subtaskDir, "output.md"), output);
    return { id: subtask.id, success: true, output };
  } catch (err: any) {
    writeFileSync(join(subtaskDir, "error.txt"), err.message ?? String(err));
    return { id: subtask.id, success: false, output: "", error: err.message };
  }
}

export async function orchestrate(specPath: string, projectDir: string): Promise<void> {
  const spec: TaskSpec = JSON.parse(readFileSync(specPath, "utf-8"));
  console.log(`Orchestrating: ${spec.title} — ${spec.subtasks.length} subtasks`);

  // Create a timestamped work directory for this run
  const workDir = join(projectDir, ".agent-work", `run-${Date.now()}`);
  mkdirSync(workDir, { recursive: true });
  writeFileSync(join(workDir, "spec.json"), JSON.stringify(spec, null, 2));

  // Fan out: all workers run in parallel
  const workerResults = await Promise.allSettled(
    spec.subtasks.map(subtask => runWorker(subtask, workDir, projectDir))
  );

  // Classify results
  const successes: WorkerResult[] = workerResults
    .filter((r): r is PromiseFulfilledResult<WorkerResult> => r.status === "fulfilled" && r.value.success)
    .map(r => r.value);

  const failures: WorkerResult[] = [
    ...workerResults
      .filter((r): r is PromiseFulfilledResult<WorkerResult> => r.status === "fulfilled" && !r.value.success)
      .map(r => r.value),
    ...workerResults
      .filter((r): r is PromiseRejectedResult => r.status === "rejected")
      .map((r, i) => ({ id: `unknown-${i}`, success: false, output: "", error: r.reason?.message })),
  ];

  if (failures.length > 0) {
    console.warn(`${failures.length} subtask(s) failed:`, failures.map(f => `${f.id}: ${f.error}`));
  }

  // Fan in: orchestrator validates and integrates
  const summaryFile = join(workDir, "summary.md");
  const aggregationPrompt = `
Review the subtask outputs in ${workDir}.

Completed subtasks: ${successes.map(s => s.id).join(", ")}
Failed subtasks: ${failures.map(f => `${f.id} (${f.error})`).join(", ") || "none"}

Acceptance criteria to verify:
${spec.acceptance_criteria.map(c => `- ${c}`).join("\n")}

For each criterion:
1. Read the relevant output files in ${workDir}
2. Verify the criterion is met
3. Note any gaps or integration issues

Write a structured report to ${summaryFile} with sections:
- PASSED criteria (list each with evidence)
- FAILED criteria (list each with what's missing)
- Integration issues (conflicts or gaps between subtasks)
- Recommended follow-up actions

Be specific — cite file paths and line ranges as evidence.
`;

  console.log("\nOrchestrator validating results...\n");
  for await (const msg of query({
    prompt: aggregationPrompt,
    options: {
      maxTurns: 20,
      cwd: projectDir,
      model: "claude-opus-4-6", // orchestrator uses the best model for final validation
      allowedTools: ["Read", "Write", "Glob", "Grep"],
    },
  })) {
    const text = await extractText(msg);
    if (text) process.stdout.write(text);
  }

  console.log(`\n\nWork artifacts: ${workDir}`);
  console.log(`Summary: ${summaryFile}`);
}
```

### Task Spec JSON Format

```json
{
  "title": "Add JWT Authentication",
  "description": "Implement JWT-based auth across the API layer",
  "acceptance_criteria": [
    "All /api routes require a valid JWT in Authorization header",
    "Token expiry returns 401 with clear error message",
    "Refresh token flow implemented and tested",
    "No hardcoded secrets — all keys read from env vars"
  ],
  "subtasks": [
    {
      "id": "auth-middleware",
      "domain": "authentication",
      "prompt": "Implement JWT verification middleware in src/middleware/auth.ts. Read existing middleware in that directory first.",
      "allowed_tools": ["Read", "Glob", "Grep", "Write", "Edit"]
    },
    {
      "id": "route-protection",
      "domain": "api",
      "prompt": "Apply auth middleware to all routes in src/routes/. Read each route file and add the middleware import and usage.",
      "allowed_tools": ["Read", "Glob", "Grep", "Edit"]
    },
    {
      "id": "auth-tests",
      "domain": "testing",
      "prompt": "Write tests for the JWT auth flow in src/__tests__/auth.test.ts. Cover: valid token, expired token, malformed token, missing header, refresh flow.",
      "allowed_tools": ["Read", "Glob", "Write", "Edit", "Bash(npm test -- --testPathPattern=auth*)"]
    }
  ]
}
```

---

## 5. Context Isolation Strategies

Context bleed between agents — where one agent's reasoning or output inadvertently influences another — is the primary source of subtle, hard-to-debug multi-agent failures. Prevent it structurally.

### Strategy 1: File-Based Context Passing (Primary)

Write all inter-agent data to files. Each agent reads only its designated file. No shared memory, no inline context embedding.

```typescript
// CORRECT: worker reads from isolated file
const prompt = `Read /tmp/agent-work/${taskId}/spec.json for your task. Implement only what's described there.`;

// INCORRECT: inlining context in the prompt
// This pollutes the context window and can cause reasoning bleed
const prompt = `Given this context: ${JSON.stringify(entireProjectState)} implement the auth layer...`;
```

Why this matters: when you inline a 20KB project context into every worker prompt, the model attends to all of it. Workers start referencing each other's domains, making changes outside their assigned scope, or producing outputs that assume knowledge they shouldn't have. File-based context gives each worker exactly what it needs, nothing more.

### Strategy 2: Structured JSON Handoffs with Validation

Define typed contracts between pipeline stages. Validate with Zod before each stage consumes the output.

```typescript
import { z } from "zod";

// Define the contract — written by stage N, read by stage N+1
const AnalysisResult = z.object({
  files_to_modify: z.array(z.string()),
  patterns_found: z.array(z.string()),
  risk_level: z.enum(["low", "medium", "high"]),
  recommended_approach: z.string(),
  estimated_changes: z.number().int().positive(),
});

type AnalysisResult = z.infer<typeof AnalysisResult>;

// Stage 1: analyzer writes result
async function runAnalysis(projectDir: string): Promise<AnalysisResult> {
  const resultFile = `${projectDir}/.agent-work/analysis.json`;
  await runAgent(
    `Analyze the codebase and write a JSON result to ${resultFile} matching this schema: ${JSON.stringify(AnalysisResult.shape)}`,
    projectDir
  );
  const raw = JSON.parse(readFileSync(resultFile, "utf-8"));
  return AnalysisResult.parse(raw); // throws if malformed
}

// Stage 2: implementer reads validated result
async function runImplementation(analysis: AnalysisResult, projectDir: string): Promise<void> {
  const contextFile = `${projectDir}/.agent-work/impl-context.json`;
  writeFileSync(contextFile, JSON.stringify(analysis, null, 2));
  await runAgent(
    `Read ${contextFile} for your context. Implement only the files listed in files_to_modify.`,
    projectDir
  );
}
```

Validation at stage boundaries prevents schema drift from accumulating silently across a long pipeline.

### Strategy 3: Sub-Process Isolation

Each `query()` call is a completely fresh process with zero knowledge of previous calls. There is no shared in-memory state between calls. The only shared state is the filesystem.

```typescript
// These two agents have completely separate context windows
// They share nothing except what they write to projectDir
const [authAnalysis, apiAnalysis] = await Promise.all([
  runAgent("Analyze src/auth/ for security vulnerabilities. Write findings to .agent-work/auth-analysis.md", projectDir),
  runAgent("Analyze src/api/ for missing input validation. Write findings to .agent-work/api-analysis.md", projectDir),
]);

// Now the aggregator reads both files — explicit handoff
await runAgent(
  "Read .agent-work/auth-analysis.md and .agent-work/api-analysis.md. Write a combined security report to .agent-work/security-report.md",
  projectDir
);
```

### Strategy 4: The Checkpoint Pattern for Long Sessions

When a single agent session accumulates too much context (typically after 15+ turns involving large file reads), performance degrades. Save state to disk and restart with a clean context.

```typescript
interface Checkpoint {
  completed_steps: string[];
  current_step: string;
  artifacts: Record<string, string>; // step -> file path
  next_task: string;
}

async function runWithCheckpointing(
  tasks: string[],
  projectDir: string
): Promise<void> {
  const checkpointFile = `${projectDir}/.agent-work/checkpoint.json`;

  for (let i = 0; i < tasks.length; i++) {
    const task = tasks[i];
    const artifactFile = `${projectDir}/.agent-work/step-${i}.md`;

    // Fresh context for each step — no memory of previous steps except checkpoint file
    await runAgent(
      `Read ${checkpointFile} to see what's been completed. Your task: ${task}. Write your output to ${artifactFile}.`,
      projectDir
    );

    // Update checkpoint
    const checkpoint: Checkpoint = {
      completed_steps: tasks.slice(0, i + 1),
      current_step: task,
      artifacts: Object.fromEntries(tasks.slice(0, i + 1).map((t, j) => [t, `step-${j}.md`])),
      next_task: tasks[i + 1] ?? "none",
    };
    writeFileSync(checkpointFile, JSON.stringify(checkpoint, null, 2));
  }
}
```

### Context Budget Estimation

Each model has a context window limit. Approximate token counts for planning:

| Content | Approximate Tokens |
|---------|-------------------|
| 1 line of code | 5–10 |
| 100-line file | 600–1,200 |
| 1,000-line file | 6,000–12,000 |
| Full project scan (Glob + Grep) | 2,000–8,000 |
| Each agent turn (system + history) | 500–2,000 |

Rule of thumb: if your task requires reading more than 10 large files plus 20 turns of conversation, checkpoint after the read phase. Pass a file list to the next agent rather than re-reading everything.

---

## 6. Named Subagents via @Mention

### Configuration

Named agents are defined in `~/.claude/settings.json` under the `agents` key. Each named agent has its own model, system prompt, and tool set.

```json
{
  "agents": {
    "reviewer": {
      "model": "claude-opus-4-6",
      "systemPrompt": "You are a strict code reviewer. Focus on: 1) security vulnerabilities, 2) correctness and edge cases, 3) maintainability and naming clarity. Be specific — cite file paths and line numbers. Do not suggest stylistic changes that have no correctness impact.",
      "allowedTools": ["Read", "Glob", "Grep"]
    },
    "tester": {
      "model": "claude-sonnet-4-6",
      "systemPrompt": "You write comprehensive tests. For every function you test: cover the happy path, at least 2 edge cases, and at least 1 failure mode. Prefer unit tests over integration tests unless integration is the only meaningful test. Always run tests after writing them.",
      "allowedTools": ["Read", "Write", "Edit", "Glob", "Grep", "Bash(npm test*)"]
    },
    "deployer": {
      "model": "claude-sonnet-4-6",
      "systemPrompt": "You handle deployment tasks. You are conservative — always read current state before making changes. Never deploy without confirming the build passes. Write a deployment log entry to .agent-work/deploy.log after every action.",
      "allowedTools": [
        "Read", "Bash(npm run build)", "Bash(npm test -- --ci)",
        "Bash(git status)", "Bash(git log --oneline -10)",
        "Bash(vercel deploy*)", "Write"
      ]
    },
    "analyst": {
      "model": "claude-opus-4-6",
      "systemPrompt": "You analyze codebases and produce structured reports. Always write your analysis to a file — never just output to chat. Format reports as markdown with clear sections, specific evidence, and actionable recommendations.",
      "allowedTools": ["Read", "Glob", "Grep", "Write"]
    }
  }
}
```

### Usage in Conversations

Mention named agents with `@` in any Claude Code chat:

```
@reviewer review src/auth/jwt.ts for security issues
@tester write tests for the JWT middleware I just implemented
@analyst analyze the API layer and identify performance bottlenecks
```

Named agents spawn inline in the current conversation. They receive the full conversation context at spawn time — they can see what you've been discussing.

### Named vs Anonymous Agents

**Use named agents when:**
- The role recurs across projects (reviewer, tester, deployer)
- You want consistent behavior enforced by a system prompt
- The tool set should be locked (e.g., reviewer never writes files)
- You want to reference the agent by name in team documentation

**Use anonymous (inline SDK) agents when:**
- The task is one-off or project-specific
- You need dynamic tool sets based on runtime conditions
- The agent is spawned programmatically, not interactively
- You need full control over model and options at call time

**Key difference**: named agents receive the full conversation context. Anonymous SDK agents (`query()` calls) start with only what you pass in `prompt`. For isolation-critical workflows, anonymous SDK agents are safer.

---

## 7. Background Agents

Headless execution without an interactive terminal. Background agents run to completion without any human input and are the foundation of automated agentic pipelines.

### CLI Invocation

```bash
# Basic headless run
claude --headless \
  --max-turns 50 \
  --allowedTools "Read,Write,Edit,Glob,Grep,Bash(git *),Bash(npm *)" \
  "Implement the feature described in .agent-work/spec.md. Write a completion report to .agent-work/result.md"

# With JSON output capture
claude --headless \
  --output-format json \
  --max-turns 30 \
  "Run all tests and write a report to test-report.json" \
  > .agent-work/agent-output.json 2>&1

# With permission mode
claude --headless \
  --permission-mode acceptEdits \
  --max-turns 80 \
  --allowedTools "Read,Write,Edit,Glob,Grep,Bash(npm run build),Bash(npm test -- --ci)" \
  "Refactor src/api/ to use the new BaseController pattern. See .agent-work/refactor-spec.md for details."

# Model selection for background tasks
claude --headless \
  --model claude-sonnet-4-6 \
  --max-turns 20 \
  "Generate TypeScript types from the OpenAPI spec at api/openapi.yaml. Write to src/types/api.ts"
```

### Programmatic Background Agent

```typescript
import { spawn } from "child_process";
import { appendFileSync, writeFileSync } from "fs";

interface BackgroundAgentResult {
  exitCode: number;
  logFile: string;
}

function spawnBackgroundAgent(
  prompt: string,
  projectDir: string,
  options: {
    maxTurns?: number;
    model?: string;
    allowedTools?: string[];
    logFile?: string;
  } = {}
): Promise<BackgroundAgentResult> {
  const {
    maxTurns = 50,
    model = "claude-sonnet-4-6",
    allowedTools = ["Read", "Write", "Edit", "Glob", "Grep"],
    logFile = `${projectDir}/.agent-work/agent-${Date.now()}.log`,
  } = options;

  return new Promise((resolve, reject) => {
    const args = [
      "--headless",
      "--max-turns", String(maxTurns),
      "--model", model,
      "--allowedTools", allowedTools.join(","),
      "--permission-mode", "acceptEdits",
      prompt,
    ];

    const proc = spawn("claude", args, {
      cwd: projectDir,
      stdio: ["ignore", "pipe", "pipe"],
      env: { ...process.env },
    });

    writeFileSync(logFile, `Agent started: ${new Date().toISOString()}\nPrompt: ${prompt}\n\n`);

    proc.stdout.on("data", (data: Buffer) => {
      appendFileSync(logFile, data.toString());
    });

    proc.stderr.on("data", (data: Buffer) => {
      appendFileSync(logFile, `[stderr] ${data.toString()}`);
    });

    proc.on("close", (code: number | null) => {
      appendFileSync(logFile, `\nAgent exited: ${new Date().toISOString()} (code ${code})\n`);
      resolve({ exitCode: code ?? 0, logFile });
    });

    proc.on("error", (err: Error) => {
      appendFileSync(logFile, `[error] ${err.message}\n`);
      reject(err);
    });
  });
}
```

### Progress Checkpointing for Background Agents

Instruct background agents to write checkpoints so the run can be monitored and resumed:

```typescript
const prompt = `
You are running headless. After completing each step, write a checkpoint to .agent-work/progress.json:
{
  "step": "<step_name>",
  "status": "completed" | "failed",
  "timestamp": "<ISO timestamp>",
  "artifact": "<path to output file if any>",
  "notes": "<brief description of what was done>"
}

Append to the JSON array — do not overwrite previous entries.

Tasks:
1. Read src/ directory structure and write a file inventory to .agent-work/inventory.md
2. Identify all TODO comments and write them to .agent-work/todos.md
3. For each TODO that has a clear fix, implement the fix
4. Run npm test and write results to .agent-work/test-results.md
`;

await spawnBackgroundAgent(prompt, "/path/to/project", { maxTurns: 80 });
```

Monitoring from a separate process:

```typescript
import { readFileSync, existsSync } from "fs";

function readProgress(projectDir: string): Array<{ step: string; status: string; timestamp: string }> {
  const progressFile = `${projectDir}/.agent-work/progress.json`;
  if (!existsSync(progressFile)) return [];
  try {
    return JSON.parse(readFileSync(progressFile, "utf-8"));
  } catch {
    return [];
  }
}

// Poll every 30 seconds
setInterval(() => {
  const progress = readProgress("/path/to/project");
  const latest = progress[progress.length - 1];
  if (latest) console.log(`Last checkpoint: ${latest.step} (${latest.status}) at ${latest.timestamp}`);
}, 30_000);
```

### Webhook Completion Notification

```typescript
// Agent writes completion signal; external service polls or watches
const COMPLETION_FILE = ".agent-work/completed.json";

// In the agent prompt:
const completionPrompt = `
After finishing all tasks, write a completion signal to ${COMPLETION_FILE}:
{
  "success": true | false,
  "timestamp": "<ISO timestamp>",
  "summary": "<one paragraph summary>",
  "artifacts": ["<list of output files>"],
  "errors": ["<any errors encountered>"]
}
`;

// External notification service (polls, then POSTs to webhook)
async function waitForCompletion(projectDir: string, webhookUrl: string): Promise<void> {
  const completionFile = `${projectDir}/.agent-work/completed.json`;
  while (true) {
    if (existsSync(completionFile)) {
      const result = JSON.parse(readFileSync(completionFile, "utf-8"));
      await fetch(webhookUrl, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ event: "agent_completed", ...result }),
      });
      return;
    }
    await new Promise(r => setTimeout(r, 5_000));
  }
}
```

### Cron-Triggered Background Agents

```bash
# Nightly code quality review
0 2 * * * cd /path/to/project && claude --headless --max-turns 30 \
  "Review all files modified in the last 24 hours. Write a report to .agent-work/nightly-review-$(date +\%Y\%m\%d).md. Include: code quality issues, potential bugs, security concerns." \
  >> /var/log/claude-agent.log 2>&1

# Hourly dependency check
0 * * * * cd /path/to/project && claude --headless --max-turns 10 \
  --allowedTools "Bash(npm audit),Bash(npm outdated),Write" \
  "Run npm audit and npm outdated. If critical vulnerabilities found, write an alert to .agent-work/security-alert.md"
```

---

## 8. Parallel Agent Execution Patterns

### Pattern 1: Independent Parallel Tasks

Use when tasks are fully independent — no shared files, no ordering constraint.

```typescript
const [authAnalysis, apiAnalysis, testAnalysis] = await Promise.all([
  runAgent("Analyze src/auth/ for security vulnerabilities. Write findings to .agent-work/auth.md", projectDir),
  runAgent("Analyze src/api/ for missing input validation. Write findings to .agent-work/api.md", projectDir),
  runAgent("Analyze test coverage gaps. Write findings to .agent-work/coverage.md", projectDir),
]);
// All three complete before proceeding — output files are independent
```

### Pattern 2: Rate-Limited Parallel Execution

When you have many tasks but need to respect API rate limits or avoid overwhelming the model service.

```typescript
import pLimit from "p-limit"; // npm install p-limit

const limit = pLimit(3); // max 3 concurrent agents

const filesToReview = [
  "src/auth/jwt.ts",
  "src/auth/session.ts",
  "src/api/users.ts",
  "src/api/payments.ts",
  "src/api/orders.ts",
  // ... 50 more files
];

const results = await Promise.all(
  filesToReview.map(file =>
    limit(() =>
      runAgent(
        `Review ${file} for security issues and code quality. Write findings to .agent-work/reviews/${file.replace(/\//g, "-")}.md`,
        projectDir
      )
    )
  )
);
```

`pLimit` ensures you never have more than `N` agents running simultaneously. Set `N` based on your API tier's concurrent request limit. For Anthropic's standard tier, start at 3–5 and increase if you're not hitting rate limits.

### Pattern 3: Fan-Out / Fan-In with Aggregation

The standard pattern for multi-domain tasks: decompose, parallelize, aggregate.

```typescript
async function fanOutFanIn(spec: TaskSpec, projectDir: string): Promise<void> {
  const workDir = `${projectDir}/.agent-work/${Date.now()}`;
  mkdirSync(workDir, { recursive: true });

  // Fan out: workers run in parallel
  const subtaskResults = await Promise.allSettled(
    spec.subtasks.map(task => runWorker(task, workDir, projectDir))
  );

  // Classify outcomes
  const successes = subtaskResults
    .filter((r): r is PromiseFulfilledResult<WorkerResult> => r.status === "fulfilled" && r.value.success)
    .map(r => r.value);

  const failures = subtaskResults
    .filter((r): r is PromiseRejectedResult | PromiseFulfilledResult<WorkerResult> =>
      r.status === "rejected" || (r.status === "fulfilled" && !r.value.success)
    );

  console.log(`Fan-out complete: ${successes.length} succeeded, ${failures.length} failed`);

  // Fan in: aggregator sees all results and validates integration
  await runAgent(`
Read all output files in ${workDir}.
Verify:
1. All acceptance criteria are met (list in spec.json)
2. No integration conflicts between the implemented components
3. Types and interfaces are consistent across boundaries

Write a validation report to ${workDir}/validation.md.
If any criterion fails or conflict found, be specific: file path, line range, what's wrong.
`, projectDir);
}
```

### Pattern 4: Parallel File Editing with Conflict Prevention

When workers need to edit source files (not just write to work directories), give each worker exclusive ownership of its files. The orchestrator assigns ownership before workers start.

```typescript
interface FileOwnership {
  workerId: string;
  files: string[];
}

async function assignFileOwnership(
  subtasks: SubTask[],
  projectDir: string
): Promise<Map<string, FileOwnership>> {
  // Orchestrator determines ownership to prevent conflicts
  const ownershipFile = `${projectDir}/.agent-work/ownership.json`;

  await runAgent(`
Analyze these subtasks and assign non-overlapping file ownership:
${JSON.stringify(subtasks, null, 2)}

For each subtask, determine which files it will need to modify.
Ensure no file appears in more than one subtask's list.
If two subtasks need the same file, assign it to the more relevant one.
Write the ownership map to ${ownershipFile} as:
[{ "workerId": "...", "files": ["src/..."] }]
`, projectDir);

  const ownership: FileOwnership[] = JSON.parse(readFileSync(ownershipFile, "utf-8"));
  return new Map(ownership.map(o => [o.workerId, o]));
}

// Workers receive explicit file lists — they only touch their assigned files
async function runOwnedWorker(
  subtask: SubTask,
  ownership: FileOwnership,
  projectDir: string
): Promise<void> {
  const prompt = `
Your task: ${subtask.prompt}
Your assigned files (ONLY modify these): ${ownership.files.join(", ")}
Do not touch any file not in your assigned list, even if it seems relevant.
Write a change summary to .agent-work/${subtask.id}/changes.md.
`;
  await runAgent(prompt, projectDir);
}
```

### Pattern 5: Pipeline (Sequential with Parallel Stages)

Some pipelines have stages that must be sequential but can parallelize within each stage.

```typescript
// Stage 1 (parallel): analysis
const [authAnalysis, apiAnalysis] = await Promise.all([
  runAgent("Analyze auth layer", projectDir),
  runAgent("Analyze API layer", projectDir),
]);

// Stage 2 (sequential): requires stage 1 outputs
await runAgent("Read .agent-work/auth.md and .agent-work/api.md. Design the integration interface.", projectDir);

// Stage 3 (parallel): implementation — can happen after design is written
const [authImpl, apiImpl] = await Promise.all([
  runAgent("Implement auth changes per .agent-work/design.md", projectDir),
  runAgent("Implement API changes per .agent-work/design.md", projectDir),
]);

// Stage 4 (sequential): testing depends on both implementations
await runAgent("Write integration tests for the auth+API changes. Run them with npm test.", projectDir);
```

---

## 9. Error Handling and Retry Patterns

### SDK Error Types

```typescript
import {
  ClaudeError,       // base class for all SDK errors
  MaxTurnsError,     // agent hit maxTurns without completing
  TimeoutError,      // AbortController fired or request timed out
  APIError,          // HTTP error from Anthropic API (has .status and .body)
} from "@anthropic-ai/claude-code";
```

### Retry with Adaptive Turn Budget

```typescript
async function runWithRetry(
  prompt: string,
  options: SDKOptions,
  maxAttempts = 3
): Promise<string> {
  let lastError: Error | null = null;
  let currentOptions = { ...options };

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const output: string[] = [];

      for await (const msg of query({ prompt, options: currentOptions })) {
        if (msg.type === "assistant") {
          output.push(
            msg.message.content
              .filter((b: any) => b.type === "text")
              .map((b: any) => b.text as string)
              .join("")
          );
        }
        if (msg.type === "result" && msg.subtype === "error_max_turns") {
          throw new MaxTurnsError(`Max turns (${currentOptions.maxTurns}) exceeded on attempt ${attempt}`);
        }
      }

      return output.join("\n");
    } catch (err: any) {
      lastError = err;
      console.warn(`Attempt ${attempt}/${maxAttempts} failed: ${err.message}`);

      if (err instanceof MaxTurnsError) {
        // Increase turn budget — task was too complex for current budget
        currentOptions = {
          ...currentOptions,
          maxTurns: (currentOptions.maxTurns ?? 20) * 2,
        };
        console.log(`Increasing maxTurns to ${currentOptions.maxTurns} for retry`);
      } else if (err instanceof APIError && err.status === 529) {
        // Service overloaded — exponential backoff
        const delayMs = Math.min(1000 * Math.pow(2, attempt), 30_000);
        console.log(`Rate limited. Waiting ${delayMs}ms before retry.`);
        await new Promise(r => setTimeout(r, delayMs));
      } else if (err instanceof TimeoutError) {
        // Increase timeout and retry
        currentOptions = { ...currentOptions };
        // AbortController timeout — recreate in calling code
        throw err; // let caller decide
      } else {
        throw err; // non-retryable: programming error, auth failure, etc.
      }
    }
  }

  throw lastError ?? new Error("All attempts exhausted");
}
```

### Circuit Breaker

Prevents cascading failures when an agent service is consistently failing.

```typescript
type CircuitState = "CLOSED" | "OPEN" | "HALF_OPEN";

class AgentCircuitBreaker {
  private state: CircuitState = "CLOSED";
  private failures = 0;
  private lastFailureTime = 0;
  private successes = 0;

  private readonly failureThreshold: number;
  private readonly resetTimeoutMs: number;
  private readonly halfOpenSuccessThreshold: number;

  constructor(options: {
    failureThreshold?: number;     // default: 5
    resetTimeoutMs?: number;       // default: 60_000
    halfOpenSuccessThreshold?: number; // default: 2
  } = {}) {
    this.failureThreshold = options.failureThreshold ?? 5;
    this.resetTimeoutMs = options.resetTimeoutMs ?? 60_000;
    this.halfOpenSuccessThreshold = options.halfOpenSuccessThreshold ?? 2;
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "OPEN") {
      const elapsed = Date.now() - this.lastFailureTime;
      if (elapsed < this.resetTimeoutMs) {
        throw new Error(`Circuit breaker OPEN — ${this.failures} failures. Retry after ${Math.ceil((this.resetTimeoutMs - elapsed) / 1000)}s`);
      }
      this.state = "HALF_OPEN";
      this.successes = 0;
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private onSuccess(): void {
    if (this.state === "HALF_OPEN") {
      this.successes++;
      if (this.successes >= this.halfOpenSuccessThreshold) {
        this.state = "CLOSED";
        this.failures = 0;
        console.log("Circuit breaker CLOSED — service recovered");
      }
    } else {
      this.failures = 0;
    }
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailureTime = Date.now();
    if (this.failures >= this.failureThreshold) {
      this.state = "OPEN";
      console.error(`Circuit breaker OPEN after ${this.failures} failures`);
    }
  }

  get currentState(): CircuitState {
    return this.state;
  }
}

// Usage
const breaker = new AgentCircuitBreaker({ failureThreshold: 3, resetTimeoutMs: 30_000 });

async function protectedAgentCall(prompt: string, projectDir: string): Promise<string> {
  return breaker.execute(() => runAgent(prompt, projectDir));
}
```

### Structured Error Reporting

When workers fail, write structured error artifacts so the orchestrator can make informed retry decisions.

```typescript
interface AgentFailure {
  task_id: string;
  error_type: "max_turns" | "api_error" | "tool_failure" | "validation_error" | "unknown";
  message: string;
  turns_completed?: number;
  last_tool_call?: string;
  partial_output?: string;
  retry_recommended: boolean;
  retry_strategy?: "increase_turns" | "simplify_task" | "check_permissions" | "wait_and_retry";
}

async function runWorkerWithStructuredErrors(
  task: SubTask,
  projectDir: string
): Promise<{ success: true; output: string } | { success: false; failure: AgentFailure }> {
  try {
    const output = await runAgent(task.prompt, projectDir);
    return { success: true, output };
  } catch (err: any) {
    const failure: AgentFailure = {
      task_id: task.id,
      error_type: "unknown",
      message: err.message ?? String(err),
      retry_recommended: false,
    };

    if (err instanceof MaxTurnsError) {
      failure.error_type = "max_turns";
      failure.retry_recommended = true;
      failure.retry_strategy = "increase_turns";
    } else if (err instanceof APIError) {
      failure.error_type = "api_error";
      failure.retry_recommended = err.status === 529 || err.status >= 500;
      failure.retry_strategy = "wait_and_retry";
    }

    writeFileSync(
      `${projectDir}/.agent-work/${task.id}/failure.json`,
      JSON.stringify(failure, null, 2)
    );

    return { success: false, failure };
  }
}
```

---

## 10. Inline MCP Server Configuration

Pass MCP servers directly in `SDKOptions.mcpServers` — no `mcp.json` file required. This is the correct approach for programmatic agent use.

### Basic GitHub MCP Integration

```typescript
import { query } from "@anthropic-ai/claude-code";

for await (const msg of query({
  prompt: "Create a GitHub issue for each TODO comment in src/. Use the format: 'TODO: <description>' as the issue title.",
  options: {
    maxTurns: 30,
    cwd: "/path/to/project",
    mcpServers: [{
      name: "github",
      transport: {
        type: "stdio",
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-github"],
        env: {
          GITHUB_TOKEN: process.env.GITHUB_TOKEN!,
          GITHUB_OWNER: "your-org",
          GITHUB_REPO: "your-repo",
        },
      },
    }],
    allowedTools: ["Read", "Grep", "mcp__github__create_issue", "mcp__github__list_issues"],
  },
})) {
  if (msg.type === "assistant") {
    process.stdout.write(
      msg.message.content.filter((b: any) => b.type === "text").map((b: any) => b.text).join("")
    );
  }
}
```

### Multiple MCP Servers

```typescript
for await (const msg of query({
  prompt: "Check Stripe for failed payments in the last 24h, create a PagerDuty incident if any found, and write a report to .agent-work/payment-report.md",
  options: {
    maxTurns: 20,
    mcpServers: [
      {
        name: "stripe",
        transport: {
          type: "stdio",
          command: "npx",
          args: ["-y", "@modelcontextprotocol/server-stripe"],
          env: { STRIPE_API_KEY: process.env.STRIPE_API_KEY! },
        },
      },
      {
        name: "pagerduty",
        transport: {
          type: "stdio",
          command: "npx",
          args: ["-y", "@company/mcp-server-pagerduty"],
          env: { PAGERDUTY_TOKEN: process.env.PAGERDUTY_TOKEN! },
        },
      },
    ],
    allowedTools: [
      "Write",
      "mcp__stripe__list_charges",
      "mcp__stripe__retrieve_charge",
      "mcp__pagerduty__create_incident",
    ],
  },
})) { ... }
```

### SSE Transport (HTTP-based MCP servers)

```typescript
for await (const msg of query({
  prompt: "Query the internal analytics database for this week's conversion funnel",
  options: {
    mcpServers: [{
      name: "analytics-db",
      transport: {
        type: "sse",
        url: "https://internal-mcp.company.com/analytics",
        headers: {
          Authorization: `Bearer ${process.env.INTERNAL_MCP_TOKEN}`,
        },
      },
    }],
    allowedTools: ["Write", "mcp__analytics-db__query", "mcp__analytics-db__list_tables"],
  },
})) { ... }
```

### Tool Naming Convention for MCP

MCP tools follow the pattern `mcp__<server-name>__<tool-name>`. When you name your server `"github"`, the create issue tool becomes `mcp__github__create_issue`. Always use exact names in `allowedTools` — the SDK enforces this whitelist strictly.

To discover what tools an MCP server exposes, run it without an `allowedTools` filter on a test prompt and inspect the `tool_use` messages in the stream.

---

## 11. Model Selection Strategy

### Model Capability Reference (2026)

| Model | Reasoning | Speed | Cost | Context |
|-------|-----------|-------|------|---------|
| claude-opus-4-6 | Highest | Slowest | Highest | 200K tokens |
| claude-sonnet-4-6 | High | Fast | Medium | 200K tokens |
| claude-haiku-4-5 | Good | Fastest | Lowest | 200K tokens |

### Role-Based Selection Matrix

| Agent Role | Recommended Model | Rationale |
|------------|------------------|-----------|
| Orchestrator (planning, decomposition) | claude-opus-4-6 | Complex multi-domain reasoning; runs once per task |
| Worker (focused implementation) | claude-sonnet-4-6 | Capable for bounded tasks; cost scales with worker count |
| Code reviewer / security auditor | claude-opus-4-6 | Catches subtle issues; high-value role justifies cost |
| Test generation | claude-sonnet-4-6 | Reliable coverage; cost-efficient at scale |
| Documentation writer | claude-sonnet-4-6 | High quality; doesn't need opus-level reasoning |
| Simple file transforms | claude-haiku-4-5 | Deterministic tasks; cheapest option |
| Background monitoring / polling | claude-haiku-4-5 | Runs constantly; minimize cost |
| Final validation / QA | claude-opus-4-6 | One-time cost; catches what workers miss |

### Cost Optimization Principles

**1. Orchestrator runs once, workers run many times.** Spend on claude-opus-4-6 for planning (1 call per task). Use claude-sonnet-4-6 for the N worker calls that do the actual work. Total cost = `opus_price * 1 + sonnet_price * N` — much cheaper than `opus_price * N`.

**2. Scope `allowedTools` tightly.** An agent with only `["Read", "Grep"]` can't accidentally make expensive tool calls. Fewer tools also means the model spends fewer tokens reasoning about which tool to use.

**3. Set `maxTurns` to match task complexity.** An agent with `maxTurns: 5` on a simple file transform uses far fewer tokens than one with `maxTurns: 50`. Set the budget based on actual task complexity, not a conservative maximum.

**4. Use `appendSystemPrompt` instead of `systemPrompt` for customization.** The default system prompt is optimized for tool use. Replacing it entirely (`systemPrompt`) can degrade performance. Append custom instructions instead.

**5. File reads are cheap; LLM turns are expensive.** Structure prompts to minimize turns, not file reads. "Read all these files, then write the implementation in one response" is cheaper than 10 turns of read-respond-read-respond.

### Model Override in Settings vs SDK

Setting model in `~/.claude/settings.json` applies globally to all Claude Code interactions:

```json
{
  "model": "claude-sonnet-4-6"
}
```

Overriding in `SDKOptions` applies only to that specific agent call:

```typescript
for await (const msg of query({
  prompt: "...",
  options: { model: "claude-opus-4-6" }, // overrides settings.json for this call only
})) { ... }
```

For orchestrator/worker systems: set a cheaper default in `settings.json` and override to `claude-opus-4-6` only for orchestrator and review calls.

---

## 12. Production Deployment Checklist

### Security

- [ ] `permissionMode: "default"` unless you have an explicit reason for `acceptEdits`
- [ ] `allowedTools` whitelist defined — never rely on `disallowedTools` alone
- [ ] No `ANTHROPIC_API_KEY` in source code — use env vars or secrets manager
- [ ] `bypassPermissions` never used outside of fully sandboxed containers
- [ ] MCP server env vars (GITHUB_TOKEN, etc.) injected at runtime, not hardcoded
- [ ] `cwd` scoped to project directory — agents should not traverse above it

### Reliability

- [ ] `maxTurns` set explicitly on every `query()` call
- [ ] `AbortController` with timeout on every `query()` call
- [ ] Retry logic with exponential backoff for `APIError` (status 529, 5xx)
- [ ] Circuit breaker for multi-agent pipelines
- [ ] Structured error artifacts written on failure for orchestrator inspection
- [ ] Progress checkpoints for agents running > 20 turns

### Observability

- [ ] All `tool_use` messages logged with task ID and timestamp
- [ ] `tool_result` content logged for debugging failed runs
- [ ] Work directory preserved after run (not cleaned up immediately)
- [ ] Summary report written to known path after orchestrator validation
- [ ] Exit code checked when using CLI `--headless` invocation

### Cost Control

- [ ] `model` explicitly set — don't rely on default changing
- [ ] Worker agents use `claude-sonnet-4-6`, not `claude-opus-4-6`
- [ ] `maxTurns` proportional to task complexity (5 for simple, 30 for complex, 80 for full-codebase)
- [ ] Parallel concurrency limited with `p-limit` to avoid rate limits
- [ ] Token usage logged from `system/result` message for cost attribution
