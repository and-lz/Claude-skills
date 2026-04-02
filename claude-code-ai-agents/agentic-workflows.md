# Agentic Workflows & Pipeline Design

A dense reference for senior engineers designing multi-agent pipelines, SDLC automation, and production-grade agentic workflows with Claude Code in 2026. Covers spec-driven development, the 5-stage agentic SDLC, seven named pipeline patterns, long-running agent management, structured handoffs, human-in-the-loop design, and cost optimization.

This file assumes familiarity with the Agent SDK (`query()`, `SDKMessage`, `SDKOptions`). If you need the SDK reference, read `agent-sdk-orchestration.md` first. If you need Claude Code customization (CLAUDE.md, hooks, settings), read `claude-code-mastery.md`.

---

## 1. Spec-Driven Development

### Why specs replace ad hoc prompts

Ad hoc prompts — "build me a login page" or "refactor the auth module" — produce non-reproducible, non-auditable, non-debuggable outputs. Each run generates subtly different results. There is no baseline to regression-test against, no structured input to replay, no diff between "what we asked for" and "what we got." When a pipeline fails, you cannot point to the specification that was violated because there was no specification.

Spec-driven development inverts this. The spec is the contract: a structured, versioned, machine-parseable document that defines inputs, expected outputs, constraints, and success criteria before the agent runs. The agent is the executor of the spec, not the inventor of requirements. This gives you:

- **Reproducibility**: The same spec produces equivalent outputs across runs, models, and agent versions. You can replay a spec against a new model to validate behavior.
- **Auditability**: Every pipeline run can be traced to a specific spec version in git. Compliance reviews can inspect the spec to verify what was requested.
- **Iteration**: Changing a spec field (adding a constraint, narrowing scope, changing the output format) is a surgical edit. Changing an ad hoc prompt requires re-reasoning about the entire request.
- **Evaluation**: Specs define expected outputs. Eval harnesses compare actual outputs against those expectations. Without a spec, there is no ground truth.
- **Team alignment**: A spec reviewed by three engineers produces consensus before any code is written. An ad hoc prompt reviewed by three engineers produces three different interpretations.

### Three spec formats

Choose the format based on who writes and consumes the spec.

#### 1. Structured JSON

Best for: pipelines where specs are generated programmatically, consumed by automation, and stored in databases or APIs. Maximum machine-parseability.

```json
{
  "$schema": "https://example.com/schemas/task-spec-v2.json",
  "id": "TASK-2026-0401-jwt-refresh",
  "version": "2.1.0",
  "title": "Implement JWT refresh token rotation",
  "description": "Add secure refresh token rotation to the existing JWT auth system. Tokens must be rotated on every refresh request. Old tokens must be invalidated immediately.",
  "inputs": {
    "source_files": ["src/lib/auth/jwt.ts", "src/lib/auth/session.ts"],
    "schema_files": ["prisma/schema.prisma"],
    "reference_docs": ["docs/specs/auth-v2.md"]
  },
  "outputs": {
    "modified_files": ["src/lib/auth/jwt.ts", "src/lib/auth/session.ts", "prisma/schema.prisma"],
    "new_files": ["src/lib/auth/refresh-rotation.ts", "src/lib/auth/__tests__/refresh-rotation.test.ts"],
    "artifacts": ["test-report.json"]
  },
  "constraints": {
    "backward_compatible": true,
    "max_files_changed": 8,
    "required_test_coverage": 0.9,
    "no_breaking_api_changes": true
  },
  "acceptance_criteria": [
    "Refresh tokens rotate on every /api/auth/refresh call",
    "Old refresh token is invalidated within 1 second of rotation",
    "Concurrent refresh requests with the same token result in exactly one success and N-1 failures",
    "Token family tracking detects reuse of rotated tokens and invalidates the entire family",
    "All existing sessions continue to work without forced logout"
  ],
  "agent_config": {
    "model": "claude-sonnet-4-6",
    "max_turns": 30,
    "allowed_tools": ["Read", "Glob", "Grep", "Write", "Edit", "Bash"],
    "permission_mode": "acceptEdits",
    "timeout_ms": 600000
  },
  "metadata": {
    "author": "andre@example.com",
    "created": "2026-04-01T10:00:00Z",
    "priority": "high",
    "labels": ["security", "auth", "backend"]
  }
}
```

#### 2. Markdown with YAML frontmatter

Best for: specs written by humans, reviewed in PRs, and consumed by agents that parse the frontmatter for machine-readable fields and the body for natural-language context.

```markdown
---
id: TASK-2026-0401-jwt-refresh
version: 2.1.0
model: claude-sonnet-4-6
max_turns: 30
allowed_tools: [Read, Glob, Grep, Write, Edit, Bash]
priority: high
labels: [security, auth, backend]
inputs:
  - src/lib/auth/jwt.ts
  - src/lib/auth/session.ts
  - prisma/schema.prisma
outputs:
  - src/lib/auth/refresh-rotation.ts
  - src/lib/auth/__tests__/refresh-rotation.test.ts
  - test-report.json
---

# JWT Refresh Token Rotation

## Goal

Add secure refresh token rotation to the existing JWT auth system.
Tokens rotate on every refresh. Old tokens invalidate immediately.

## Acceptance Criteria

1. Refresh tokens rotate on every `/api/auth/refresh` call
2. Old refresh token invalidated within 1 second
3. Concurrent refresh with same token: exactly 1 success, rest fail
4. Token family tracking detects reuse, invalidates entire family
5. Existing sessions unaffected — no forced logouts

## Context

Current auth: `src/lib/auth/jwt.ts` (lines 1-120).
Session store: `src/lib/auth/session.ts` (Prisma-backed).
Schema: `prisma/schema.prisma` — add `RefreshTokenFamily` model.

## Constraints

- Backward compatible — no breaking API changes
- Max 8 files changed
- 90%+ test coverage on new code
```

#### 3. TypeScript interface

Best for: codebases where specs are constructed in code, validated at compile time, and passed directly to pipeline functions. Maximum type safety.

```typescript
// src/pipeline/types/task-spec.ts

import { z } from "zod";

/** Defines the complete contract for an agentic task. */
export interface TaskSpec {
  /** Unique identifier for this task. Format: TASK-YYYY-MMDD-slug */
  id: string;

  /** Semantic version of this spec. Bump major on output format changes. */
  version: string;

  /** Human-readable title. Under 80 characters. */
  title: string;

  /** Detailed description of what the agent should accomplish. */
  description: string;

  /** Input files and references the agent needs to read. */
  inputs: {
    /** Existing source files the agent must read before starting. */
    source_files: string[];
    /** Schema or type definition files that define data structures. */
    schema_files?: string[];
    /** Documentation or design docs for additional context. */
    reference_docs?: string[];
    /** Environment variables required (names only, not values). */
    required_env_vars?: string[];
  };

  /** Expected outputs the agent must produce. */
  outputs: {
    /** Files that should be modified (must already exist). */
    modified_files: string[];
    /** Files that should be created (must not already exist). */
    new_files: string[];
    /** Non-code artifacts: test reports, docs, manifests. */
    artifacts?: string[];
  };

  /** Hard constraints the agent must not violate. */
  constraints: {
    /** If true, no existing public API signatures may change. */
    backward_compatible: boolean;
    /** Maximum number of files the agent may modify or create. */
    max_files_changed: number;
    /** Minimum test coverage ratio (0.0-1.0) for new/changed code. */
    required_test_coverage?: number;
    /** If true, no existing tests may be deleted or weakened. */
    no_breaking_api_changes?: boolean;
    /** Maximum allowed increase in bundle size (bytes). */
    max_bundle_size_increase?: number;
  };

  /** Human-readable success conditions. Each must be independently verifiable. */
  acceptance_criteria: string[];

  /** Agent runtime configuration. */
  agent_config: {
    model: "claude-opus-4-6" | "claude-sonnet-4-6" | "claude-haiku-4-5";
    max_turns: number;
    allowed_tools: string[];
    permission_mode: "default" | "acceptEdits" | "bypassPermissions";
    timeout_ms: number;
    /** Optional system prompt additions specific to this task. */
    append_system_prompt?: string;
  };

  /** Tracking metadata. */
  metadata: {
    author: string;
    created: string;
    priority: "critical" | "high" | "medium" | "low";
    labels: string[];
    /** Git issue or ticket URL this spec implements. */
    issue_url?: string;
    /** Parent spec ID if this is a sub-task. */
    parent_spec_id?: string;
  };
}

/** Zod runtime validator derived from the interface. */
export const TaskSpecSchema = z.object({
  id: z.string().regex(/^TASK-\d{4}-\d{4}-[\w-]+$/),
  version: z.string().regex(/^\d+\.\d+\.\d+$/),
  title: z.string().max(80),
  description: z.string().min(20),
  inputs: z.object({
    source_files: z.array(z.string()).min(1),
    schema_files: z.array(z.string()).optional(),
    reference_docs: z.array(z.string()).optional(),
    required_env_vars: z.array(z.string()).optional(),
  }),
  outputs: z.object({
    modified_files: z.array(z.string()),
    new_files: z.array(z.string()),
    artifacts: z.array(z.string()).optional(),
  }),
  constraints: z.object({
    backward_compatible: z.boolean(),
    max_files_changed: z.number().int().positive(),
    required_test_coverage: z.number().min(0).max(1).optional(),
    no_breaking_api_changes: z.boolean().optional(),
    max_bundle_size_increase: z.number().int().optional(),
  }),
  acceptance_criteria: z.array(z.string()).min(1),
  agent_config: z.object({
    model: z.enum(["claude-opus-4-6", "claude-sonnet-4-6", "claude-haiku-4-5"]),
    max_turns: z.number().int().min(1).max(200),
    allowed_tools: z.array(z.string()).min(1),
    permission_mode: z.enum(["default", "acceptEdits", "bypassPermissions"]),
    timeout_ms: z.number().int().min(10000).max(3600000),
    append_system_prompt: z.string().optional(),
  }),
  metadata: z.object({
    author: z.string().email(),
    created: z.string().datetime(),
    priority: z.enum(["critical", "high", "medium", "low"]),
    labels: z.array(z.string()),
    issue_url: z.string().url().optional(),
    parent_spec_id: z.string().optional(),
  }),
}) satisfies z.ZodType<TaskSpec>;
```

### The spec-to-agent-to-eval-to-merge loop

Every spec-driven task follows a five-step loop:

1. **Write the spec.** A human (or a requirements agent) authors a `TaskSpec` in JSON, markdown+YAML, or TypeScript. The spec is committed to git under `docs/specs/` and reviewed in a PR before execution.

2. **Run the agent against the spec.** A pipeline function reads the spec, constructs a structured prompt from its fields, and calls `query()` with the spec's `agent_config`. The agent executes within the boundaries defined by the spec — it reads only the files listed in `inputs`, produces only the files listed in `outputs`, and operates within the `constraints`.

3. **Evaluate the output.** An eval harness (or a reviewer agent) checks each `acceptance_criteria` item against the actual output. File existence is verified. Tests are run. Coverage is measured. The eval produces a structured `EvalResult` with pass/fail per criterion.

4. **Gate on eval results.** If all criteria pass, the output proceeds to merge. If any criterion fails, the spec and eval feedback are fed back to the agent for a retry (up to a configured retry budget). If retries are exhausted, the task escalates to a human.

5. **Merge and archive.** The agent's output is merged (via PR). The spec, eval results, and agent session log are archived together as an auditable record. The spec version is tagged in git.

### TypeScript: `runFromSpec()` function

```typescript
// src/pipeline/run-from-spec.ts

import { query, type SDKMessage } from "@anthropic-ai/claude-code";
import { readFile } from "fs/promises";
import { TaskSpecSchema, type TaskSpec } from "./types/task-spec";

/**
 * Reads a TaskSpec from a JSON file, validates it, constructs a structured
 * prompt, and runs a Claude Code agent against it.
 *
 * Returns the agent's final text output and all messages for logging.
 */
export async function runFromSpec(specPath: string): Promise<{
  output: string;
  messages: SDKMessage[];
  turns: number;
}> {
  // 1. Read and validate the spec
  const raw = JSON.parse(await readFile(specPath, "utf-8"));
  const spec = TaskSpecSchema.parse(raw);

  // 2. Build a structured prompt from the spec
  const prompt = buildPromptFromSpec(spec);

  // 3. Run the agent
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), spec.agent_config.timeout_ms);

  const messages: SDKMessage[] = [];
  const textOutput: string[] = [];
  let turns = 0;

  try {
    for await (const msg of query({
      prompt,
      abortController: controller,
      options: {
        maxTurns: spec.agent_config.max_turns,
        allowedTools: spec.agent_config.allowed_tools,
        permissionMode: spec.agent_config.permission_mode,
        model: spec.agent_config.model,
        appendSystemPrompt: spec.agent_config.append_system_prompt,
      },
    })) {
      messages.push(msg);

      if (msg.type === "assistant") {
        turns++;
        const text = msg.message.content
          .filter((b: any) => b.type === "text")
          .map((b: any) => b.text)
          .join("");
        if (text) textOutput.push(text);
      }

      if (msg.type === "result") {
        if (msg.subtype === "error_max_turns") {
          console.warn(`[${spec.id}] Agent exhausted max turns (${spec.agent_config.max_turns})`);
        }
        if (msg.subtype === "error_during_turn") {
          console.error(`[${spec.id}] Agent error:`, msg);
        }
      }
    }
  } finally {
    clearTimeout(timeout);
  }

  return {
    output: textOutput.join("\n"),
    messages,
    turns,
  };
}

/**
 * Converts a TaskSpec into a structured natural-language prompt
 * that gives the agent all necessary context and constraints.
 */
function buildPromptFromSpec(spec: TaskSpec): string {
  return `You are executing task ${spec.id} (v${spec.version}).

## Task
${spec.title}

## Description
${spec.description}

## Input Files to Read First
${spec.inputs.source_files.map((f) => `- ${f}`).join("\n")}
${spec.inputs.schema_files?.map((f) => `- ${f} (schema)`).join("\n") ?? ""}
${spec.inputs.reference_docs?.map((f) => `- ${f} (reference)`).join("\n") ?? ""}

## Expected Outputs
### Files to Modify
${spec.outputs.modified_files.map((f) => `- ${f}`).join("\n")}

### Files to Create
${spec.outputs.new_files.map((f) => `- ${f}`).join("\n")}

### Artifacts to Produce
${spec.outputs.artifacts?.map((f) => `- ${f}`).join("\n") ?? "None"}

## Hard Constraints — Do Not Violate
- Backward compatible: ${spec.constraints.backward_compatible ? "YES — no existing public APIs may change" : "Breaking changes allowed"}
- Max files changed: ${spec.constraints.max_files_changed}
${spec.constraints.required_test_coverage ? `- Required test coverage: ${(spec.constraints.required_test_coverage * 100).toFixed(0)}%` : ""}
${spec.constraints.no_breaking_api_changes ? "- No breaking API changes allowed" : ""}

## Acceptance Criteria — All Must Pass
${spec.acceptance_criteria.map((c, i) => `${i + 1}. ${c}`).join("\n")}

## Instructions
1. Read all input files listed above before making any changes.
2. Produce exactly the files listed in Expected Outputs.
3. Do not modify files not listed in Expected Outputs unless absolutely necessary (and stay within max_files_changed).
4. Write tests that verify each acceptance criterion.
5. After completing all changes, produce a summary listing: files changed, tests added, and which acceptance criteria are covered.`;
}
```

---

## 2. The 5-Stage Agentic SDLC

The agentic SDLC maps the traditional software development lifecycle onto a pipeline of specialized agents, each responsible for one stage. The key insight: each stage has a different context requirement, different tool needs, and a different risk profile — so each stage uses a different agent configuration.

```
 ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
 │ REQUIREMENTS│───>│   DESIGN    │───>│ IMPLEMENT   │───>│    TEST     │───>│   DEPLOY    │
 │   Agent     │    │   Agent     │    │  Workers    │    │   Agent     │    │   Agent     │
 │             │    │             │    │ (parallel)  │    │             │    │             │
 │ Reads issue │    │ Cross-refs  │    │ Each worker │    │ Runs suite  │    │ Creates PR  │
 │ Writes spec │    │ architect   │    │ owns files  │    │ Writes JSON │    │ Deploys via │
 │             │    │ skill P1+P2 │    │             │    │ report      │    │ MCP servers │
 └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
        │                  │                  │                  │                  │
        ▼                  ▼                  ▼                  ▼                  ▼
   TaskSpec.json     plan/<slug>.md     src/ changes      test-report.json    PR + deploy URL
                                                                              
   ◄── HUMAN GATE ──► ◄── HUMAN GATE ──►              ◄── HUMAN GATE ──►
```

Three human gates: after requirements (approve spec), after design (approve plan), and after testing (approve for deployment). Implementation and testing are autonomous within the constraints set by the approved spec and plan.

### Stage 1: Requirements Agent

The requirements agent reads an issue (GitHub issue, Jira ticket, Slack thread — via MCP), asks clarifying questions if needed, and produces a `TaskSpec`.

**Prompt template:**

```typescript
const requirementsPrompt = `You are a requirements analyst agent.

## Input
GitHub Issue: ${issueUrl}

## Instructions
1. Read the issue using the GitHub MCP tool.
2. Read all comments on the issue for additional context.
3. Read the referenced files mentioned in the issue.
4. Identify ambiguities. If critical information is missing, list what you need
   under "## Blockers" and stop. Do not guess.
5. Produce a TaskSpec JSON file at docs/specs/${specId}.json that includes:
   - All input files the implementation agent will need
   - All expected output files
   - Hard constraints derived from the issue requirements
   - Acceptance criteria: one per testable requirement, written as assertions
6. Validate that every acceptance criterion is independently verifiable —
   no vague criteria like "works correctly" or "is well-designed."

## Output
Write the spec to docs/specs/${specId}.json and summarize what you produced.`;
```

**Agent config for requirements stage:**

```typescript
const requirementsConfig: SDKOptions = {
  maxTurns: 15,
  allowedTools: [
    "Read", "Glob", "Grep", "Write",
    "mcp__github__get_issue",
    "mcp__github__list_issue_comments",
  ],
  permissionMode: "acceptEdits",
  model: "claude-sonnet-4-6", // Sonnet is sufficient for analysis tasks
};
```

### Stage 2: Design / Architect Agent

The architect agent reads the approved `TaskSpec` and produces a detailed implementation plan. This stage directly corresponds to the `architect` skill's Phase 1 (Context Refinement) and Phase 2 (Architectural Plan). The architect agent performs codebase pattern discovery, maps the blast radius, identifies dependencies, and produces a step-by-step plan where each step modifies 3-4 files and leaves the codebase buildable.

**Prompt template:**

```typescript
const architectPrompt = `You are an architect agent. You have two jobs:
1. Context gathering (architect skill Phase 1)
2. Plan production (architect skill Phase 2)

## Input Spec
${JSON.stringify(spec, null, 2)}

## Phase 1: Context
- Read every file listed in spec.inputs.source_files
- Use Grep/Glob to find related patterns, existing utilities, naming conventions
- Identify the dependency chain: which files must change first
- Map the blast radius: every file affected, including ripple effects
- Check for concurrent development: are other specs touching the same files?

## Phase 2: Plan
- Produce an ordered list of implementation steps
- Each step: description, files to modify/create, verification criterion
- Steps ordered by dependency chain (shared interfaces first)
- Each step leaves the codebase buildable (no broken intermediate states)
- Address cross-cutting concerns: security, testing, a11y, performance
- Write the plan to docs/plans/${planSlug}.md

## Constraints
- Do not write any implementation code. Plan only.
- Reference existing utilities and patterns discovered in Phase 1.
- Every file path in the plan must exist or be explicitly marked (create).`;
```

**Agent config for architect stage:**

```typescript
const architectConfig: SDKOptions = {
  maxTurns: 25,
  allowedTools: ["Read", "Glob", "Grep", "Write"], // Read-only + write plan doc
  permissionMode: "acceptEdits",
  model: "claude-sonnet-4-6",
};
```

### Stage 3: Parallel Implementation Workers

The implementation stage fans out to parallel workers, each responsible for a subset of the plan steps. Workers are isolated: each gets its own working directory (via git worktree or copy), its own set of files, and no shared mutable state with siblings. The orchestrator assigns steps to workers and aggregates their outputs.

**Prompt template per worker:**

```typescript
function buildWorkerPrompt(spec: TaskSpec, steps: PlanStep[], workerIndex: number): string {
  return `You are implementation worker #${workerIndex}.

## Your Assigned Steps
${steps.map((s, i) => `### Step ${s.number}: ${s.description}
Files: ${s.files.map((f) => `${f.path} (${f.action})`).join(", ")}
Verification: ${s.verification}`).join("\n\n")}

## Constraints from Spec
- Backward compatible: ${spec.constraints.backward_compatible}
- Follow existing patterns found in the codebase
- Write tests for each step's verification criterion
- Do not modify files outside your assigned steps

## Instructions
1. Read the files you need to modify.
2. Implement each step in order.
3. After each step, verify the criterion is met.
4. After all steps, run the relevant test file(s).
5. Produce a summary: files changed, tests added, status per step.`;
}
```

**Parallel execution:**

```typescript
import { query, type SDKMessage } from "@anthropic-ai/claude-code";

interface WorkerResult {
  workerIndex: number;
  output: string;
  messages: SDKMessage[];
  success: boolean;
}

async function runParallelWorkers(
  spec: TaskSpec,
  plan: Plan,
  workerCount: number = 3,
): Promise<WorkerResult[]> {
  // Partition plan steps across workers, respecting dependency order
  const stepGroups = partitionSteps(plan.steps, workerCount);

  const workerPromises: Promise<WorkerResult>[] = stepGroups.map(
    async (steps, index) => {
      const prompt = buildWorkerPrompt(spec, steps, index);
      const controller = new AbortController();
      const timeout = setTimeout(() => controller.abort(), spec.agent_config.timeout_ms);

      const messages: SDKMessage[] = [];
      const textOutput: string[] = [];

      try {
        for await (const msg of query({
          prompt,
          abortController: controller,
          options: {
            maxTurns: Math.ceil(spec.agent_config.max_turns / workerCount),
            allowedTools: spec.agent_config.allowed_tools,
            permissionMode: spec.agent_config.permission_mode,
            model: spec.agent_config.model,
            cwd: `/tmp/worktree-${spec.id}-worker-${index}`,
          },
        })) {
          messages.push(msg);
          if (msg.type === "assistant") {
            const text = msg.message.content
              .filter((b: any) => b.type === "text")
              .map((b: any) => b.text)
              .join("");
            if (text) textOutput.push(text);
          }
        }

        return {
          workerIndex: index,
          output: textOutput.join("\n"),
          messages,
          success: true,
        };
      } catch (err) {
        return {
          workerIndex: index,
          output: `Worker ${index} failed: ${err}`,
          messages,
          success: false,
        };
      } finally {
        clearTimeout(timeout);
      }
    },
  );

  // All workers run in parallel — Promise.all
  return Promise.all(workerPromises);
}

/**
 * Partition plan steps into N groups, keeping dependent steps together.
 * Steps with sequential dependencies stay in the same group.
 */
function partitionSteps(steps: PlanStep[], n: number): PlanStep[][] {
  const groups: PlanStep[][] = Array.from({ length: n }, () => []);
  // Simple round-robin; real implementation should respect dependency DAG
  for (let i = 0; i < steps.length; i++) {
    groups[i % n].push(steps[i]);
  }
  return groups;
}
```

### Stage 4: Test Agent

The test agent runs the full test suite against the combined output of all workers, produces a structured test report, and flags any failing acceptance criteria.

**Prompt template:**

```typescript
const testPrompt = `You are a test agent.

## Input
Task spec: docs/specs/${spec.id}.json
Implementation complete. All workers have finished.

## Instructions
1. Run the full test suite: pnpm test --coverage --json --outputFile=test-report.json
2. Parse the test results.
3. For each acceptance criterion in the spec, determine:
   - PASS: A test explicitly verifies this criterion and passes
   - FAIL: A test explicitly verifies this criterion and fails
   - UNTESTED: No test covers this criterion
4. Check code coverage meets the spec threshold (${spec.constraints.required_test_coverage}).
5. Write a structured report to test-report.json:

{
  "spec_id": "${spec.id}",
  "timestamp": "<ISO 8601>",
  "overall_status": "pass | fail",
  "test_results": { "total": N, "passed": N, "failed": N, "skipped": N },
  "coverage": { "statements": N, "branches": N, "functions": N, "lines": N },
  "acceptance_criteria": [
    { "criterion": "...", "status": "pass | fail | untested", "test_file": "...", "details": "..." }
  ],
  "blocking_issues": ["..."]
}

6. If any criterion is FAIL or UNTESTED, list specific remediation steps.`;
```

**Agent config for test stage:**

```typescript
const testConfig: SDKOptions = {
  maxTurns: 20,
  allowedTools: ["Read", "Glob", "Grep", "Bash", "Write"], // Bash needed for running tests
  permissionMode: "default",
  model: "claude-sonnet-4-6",
};
```

### Stage 5: Deploy Agent

The deploy agent creates a PR via GitHub MCP, triggers a preview deployment via Vercel MCP, and reports the results. This stage requires write access to external systems and is the highest-risk stage — it must be gated by human approval of the test report.

**Prompt template:**

```typescript
const deployPrompt = `You are a deploy agent.

## Input
Task spec: docs/specs/${spec.id}.json
Test report: test-report.json (must show overall_status: "pass")

## Pre-conditions
- Verify test-report.json exists and shows overall_status: "pass"
- If tests did not pass, STOP and report the failure. Do not proceed.

## Instructions
1. Create a feature branch: feat/${spec.id}
2. Stage all changed files (only files listed in spec.outputs)
3. Commit with message: "feat(${spec.metadata.labels[0]}): ${spec.title}"
4. Push the branch
5. Create a PR using GitHub MCP:
   - Title: ${spec.title}
   - Body: Include spec ID, acceptance criteria results from test-report.json, coverage numbers
   - Labels: ${spec.metadata.labels.join(", ")}
   - Link to issue: ${spec.metadata.issue_url ?? "N/A"}
6. Trigger a preview deployment via Vercel MCP
7. Add a PR comment with the preview URL
8. Report: PR URL, preview URL, deployment status`;
```

**Agent config for deploy stage:**

```typescript
const deployConfig: SDKOptions = {
  maxTurns: 15,
  allowedTools: [
    "Read", "Bash",
    "mcp__github__create_pull_request",
    "mcp__github__create_or_update_file",
    "mcp__github__push_files",
    "mcp__github__add_issue_comment",
    "mcp__vercel__create_deployment",
  ],
  permissionMode: "default",
  model: "claude-haiku-4-5", // Haiku is sufficient for mechanical deploy tasks
};
```

---

## 3. Seven Named Pipeline Patterns

### Pattern 1: Sequential Pipeline

```
  ┌───────┐     ┌───────┐     ┌───────┐
  │ Agent │────>│ Agent │────>│ Agent │
  │   A   │     │   B   │     │   C   │
  └───────┘     └───────┘     └───────┘
  analyze        transform     validate
```

**When to use:** Tasks where each stage depends on the complete output of the previous stage. Requirements-first workflows. Data transformation chains. Any workflow where the stages are inherently sequential and cannot be parallelized.

**Key property:** Each agent reads the previous agent's output file. Context does not carry between agents — only the artifact (file) does. This means each agent gets a fresh context window.

```typescript
// src/pipeline/patterns/sequential.ts

import { query, type SDKMessage } from "@anthropic-ai/claude-code";

interface StageConfig {
  name: string;
  prompt: (previousOutput: string) => string;
  outputFile: string;
  maxTurns: number;
  allowedTools: string[];
  model: "claude-opus-4-6" | "claude-sonnet-4-6" | "claude-haiku-4-5";
}

async function runSequentialPipeline(
  stages: StageConfig[],
  initialInput: string,
  workDir: string,
): Promise<Map<string, string>> {
  const results = new Map<string, string>();
  let previousOutput = initialInput;

  for (const stage of stages) {
    console.log(`[pipeline] Starting stage: ${stage.name}`);
    const startTime = Date.now();

    const output: string[] = [];

    for await (const msg of query({
      prompt: stage.prompt(previousOutput),
      options: {
        maxTurns: stage.maxTurns,
        allowedTools: stage.allowedTools,
        model: stage.model,
        cwd: workDir,
        permissionMode: "acceptEdits",
      },
    })) {
      if (msg.type === "assistant") {
        const text = msg.message.content
          .filter((b: any) => b.type === "text")
          .map((b: any) => b.text)
          .join("");
        if (text) output.push(text);
      }
    }

    const elapsed = Date.now() - startTime;
    console.log(`[pipeline] Stage ${stage.name} completed in ${elapsed}ms`);

    previousOutput = output.join("\n");
    results.set(stage.name, previousOutput);
  }

  return results;
}

// Usage: analyze → transform → validate
const stages: StageConfig[] = [
  {
    name: "analyze",
    prompt: (input) => `Analyze this codebase issue and produce a structured analysis:\n\n${input}`,
    outputFile: "analysis.json",
    maxTurns: 15,
    allowedTools: ["Read", "Glob", "Grep"],
    model: "claude-sonnet-4-6",
  },
  {
    name: "transform",
    prompt: (analysis) => `Based on this analysis, implement the required changes:\n\n${analysis}`,
    outputFile: "changes.patch",
    maxTurns: 25,
    allowedTools: ["Read", "Glob", "Grep", "Write", "Edit"],
    model: "claude-sonnet-4-6",
  },
  {
    name: "validate",
    prompt: (changes) => `Validate these changes pass all tests and meet requirements:\n\n${changes}`,
    outputFile: "validation.json",
    maxTurns: 10,
    allowedTools: ["Read", "Bash"],
    model: "claude-haiku-4-5",
  },
];
```

### Pattern 2: Fan-Out / Fan-In

```
                    ┌──────────┐
               ┌───>│ Worker 1 │───┐
  ┌────────┐   │    └──────────┘   │    ┌────────────┐
  │ Orch.  │───┤    ┌──────────┐   ├───>│ Aggregator │
  │        │   ├───>│ Worker 2 │───┤    │            │
  └────────┘   │    └──────────┘   │    └────────────┘
               │    ┌──────────┐   │
               └───>│ Worker 3 │───┘
                    └──────────┘
```

**When to use:** Independent subtasks that can be parallelized. Processing multiple files or modules that don't depend on each other. Running the same analysis across different parts of a codebase. The orchestrator splits work; workers execute in parallel; the aggregator combines results.

**Key property:** Workers must have isolated working directories. No shared mutable state. The aggregator is a separate agent (or function) that merges worker outputs.

```typescript
// src/pipeline/patterns/fan-out-fan-in.ts

import { query, type SDKMessage } from "@anthropic-ai/claude-code";

interface WorkUnit {
  id: string;
  files: string[];
  instruction: string;
}

interface WorkResult {
  id: string;
  output: string;
  success: boolean;
  durationMs: number;
}

async function fanOutFanIn(
  workUnits: WorkUnit[],
  aggregatorPrompt: (results: WorkResult[]) => string,
  workDir: string,
): Promise<string> {
  // Fan-out: run all workers in parallel
  const workerResults = await Promise.all(
    workUnits.map(async (unit): Promise<WorkResult> => {
      const start = Date.now();
      const output: string[] = [];
      let success = true;

      try {
        for await (const msg of query({
          prompt: `Process these files: ${unit.files.join(", ")}\n\n${unit.instruction}`,
          options: {
            maxTurns: 15,
            allowedTools: ["Read", "Glob", "Grep", "Write", "Edit"],
            model: "claude-sonnet-4-6",
            cwd: `${workDir}/worker-${unit.id}`,
            permissionMode: "acceptEdits",
          },
        })) {
          if (msg.type === "assistant") {
            const text = msg.message.content
              .filter((b: any) => b.type === "text")
              .map((b: any) => b.text)
              .join("");
            if (text) output.push(text);
          }
          if (msg.type === "result" && msg.subtype?.startsWith("error")) {
            success = false;
          }
        }
      } catch {
        success = false;
      }

      return {
        id: unit.id,
        output: output.join("\n"),
        success,
        durationMs: Date.now() - start,
      };
    }),
  );

  // Fan-in: aggregate results
  const aggregated: string[] = [];
  for await (const msg of query({
    prompt: aggregatorPrompt(workerResults),
    options: {
      maxTurns: 10,
      allowedTools: ["Read", "Write"],
      model: "claude-sonnet-4-6",
      cwd: workDir,
      permissionMode: "acceptEdits",
    },
  })) {
    if (msg.type === "assistant") {
      const text = msg.message.content
        .filter((b: any) => b.type === "text")
        .map((b: any) => b.text)
        .join("");
      if (text) aggregated.push(text);
    }
  }

  return aggregated.join("\n");
}
```

### Pattern 3: Hierarchical Delegation

```
                    ┌──────────────┐
                    │  Lead Agent  │
                    │ (strategy)   │
                    └──────┬───────┘
               ┌───────────┼───────────┐
               ▼           ▼           ▼
        ┌────────────┐ ┌────────────┐ ┌────────────┐
        │ Sub-Orch 1 │ │ Sub-Orch 2 │ │ Sub-Orch 3 │
        │ (frontend) │ │ (backend)  │ │ (infra)    │
        └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
          ┌───┴───┐     ┌───┴───┐     ┌───┴───┐
          ▼       ▼     ▼       ▼     ▼       ▼
        ┌───┐ ┌───┐  ┌───┐ ┌───┐  ┌───┐ ┌───┐
        │W1 │ │W2 │  │W3 │ │W4 │  │W5 │ │W6 │
        └───┘ └───┘  └───┘ └───┘  └───┘ └───┘
```

**When to use:** Organization-scale tasks that span multiple domains (frontend, backend, infrastructure). The lead agent sets strategy and delegates domains to sub-orchestrators. Sub-orchestrators manage workers within their domain. This creates a coordination hierarchy that mirrors how engineering teams work.

**Key property:** The lead agent's context contains only coordination state — it never holds the full implementation details. Each sub-orchestrator is a complete orchestrator within its domain. Workers at the leaf level do focused implementation.

```typescript
// src/pipeline/patterns/hierarchical.ts

import { query } from "@anthropic-ai/claude-code";

interface DomainConfig {
  name: string;
  description: string;
  workerSpecs: { id: string; files: string[]; task: string }[];
}

async function hierarchicalDelegation(
  taskDescription: string,
  domains: DomainConfig[],
  workDir: string,
): Promise<Map<string, string>> {
  // Lead agent: produce domain-level work breakdown
  const leadOutput: string[] = [];
  for await (const msg of query({
    prompt: `You are the lead architect for: ${taskDescription}

Domains assigned to sub-teams:
${domains.map((d) => `- ${d.name}: ${d.description}`).join("\n")}

Produce a coordination plan:
1. Shared interfaces/types that must be agreed upon first
2. Per-domain execution order (which domain blocks which)
3. Integration points between domains
4. Cross-domain acceptance criteria

Output as structured JSON under "coordination-plan.json".`,
    options: {
      maxTurns: 10,
      allowedTools: ["Read", "Glob", "Grep", "Write"],
      model: "claude-opus-4-6", // Opus for strategic planning
      cwd: workDir,
    },
  })) {
    if (msg.type === "assistant") {
      const text = msg.message.content
        .filter((b: any) => b.type === "text")
        .map((b: any) => b.text)
        .join("");
      if (text) leadOutput.push(text);
    }
  }

  // Sub-orchestrators: each manages its domain's workers
  const domainResults = await Promise.all(
    domains.map(async (domain) => {
      // Sub-orchestrator plans within its domain
      const subOrchOutput: string[] = [];
      for await (const msg of query({
        prompt: `You are the ${domain.name} sub-orchestrator.
Domain: ${domain.description}
Coordination plan: Read coordination-plan.json for cross-domain constraints.

Plan the implementation for your ${domain.workerSpecs.length} workers:
${domain.workerSpecs.map((w) => `- Worker ${w.id}: ${w.task} (files: ${w.files.join(", ")})`).join("\n")}

Produce per-worker instructions that respect the coordination plan.`,
        options: {
          maxTurns: 10,
          allowedTools: ["Read", "Glob", "Grep", "Write"],
          model: "claude-sonnet-4-6",
          cwd: workDir,
        },
      })) {
        if (msg.type === "assistant") {
          const text = msg.message.content
            .filter((b: any) => b.type === "text")
            .map((b: any) => b.text)
            .join("");
          if (text) subOrchOutput.push(text);
        }
      }

      // Workers: execute in parallel within the domain
      const workerResults = await Promise.all(
        domain.workerSpecs.map(async (worker) => {
          const output: string[] = [];
          for await (const msg of query({
            prompt: `You are worker ${worker.id} in the ${domain.name} domain.
Task: ${worker.task}
Files: ${worker.files.join(", ")}
Read coordination-plan.json for constraints.
Implement your assigned task.`,
            options: {
              maxTurns: 20,
              allowedTools: ["Read", "Glob", "Grep", "Write", "Edit"],
              model: "claude-sonnet-4-6",
              cwd: `${workDir}/worker-${worker.id}`,
              permissionMode: "acceptEdits",
            },
          })) {
            if (msg.type === "assistant") {
              const text = msg.message.content
                .filter((b: any) => b.type === "text")
                .map((b: any) => b.text)
                .join("");
              if (text) output.push(text);
            }
          }
          return { id: worker.id, output: output.join("\n") };
        }),
      );

      return { domain: domain.name, workers: workerResults };
    }),
  );

  const results = new Map<string, string>();
  for (const dr of domainResults) {
    for (const wr of dr.workers) {
      results.set(`${dr.domain}/${wr.id}`, wr.output);
    }
  }
  return results;
}
```

### Pattern 4: Critic / Reviewer Loop

```
  ┌──────────┐     ┌──────────┐
  │  Worker   │────>│  Critic  │
  │ (writes)  │     │(reviews) │
  └─────▲─────┘     └────┬─────┘
        │                 │
        │   feedback      │ pass/fail
        └─────────────────┘
              max N rounds
```

**When to use:** Code quality enforcement. Security review. Any task where iterative refinement based on structured feedback produces better output than a single pass. The critic agent reviews the worker's output against criteria and returns structured feedback. The worker revises. The loop continues until the critic passes all criteria or N rounds are exhausted.

**Key property:** The critic must have different (usually stricter) criteria than the worker's own judgment. Using the same model for both worker and critic is fine — the separation of roles (generate vs evaluate) produces different behavior even with the same model.

```typescript
// src/pipeline/patterns/critic-loop.ts

import { query } from "@anthropic-ai/claude-code";

interface CriticFeedback {
  pass: boolean;
  issues: { severity: "critical" | "major" | "minor"; description: string; file?: string; line?: number }[];
  summary: string;
}

async function criticReviewerLoop(
  task: string,
  criteria: string[],
  workDir: string,
  maxRounds: number = 3,
): Promise<{ finalOutput: string; rounds: number; feedback: CriticFeedback[] }> {
  const allFeedback: CriticFeedback[] = [];
  let workerOutput = "";
  let previousFeedback = "";

  for (let round = 1; round <= maxRounds; round++) {
    console.log(`[critic-loop] Round ${round}/${maxRounds}`);

    // Worker: implement (or revise)
    const workerParts: string[] = [];
    const workerPrompt = round === 1
      ? `Implement the following task:\n${task}`
      : `Revise your implementation based on this critic feedback:\n${previousFeedback}\n\nOriginal task:\n${task}`;

    for await (const msg of query({
      prompt: workerPrompt,
      options: {
        maxTurns: 20,
        allowedTools: ["Read", "Glob", "Grep", "Write", "Edit", "Bash"],
        model: "claude-sonnet-4-6",
        cwd: workDir,
        permissionMode: "acceptEdits",
      },
    })) {
      if (msg.type === "assistant") {
        const text = msg.message.content
          .filter((b: any) => b.type === "text")
          .map((b: any) => b.text)
          .join("");
        if (text) workerParts.push(text);
      }
    }
    workerOutput = workerParts.join("\n");

    // Critic: review against criteria
    const criticParts: string[] = [];
    for await (const msg of query({
      prompt: `You are a code reviewer. Review the implementation against these criteria:
${criteria.map((c, i) => `${i + 1}. ${c}`).join("\n")}

The worker just completed round ${round}. Review all changed files in the working directory.

Respond with ONLY a JSON object matching this schema:
{
  "pass": boolean,
  "issues": [{ "severity": "critical|major|minor", "description": "...", "file": "...", "line": N }],
  "summary": "..."
}

Be strict. "pass" is true ONLY if zero critical and zero major issues remain.`,
      options: {
        maxTurns: 10,
        allowedTools: ["Read", "Glob", "Grep"],
        model: "claude-sonnet-4-6",
        cwd: workDir,
      },
    })) {
      if (msg.type === "assistant") {
        const text = msg.message.content
          .filter((b: any) => b.type === "text")
          .map((b: any) => b.text)
          .join("");
        if (text) criticParts.push(text);
      }
    }

    // Parse critic feedback (extract JSON from response)
    const criticText = criticParts.join("\n");
    const jsonMatch = criticText.match(/\{[\s\S]*\}/);
    if (!jsonMatch) {
      throw new Error(`Critic did not return valid JSON in round ${round}`);
    }

    const feedback: CriticFeedback = JSON.parse(jsonMatch[0]);
    allFeedback.push(feedback);

    if (feedback.pass) {
      console.log(`[critic-loop] Passed in round ${round}`);
      return { finalOutput: workerOutput, rounds: round, feedback: allFeedback };
    }

    previousFeedback = JSON.stringify(feedback, null, 2);
  }

  console.warn(`[critic-loop] Did not pass after ${maxRounds} rounds — escalating to human`);
  return { finalOutput: workerOutput, rounds: maxRounds, feedback: allFeedback };
}
```

### Pattern 5: Debate / Consensus

```
  ┌───────────┐            ┌───────────┐
  │  Agent A  │            │  Agent B  │
  │ (proposal │◄──────────►│ (proposal │
  │   alpha)  │            │   beta)   │
  └─────┬─────┘            └─────┬─────┘
        │                        │
        ▼                        ▼
  ┌──────────────────────────────────────┐
  │              Arbiter                  │
  │  (compares proposals, picks winner   │
  │   or synthesizes best of both)       │
  └──────────────────────────────────────┘
```

**When to use:** Architectural decisions where multiple valid approaches exist. Design reviews. API design where two patterns are both reasonable. The debate pattern reduces single-agent bias by forcing two independent proposals and a structured comparison.

**Key property:** Agents A and B must not see each other's output until the arbiter phase. Independent proposals prevent anchoring bias. The arbiter sees both proposals simultaneously and picks based on explicit criteria.

```typescript
// src/pipeline/patterns/debate.ts

import { query } from "@anthropic-ai/claude-code";

interface Proposal {
  agent: string;
  approach: string;
  tradeoffs: string;
  output: string;
}

async function debateConsensus(
  problem: string,
  evaluationCriteria: string[],
  workDir: string,
): Promise<{ winner: "A" | "B" | "synthesis"; rationale: string }> {
  // Run agents A and B in parallel — they cannot see each other
  const [proposalA, proposalB] = await Promise.all([
    runProposalAgent("A", problem, workDir),
    runProposalAgent("B", problem, workDir),
  ]);

  // Arbiter: compare and decide
  const arbiterParts: string[] = [];
  for await (const msg of query({
    prompt: `You are an arbiter comparing two proposals for: ${problem}

## Evaluation Criteria
${evaluationCriteria.map((c, i) => `${i + 1}. ${c}`).join("\n")}

## Proposal A
${proposalA.output}

## Proposal B
${proposalB.output}

## Instructions
1. Score each proposal against each criterion (1-10).
2. Identify the strengths and weaknesses of each.
3. Decide: pick A, pick B, or synthesize the best elements of both.
4. Respond with JSON: { "winner": "A" | "B" | "synthesis", "rationale": "...", "scores": { "A": {...}, "B": {...} } }`,
    options: {
      maxTurns: 10,
      allowedTools: ["Read"],
      model: "claude-opus-4-6", // Opus for judgment tasks
      cwd: workDir,
    },
  })) {
    if (msg.type === "assistant") {
      const text = msg.message.content
        .filter((b: any) => b.type === "text")
        .map((b: any) => b.text)
        .join("");
      if (text) arbiterParts.push(text);
    }
  }

  const arbiterText = arbiterParts.join("\n");
  const jsonMatch = arbiterText.match(/\{[\s\S]*\}/);
  return JSON.parse(jsonMatch![0]);
}

async function runProposalAgent(
  agentId: string,
  problem: string,
  workDir: string,
): Promise<Proposal> {
  const parts: string[] = [];
  for await (const msg of query({
    prompt: `You are proposal agent ${agentId}. Propose a solution for:
${problem}

Structure your response:
1. Approach: describe your design in detail
2. Tradeoffs: what are the pros and cons of your approach
3. Implementation sketch: key code structures, not full implementation`,
    options: {
      maxTurns: 15,
      allowedTools: ["Read", "Glob", "Grep"],
      model: "claude-sonnet-4-6",
      cwd: workDir,
    },
  })) {
    if (msg.type === "assistant") {
      const text = msg.message.content
        .filter((b: any) => b.type === "text")
        .map((b: any) => b.text)
        .join("");
      if (text) parts.push(text);
    }
  }

  return { agent: agentId, approach: "", tradeoffs: "", output: parts.join("\n") };
}
```

### Pattern 6: Incremental Refinement

```
  ┌───────┐     ┌──────┐     ┌───────┐     ┌──────┐     ┌───────┐     ┌──────┐
  │ Agent │────>│ Eval │──X──│ Agent │────>│ Eval │──X──│ Agent │────>│ Eval │──✓
  │ (v1)  │     │      │fail │ (v2)  │     │      │fail │ (v3)  │     │      │pass
  └───────┘     └──────┘     └───────┘     └──────┘     └───────┘     └──────┘
                  │                          │
                  └── feedback ──┘           └── feedback ──┘
```

**When to use:** Tasks where first-pass output is unlikely to meet all criteria. Code generation where compilation or test failures provide actionable feedback. Any iterative improvement loop where the eval function provides structured, machine-readable feedback that the agent can act on.

**Key property:** The eval function must produce structured feedback — not just pass/fail, but what specifically failed and how to fix it. The agent receives the eval output as context for the next attempt.

```typescript
// src/pipeline/patterns/incremental-refinement.ts

import { query } from "@anthropic-ai/claude-code";

interface EvalResult {
  pass: boolean;
  score: number; // 0.0-1.0
  failures: { test: string; error: string; suggestion: string }[];
}

async function incrementalRefinement(
  task: string,
  evalFn: (workDir: string) => Promise<EvalResult>,
  workDir: string,
  maxAttempts: number = 5,
  passThreshold: number = 1.0,
): Promise<{ finalScore: number; attempts: number; passed: boolean }> {
  let feedback = "";
  let bestScore = 0;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    console.log(`[refinement] Attempt ${attempt}/${maxAttempts}`);

    // Agent: implement or refine
    const prompt = attempt === 1
      ? `Implement: ${task}`
      : `Your previous attempt scored ${bestScore.toFixed(2)}. Fix these failures:\n${feedback}\n\nOriginal task: ${task}`;

    for await (const msg of query({
      prompt,
      options: {
        maxTurns: 20,
        allowedTools: ["Read", "Glob", "Grep", "Write", "Edit", "Bash"],
        model: "claude-sonnet-4-6",
        cwd: workDir,
        permissionMode: "acceptEdits",
      },
    })) {
      // consume stream
    }

    // Eval: score the output
    const evalResult = await evalFn(workDir);
    bestScore = Math.max(bestScore, evalResult.score);

    if (evalResult.pass || evalResult.score >= passThreshold) {
      return { finalScore: evalResult.score, attempts: attempt, passed: true };
    }

    // Structured feedback for next attempt
    feedback = evalResult.failures
      .map((f) => `FAIL: ${f.test}\n  Error: ${f.error}\n  Fix: ${f.suggestion}`)
      .join("\n\n");
  }

  return { finalScore: bestScore, attempts: maxAttempts, passed: false };
}
```

### Pattern 7: Watchdog

```
  ┌──────────┐                    ┌──────────┐
  │  Worker   │──── runs ────────>│  Output  │
  │  Agent    │                    │          │
  └─────▲─────┘                    └──────────┘
        │ intervene
  ┌─────┴──────┐
  │  Watchdog  │◄─── monitors worker messages
  │  Agent     │     detects: error loops,
  │            │     hallucination, scope creep,
  └────────────┘     repeated failures
```

**When to use:** Long-running agents that might get stuck in error loops, hallucinate file paths, repeatedly try the same failing approach, or drift from the task scope. The watchdog is a lightweight agent (or function) that monitors the worker's message stream and intervenes when it detects pathological patterns.

**Key property:** The watchdog is passive — it observes the worker's output stream, not the worker's context. It does not share the worker's context window. Intervention means aborting the worker and restarting with corrective instructions, not modifying the worker's ongoing conversation.

```typescript
// src/pipeline/patterns/watchdog.ts

import { query, type SDKMessage } from "@anthropic-ai/claude-code";

interface WatchdogConfig {
  /** Max consecutive tool errors before intervention */
  maxConsecutiveErrors: number;
  /** Max times the same file can be edited (suggests thrashing) */
  maxSameFileEdits: number;
  /** Max turns without producing any file changes */
  maxIdleTurns: number;
  /** Patterns that indicate hallucination */
  halluccinationPatterns: RegExp[];
}

const DEFAULT_WATCHDOG_CONFIG: WatchdogConfig = {
  maxConsecutiveErrors: 5,
  maxSameFileEdits: 8,
  maxIdleTurns: 10,
  halluccinationPatterns: [
    /File not found:.*(?:does not exist|No such file)/,
    /Error: ENOENT/,
  ],
};

interface WatchdogState {
  consecutiveErrors: number;
  fileEditCounts: Map<string, number>;
  turnsSinceLastEdit: number;
  interventionReason: string | null;
}

function createWatchdog(config: WatchdogConfig = DEFAULT_WATCHDOG_CONFIG) {
  const state: WatchdogState = {
    consecutiveErrors: 0,
    fileEditCounts: new Map(),
    turnsSinceLastEdit: 0,
    interventionReason: null,
  };

  function observe(msg: SDKMessage): boolean {
    // Check for tool errors
    if (msg.type === "tool_result") {
      const content = typeof msg.content === "string" ? msg.content : JSON.stringify(msg.content);
      const isError = msg.is_error || content.includes("Error") || content.includes("ENOENT");

      if (isError) {
        state.consecutiveErrors++;
        if (state.consecutiveErrors >= config.maxConsecutiveErrors) {
          state.interventionReason = `${state.consecutiveErrors} consecutive tool errors — likely stuck in error loop`;
          return true; // intervene
        }

        // Check hallucination patterns
        for (const pattern of config.halluccinationPatterns) {
          if (pattern.test(content)) {
            state.interventionReason = `Hallucination detected: accessing non-existent resources`;
            return true;
          }
        }
      } else {
        state.consecutiveErrors = 0;
      }
    }

    // Track file edits
    if (msg.type === "tool_use" && (msg.name === "Edit" || msg.name === "Write")) {
      const filePath = msg.input?.file_path;
      if (filePath) {
        const count = (state.fileEditCounts.get(filePath) ?? 0) + 1;
        state.fileEditCounts.set(filePath, count);
        state.turnsSinceLastEdit = 0;

        if (count >= config.maxSameFileEdits) {
          state.interventionReason = `File ${filePath} edited ${count} times — likely thrashing`;
          return true;
        }
      }
    } else if (msg.type === "assistant") {
      state.turnsSinceLastEdit++;
      if (state.turnsSinceLastEdit >= config.maxIdleTurns) {
        state.interventionReason = `${state.turnsSinceLastEdit} turns without file changes — likely stuck`;
        return true;
      }
    }

    return false; // no intervention needed
  }

  return { observe, state };
}

async function runWithWatchdog(
  task: string,
  workDir: string,
  watchdogConfig?: WatchdogConfig,
  maxRestarts: number = 2,
): Promise<{ output: string; restarts: number; watchdogInterventions: string[] }> {
  const interventions: string[] = [];
  let output = "";

  for (let attempt = 0; attempt <= maxRestarts; attempt++) {
    const watchdog = createWatchdog(watchdogConfig);
    const controller = new AbortController();
    const parts: string[] = [];

    const correctionContext = interventions.length > 0
      ? `\n\nPREVIOUS ATTEMPT FAILED. Watchdog intervention: ${interventions[interventions.length - 1]}\nAvoid the same mistake. Try a different approach.`
      : "";

    try {
      for await (const msg of query({
        prompt: `${task}${correctionContext}`,
        abortController: controller,
        options: {
          maxTurns: 30,
          allowedTools: ["Read", "Glob", "Grep", "Write", "Edit", "Bash"],
          model: "claude-sonnet-4-6",
          cwd: workDir,
          permissionMode: "acceptEdits",
        },
      })) {
        // Watchdog observes every message
        const shouldIntervene = watchdog.observe(msg);

        if (shouldIntervene) {
          console.warn(`[watchdog] Intervening: ${watchdog.state.interventionReason}`);
          interventions.push(watchdog.state.interventionReason!);
          controller.abort();
          break;
        }

        if (msg.type === "assistant") {
          const text = msg.message.content
            .filter((b: any) => b.type === "text")
            .map((b: any) => b.text)
            .join("");
          if (text) parts.push(text);
        }
      }

      output = parts.join("\n");

      // If we got here without intervention, we're done
      if (!watchdog.state.interventionReason) {
        return { output, restarts: attempt, watchdogInterventions: interventions };
      }
    } catch {
      // AbortError from watchdog intervention — continue to next attempt
    }
  }

  return { output, restarts: maxRestarts, watchdogInterventions: interventions };
}
```

---

## 4. Long-Running Agent Management

Production pipelines run for minutes to hours. Agents hit timeouts, get interrupted by deploys, encounter transient network failures, and run out of context window. Long-running agent management is about designing for these realities — not hoping they won't happen.

### AgentCheckpoint Interface

A checkpoint captures the complete resumable state of a pipeline at a point in time. If the process dies, the pipeline can resume from the last checkpoint without re-doing completed work.

```typescript
// src/pipeline/checkpoint.ts

import { writeFile, readFile, mkdir } from "fs/promises";
import { join } from "path";

export interface AgentCheckpoint {
  /** Unique identifier for this pipeline run */
  task_id: string;

  /** Current stage in the pipeline (e.g., "implement", "test", "deploy") */
  stage: string;

  /** Steps that have been completed successfully */
  completed_steps: {
    step_id: string;
    completed_at: string;
    output_summary: string;
    artifacts: string[]; // file paths produced by this step
  }[];

  /** Steps that remain to be executed */
  pending_steps: {
    step_id: string;
    description: string;
    dependencies: string[]; // step_ids that must complete first
  }[];

  /** File artifacts produced so far (absolute paths) */
  artifacts: Record<string, string>;

  /** Number of errors encountered (for retry budget tracking) */
  error_count: number;

  /** Last error message, if any */
  last_error?: string;

  /** ISO timestamp of last checkpoint write */
  last_checkpoint_at: string;

  /** Total tokens consumed across all stages so far */
  total_tokens_used: number;

  /** Elapsed wall-clock time in ms since pipeline start */
  elapsed_ms: number;
}

const CHECKPOINT_DIR = ".pipeline/checkpoints";

export async function writeCheckpoint(
  workDir: string,
  checkpoint: AgentCheckpoint,
): Promise<void> {
  const dir = join(workDir, CHECKPOINT_DIR);
  await mkdir(dir, { recursive: true });
  const path = join(dir, `${checkpoint.task_id}.json`);
  await writeFile(path, JSON.stringify(checkpoint, null, 2));
  console.log(`[checkpoint] Written: ${path} (stage: ${checkpoint.stage}, ${checkpoint.completed_steps.length} steps done)`);
}

export async function readCheckpoint(
  workDir: string,
  taskId: string,
): Promise<AgentCheckpoint | null> {
  try {
    const path = join(workDir, CHECKPOINT_DIR, `${taskId}.json`);
    const raw = await readFile(path, "utf-8");
    return JSON.parse(raw) as AgentCheckpoint;
  } catch {
    return null; // no checkpoint found — fresh start
  }
}

/**
 * Builds a resume prompt that tells the agent what has been completed,
 * what remains, and what the last error was (if any).
 */
export function buildResumePrompt(
  checkpoint: AgentCheckpoint,
  originalTask: string,
): string {
  const completedSummary = checkpoint.completed_steps
    .map((s) => `- [DONE] ${s.step_id}: ${s.output_summary}`)
    .join("\n");

  const pendingSummary = checkpoint.pending_steps
    .map((s) => `- [TODO] ${s.step_id}: ${s.description}`)
    .join("\n");

  const errorContext = checkpoint.last_error
    ? `\n## Last Error (attempt to avoid repeating)\n${checkpoint.last_error}\n`
    : "";

  return `You are resuming a partially completed task.

## Original Task
${originalTask}

## Current Stage: ${checkpoint.stage}

## Completed Steps
${completedSummary || "None yet"}

## Pending Steps
${pendingSummary}

## Artifacts Produced So Far
${Object.entries(checkpoint.artifacts)
  .map(([name, path]) => `- ${name}: ${path}`)
  .join("\n") || "None"}
${errorContext}
## Instructions
1. Do NOT redo completed steps — their artifacts already exist.
2. Start from the first pending step.
3. If the last error is relevant, use a different approach.
4. Continue until all pending steps are done or you hit a blocker.`;
}
```

### Job Queue Integration with BullMQ

For production pipelines, use a proper job queue to manage agent runs. BullMQ provides persistence (Redis-backed), retry logic, rate limiting, and dead letter queues — all critical for reliable long-running agent management.

```typescript
// src/pipeline/queue.ts

import { Queue, Worker, Job } from "bullmq";
import { runFromSpec } from "./run-from-spec";
import { writeCheckpoint, readCheckpoint, buildResumePrompt } from "./checkpoint";
import type { TaskSpec } from "./types/task-spec";
import type { AgentCheckpoint } from "./checkpoint";

const REDIS_CONFIG = {
  host: process.env.REDIS_HOST ?? "localhost",
  port: parseInt(process.env.REDIS_PORT ?? "6379"),
};

// Queue: accepts task specs for processing
export const pipelineQueue = new Queue<{ specPath: string; workDir: string }>(
  "agent-pipeline",
  { connection: REDIS_CONFIG },
);

// Worker: processes specs from the queue
export const pipelineWorker = new Worker<{ specPath: string; workDir: string }>(
  "agent-pipeline",
  async (job: Job) => {
    const { specPath, workDir } = job.data;
    const taskId = job.id!;

    // Check for existing checkpoint (resume after crash)
    const existingCheckpoint = await readCheckpoint(workDir, taskId);
    if (existingCheckpoint && existingCheckpoint.pending_steps.length > 0) {
      console.log(`[queue] Resuming ${taskId} from checkpoint (${existingCheckpoint.completed_steps.length} steps done)`);
      // Resume logic: rebuild prompt with checkpoint context
    }

    // Report progress back to the queue
    await job.updateProgress({ stage: "starting", timestamp: new Date().toISOString() });

    try {
      const result = await runFromSpec(specPath);

      await job.updateProgress({
        stage: "complete",
        turns: result.turns,
        timestamp: new Date().toISOString(),
      });

      return { success: true, output: result.output, turns: result.turns };
    } catch (error) {
      // Write checkpoint so the job can resume on retry
      const checkpoint: AgentCheckpoint = {
        task_id: taskId,
        stage: "failed",
        completed_steps: [],
        pending_steps: [],
        artifacts: {},
        error_count: (existingCheckpoint?.error_count ?? 0) + 1,
        last_error: error instanceof Error ? error.message : String(error),
        last_checkpoint_at: new Date().toISOString(),
        total_tokens_used: 0,
        elapsed_ms: 0,
      };
      await writeCheckpoint(workDir, checkpoint);
      throw error; // BullMQ will retry based on queue config
    }
  },
  {
    connection: REDIS_CONFIG,
    concurrency: 3, // max 3 parallel agent runs
    limiter: {
      max: 10,
      duration: 60000, // max 10 jobs per minute (rate limiting)
    },
  },
);

// Enqueue a task
export async function enqueueTask(specPath: string, workDir: string): Promise<string> {
  const job = await pipelineQueue.add(
    "run-spec",
    { specPath, workDir },
    {
      attempts: 3,              // retry up to 3 times
      backoff: {
        type: "exponential",
        delay: 30000,           // 30s, 60s, 120s
      },
      removeOnComplete: { age: 86400 },  // keep completed jobs for 24h
      removeOnFail: { age: 604800 },     // keep failed jobs for 7 days
    },
  );
  return job.id!;
}
```

### Timeout Handling and Graceful Interruption

Agents must handle timeouts gracefully — writing a checkpoint before dying, not leaving files in a half-written state, and providing enough context for a resume.

```typescript
// src/pipeline/timeout.ts

import { query } from "@anthropic-ai/claude-code";
import { writeCheckpoint, type AgentCheckpoint } from "./checkpoint";

/**
 * Runs an agent with a hard timeout. On timeout, writes a checkpoint
 * and returns a partial result rather than crashing.
 */
async function runWithTimeout(
  prompt: string,
  workDir: string,
  timeoutMs: number,
  taskId: string,
): Promise<{ output: string; timedOut: boolean }> {
  const controller = new AbortController();
  const startTime = Date.now();
  const output: string[] = [];
  let timedOut = false;

  // Hard timeout: abort the agent
  const timer = setTimeout(() => {
    timedOut = true;
    controller.abort();
  }, timeoutMs);

  // Soft timeout: warn the agent 60s before hard timeout
  const softTimer = setTimeout(async () => {
    // The agent can't receive mid-stream messages, but we can log for observability
    console.warn(`[timeout] ${taskId}: ${timeoutMs - 60000}ms elapsed, 60s remaining`);
  }, timeoutMs - 60000);

  try {
    for await (const msg of query({
      prompt,
      abortController: controller,
      options: {
        maxTurns: 50,
        allowedTools: ["Read", "Glob", "Grep", "Write", "Edit", "Bash"],
        model: "claude-sonnet-4-6",
        cwd: workDir,
        permissionMode: "acceptEdits",
      },
    })) {
      if (msg.type === "assistant") {
        const text = msg.message.content
          .filter((b: any) => b.type === "text")
          .map((b: any) => b.text)
          .join("");
        if (text) output.push(text);
      }
    }
  } catch (err: any) {
    if (err.name === "AbortError" || timedOut) {
      console.warn(`[timeout] ${taskId}: Hard timeout after ${timeoutMs}ms`);

      // Write checkpoint so the task can resume
      const checkpoint: AgentCheckpoint = {
        task_id: taskId,
        stage: "timed_out",
        completed_steps: [], // would be populated from parsed output in production
        pending_steps: [],
        artifacts: {},
        error_count: 0,
        last_error: `Hard timeout after ${timeoutMs}ms`,
        last_checkpoint_at: new Date().toISOString(),
        total_tokens_used: 0,
        elapsed_ms: Date.now() - startTime,
      };
      await writeCheckpoint(workDir, checkpoint);
    } else {
      throw err;
    }
  } finally {
    clearTimeout(timer);
    clearTimeout(softTimer);
  }

  return { output: output.join("\n"), timedOut };
}
```

### Heartbeat Pattern

For truly long-running agents (10+ minutes), use a heartbeat file that external monitoring can check. If the heartbeat stops updating, the monitor knows the agent is hung.

```typescript
// src/pipeline/heartbeat.ts

import { writeFile, readFile } from "fs/promises";
import { join } from "path";

interface HeartbeatData {
  task_id: string;
  last_update: string;
  current_activity: string;
  turns_completed: number;
  files_modified: number;
}

export class Heartbeat {
  private intervalId: NodeJS.Timeout | null = null;
  private data: HeartbeatData;
  private filePath: string;

  constructor(taskId: string, workDir: string) {
    this.filePath = join(workDir, ".pipeline", `heartbeat-${taskId}.json`);
    this.data = {
      task_id: taskId,
      last_update: new Date().toISOString(),
      current_activity: "initializing",
      turns_completed: 0,
      files_modified: 0,
    };
  }

  /** Start emitting heartbeats every `intervalMs` (default: 10s) */
  start(intervalMs: number = 10_000): void {
    this.write(); // immediate first write
    this.intervalId = setInterval(() => this.write(), intervalMs);
  }

  /** Update heartbeat data from agent message stream */
  update(activity: string, turns?: number, files?: number): void {
    this.data.last_update = new Date().toISOString();
    this.data.current_activity = activity;
    if (turns !== undefined) this.data.turns_completed = turns;
    if (files !== undefined) this.data.files_modified = files;
  }

  /** Stop heartbeat emission */
  stop(): void {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  private async write(): Promise<void> {
    this.data.last_update = new Date().toISOString();
    await writeFile(this.filePath, JSON.stringify(this.data, null, 2));
  }
}

/**
 * External monitor: check if an agent is still alive.
 * Returns false if heartbeat is stale (older than threshold).
 */
export async function isAgentAlive(
  workDir: string,
  taskId: string,
  staleThresholdMs: number = 60_000,
): Promise<{ alive: boolean; lastSeen: string; activity: string }> {
  try {
    const path = join(workDir, ".pipeline", `heartbeat-${taskId}.json`);
    const raw = await readFile(path, "utf-8");
    const data: HeartbeatData = JSON.parse(raw);
    const age = Date.now() - new Date(data.last_update).getTime();

    return {
      alive: age < staleThresholdMs,
      lastSeen: data.last_update,
      activity: data.current_activity,
    };
  } catch {
    return { alive: false, lastSeen: "never", activity: "unknown" };
  }
}
```

---

## 5. Structured Handoffs Between Agents

Every inter-agent communication must be typed, validated, and versioned. Plain text handoffs between agents are undebuggable, unversionable, and fragile. Structured JSON with runtime validation is the standard.

### Zod Schema for Agent Output

```typescript
// src/pipeline/schemas/analysis-output.ts

import { z } from "zod";

export const AnalysisOutputSchema = z.object({
  /** One-paragraph summary of the analysis findings */
  summary: z.string().min(20).max(2000),

  /** Files that will be affected by the proposed changes */
  files_affected: z.array(
    z.object({
      path: z.string(),
      action: z.enum(["modify", "create", "delete", "rename"]),
      reason: z.string(),
    }),
  ),

  /** Overall risk assessment */
  risk_level: z.enum(["low", "medium", "high", "critical"]),

  /** Issues that must be resolved before proceeding */
  blocking_issues: z.array(
    z.object({
      id: z.string(),
      severity: z.enum(["critical", "major", "minor"]),
      description: z.string(),
      suggested_resolution: z.string().optional(),
    }),
  ),

  /** Non-blocking observations and recommendations */
  recommendations: z.array(z.string()).optional(),

  /** Estimated complexity (affects worker count and timeout) */
  estimated_complexity: z.enum(["trivial", "simple", "moderate", "complex", "very_complex"]),

  /** Dependencies on external systems or teams */
  external_dependencies: z.array(
    z.object({
      system: z.string(),
      dependency_type: z.enum(["blocking", "informational"]),
      description: z.string(),
    }),
  ).optional(),
});

export type AnalysisOutput = z.infer<typeof AnalysisOutputSchema>;
```

### Safe Parse Validation Before Handoff

Never pass raw text between agents. Always validate with `safeParse` and handle validation failures explicitly.

```typescript
// src/pipeline/handoff.ts

import { z } from "zod";
import { AnalysisOutputSchema, type AnalysisOutput } from "./schemas/analysis-output";

/**
 * Extracts and validates structured JSON from an agent's text output.
 * Returns the validated data or a structured error.
 */
export function parseAgentOutput<T>(
  rawOutput: string,
  schema: z.ZodType<T>,
  agentName: string,
): { success: true; data: T } | { success: false; error: string } {
  // Extract JSON from potentially mixed text/JSON output
  const jsonMatch = rawOutput.match(/```json\s*([\s\S]*?)```/) ?? rawOutput.match(/(\{[\s\S]*\})/);

  if (!jsonMatch) {
    return {
      success: false,
      error: `[handoff] ${agentName} did not produce JSON output. Raw output length: ${rawOutput.length} chars`,
    };
  }

  let parsed: unknown;
  try {
    parsed = JSON.parse(jsonMatch[1]);
  } catch (parseErr) {
    return {
      success: false,
      error: `[handoff] ${agentName} produced invalid JSON: ${parseErr}`,
    };
  }

  const result = schema.safeParse(parsed);

  if (!result.success) {
    const issues = result.error.issues
      .map((i) => `  - ${i.path.join(".")}: ${i.message}`)
      .join("\n");
    return {
      success: false,
      error: `[handoff] ${agentName} output failed schema validation:\n${issues}`,
    };
  }

  return { success: true, data: result.data };
}

// Usage in pipeline:
// const analysisResult = parseAgentOutput(agentOutput, AnalysisOutputSchema, "analysis-agent");
// if (!analysisResult.success) {
//   // Retry the agent with the validation error as feedback, or escalate
//   console.error(analysisResult.error);
// }
```

### Agent Output State Union

Every agent in a pipeline resolves to one of four states. Downstream agents must handle all four.

```typescript
// src/pipeline/types/agent-state.ts

/** Represents the possible terminal states of an agent run */
export type AgentOutputState<T> =
  | {
      status: "success";
      data: T;
      metadata: { turns: number; tokens_used: number; duration_ms: number };
    }
  | {
      status: "failure";
      error: string;
      /** Structured error code for programmatic handling */
      error_code: "max_turns" | "timeout" | "tool_error" | "validation_error" | "model_refusal" | "abort";
      /** Partial output produced before failure, if any */
      partial_data?: Partial<T>;
    }
  | {
      status: "partial";
      data: Partial<T>;
      /** What was completed and what remains */
      completed: string[];
      remaining: string[];
      /** Why the agent stopped before completion */
      reason: string;
    }
  | {
      status: "needs_human";
      /** Specific question or decision the agent needs help with */
      question: string;
      /** Context for the human reviewer */
      context: string;
      /** What the agent would do if it had to guess (helps human decide) */
      suggested_action?: string;
      /** Partial work completed so far */
      partial_data?: Partial<T>;
    };

/**
 * Type guard for checking agent output state.
 * Use in pipeline orchestrators to branch on state.
 */
export function isSuccess<T>(state: AgentOutputState<T>): state is Extract<AgentOutputState<T>, { status: "success" }> {
  return state.status === "success";
}

export function needsHuman<T>(state: AgentOutputState<T>): state is Extract<AgentOutputState<T>, { status: "needs_human" }> {
  return state.status === "needs_human";
}
```

### Versioned Handoff Schemas

When a pipeline runs for months, the handoff schema will evolve. Version the schema and include a migration path.

```typescript
// src/pipeline/schemas/versioned.ts

import { z } from "zod";

/**
 * Every handoff schema includes a version field.
 * The pipeline validates the version before parsing.
 */
export const VersionedEnvelope = z.object({
  schema_version: z.string().regex(/^\d+\.\d+\.\d+$/),
  produced_by: z.string(), // agent name that produced this
  produced_at: z.string().datetime(),
  payload: z.unknown(), // validated separately by version-specific schema
});

type SchemaRegistry = Map<string, z.ZodType<any>>;

const registry: SchemaRegistry = new Map();

/** Register a schema version */
export function registerSchema(version: string, schema: z.ZodType<any>): void {
  registry.set(version, schema);
}

/** Parse a versioned handoff, selecting the correct schema */
export function parseVersionedHandoff<T>(
  raw: string,
  expectedVersionPrefix: string,
): { success: true; data: T; version: string } | { success: false; error: string } {
  let envelope: z.infer<typeof VersionedEnvelope>;
  try {
    envelope = VersionedEnvelope.parse(JSON.parse(raw));
  } catch (err) {
    return { success: false, error: `Invalid envelope: ${err}` };
  }

  const schema = registry.get(envelope.schema_version);
  if (!schema) {
    return { success: false, error: `Unknown schema version: ${envelope.schema_version}. Known: ${[...registry.keys()].join(", ")}` };
  }

  if (!envelope.schema_version.startsWith(expectedVersionPrefix)) {
    return { success: false, error: `Version mismatch: expected ${expectedVersionPrefix}.x, got ${envelope.schema_version}` };
  }

  const result = schema.safeParse(envelope.payload);
  if (!result.success) {
    return { success: false, error: `Payload validation failed: ${result.error.message}` };
  }

  return { success: true, data: result.data, version: envelope.schema_version };
}
```

---

## 6. Human-in-the-Loop Design

Agents automate execution, not judgment. The question is never whether to include humans — it is where, when, and how. A pipeline with no human checkpoints is a pipeline that converts a misunderstood prompt into a production incident at maximum speed.

### Four Checkpoint Types

| Type | Purpose | Blocking? | Example |
|------|---------|-----------|---------|
| **Approval** | Human must explicitly approve before the pipeline proceeds | Yes | "Approve this deployment plan before I implement it" |
| **Review** | Human reviews output; pipeline pauses but can auto-proceed on timeout | Soft | "Review these code changes. Proceeding in 4 hours if no feedback." |
| **Escalation** | Agent is stuck or uncertain; needs human expertise | Yes | "I found conflicting requirements in the spec. Which interpretation is correct?" |
| **Information** | Agent needs a fact it cannot determine autonomously | Yes | "What is the production database connection string for the payments service?" |

### When to Interrupt vs Proceed Autonomously

**Always interrupt (require human approval):**
- Before deploying to any environment (staging, production)
- Before modifying database schemas in shared environments
- Before deleting files, branches, or resources that cannot be trivially recreated
- When the agent detects conflicting requirements or ambiguous acceptance criteria
- When estimated cost exceeds a configured threshold
- When the agent's confidence in its approach is low (self-assessed)

**Safe to proceed autonomously:**
- Reading files and analyzing code
- Running tests in an isolated environment
- Generating code that has not yet been committed
- Writing to temporary or staging files
- Producing analysis reports and recommendations
- Steps that are bounded by `maxTurns` and `allowedTools`

### Async Approval Pattern

For pipelines that run in the background (CI, scheduled jobs, headless agents), the approval pattern is: write a request, notify the human, poll for a response, and timeout gracefully.

```typescript
// src/pipeline/approval.ts

import { writeFile, readFile, unlink } from "fs/promises";
import { join } from "path";

interface ApprovalRequest {
  request_id: string;
  pipeline_id: string;
  stage: string;
  checkpoint_type: "approval" | "review" | "escalation" | "information";
  summary: string;
  details: string;
  requested_at: string;
  timeout_at: string;
  /** What happens if the human doesn't respond by timeout */
  timeout_action: "proceed" | "abort" | "retry";
}

interface ApprovalResponse {
  request_id: string;
  decision: "approve" | "reject" | "modify";
  feedback?: string;
  responded_at: string;
  responder: string;
}

const APPROVAL_DIR = ".pipeline/approvals";

export async function requestApproval(
  workDir: string,
  request: ApprovalRequest,
): Promise<void> {
  const dir = join(workDir, APPROVAL_DIR);
  const path = join(dir, `${request.request_id}.request.json`);
  await writeFile(path, JSON.stringify(request, null, 2));

  // Notify via configured channel (Slack, email, GitHub comment, etc.)
  await notifyHuman(request);
}

export async function pollForApproval(
  workDir: string,
  requestId: string,
  pollIntervalMs: number = 30_000,
  timeoutMs: number = 14_400_000, // 4 hours default
): Promise<ApprovalResponse | "timeout"> {
  const responsePath = join(workDir, APPROVAL_DIR, `${requestId}.response.json`);
  const startTime = Date.now();

  while (Date.now() - startTime < timeoutMs) {
    try {
      const raw = await readFile(responsePath, "utf-8");
      const response: ApprovalResponse = JSON.parse(raw);
      // Clean up
      await unlink(responsePath);
      return response;
    } catch {
      // No response yet — wait and retry
      await new Promise((resolve) => setTimeout(resolve, pollIntervalMs));
    }
  }

  return "timeout";
}

async function notifyHuman(request: ApprovalRequest): Promise<void> {
  // Implementation depends on your notification channel:
  // - Slack webhook: POST to a webhook URL with the request summary
  // - GitHub: Create a PR comment or issue comment
  // - Email: Send via SES/SendGrid
  // - Dashboard: Write to a database that the approval UI reads
  console.log(`[approval] Notification sent for ${request.request_id}: ${request.summary}`);
}

// Usage in pipeline:
//
// await requestApproval(workDir, {
//   request_id: "apr-001",
//   pipeline_id: spec.id,
//   stage: "pre-deploy",
//   checkpoint_type: "approval",
//   summary: "Deploy JWT refresh rotation to staging",
//   details: "12 files changed, all tests pass, coverage 94%",
//   requested_at: new Date().toISOString(),
//   timeout_at: new Date(Date.now() + 4 * 3600 * 1000).toISOString(),
//   timeout_action: "abort",
// });
//
// const response = await pollForApproval(workDir, "apr-001");
// if (response === "timeout") {
//   console.log("Approval timed out — aborting pipeline");
//   return;
// }
// if (response.decision === "reject") {
//   console.log(`Rejected by ${response.responder}: ${response.feedback}`);
//   return;
// }
// // Proceed with deploy...
```

### Time-Bounded Autonomy

The "proceed for N hours then checkpoint" pattern gives agents autonomy within a time window, then forces a human review regardless of progress. This balances speed (no constant interruptions) with oversight (guaranteed periodic review).

```typescript
// src/pipeline/time-bounded.ts

import { query } from "@anthropic-ai/claude-code";
import { requestApproval, pollForApproval } from "./approval";

interface TimeBoundedConfig {
  /** How long the agent can run autonomously before requiring a checkpoint */
  autonomy_window_ms: number;
  /** Maximum total runtime including human review time */
  total_timeout_ms: number;
  /** Maximum number of autonomy windows before escalation */
  max_windows: number;
}

async function runTimeBounded(
  task: string,
  workDir: string,
  config: TimeBoundedConfig,
): Promise<void> {
  let windowsUsed = 0;
  let cumulativeOutput = "";

  while (windowsUsed < config.max_windows) {
    windowsUsed++;

    // Run agent for one autonomy window
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), config.autonomy_window_ms);
    const parts: string[] = [];

    try {
      for await (const msg of query({
        prompt: windowsUsed === 1
          ? task
          : `Continue the task. Previous progress:\n${cumulativeOutput}\n\nOriginal task: ${task}`,
        abortController: controller,
        options: {
          maxTurns: 30,
          allowedTools: ["Read", "Glob", "Grep", "Write", "Edit", "Bash"],
          model: "claude-sonnet-4-6",
          cwd: workDir,
          permissionMode: "acceptEdits",
        },
      })) {
        if (msg.type === "assistant") {
          const text = msg.message.content
            .filter((b: any) => b.type === "text")
            .map((b: any) => b.text)
            .join("");
          if (text) parts.push(text);
        }

        // Check if agent signaled completion
        if (msg.type === "result" && !msg.subtype?.startsWith("error")) {
          clearTimeout(timer);
          console.log(`[time-bounded] Agent completed in window ${windowsUsed}`);
          return; // done
        }
      }
    } catch {
      // Timeout — expected when the autonomy window expires
    } finally {
      clearTimeout(timer);
    }

    cumulativeOutput += parts.join("\n");

    // Checkpoint: request human review
    console.log(`[time-bounded] Window ${windowsUsed}/${config.max_windows} expired. Requesting review.`);

    await requestApproval(workDir, {
      request_id: `tbr-${windowsUsed}`,
      pipeline_id: "time-bounded-run",
      stage: `window-${windowsUsed}`,
      checkpoint_type: "review",
      summary: `Agent ran for ${config.autonomy_window_ms / 60000} minutes. Review progress.`,
      details: cumulativeOutput.slice(-2000), // last 2000 chars of output
      requested_at: new Date().toISOString(),
      timeout_at: new Date(Date.now() + 4 * 3600 * 1000).toISOString(),
      timeout_action: "proceed", // auto-continue after 4h if no response
    });

    const response = await pollForApproval(workDir, `tbr-${windowsUsed}`);
    if (response !== "timeout" && response.decision === "reject") {
      console.log(`[time-bounded] Human rejected continuation: ${response.feedback}`);
      return;
    }
  }

  console.warn(`[time-bounded] Exhausted ${config.max_windows} windows without completion — escalating`);
}
```

### Escalation Paths

Define explicit escalation paths before you need them. The chain: agent detects uncertainty -> structured escalation request -> human reviews -> human overrides or provides guidance -> agent resumes -> audit trail records the override.

```
Agent detects uncertainty
        │
        ▼
Write escalation request (JSON)
        │
        ▼
Notify human (Slack / GitHub / email)
        │
        ▼
Human reviews context + agent's suggested action
        │
        ├── Approve suggestion → agent proceeds with suggested action
        │
        ├── Override → agent proceeds with human's alternative
        │
        └── Reject → pipeline aborts with human's rationale
        │
        ▼
Audit log records: who decided, what, when, why
```

Every escalation is logged: the agent's uncertainty, the human's decision, and the outcome. This audit trail is essential for compliance, post-incident review, and pipeline improvement.

---

## 7. Cost and Performance Optimization

### Token Cost by Pipeline Stage

Estimates based on typical SDLC pipeline runs (April 2026 pricing):

| Stage | Model | Avg Input Tokens | Avg Output Tokens | Est. Cost/Run |
|-------|-------|-----------------|-------------------|---------------|
| Requirements | claude-sonnet-4-6 | ~8,000 | ~3,000 | $0.06 |
| Architect | claude-sonnet-4-6 | ~25,000 | ~8,000 | $0.18 |
| Implement (per worker) | claude-sonnet-4-6 | ~40,000 | ~15,000 | $0.30 |
| Implement (3 workers) | claude-sonnet-4-6 | ~120,000 | ~45,000 | $0.90 |
| Test | claude-sonnet-4-6 | ~15,000 | ~5,000 | $0.10 |
| Deploy | claude-haiku-4-5 | ~5,000 | ~2,000 | $0.01 |
| **Full pipeline** | **mixed** | **~173,000** | **~63,000** | **~$1.25** |

These are averages. Complex tasks with multiple retry rounds, large codebases, or many files can cost 5-10x more. The critic loop pattern (Pattern 4) adds cost proportional to the number of rounds.

### Five Cost Optimization Principles

**1. Route by complexity — use the cheapest model that works.**

Not every stage needs Opus. Requirements analysis, test execution, and deployment are mechanical tasks where Sonnet or Haiku produces equivalent results at a fraction of the cost. Reserve Opus for judgment tasks: architectural decisions, ambiguity resolution, the arbiter role in debate patterns.

```typescript
// Model selection by task type
const MODEL_ROUTING: Record<string, string> = {
  "requirements-analysis": "claude-sonnet-4-6",    // analysis, structured output
  "architecture-planning":  "claude-sonnet-4-6",    // planning, reasoning
  "implementation":         "claude-sonnet-4-6",    // code generation
  "code-review":            "claude-sonnet-4-6",    // evaluation
  "test-execution":         "claude-haiku-4-5",     // mechanical test running
  "deployment":             "claude-haiku-4-5",     // mechanical deploy steps
  "strategic-decision":     "claude-opus-4-6",      // high-stakes judgment
  "arbiter-debate":         "claude-opus-4-6",      // comparison and selection
};
```

**2. Cache analysis outputs — don't re-analyze unchanged files.**

If the codebase hasn't changed since the last analysis, the analysis output is still valid. Cache analysis results keyed by file content hashes. Invalidate on change. This eliminates the most expensive stage (analysis + planning) on iterative runs.

```typescript
import { createHash } from "crypto";
import { readFile } from "fs/promises";

async function contentHash(filePaths: string[]): Promise<string> {
  const hash = createHash("sha256");
  for (const p of filePaths.sort()) {
    hash.update(await readFile(p));
  }
  return hash.digest("hex");
}

// Cache key: hash of all input files
// Cache value: analysis output JSON
// Invalidation: any input file changes → cache miss → re-analyze
```

**3. Scope context ruthlessly — every token in the prompt should earn its place.**

Do not include entire files when the agent only needs specific functions. Do not include the full git log when the agent only needs the last 5 commits. Do not include all reference docs when the agent only needs the auth section. Use `file:line` references and extract relevant sections rather than dumping full files into context.

**4. Set maxTurns based on task complexity — not a generous default.**

A deployment agent needs 10-15 turns. A complex implementation agent might need 30-40. Setting `maxTurns: 100` "just to be safe" means a confused agent can burn through 100 turns of expensive API calls before you discover it's stuck. Measure actual turn counts for each stage and set `maxTurns` to 1.5x the P95 observed value.

**5. Checkpoint and resume — don't restart from scratch on failure.**

A 30-minute pipeline that fails at minute 28 should not restart from minute 0. Checkpoint after each stage. On failure, resume from the last successful checkpoint. This cuts retry cost by the percentage of work already completed.

### Monthly Cost Formula

```
Monthly cost = (runs_per_day) * 30 * (avg_cost_per_run) * (1 + retry_rate)

Example:
- 20 pipeline runs per day (a medium team)
- $1.25 average cost per run
- 15% retry rate (some runs need 1 retry)
- Monthly: 20 * 30 * $1.25 * 1.15 = $862.50

With cost optimization applied:
- Model routing saves ~30% (Haiku for mechanical stages)
- Analysis caching saves ~20% on iterative runs
- Context scoping saves ~15% on token usage
- Optimized monthly: ~$475
```

### Parallelism vs Context Cost Tradeoffs

Parallelism (fan-out) trades latency for cost. Three parallel workers each consume their own context window — the total tokens used is roughly 3x a sequential approach. But the wall-clock time is roughly 1/3.

| Approach | Wall-Clock Time | Total Tokens | Cost | When to Choose |
|----------|----------------|--------------|------|----------------|
| Sequential (1 agent) | 3x | 1x | $ | Budget-constrained, low urgency |
| Parallel (3 workers) | 1x | ~2.5x | $$$ | Time-sensitive, budget available |
| Hybrid (2 workers) | 1.5x | ~1.8x | $$ | Balanced tradeoff |

The hybrid approach: parallelize only the stages that are truly independent. Keep sequential execution for stages with dependencies. Most real pipelines have a dependency graph that's partially parallelizable — find the critical path and parallelize only the non-critical branches.

Another tradeoff: parallel workers have smaller individual context windows (each sees only its assigned files), which can improve quality for focused tasks. A single agent trying to hold the entire codebase in context makes worse decisions than three workers each focused on their domain. Parallelism sometimes improves both speed AND quality — at the cost of higher total tokens.

---

## Appendix: Combining Patterns

Real pipelines combine multiple patterns. A production SDLC pipeline might use:

1. **Sequential** for the requirements -> architecture -> implementation -> test -> deploy flow
2. **Fan-out/Fan-in** within the implementation stage (parallel workers)
3. **Critic loop** within each worker (code review before merging worker output)
4. **Watchdog** monitoring each long-running worker for error loops
5. **Debate** for the architecture stage when multiple approaches are viable
6. **Incremental refinement** for the test stage (fix failing tests iteratively)
7. **Human-in-the-loop** gates between architecture and implementation, and between testing and deployment

The patterns are composable building blocks. The art is knowing which combinations serve your specific reliability, cost, and latency requirements. Start simple (sequential + human gates), add complexity only when measured problems justify it.
