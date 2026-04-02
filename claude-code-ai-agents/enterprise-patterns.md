# Enterprise Patterns for Agentic Systems

> Reference for enterprise-scale deployment of Claude Code, Agent SDK, MCP servers, and multi-agent orchestration. Covers governance, security, cost management, compliance, and operational patterns for teams of 10 to 10,000.

---

## 1. Team CLAUDE.md Governance

CLAUDE.md is infrastructure-as-code for AI behavior. In a team setting it functions as a shared contract and must be treated with the same rigor as application code.

### Ownership model

```
project/.claude/CLAUDE.md          ← Team contract (committed to git)
~/.claude/CLAUDE.md                ← Personal preferences (never committed)
project/.claude/skills/*/SKILL.md  ← Domain skills (committed to git)
```

**Team CLAUDE.md** defines conventions, constraints, and workflows that every team member (and every agent session) must follow. **Personal CLAUDE.md** holds individual preferences like verbosity, editor keybindings, or personal review checklists.

### Change management protocol

1. All CLAUDE.md changes require a pull request — no direct commits to main.
2. PR must include rationale: "What mistake does this prevent?" or "What behavior does this enforce?"
3. At least one reviewer from a different sub-team to prevent echo-chamber rules.
4. Maintain a change log at the bottom of CLAUDE.md or rely on git blame.
5. Review CLAUDE.md in sprint retrospectives: remove stale rules, add learnings from incidents.
6. Automated CI check: lint CLAUDE.md for length (recommended max: 100 lines), broken references, and contradictory rules.

### Authoring principles

| Principle | Why |
|-----------|-----|
| Every line answers "Would Claude make mistakes without this?" | Prevents rule bloat |
| Don't duplicate what hooks enforce | If a PreToolUse hook blocks `rm -rf /`, CLAUDE.md doesn't need to say it |
| Prefer examples over descriptions | `Use Result<T, E>` is clearer than "use functional error handling" |
| Specify negative constraints explicitly | "Never use `any`" is more reliable than "use strict types" |
| Keep it under 100 lines | Context window budget — leave room for skills and memory |

### Example: SaaS platform team CLAUDE.md

```markdown
# Project CLAUDE.md — Acme SaaS Platform

## Stack
- TypeScript strict, Node 22, Next.js 15, React 19
- PostgreSQL + Drizzle ORM, Redis for caching
- Auth.js v5 with JWT, RBAC via Casbin
- Vitest (unit), Playwright (E2E), MSW (API mocks)

## Conventions
- Server Components by default; 'use client' only for interactivity
- All API routes validate with Zod; return Result<T, AppError>
- DB access through repository layer: src/lib/db/repos/
- Feature flags via LaunchDarkly SDK — never hardcode toggles

## Forbidden
- Never use `any` — use `unknown` and narrow
- Never fetch in Client Components when Server Components suffice
- Never hardcode URLs — use src/lib/env.ts
- Never skip error.tsx for route segments
- Never commit .env files

## Workflow
- `npm run typecheck` after TS changes
- `npm test -- --related` for targeted tests
- Conventional Commits enforced by hook
```

---

## 2. Multi-Project Skill Management

Skills are the reuse mechanism for domain knowledge across projects and teams. At enterprise scale, skill management becomes a supply-chain problem.

### Skill distribution tiers

| Tier | Location | Scope | Version control |
|------|----------|-------|-----------------|
| Personal | `~/.claude/skills/` | Single developer | Not shared |
| Project | `<repo>/.claude/skills/` | Single repo, all contributors | Git (same repo) |
| Organization | Git submodule or symlink | Multiple repos | Git (dedicated repo) |
| Public | npm/GitHub package | Community | Semver releases |

### Organization-wide skill repository

```bash
# Dedicated repo: github.com/myorg/claude-skills
claude-skills/
├── SKILL.md                    # Org-wide standards index
├── code-review/
│   ├── SKILL.md
│   └── review-checklist.md
├── security-audit/
│   ├── SKILL.md
│   └── owasp-patterns.md
├── api-design/
│   ├── SKILL.md
│   └── rest-conventions.md
└── incident-response/
    ├── SKILL.md
    └── runbook-template.md
```

### Integration strategies

**Git submodules** (recommended for strict version control):
```bash
# Add org skills as a submodule — pinned to a specific commit
git submodule add https://github.com/myorg/claude-skills .claude/skills/org-standards
git submodule update --init

# Update to latest approved version
cd .claude/skills/org-standards && git checkout v2.3.0
```

**Symlinks** (for mono-repo or shared filesystem):
```bash
ln -s /shared/team-skills/security-audit .claude/skills/security-audit
```

**CI-managed copy** (for air-gapped or highly controlled environments):
```yaml
# In CI pipeline
- name: Sync org skills
  run: |
    curl -L https://github.com/myorg/claude-skills/archive/v2.3.0.tar.gz | tar xz
    cp -r claude-skills-2.3.0/* .claude/skills/org-standards/
```

### Skill contribution workflow

1. Developer identifies a repeating pattern (3+ occurrences = skill candidate).
2. Branch: `feature/skill-<domain>-<name>`.
3. Write SKILL.md with clear trigger conditions and reference files.
4. Peer review by domain expert + one person from a different team.
5. Merge to org skill repo with semver tag.
6. Downstream projects update submodule/pin to new version.

---

## 3. MCP Server Governance

MCP servers extend Claude's capabilities but each server is an attack surface and a reliability risk. Enterprise governance requires a registry, review process, and runtime controls.

### Central registry

```json
{
  "approved_servers": {
    "github": {
      "package": "@modelcontextprotocol/server-github@1.2.3",
      "required_scopes": ["repo", "issues"],
      "max_scope": "write",
      "approved_by": "security-team",
      "last_reviewed": "2026-03-15",
      "risk_tier": "medium"
    },
    "postgres": {
      "package": "@modelcontextprotocol/server-postgres@0.9.1",
      "constraint": "read-only-connection-string",
      "approved_by": "dba-team",
      "last_reviewed": "2026-03-10",
      "risk_tier": "high"
    },
    "stripe": {
      "package": "@stripe/mcp-server@2.0.1",
      "constraint": "test-mode-only-in-dev",
      "approved_by": "security-team",
      "last_reviewed": "2026-03-01",
      "risk_tier": "high"
    }
  },
  "denied_servers": {
    "arbitrary-shell-mcp": {
      "reason": "Unrestricted shell access bypasses all controls",
      "denied_by": "security-team",
      "date": "2026-02-20"
    }
  }
}
```

### Version pinning — mandatory

```json
// CORRECT: pinned version
{ "args": ["-y", "@modelcontextprotocol/server-github@1.2.3"] }

// WRONG: unpinned — breaks without warning, may introduce vulnerabilities
{ "args": ["-y", "@modelcontextprotocol/server-github"] }
```

### Review process for new MCP servers

1. **Capability audit**: enumerate every tool the server exposes. Document read vs. write vs. delete capabilities.
2. **Data flow analysis**: what data enters the server? What leaves? Does it phone home?
3. **Least privilege verification**: can scopes be narrowed? Can write tools be disabled?
4. **Dependency scan**: check npm audit / snyk for known vulnerabilities.
5. **Sandbox test**: run the server in an isolated environment with synthetic data.
6. **Approval**: security team signs off, adds to registry with version pin and review date.
7. **Ongoing**: re-review on version bumps, quarterly even if unchanged.

### Risk tiering

| Tier | Examples | Controls |
|------|----------|----------|
| Low | Filesystem read-only, linter | Standard review |
| Medium | GitHub (write), Jira | Security review + scope limits |
| High | Database, cloud infra, payment | Security review + DBA/SRE sign-off + audit logging |
| Critical | Production deploy, secrets manager | CISO approval + real-time monitoring + break-glass only |

---

## 4. SSO/OIDC Integration

For MCP servers that access company resources, authentication must flow through the organization's identity provider — never through long-lived API keys stored in config files.

### OAuth flow for MCP

```
Developer → Claude Code → MCP Server → Identity Provider (Okta/Azure AD/Auth0)
                                     ← Short-lived token (scoped to user + role)
                                     → MCP Server enforces per-tool permissions
```

### Token lifecycle management

- **Access tokens**: 1 hour max lifetime. Scoped to specific MCP server capabilities.
- **Refresh tokens**: 8 hours or session-length. Stored in OS keychain, never in plaintext config.
- **Token claims**: include user ID, team, role, and allowed MCP tool scopes.
- **Revocation**: SSO provider can revoke tokens instantly (e.g., on offboarding).

### Implementation pattern

```typescript
// MCP server middleware: verify token on every tool call
async function authMiddleware(request: CallToolRequest): Promise<AuthContext | ErrorResponse> {
  const token = request.meta?.authToken;

  if (!token) {
    return {
      isError: true,
      content: [{
        type: "text",
        text: JSON.stringify({
          error: "auth_required",
          auth_url: `https://sso.company.com/oauth/authorize?client_id=${CLIENT_ID}&scope=mcp:tools`,
        }),
      }],
    };
  }

  // Verify JWT against company JWKS endpoint
  const claims = await verifyJWT(token, { jwksUri: "https://sso.company.com/.well-known/jwks.json" });

  // Check tool-level permissions from token claims
  const allowedTools = claims["mcp:tools"] as string[];
  if (!allowedTools.includes(request.params.name)) {
    return {
      isError: true,
      content: [{ type: "text", text: `Forbidden: ${request.params.name} not in user's allowed tools` }],
    };
  }

  return { userId: claims.sub, role: claims.role, team: claims.team };
}
```

### Service account pattern (for CI/CD agents)

```bash
# CI agent authenticates via machine-to-machine OAuth (client_credentials grant)
export MCP_AUTH_TOKEN=$(curl -s -X POST https://sso.company.com/oauth/token \
  -d "grant_type=client_credentials&client_id=$CI_CLIENT_ID&client_secret=$CI_CLIENT_SECRET&scope=mcp:read" \
  | jq -r '.access_token')
```

---

## 5. Role-Based Tool Access

Claude Code's `settings.json` permission system maps directly to organizational roles. Define profiles per role and distribute via team infrastructure.

### Permission profiles

```jsonc
// role-junior.settings.json — Safe defaults, read-heavy, write-limited
{
  "permissions": {
    "allow": [
      "Read(*)", "Glob(*)", "Grep(*)",
      "Write(src/**)", "Edit(src/**)",
      "Bash(npm test*)", "Bash(npm run lint*)", "Bash(npm run typecheck*)"
    ],
    "deny": [
      "Bash(git push*)", "Bash(git rebase*)", "Bash(npm publish*)",
      "Edit(.claude/*)", "Edit(infra/*)", "Edit(*.env*)",
      "Bash(rm -rf*)", "Bash(docker *)"
    ]
  }
}

// role-senior.settings.json — Full dev access, infra restricted
{
  "permissions": {
    "allow": [
      "Read(*)", "Write(*)", "Edit(*)",
      "Bash(npm *)", "Bash(git *)", "Bash(docker compose*)"
    ],
    "deny": [
      "Bash(rm -rf /)", "Edit(infra/terraform/*.tf)",
      "Bash(kubectl * --namespace=production*)",
      "Bash(npm publish*)"
    ]
  }
}

// role-ci.settings.json — Headless agent: test + build only
{
  "permissions": {
    "allow": [
      "Read(*)", "Glob(*)", "Grep(*)",
      "Bash(npm test*)", "Bash(npm run build*)", "Bash(npm run lint*)"
    ],
    "deny": [
      "Write(*)", "Edit(*)",
      "Bash(git push*)", "Bash(npm publish*)", "Bash(curl*)"
    ]
  }
}

// role-auditor.settings.json — Read-only, zero write surface
{
  "permissions": {
    "allow": ["Read(*)", "Glob(*)", "Grep(*)"],
    "deny": ["Write(*)", "Edit(*)", "Bash(*)"]
  }
}
```

### Distribution and enforcement

```bash
# Team maintains canonical role files
team-configs/
├── settings.junior.json
├── settings.senior.json
├── settings.ci.json
├── settings.auditor.json
└── settings.sre.json

# Onboarding script assigns role
#!/bin/bash
ROLE=${1:-junior}
cp "team-configs/settings.${ROLE}.json" .claude/settings.json
echo "Applied ${ROLE} permissions profile"
```

### Escalation pattern

When a junior developer needs a temporarily elevated permission:
1. Create a time-boxed override in personal settings with a comment explaining why.
2. The PreToolUse hook logs the escalation.
3. Override auto-expires (CI check validates no stale escalations exist).

---

## 6. Cost Controls

AI agent spend can grow non-linearly. Enterprise cost management requires budgets at multiple granularities: per-session, per-developer, per-team, and per-month.

### Token budget enforcement

```typescript
import { query, type Message } from "@anthropic-ai/claude-code";

interface BudgetConfig {
  maxInputTokens: number;
  maxOutputTokens: number;
  maxTurns: number;
  maxDurationMs: number;
  alertThresholdPct: number; // e.g. 0.8 = alert at 80%
}

const BUDGETS: Record<string, BudgetConfig> = {
  development: { maxInputTokens: 500_000, maxOutputTokens: 200_000, maxTurns: 50, maxDurationMs: 600_000, alertThresholdPct: 0.8 },
  ci_triage:   { maxInputTokens: 100_000, maxOutputTokens: 50_000,  maxTurns: 20, maxDurationMs: 180_000, alertThresholdPct: 0.9 },
  background:  { maxInputTokens: 200_000, maxOutputTokens: 100_000, maxTurns: 30, maxDurationMs: 300_000, alertThresholdPct: 0.8 },
};

async function runWithBudget(prompt: string, tier: string): Promise<string> {
  const budget = BUDGETS[tier];
  let inputTokens = 0, outputTokens = 0, turns = 0;
  const startTime = Date.now();
  let result = "";

  for await (const msg of query({ prompt, options: { maxTurns: budget.maxTurns } })) {
    turns++;
    if (msg.type === "result") {
      inputTokens += msg.usage?.input_tokens ?? 0;
      outputTokens += msg.usage?.output_tokens ?? 0;
      result = msg.text;
    }

    const elapsed = Date.now() - startTime;
    if (inputTokens > budget.maxInputTokens || outputTokens > budget.maxOutputTokens || elapsed > budget.maxDurationMs) {
      logBudgetExceeded({ tier, inputTokens, outputTokens, turns, elapsed });
      break;
    }
  }

  trackSpend(tier, inputTokens, outputTokens);
  return result;
}
```

### Model tier policies

| Use case | Model | Input cost/1M | Output cost/1M | Session budget | Daily cap |
|----------|-------|---------------|----------------|----------------|-----------|
| Interactive dev | claude-sonnet-4-6 | $3 | $15 | 500K tokens | Unlimited |
| Code review | claude-opus-4-6 | $15 | $75 | 300K tokens | 20 sessions |
| CI triage | claude-haiku-4-5 | $0.25 | $1.25 | 100K tokens | 200 sessions |
| Background agents | claude-haiku-4-5 | $0.25 | $1.25 | 200K tokens | 100 sessions |
| Architecture review | claude-opus-4-6 | $15 | $75 | 1M tokens | 5 sessions |

### Spend tracking and alerting

```typescript
import { existsSync, readFileSync, writeFileSync, mkdirSync } from "node:fs";
import { join } from "node:path";

interface DailySpend {
  total_usd: number;
  by_model: Record<string, number>;
  by_team: Record<string, number>;
  session_count: number;
}

function trackSpend(tier: string, inputTokens: number, outputTokens: number): void {
  const costs: Record<string, { input: number; output: number }> = {
    "claude-opus-4-6":  { input: 15, output: 75 },
    "claude-sonnet-4-6": { input: 3, output: 15 },
    "claude-haiku-4-5":  { input: 0.25, output: 1.25 },
  };

  const model = tierToModel(tier);
  const rate = costs[model] ?? costs["claude-sonnet-4-6"];
  const cost = (inputTokens / 1_000_000) * rate.input + (outputTokens / 1_000_000) * rate.output;

  const dir = join(process.env.HOME!, ".claude", "spend");
  mkdirSync(dir, { recursive: true });

  const today = new Date().toISOString().split("T")[0];
  const file = join(dir, `${today}.json`);

  const data: DailySpend = existsSync(file)
    ? JSON.parse(readFileSync(file, "utf-8"))
    : { total_usd: 0, by_model: {}, by_team: {}, session_count: 0 };

  data.total_usd += cost;
  data.by_model[model] = (data.by_model[model] ?? 0) + cost;
  data.session_count++;
  writeFileSync(file, JSON.stringify(data, null, 2));

  // Alert thresholds
  const DAILY_WARN = 50;   // $50/day warning
  const DAILY_HARD = 200;  // $200/day hard stop
  if (data.total_usd > DAILY_HARD) {
    notifyAndBlock(`HARD STOP: Daily AI spend $${data.total_usd.toFixed(2)} exceeds $${DAILY_HARD}`);
  } else if (data.total_usd > DAILY_WARN) {
    notifySlack(`WARNING: Daily AI spend $${data.total_usd.toFixed(2)} exceeds $${DAILY_WARN}`);
  }
}
```

### Monthly budget dashboard (CLI)

```bash
# Monthly spend summary
jq -s '
  reduce .[] as $day ({};
    .total += $day.total_usd |
    reduce (($day.by_model // {}) | to_entries[]) as $e (.;
      .by_model[$e.key] += $e.value
    )
  ) | {
    total: (.total | . * 100 | round / 100),
    by_model: (.by_model | to_entries | map({key, value: (.value * 100 | round / 100)}) | from_entries)
  }
' ~/.claude/spend/2026-04-*.json
```

---

## 7. Audit Logging Infrastructure

Every tool invocation in an enterprise agent deployment must be logged, searchable, and retainable per compliance requirements.

### Log architecture

```
Claude Code session
  └→ PostToolUse hook (settings.json)
       └→ Appends JSONL to ~/.claude/audit-logs/<date>.jsonl
            └→ Log shipper (Fluentd/Vector/Filebeat)
                 └→ Central store (Elasticsearch / Loki / Datadog)
                      └→ Dashboards + alerts (Grafana / Kibana)
```

### Standardized log schema

```typescript
interface AgentAuditEntry {
  // Identity
  timestamp: string;              // ISO 8601 with timezone
  session_id: string;             // Unique per Claude session
  agent_type: "interactive" | "ci" | "background" | "scheduled";
  user_id: string;                // SSO identity
  team: string;
  project: string;                // Git repo name or project identifier

  // Action
  tool_name: string;              // "Edit", "Bash", "mcp__github__create_issue"
  tool_input_hash: string;        // SHA-256 of full input (don't log raw — may contain secrets)
  tool_input_summary: string;     // First 200 chars, PII-scrubbed
  tool_result_status: "success" | "error" | "denied" | "timeout";
  tool_duration_ms: number;

  // Cost
  model: string;
  input_tokens: number;
  output_tokens: number;
  estimated_cost_usd: number;

  // Context
  git_branch: string;
  git_commit_sha: string;
  permission_profile: string;     // "junior", "senior", "ci", etc.
}
```

### Hook implementation

```json
// settings.json — audit logging hook
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "*",
        "command": "node /shared/scripts/audit-log.mjs"
      }
    ]
  }
}
```

### Retention policy

| Log category | Retention | Rationale |
|-------------|-----------|-----------|
| Development (interactive) | 30 days | Debugging, not compliance |
| CI/CD agent logs | 90 days | Build reproducibility |
| Production-touching agents | 1 year | SOC 2 / audit trail |
| Security-sensitive (infra, secrets) | 2 years | Compliance + forensics |
| Cost/billing logs | Indefinite | Budget trend analysis |

### Analysis queries

```bash
# Total estimated cost today
jq -s 'map(.estimated_cost_usd) | add // 0 | . * 100 | round / 100' \
  ~/.claude/audit-logs/2026-04-01.jsonl

# Top 10 most-used tools this week
cat ~/.claude/audit-logs/2026-03-2*.jsonl | \
  jq -s 'group_by(.tool_name) | map({tool: .[0].tool_name, count: length}) | sort_by(-.count) | .[0:10]'

# Failed tool calls with error context
jq -s 'map(select(.tool_result_status == "error"))' ~/.claude/audit-logs/2026-04-01.jsonl

# Sessions exceeding 50 turns (potential runaway agents)
jq -s 'group_by(.session_id) | map(select(length > 50)) | map({session: .[0].session_id, turns: length, user: .[0].user_id})' \
  ~/.claude/audit-logs/2026-04-01.jsonl

# Cost by team this month
cat ~/.claude/audit-logs/2026-04-*.jsonl | \
  jq -s 'group_by(.team) | map({team: .[0].team, cost: (map(.estimated_cost_usd) | add | . * 100 | round / 100)})'
```

---

## 8. Agent Deployment Patterns

Treating agent configurations (CLAUDE.md, skills, hooks, settings.json, mcp.json) as deployable artifacts enables staged rollouts, canary testing, and rollback.

### Staging-to-production promotion

```
1. Develop   → Branch with new skill/hook/CLAUDE.md changes
2. Test      → Run eval suite (automated prompts with expected outcomes)
3. Shadow    → Run new config alongside old config, compare tool call patterns
4. Canary    → Deploy to 10% of team, monitor cost and error rates
5. Rollout   → 50% → 100% over 48 hours
6. Rollback  → `git revert` the config commit, immediate effect on next session
```

### Eval suite for agent configs

```typescript
// eval/test-cases.ts — validate agent behavior against known scenarios
const testCases = [
  {
    name: "should use Server Component for data fetch",
    prompt: "Add a page that displays a list of users from the API",
    expectTool: "Write",
    expectPattern: /async function.*Page/,      // Server Component pattern
    denyPattern: /'use client'/,                 // Should NOT add use client
  },
  {
    name: "should validate API input with Zod",
    prompt: "Create a POST endpoint for creating a user",
    expectTool: "Write",
    expectPattern: /z\.object/,                  // Zod schema
    denyPattern: /JSON\.parse\(.*body\)/,        // Raw parse without validation
  },
  {
    name: "should refuse to edit terraform directly",
    prompt: "Change the RDS instance type to db.r6g.xlarge",
    expectDenied: true,                          // Permission system should block
  },
];
```

### Canary deployments for MCP servers

```json
{
  "mcpServers": {
    "github-stable": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github@1.2.3"]
    },
    "github-canary": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github@1.3.0-rc.1"],
      "disabled": true
    }
  }
}
```

Enable canary for specific developers:
```bash
# Developer opts in to canary
jq '.mcpServers["github-canary"].disabled = false | .mcpServers["github-stable"].disabled = true' \
  .claude/mcp.json > tmp && mv tmp .claude/mcp.json
```

### Blue-green for headless agents

```typescript
// Two agent pools: blue (current) and green (new config)
const blueConfig = loadConfig("v2.3.0");  // Current production
const greenConfig = loadConfig("v2.4.0"); // New version

// Route traffic
const config = Math.random() < canaryPercent ? greenConfig : blueConfig;

const result = await query({
  prompt: task.prompt,
  options: { ...config, maxTurns: config.maxTurns },
});

// Compare outcomes
logDeploymentMetrics({
  version: config.version,
  success: result.exitCode === 0,
  tokens: result.totalTokens,
  duration: result.durationMs,
});
```

---

## 9. Knowledge Management at Scale

Enterprise knowledge management for AI agents spans team conventions, domain expertise, architectural decisions, and operational runbooks.

### Knowledge taxonomy

| Type | Storage | Update frequency | Owner |
|------|---------|-----------------|-------|
| Team conventions | CLAUDE.md | Monthly / retro | Tech lead |
| Domain knowledge | Skills (SKILL.md + refs) | Per feature cycle | Domain experts |
| Architecture decisions | ADRs in git (`docs/decisions/`) | Per decision | Architects |
| Incident learnings | Skills or CLAUDE.md addendum | Post-incident | SRE / on-call |
| API contracts | OpenAPI specs + MCP tools | Per API change | API owners |
| Onboarding context | Dedicated onboarding skill | Quarterly | Engineering manager |

### Architecture Decision Records (ADRs) as agent context

```markdown
<!-- docs/decisions/0015-use-drizzle-orm.md -->
# ADR-0015: Use Drizzle ORM instead of Prisma

## Status: Accepted (2026-02-15)

## Context
Prisma's generated client adds 10MB+ to Lambda cold starts.

## Decision
Migrate to Drizzle ORM for type-safe SQL with zero runtime overhead.

## Consequences
- All new database code uses Drizzle schema definitions
- Existing Prisma code migrated incrementally (tracked in JIRA-4521)
- Claude should generate Drizzle patterns, never Prisma
```

Reference ADRs from CLAUDE.md:
```markdown
## Architecture
- ORM: Drizzle (see docs/decisions/0015-use-drizzle-orm.md) — never generate Prisma code
```

### Skill contribution pipeline

```
Identify pattern → Draft skill → Peer review → Publish → Measure adoption
     (3+ uses)       (branch)      (PR)       (merge)    (audit logs)
```

Metrics to track:
- Skill activation frequency (from audit logs: how often does a skill's trigger fire?)
- Error rate with vs. without skill active
- Token usage with vs. without skill (skills that increase cost without improving quality should be trimmed)

### Cross-team knowledge sharing

```bash
# Org-wide skill registry (simple: a JSON file in the org skills repo)
{
  "skills": [
    {
      "name": "api-design",
      "owner": "platform-team",
      "version": "1.2.0",
      "description": "REST API design conventions, pagination, error format",
      "adopted_by": ["team-a", "team-b", "team-c"]
    },
    {
      "name": "security-audit",
      "owner": "security-team",
      "version": "2.0.0",
      "description": "OWASP top 10 patterns, dependency scanning, secret detection",
      "adopted_by": ["all"]
    }
  ]
}
```

---

## 10. Compliance and Regulatory Considerations

Deploying AI agents in regulated environments (finance, healthcare, government) requires explicit controls for data handling, auditability, and access governance.

### Data residency and processing

- **Know your data flows**: Claude API calls transmit prompt context to Anthropic's infrastructure. Map every data flow.
- **Data classification**: classify what enters agent context as Public, Internal, Confidential, or Restricted.
- **Restricted data**: PII, PHI, PCI, and trade secrets must never enter agent context unless architecturally necessary and approved by legal/compliance.
- **MCP as a boundary**: use MCP servers to query sensitive data and return only necessary fields — the raw data stays in your infrastructure.

```typescript
// MCP tool that filters sensitive fields before returning to agent context
server.tool("get_customer", { id: z.string() }, async ({ id }) => {
  const customer = await db.customers.findById(id);
  // Return only non-sensitive fields
  return {
    content: [{
      type: "text",
      text: JSON.stringify({
        id: customer.id,
        plan: customer.plan,
        region: customer.region,
        // Explicitly exclude: name, email, phone, payment_method
      }),
    }],
  };
});
```

### PII detection hook

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "*",
        "command": "node /shared/scripts/pii-scanner.mjs"
      }
    ]
  }
}
```

```typescript
// pii-scanner.mjs — scan tool results for PII patterns
const PII_PATTERNS = [
  { name: "SSN", pattern: /\b\d{3}-\d{2}-\d{4}\b/ },
  { name: "Credit Card", pattern: /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/ },
  { name: "Email", pattern: /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z]{2,}\b/i },
  { name: "Phone", pattern: /\b\+?1?[-.\s]?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b/ },
];

const toolResult = JSON.parse(process.env.TOOL_RESULT ?? "{}");
const text = JSON.stringify(toolResult);

for (const { name, pattern } of PII_PATTERNS) {
  if (pattern.test(text)) {
    console.error(`WARNING: Potential ${name} detected in tool result. Review before proceeding.`);
    // Optionally: block the result, redact, or require confirmation
  }
}
```

### GDPR / CCPA compliance

| Requirement | Agent implication | Control |
|------------|-------------------|---------|
| Right to access | User can request all agent logs referencing them | Audit logs searchable by user_id |
| Right to erasure | Logs must be deletable on request | Retention policy + deletion script |
| Data minimization | Only collect what's needed | PII filter hooks, scoped MCP tools |
| Purpose limitation | Agent data used only for stated purpose | Documented in data processing agreement |
| Consent | User aware of AI processing | Disclosed in terms of service |

### SOC 2 mapping

| SOC 2 criterion | Agent control |
|-----------------|---------------|
| CC6.1 — Logical access | Role-based settings.json profiles |
| CC6.2 — Authentication | SSO/OIDC for MCP servers |
| CC6.3 — Authorization | PreToolUse permission hooks |
| CC7.1 — Configuration management | CLAUDE.md + skills versioned in git, PR-reviewed |
| CC7.2 — Change management | Staging → canary → production promotion |
| CC8.1 — Monitoring | PostToolUse audit logging, cost alerts |
| CC8.2 — Incident response | Runaway agent detection (turn count, token budget, duration limits) |

### HIPAA considerations (healthcare)

- **Business Associate Agreement (BAA)**: required with Anthropic if PHI may enter context.
- **Minimum necessary**: MCP tools must return only the minimum data needed.
- **Access logging**: every agent access to patient data must be logged with user identity and purpose.
- **Encryption**: all agent-to-MCP communication must be encrypted in transit (HTTPS/TLS for remote MCP, stdio for local).

---

## 11. Multi-Agent Coordination at Scale

Enterprise workflows often require multiple agents operating on different aspects of a task, coordinated by an orchestrator.

### Fan-out / fan-in pattern

```typescript
import { query } from "@anthropic-ai/claude-code";

interface SubTask {
  id: string;
  prompt: string;
  model: string;
  maxTurns: number;
  workdir?: string;
}

async function orchestrate(tasks: SubTask[]): Promise<Map<string, string>> {
  const results = new Map<string, string>();

  // Fan-out: run sub-agents in parallel
  const promises = tasks.map(async (task) => {
    const output = await runWithBudget(task.prompt, task.model, task.maxTurns);
    return { id: task.id, output };
  });

  const settled = await Promise.allSettled(promises);

  for (const result of settled) {
    if (result.status === "fulfilled") {
      results.set(result.value.id, result.value.output);
    } else {
      results.set("error", result.reason.message);
    }
  }

  return results;
}

// Example: parallel code review
const reviewTasks: SubTask[] = [
  { id: "security", prompt: "Review for security vulnerabilities: ...", model: "opus", maxTurns: 10 },
  { id: "performance", prompt: "Review for performance issues: ...", model: "sonnet", maxTurns: 10 },
  { id: "style", prompt: "Review for style and convention compliance: ...", model: "haiku", maxTurns: 5 },
];

const reviews = await orchestrate(reviewTasks);
```

### Pipeline pattern (sequential agents)

```typescript
// Each agent's output feeds the next agent's input
const pipeline = [
  { name: "architect",   prompt: (input: string) => `Design the architecture for: ${input}`, model: "opus" },
  { name: "implementer", prompt: (input: string) => `Implement this design:\n${input}`, model: "sonnet" },
  { name: "reviewer",    prompt: (input: string) => `Review this implementation:\n${input}`, model: "opus" },
  { name: "tester",      prompt: (input: string) => `Write tests for:\n${input}`, model: "sonnet" },
];

let context = userRequest;
for (const stage of pipeline) {
  context = await runAgent(stage.prompt(context), stage.model);
  logPipelineStage(stage.name, context);
}
```

### Supervisor pattern

A supervisor agent monitors sub-agents and intervenes on failure:

```typescript
async function supervisedExecution(task: string, maxRetries: number = 2): Promise<string> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const result = await runAgent(task, "sonnet");

    // Supervisor evaluates the result
    const evaluation = await runAgent(
      `Evaluate this agent output. Is it correct and complete?\n\nTask: ${task}\n\nOutput: ${result}\n\nRespond with JSON: { "pass": boolean, "feedback": string }`,
      "opus"
    );

    const parsed = JSON.parse(evaluation);
    if (parsed.pass) return result;

    // Retry with feedback
    task = `${task}\n\nPrevious attempt feedback: ${parsed.feedback}\nPlease address the feedback and try again.`;
  }

  throw new Error(`Task failed after ${maxRetries} retries`);
}
```

---

## 12. Disaster Recovery and Incident Response

### Runaway agent detection

```typescript
// Guardrails that should be enforced at the orchestration layer
const GUARDRAILS = {
  maxTurnsPerSession: 100,        // No single session should exceed 100 tool calls
  maxTokensPerSession: 2_000_000, // Hard ceiling
  maxCostPerSession: 10.00,       // $10 per session max
  maxConcurrentAgents: 20,        // Org-wide concurrency limit
  maxWriteOpsPerMinute: 30,       // Rate limit destructive operations
  circuitBreakerErrorRate: 0.5,   // Trip if >50% of recent calls fail
};
```

### Incident playbook

1. **Detection**: audit log alerts on anomalous patterns (sudden cost spike, high error rate, unusual tool calls).
2. **Containment**: kill running agent sessions, disable the problematic MCP server or hook.
3. **Assessment**: review audit logs for the session — what tools were called, what data was accessed?
4. **Remediation**: fix the root cause (skill bug, bad CLAUDE.md rule, MCP server regression).
5. **Post-incident**: update skills/CLAUDE.md with the learning, add an eval test case to prevent recurrence.

### Rollback procedure

```bash
# Immediate rollback of agent config to last known good
git log --oneline .claude/         # Find last known good commit
git checkout <good-commit> -- .claude/CLAUDE.md .claude/settings.json .claude/mcp.json
git commit -m "rollback: revert agent config to <good-commit> due to incident"

# For org-wide skill rollback
cd .claude/skills/org-standards
git checkout v2.2.0              # Previous stable version
cd ../..
git add .claude/skills/org-standards
git commit -m "rollback: revert org skills to v2.2.0"
```

---

## Summary: Enterprise Readiness Checklist

| Area | Control | Status |
|------|---------|--------|
| Governance | CLAUDE.md in git, PR-reviewed | |
| Skills | Org skill repo with semver, contribution pipeline | |
| MCP | Central registry, version-pinned, security-reviewed | |
| Auth | SSO/OIDC for sensitive MCP servers | |
| Access | Role-based settings.json profiles | |
| Cost | Per-session budgets, daily caps, monthly dashboards | |
| Audit | Structured JSONL logging, central aggregation | |
| Deployment | Staging → canary → production, eval suite | |
| Knowledge | ADRs + skills + CLAUDE.md, retro-driven updates | |
| Compliance | Data classification, PII hooks, SOC 2 mapping | |
| Multi-agent | Orchestration patterns, supervisor, budget per agent | |
| DR | Runaway detection, rollback procedure, incident playbook | |
