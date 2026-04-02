# MCP Servers — Consuming Existing Servers

A dense reference for senior engineers connecting Claude Code to production MCP servers in 2026. Covers mcp.json configuration, tool discovery, and deep integration guides for GitHub, Figma, Stripe, Vercel, and Firebase MCP servers — plus multi-server composition patterns and credential management strategies.

---

## 1. mcp.json Configuration Schema

### Configuration file locations

MCP server configuration lives in `mcp.json` files at two levels:

| Level | Path | Committed to git? | Scope |
|-------|------|--------------------|-------|
| User-level | `~/.claude/mcp.json` | No | All projects for this user |
| Project-level | `<project>/.claude/mcp.json` | Yes (usually) | This project only |

**Precedence rule**: project-level configuration *merges with* user-level configuration. If both define the same server name, the project-level definition wins entirely — no partial merging of individual fields. This means you can override a user-level server with a project-specific version (different args, different env vars), but you must redeclare the full server config at the project level.

**When to use which:**
- User-level: personal API tokens (GitHub PAT, Figma token), servers you use across all projects, MCP servers for your personal workflow
- Project-level: project-specific servers (this project's Firebase project, this project's Vercel team), shared team configuration with `${ENV_VAR}` placeholders for secrets

### Full annotated mcp.json

This example shows all five major MCP servers with every configuration option that matters:

```jsonc
{
  // Each key is a server name — used in tool namespacing as mcp__<name>__<tool>
  "mcpServers": {
    // ─── GitHub: stdio transport ───
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@anthropic-ai/github-mcp-server@1.2.0"  // Always pin version
      ],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"  // ${} syntax reads from process environment
      }
    },

    // ─── Figma: SSE transport (remote server) ───
    "figma": {
      "type": "sse",
      "url": "https://figma-mcp.example.com/sse",
      "headers": {
        "Authorization": "Bearer ${FIGMA_ACCESS_TOKEN}"
      }
    },

    // ─── Stripe: stdio transport ───
    "stripe": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@anthropic-ai/stripe-mcp-server@2.0.1"
      ],
      "env": {
        "STRIPE_SECRET_KEY": "${STRIPE_TEST_SECRET_KEY}"  // ALWAYS test key — see section 5
      }
    },

    // ─── Vercel: stdio transport ───
    "vercel": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@anthropic-ai/vercel-mcp-server@1.1.0"
      ],
      "env": {
        "VERCEL_TOKEN": "${VERCEL_TOKEN}",
        "VERCEL_TEAM_ID": "${VERCEL_TEAM_ID}"  // Optional — scopes to a specific team
      }
    },

    // ─── Firebase: stdio via firebase-tools ───
    "firebase": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "firebase-tools@14.2.0",
        "experimental:mcp"
      ],
      "env": {
        "GOOGLE_APPLICATION_CREDENTIALS": "${GOOGLE_APPLICATION_CREDENTIALS}",
        "FIREBASE_PROJECT_ID": "${FIREBASE_PROJECT_ID}"
      }
    }
  }
}
```

### Transport types

**stdio** — The default and most common. Claude Code spawns the MCP server as a child process and communicates via stdin/stdout using JSON-RPC 2.0 frames. The server runs locally, starts instantly, and dies when Claude Code exits. Use for all officially packaged servers (`npx` pattern).

```jsonc
{
  "type": "stdio",
  "command": "npx",                    // or "node", "python3", "deno", etc.
  "args": ["-y", "@scope/package"],    // passed to command
  "env": { "KEY": "${VALUE}" }         // environment variables for the child process
}
```

**sse** — Server-Sent Events over HTTP. The MCP server runs as a persistent remote service. Claude Code connects via HTTP and receives events over SSE. Use for remote/shared MCP servers, servers behind auth proxies, or servers that need to maintain long-lived connections to external services.

```jsonc
{
  "type": "sse",
  "url": "https://mcp-server.example.com/sse",
  "headers": {
    "Authorization": "Bearer ${TOKEN}"
  }
}
```

**http** (Streamable HTTP) — New in 2026. The successor to SSE for remote servers. Uses standard HTTP POST for requests and supports bidirectional streaming via chunked transfer encoding. Better for environments where SSE proxying is difficult (some CDNs, corporate proxies). The server publishes a single HTTP endpoint.

```jsonc
{
  "type": "http",
  "url": "https://mcp-server.example.com/mcp",
  "headers": {
    "Authorization": "Bearer ${TOKEN}"
  }
}
```

**Which transport to choose:**
- stdio: first choice for all local servers. Zero network overhead, simple lifecycle.
- sse: remote servers that need to push data (e.g., long-running operations with progress).
- http: remote servers in environments where SSE is problematic, or when the server needs request-response semantics.

### Environment variable syntax

The `${ENV_VAR}` syntax in mcp.json reads from the process environment at server startup time. This is the **only** way to pass secrets into MCP server configuration.

```jsonc
{
  "env": {
    "GITHUB_TOKEN": "${GITHUB_TOKEN}",           // reads process.env.GITHUB_TOKEN
    "STRIPE_SECRET_KEY": "${STRIPE_TEST_SECRET_KEY}",  // reads process.env.STRIPE_TEST_SECRET_KEY
    "CUSTOM_VAR": "literal-value"                 // literal values also work (not for secrets)
  }
}
```

**NEVER hardcode tokens or secrets in mcp.json.** The file is often committed to version control. Even for user-level `~/.claude/mcp.json` which is not committed, hardcoded secrets create rotation nightmares and audit blind spots.

The typical setup: create a `.env.local` file (gitignored) at the project root or use your shell profile to export the variables. Claude Code reads the process environment when spawning MCP servers.

```bash
# .env.local (never committed)
GITHUB_TOKEN=ghp_PLACEHOLDER_REPLACE_ME
FIGMA_ACCESS_TOKEN=figd_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
STRIPE_TEST_SECRET_KEY=sk_test_PLACEHOLDER_REPLACE_ME
VERCEL_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxx
FIREBASE_PROJECT_ID=my-project-123
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
```

### Version pinning in args

Always pin MCP server package versions in `args`. An unpinned `npx -y @scope/package` fetches latest, which can introduce breaking tool schema changes mid-sprint. Pinned versions guarantee deterministic tool interfaces.

```jsonc
// BAD: unpinned — tool schemas can change without warning
"args": ["-y", "@anthropic-ai/github-mcp-server"]

// GOOD: pinned — deterministic tool interfaces
"args": ["-y", "@anthropic-ai/github-mcp-server@1.2.0"]
```

Update pinned versions deliberately, test the new tool schemas, and update any `allowedTools` lists that reference specific tool names.

---

## 2. Tool Discovery and Inspection

### The /mcp command

In Claude Code's interactive mode, the `/mcp` slash command shows the status of all configured MCP servers:

```
> /mcp

MCP Servers:
  github    ● connected   12 tools   stdio
  figma     ● connected    5 tools   sse
  stripe    ● connected    8 tools   stdio
  vercel    ● connected    6 tools   stdio
  firebase  ● connected   11 tools   stdio
```

This tells you immediately whether servers started successfully, how many tools each exposes, and which transport they're using. If a server shows `● disconnected` or `● error`, check the command/args, env vars, and that the package version exists.

### Tool namespacing

Every MCP tool is namespaced as `mcp__<server>__<tool>` where `<server>` is the key from your mcp.json and `<tool>` is the tool name defined by the MCP server. This prevents collisions when multiple servers expose similarly-named tools.

```
mcp__github__create_issue
mcp__github__list_pull_requests
mcp__figma__get_file
mcp__stripe__create_customer
mcp__vercel__list_deployments
mcp__firebase__firestore_get_document
```

This namespacing matters in three places:
1. **allowedTools** in SDK `query()` calls — you must use the full namespaced name
2. **Hooks** — PreToolUse/PostToolUse `$TOOL_NAME` env var uses the namespaced name
3. **Logging and audit** — all tool calls are logged with the namespaced name

```typescript
// allowedTools uses namespaced names
const options: SDKOptions = {
  allowedTools: [
    "Read", "Write", "Edit",  // built-in tools — no namespace
    "mcp__github__create_issue",
    "mcp__github__list_pull_requests",
    "mcp__github__get_file_contents",
    "mcp__stripe__create_product",
    "mcp__stripe__create_price",
  ],
};
```

### Programmatic inspection via Client SDK

When building agent systems, you may need to inspect available tools at runtime — for example, to dynamically build `allowedTools` lists or validate that expected tools exist before launching an agent.

```typescript
import { query, type SDKMessage } from "@anthropic-ai/claude-code";

async function discoverTools(serverName: string): Promise<string[]> {
  const tools: string[] = [];

  for await (const msg of query({
    prompt: `List all available MCP tools from the ${serverName} server. Output only the tool names, one per line.`,
    options: {
      maxTurns: 3,
      allowedTools: ["Bash"],  // minimal permissions for discovery
      model: "claude-sonnet-4-6",
    },
  })) {
    if (msg.type === "assistant") {
      const text = msg.message.content
        .filter((b: any) => b.type === "text")
        .map((b: any) => b.text)
        .join("");
      if (text) {
        tools.push(
          ...text.split("\n").filter((l: string) => l.startsWith(`mcp__${serverName}__`))
        );
      }
    }
  }

  return tools;
}
```

A more reliable approach for programmatic discovery: use the `/mcp` output or query the MCP server directly via JSON-RPC `tools/list`:

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

async function listToolsFromServer(): Promise<void> {
  const transport = new StdioClientTransport({
    command: "npx",
    args: ["-y", "@anthropic-ai/github-mcp-server@1.2.0"],
    env: { ...process.env, GITHUB_TOKEN: process.env.GITHUB_TOKEN! },
  });

  const client = new Client({ name: "tool-inspector", version: "1.0.0" });
  await client.connect(transport);

  const { tools } = await client.listTools();
  for (const tool of tools) {
    console.log(`${tool.name}: ${tool.description}`);
    console.log(`  Input schema: ${JSON.stringify(tool.inputSchema, null, 2)}`);
  }

  await client.close();
}
```

This gives you the full tool schema including input parameters, types, and descriptions — useful for building validation layers or generating typed wrappers.

---

## 3. GitHub MCP Server

### Configuration

```jsonc
// In ~/.claude/mcp.json or <project>/.claude/mcp.json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/github-mcp-server@1.2.0"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### Required token scopes

Use a **fine-grained personal access token** (not classic tokens). Fine-grained tokens scope to specific repositories and permissions, which is critical for agent security.

| Permission | Scope | Why |
|-----------|-------|-----|
| Repository contents | Read & Write | Read files, push commits, create branches |
| Issues | Read & Write | Create/update issues, add comments |
| Pull requests | Read & Write | Create PRs, list PRs, merge PRs |
| Metadata | Read | Required for all API access |

**Never grant `admin` scope to an agent.** Admin permissions allow destructive operations (delete repos, change visibility, manage webhooks) that an agent should never perform autonomously. If an agent needs admin-level actions, escalate to a human.

For read-only workflows (code analysis, issue triage, PR review), drop the Write permissions entirely:

| Permission | Scope |
|-----------|-------|
| Repository contents | Read |
| Issues | Read |
| Pull requests | Read |
| Metadata | Read |

### Key tools

```
mcp__github__create_issue
  owner: string, repo: string, title: string, body?: string, labels?: string[], assignees?: string[]

mcp__github__list_pull_requests
  owner: string, repo: string, state?: "open" | "closed" | "all", head?: string, base?: string

mcp__github__create_pull_request
  owner: string, repo: string, title: string, body?: string, head: string, base: string, draft?: boolean

mcp__github__get_file_contents
  owner: string, repo: string, path: string, ref?: string

mcp__github__push_files
  owner: string, repo: string, branch: string, files: Array<{path: string, content: string}>, message: string

mcp__github__search_repositories
  query: string, sort?: "stars" | "forks" | "updated", order?: "asc" | "desc"

mcp__github__create_branch
  owner: string, repo: string, branch: string, from_branch?: string

mcp__github__merge_pull_request
  owner: string, repo: string, pull_number: number, merge_method?: "merge" | "squash" | "rebase"

mcp__github__add_issue_comment
  owner: string, repo: string, issue_number: number, body: string

mcp__github__list_commits
  owner: string, repo: string, sha?: string, path?: string, since?: string, until?: string
```

### Workflow example: agent reads failing CI and creates issue with context

This is a common pattern — a background agent monitors CI and creates well-structured issues when builds fail:

```typescript
import { query, type SDKMessage } from "@anthropic-ai/claude-code";

async function ciFailureTriageAgent(
  owner: string,
  repo: string,
  runId: string
): Promise<void> {
  const messages: SDKMessage[] = [];

  for await (const msg of query({
    prompt: `
      A CI run failed for ${owner}/${repo} (run ID: ${runId}).

      Steps:
      1. Use mcp__github__list_commits to get the last 5 commits on the main branch
      2. Use Bash to run "gh run view ${runId} --log-failed" to get the failure logs
      3. Analyze the failure: identify the root cause, the failing test or build step,
         and which recent commit likely introduced the regression
      4. Use mcp__github__create_issue to create a well-structured issue with:
         - Title: "CI Failure: [brief description of failure]"
         - Body: root cause analysis, relevant log excerpts (max 50 lines),
           link to the failing run, suspected commit SHA
         - Labels: ["bug", "ci-failure"]
      5. If you can identify the author of the suspected commit, assign them

      Be precise. Include only relevant log lines, not the full output.
    `,
    options: {
      maxTurns: 15,
      cwd: `/tmp/ci-triage-${runId}`,
      permissionMode: "bypassPermissions",
      allowedTools: [
        "Bash",
        "mcp__github__list_commits",
        "mcp__github__create_issue",
        "mcp__github__get_file_contents",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    messages.push(msg);
  }
}
```

### Security considerations

- **Fine-grained tokens over classic tokens** — always. Classic tokens grant broad access across all repos.
- **Repository-scoped tokens** — scope the token to only the repositories the agent needs. A CI triage agent doesn't need access to your private dotfiles repo.
- **Short expiration** — set token expiration to 30-90 days max. Use your secrets manager to handle rotation.
- **Read-only by default** — start with read-only permissions. Only add write when the workflow explicitly requires it and you've verified the agent won't make destructive changes.
- **Never `admin` scope** — there is no legitimate agent use case for admin permissions. If you think you need it, you need a human in the loop instead.
- **Audit trail** — GitHub logs all API calls by token. Use a dedicated token per agent (not your personal token) so you can audit and revoke agent access independently.

---

## 4. Figma MCP Server

### Configuration

```jsonc
{
  "mcpServers": {
    "figma": {
      "type": "sse",
      "url": "https://figma-mcp.example.com/sse",
      "headers": {
        "Authorization": "Bearer ${FIGMA_ACCESS_TOKEN}"
      }
    }
  }
}
```

Figma MCP typically runs as a remote SSE server because it maintains a persistent connection to the Figma API and caches file data. If you're running the official Figma MCP locally, the stdio configuration works too:

```jsonc
{
  "mcpServers": {
    "figma": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/figma-mcp-server@1.0.3"],
      "env": {
        "FIGMA_ACCESS_TOKEN": "${FIGMA_ACCESS_TOKEN}"
      }
    }
  }
}
```

### Key tools

```
mcp__figma__get_file
  file_key: string
  Returns: full Figma file JSON (document tree, components, styles)
  WARNING: Large files return 5-50MB of JSON. Prefer get_node for targeted access.

mcp__figma__get_node
  file_key: string, node_id: string
  Returns: specific node and its children only
  USE THIS for most workflows — dramatically reduces context cost.

mcp__figma__get_components
  file_key: string
  Returns: all published components in the file with their properties

mcp__figma__get_styles
  file_key: string
  Returns: all styles (colors, text styles, effects, grids) in the file

mcp__figma__export_node_as_image
  file_key: string, node_id: string, format?: "png" | "svg" | "pdf", scale?: number
  Returns: URL to exported image
```

### Context cost warning

`get_file` on a large Figma design system file can return 10-50MB of JSON. This will obliterate your context window and cause the model to degrade or fail. **Always prefer `get_node` with a specific `node_id`** — get the node ID from Figma's URL or from a previous `get_components` call.

The workflow pattern for large files:
1. Use `get_components` to list all components and their node IDs (small response)
2. Use `get_styles` to extract design tokens (small response)
3. Use `get_node` with specific node IDs for the components you actually need

Never call `get_file` unless you're working with a small file (< 50 components) and you genuinely need the full tree.

### Cross-reference: web-frontend (Tailwind v4 @theme)

Extract Figma design tokens and convert them to Tailwind v4 `@theme` values. This is one of the highest-value Figma MCP workflows — keeping design tokens in sync between Figma and code.

```typescript
import { query } from "@anthropic-ai/claude-code";

async function syncFigmaTokensToTailwind(
  figmaFileKey: string,
  cssOutputPath: string
): Promise<void> {
  for await (const msg of query({
    prompt: `
      Extract design tokens from Figma and generate Tailwind v4 @theme CSS.

      Steps:
      1. Use mcp__figma__get_styles to get all styles from file "${figmaFileKey}"
      2. For each color style, extract the name and hex/rgba value
      3. For each text style, extract font-family, font-size, font-weight, line-height, letter-spacing
      4. For each effect style (shadows), extract the shadow values
      5. Read the existing file at "${cssOutputPath}" to understand the current structure
      6. Generate a Tailwind v4 @theme block that maps Figma tokens to CSS custom properties:

         @theme {
           --color-primary: <figma primary color>;
           --color-secondary: <figma secondary color>;
           /* ... all color tokens ... */

           --font-sans: <figma body font family>;
           --font-heading: <figma heading font family>;

           --text-xs: <figma smallest text size>;
           --text-sm: <figma small text size>;
           /* ... all text size tokens ... */

           --shadow-sm: <figma small shadow>;
           --shadow-md: <figma medium shadow>;
           /* ... all shadow tokens ... */
         }

      7. Write the generated CSS to "${cssOutputPath}"

      Follow Tailwind v4 @theme conventions exactly. Use the -- namespace prefix
      that matches Tailwind's utility class expectations (--color-*, --text-*, --shadow-*).
      Preserve any manually-added tokens in the existing file.
    `,
    options: {
      maxTurns: 15,
      allowedTools: [
        "Read", "Write", "Edit",
        "mcp__figma__get_styles",
        "mcp__figma__get_components",
        "mcp__figma__get_node",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream processing
  }
}
```

### Cross-reference: ios-26-app (SwiftUI component specs)

Extract Figma component specifications and generate SwiftUI views. The pattern mirrors the Tailwind workflow but targets Swift types:

```typescript
import { query } from "@anthropic-ai/claude-code";

async function figmaToSwiftUI(
  figmaFileKey: string,
  componentNodeId: string,
  swiftOutputDir: string
): Promise<void> {
  for await (const msg of query({
    prompt: `
      Extract the Figma component at node "${componentNodeId}" in file "${figmaFileKey}"
      and generate a SwiftUI view.

      Steps:
      1. Use mcp__figma__get_node to get the component spec (file: "${figmaFileKey}", node: "${componentNodeId}")
      2. Use mcp__figma__export_node_as_image to get a visual reference (PNG, 2x scale)
      3. Analyze the component structure:
         - Layout: HStack/VStack/ZStack mapping from Figma auto-layout
         - Typography: map Figma text styles to iOS Dynamic Type sizes
         - Colors: map Figma fills to SwiftUI Color assets or .tint
         - Spacing: map Figma padding/gap to SwiftUI .padding() values
         - Corner radius: map to .clipShape(.rect(cornerRadius:))
         - Shadows: map to .shadow()
      4. Generate a SwiftUI view in "${swiftOutputDir}/" following ios-26-app conventions:
         - Use @Observable for state (not @StateObject/@ObservedObject)
         - Use Liquid Glass materials where appropriate (iOS 26)
         - Respect Dynamic Type and accessibility sizing
         - Include a #Preview block

      Output production-quality SwiftUI, not a rough sketch.
    `,
    options: {
      maxTurns: 15,
      allowedTools: [
        "Read", "Write", "Edit",
        "mcp__figma__get_node",
        "mcp__figma__get_components",
        "mcp__figma__export_node_as_image",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream processing
  }
}
```

---

## 5. Stripe MCP Server

### Configuration

```jsonc
{
  "mcpServers": {
    "stripe": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/stripe-mcp-server@2.0.1"],
      "env": {
        "STRIPE_SECRET_KEY": "${STRIPE_TEST_SECRET_KEY}"
      }
    }
  }
}
```

### CRITICAL: Test keys only — never live keys for agents

This is the single most important rule for Stripe MCP integration: **agents must only ever use test-mode secret keys** (`sk_test_...`). Never `sk_live_...`. An agent with a live Stripe key can create real charges, modify real subscriptions, and issue real refunds. A confused or prompt-injected agent with live keys is a financial incident.

Test keys are functionally identical to live keys for development purposes — they hit the same API, return the same response shapes, and support the same operations. They just don't move real money.

**Enforcement strategies:**

1. **Environment variable naming**: Name the env var `STRIPE_TEST_SECRET_KEY` (not `STRIPE_SECRET_KEY`) to make the intent obvious.

2. **PreToolUse hook** that blocks live keys:

```bash
#!/bin/bash
# .claude/hooks/pre-tool-use/block-live-stripe.sh
# Hook for: PreToolUse on mcp__stripe__*

if [[ "$TOOL_NAME" == mcp__stripe__* ]]; then
  # Check if the Stripe key in the environment starts with sk_live_
  if [[ "${STRIPE_SECRET_KEY:-}" == sk_live_* ]] || [[ "${STRIPE_TEST_SECRET_KEY:-}" == sk_live_* ]]; then
    echo "BLOCKED: Live Stripe key detected. Agents must use test keys only (sk_test_...)." >&2
    echo "Set STRIPE_TEST_SECRET_KEY to a test-mode key and restart Claude Code." >&2
    exit 1  # Non-zero exit blocks the tool call
  fi
fi

exit 0
```

Register this hook in settings.json:

```jsonc
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__stripe__.*",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/pre-tool-use/block-live-stripe.sh"
          }
        ]
      }
    ]
  }
}
```

3. **CI/CD validation**: In your CI pipeline, scan mcp.json files for any reference to `STRIPE_SECRET_KEY` (without `TEST` in the name) and fail the build.

### Key tools

```
mcp__stripe__create_customer
  name?: string, email?: string, metadata?: Record<string, string>
  Returns: Stripe Customer object with id (cus_...)

mcp__stripe__create_product
  name: string, description?: string, metadata?: Record<string, string>
  Returns: Stripe Product object with id (prod_...)

mcp__stripe__create_price
  product: string, unit_amount: number, currency: string, recurring?: { interval: "day" | "week" | "month" | "year" }
  Returns: Stripe Price object with id (price_...)

mcp__stripe__create_payment_intent
  amount: number, currency: string, customer?: string, metadata?: Record<string, string>
  Returns: PaymentIntent object with client_secret for frontend

mcp__stripe__list_subscriptions
  customer?: string, status?: "active" | "past_due" | "canceled" | "all"
  Returns: list of Subscription objects

mcp__stripe__create_checkout_session
  line_items: Array<{ price: string, quantity: number }>, mode: "payment" | "subscription",
  success_url: string, cancel_url: string
  Returns: Checkout Session with url for redirect
```

### Workflow: agent reads pricing spec and creates Stripe products/prices

```typescript
import { query } from "@anthropic-ai/claude-code";

async function createStripeProductsFromSpec(specPath: string): Promise<void> {
  for await (const msg of query({
    prompt: `
      Read the pricing spec at "${specPath}" and create the corresponding
      Stripe products and prices.

      Steps:
      1. Read "${specPath}" — it contains a JSON array of products with:
         { name, description, prices: [{ amount_cents, currency, interval? }] }
      2. For each product in the spec:
         a. Use mcp__stripe__create_product with the name and description
         b. For each price in the product's prices array:
            - Use mcp__stripe__create_price with the product ID,
              unit_amount (in cents), currency, and recurring interval if present
      3. After creating all products and prices, output a summary table:
         | Product | Product ID | Price | Price ID | Amount | Interval |
      4. Write the mapping of spec names to Stripe IDs to "${specPath}.stripe-ids.json"
         so downstream systems can reference the Stripe objects.

      IMPORTANT: Verify you are using a test key (the key should start with sk_test_).
      If the key appears to be a live key, STOP immediately and report the error.
    `,
    options: {
      maxTurns: 25,
      allowedTools: [
        "Read", "Write",
        "mcp__stripe__create_product",
        "mcp__stripe__create_price",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream processing
  }
}
```

---

## 6. Vercel MCP Server

### Configuration

```jsonc
{
  "mcpServers": {
    "vercel": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/vercel-mcp-server@1.1.0"],
      "env": {
        "VERCEL_TOKEN": "${VERCEL_TOKEN}",
        "VERCEL_TEAM_ID": "${VERCEL_TEAM_ID}"
      }
    }
  }
}
```

Generate the token at [vercel.com/account/tokens](https://vercel.com/account/tokens). For team projects, include `VERCEL_TEAM_ID` to scope operations to the correct team. Without it, operations default to your personal account.

### Key tools

```
mcp__vercel__list_projects
  team_id?: string, limit?: number
  Returns: array of Project objects with name, id, framework, latest deployment

mcp__vercel__list_deployments
  project_id?: string, team_id?: string, state?: "BUILDING" | "READY" | "ERROR", limit?: number
  Returns: array of Deployment objects with url, state, created timestamp

mcp__vercel__create_deployment
  project_id: string, ref?: string, target?: "production" | "preview"
  Returns: Deployment object — triggers a new build

mcp__vercel__get_deployment_logs
  deployment_id: string
  Returns: build logs and runtime logs for the deployment

mcp__vercel__create_env
  project_id: string, key: string, value: string, target: ("production" | "preview" | "development")[]
  Returns: confirmation of env var creation

mcp__vercel__list_domains
  project_id: string
  Returns: array of Domain objects with name, verified status, DNS config
```

### Cross-reference: web-frontend deploy-on-green workflow

The canonical pattern for CI/CD with Claude Code agents: when tests pass on main, deploy to Vercel automatically. This integrates the GitHub MCP (to check CI status) with the Vercel MCP (to trigger deployment).

```typescript
import { query } from "@anthropic-ai/claude-code";

async function deployOnGreen(
  owner: string,
  repo: string,
  projectDir: string
): Promise<void> {
  for await (const msg of query({
    prompt: `
      Check if the latest commit on main has passing CI, and if so, deploy to Vercel.

      Steps:
      1. Use mcp__github__list_commits to get the latest commit SHA on main for ${owner}/${repo}
      2. Use Bash to run "gh run list --branch main --limit 1 --json status,conclusion"
         to check the CI status of the latest commit
      3. If CI status is not "completed" with conclusion "success", STOP and report
         "CI not green — skipping deployment"
      4. If CI is green:
         a. Use mcp__vercel__list_projects to find the project ID for this repo
         b. Use mcp__vercel__create_deployment with target "production" and the commit ref
         c. Wait 10 seconds, then use mcp__vercel__list_deployments to check the new
            deployment status
         d. If status is "ERROR", use mcp__vercel__get_deployment_logs to fetch the
            build logs and report the failure
         e. If status is "READY", report the deployment URL
      5. Use mcp__github__add_issue_comment to post the deployment status on the
         latest merged PR (find it via mcp__github__list_pull_requests with state "closed"
         and base "main")
    `,
    options: {
      maxTurns: 20,
      cwd: projectDir,
      allowedTools: [
        "Bash",
        "mcp__github__list_commits",
        "mcp__github__list_pull_requests",
        "mcp__github__add_issue_comment",
        "mcp__vercel__list_projects",
        "mcp__vercel__create_deployment",
        "mcp__vercel__list_deployments",
        "mcp__vercel__get_deployment_logs",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream processing
  }
}
```

### Deploy workflow with allowedTools scoping

When building deploy agents, scope `allowedTools` tightly. A deploy agent should not have access to `create_env` (which could overwrite production secrets) or domain management unless specifically needed:

```typescript
// Read-only deployment monitoring — safest
const monitorTools = [
  "mcp__vercel__list_projects",
  "mcp__vercel__list_deployments",
  "mcp__vercel__get_deployment_logs",
];

// Deploy with monitoring — moderate privilege
const deployTools = [
  ...monitorTools,
  "mcp__vercel__create_deployment",
];

// Full management — high privilege, require human approval
const fullTools = [
  ...deployTools,
  "mcp__vercel__create_env",
  "mcp__vercel__list_domains",
];
```

Use the narrowest set that satisfies the workflow. Escalate to a broader set only when the narrow set is insufficient and the escalation is justified.

---

## 7. Firebase MCP Server

### Configuration

```jsonc
{
  "mcpServers": {
    "firebase": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "firebase-tools@14.2.0",
        "experimental:mcp"
      ],
      "env": {
        "GOOGLE_APPLICATION_CREDENTIALS": "${GOOGLE_APPLICATION_CREDENTIALS}",
        "FIREBASE_PROJECT_ID": "${FIREBASE_PROJECT_ID}"
      }
    }
  }
}
```

`GOOGLE_APPLICATION_CREDENTIALS` points to a service account JSON key file. `FIREBASE_PROJECT_ID` specifies which Firebase project the tools operate on.

The `experimental:mcp` subcommand of `firebase-tools` launches the MCP server. This is an official Firebase feature but may change interface between major `firebase-tools` versions — hence the version pin.

### Firestore tools

```
mcp__firebase__firestore_get_document
  collection: string, document_id: string
  Returns: document data with all fields and metadata

mcp__firebase__firestore_list_documents
  collection: string, limit?: number, order_by?: string, where?: Array<{field: string, op: string, value: any}>
  Returns: array of documents matching the query

mcp__firebase__firestore_create_document
  collection: string, data: Record<string, any>, document_id?: string
  Returns: created document with generated or specified ID

mcp__firebase__firestore_update_document
  collection: string, document_id: string, data: Record<string, any>, merge?: boolean
  Returns: updated document
  NOTE: Without merge:true, this REPLACES the entire document.
        Always use merge:true unless you explicitly want a full replace.

mcp__firebase__firestore_delete_document
  collection: string, document_id: string
  Returns: confirmation of deletion
  WARNING: Irreversible. Agents should rarely have delete permissions.
```

### Auth tools

```
mcp__firebase__auth_list_users
  limit?: number, page_token?: string
  Returns: array of UserRecord objects

mcp__firebase__auth_get_user
  uid: string
  Returns: UserRecord with email, displayName, disabled status, metadata

mcp__firebase__auth_create_user
  email: string, password?: string, display_name?: string, disabled?: boolean
  Returns: created UserRecord

mcp__firebase__auth_delete_user
  uid: string
  Returns: confirmation
  WARNING: Irreversible. Deletes the user and all associated auth data.
```

### Security: least-privilege IAM service account

Never use a service account with `roles/owner` or `roles/editor` for an agent. Create a dedicated service account with only the permissions the agent needs:

```bash
# Create a dedicated service account for the agent
gcloud iam service-accounts create claude-agent \
  --display-name="Claude Code Agent" \
  --project=${FIREBASE_PROJECT_ID}

# Grant only Firestore read/write (not admin)
gcloud projects add-iam-policy-binding ${FIREBASE_PROJECT_ID} \
  --member="serviceAccount:claude-agent@${FIREBASE_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/datastore.user"

# Grant only Auth read (if the agent doesn't need to create/delete users)
gcloud projects add-iam-policy-binding ${FIREBASE_PROJECT_ID} \
  --member="serviceAccount:claude-agent@${FIREBASE_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/firebaseauth.viewer"

# Generate a key file for the service account
gcloud iam service-accounts keys create /path/to/claude-agent-key.json \
  --iam-account=claude-agent@${FIREBASE_PROJECT_ID}.iam.gserviceaccount.com
```

Then set `GOOGLE_APPLICATION_CREDENTIALS=/path/to/claude-agent-key.json` in your environment.

**IAM role reference for Firebase agents:**

| Role | Permissions | When to use |
|------|-------------|-------------|
| `roles/datastore.viewer` | Firestore read only | Read-only data analysis agents |
| `roles/datastore.user` | Firestore read + write | Agents that create/update documents |
| `roles/firebaseauth.viewer` | Auth read only | Agents that look up user info |
| `roles/firebaseauth.admin` | Auth full access | Agents that manage users (use cautiously) |
| `roles/firebase.viewer` | Read-only across Firebase | Monitoring/reporting agents |

**Never combine `roles/datastore.admin` with `roles/firebaseauth.admin`** for an agent. That combination allows deletion of all data and all users — a catastrophic blast radius for a confused agent. If an agent needs both, it should be two separate agents with separate service accounts, each with minimal permissions.

### Firestore agent workflow example

```typescript
import { query } from "@anthropic-ai/claude-code";

async function migrateFirestoreSchema(
  collection: string,
  migrationSpecPath: string
): Promise<void> {
  for await (const msg of query({
    prompt: `
      Run a Firestore schema migration as specified in "${migrationSpecPath}".

      Steps:
      1. Read the migration spec at "${migrationSpecPath}". It contains:
         { collection, add_fields: [{name, default_value, type}], rename_fields: [{old, new}] }
      2. Use mcp__firebase__firestore_list_documents to list all documents in
         collection "${collection}" (paginate with limit 100 if needed)
      3. For each document:
         a. Add any missing fields with their default values
         b. Rename fields as specified (copy value to new key, remove old key)
         c. Use mcp__firebase__firestore_update_document with merge:true
            to apply changes without overwriting unrelated fields
      4. Output a summary: total documents processed, documents modified,
         documents already conforming, any errors encountered

      IMPORTANT: Use merge:true on every update to avoid data loss.
      If any update fails, log the document ID and continue — do not abort.
    `,
    options: {
      maxTurns: 50,  // High turn limit for pagination
      allowedTools: [
        "Read",
        "mcp__firebase__firestore_list_documents",
        "mcp__firebase__firestore_get_document",
        "mcp__firebase__firestore_update_document",
      ],
      model: "claude-sonnet-4-6",
    },
  })) {
    // Stream processing
  }
}
```

---

## 8. Multi-Server Composition

### Tool namespacing prevents conflicts

When you configure multiple MCP servers, each tool is prefixed with `mcp__<serverName>__`. This means you can safely run 5+ servers simultaneously without tool name collisions. A `list` tool on GitHub (`mcp__github__list_pull_requests`) is unambiguous from a `list` tool on Vercel (`mcp__vercel__list_deployments`).

This namespacing is automatic — you don't configure it. The server name in your mcp.json becomes the namespace.

### Context cost of tool schemas

Every MCP tool's schema (name, description, input parameters, types) is included in the system prompt. This is invisible but real context consumption. Rough estimates:

| Servers | Approx. tool count | Schema overhead |
|---------|-------------------|-----------------|
| 1 server | 5-12 tools | ~2-4K tokens |
| 2 servers | 15-25 tools | ~5-8K tokens |
| 3 servers | 25-40 tools | ~8-15K tokens |
| 5 servers | 50-70 tools | ~15-25K tokens |
| 5+ servers + built-in tools | 80-100+ tools | ~25-40K tokens |

At 5 servers with all built-in tools, you're spending 25-40K tokens just on tool schemas before the conversation even starts. On a 200K context window, that's 12-20% of your budget consumed by tool definitions. This directly reduces the context available for actual work — file contents, conversation history, reasoning.

**Mitigation strategies:**

1. **Don't configure servers you're not using.** If a project doesn't need Stripe, don't include it in project-level mcp.json.

2. **Use `allowedTools` in SDK agents to scope tool availability.** Even though all tools are loaded, the agent only sees the ones in `allowedTools`:

```typescript
// Agent sees ONLY these tools — not the full 70+ tool catalog
const options: SDKOptions = {
  allowedTools: [
    "Read", "Write", "Edit", "Glob", "Grep",
    "mcp__github__create_pull_request",
    "mcp__github__push_files",
  ],
};
```

3. **Inline MCP override in SDK for context-constrained agents.** Instead of loading servers from mcp.json (which loads ALL configured servers), pass only the servers each agent needs via `mcpServers` in the SDK:

```typescript
import { query, type SDKOptions } from "@anthropic-ai/claude-code";

const options: SDKOptions = {
  maxTurns: 10,
  allowedTools: [
    "Read", "Write",
    "mcp__github__create_issue",
    "mcp__github__add_issue_comment",
  ],
  mcpServers: {
    github: {
      type: "stdio",
      command: "npx",
      args: ["-y", "@anthropic-ai/github-mcp-server@1.2.0"],
      env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN! },
    },
    // Only GitHub — no Figma, Stripe, Vercel, Firebase schemas loaded
  },
};
```

This is essential for subagents in multi-agent systems. The orchestrator might configure 5 servers, but each worker agent should only receive the 1-2 servers it actually needs. This dramatically reduces per-agent context overhead.

### Cross-server workflow example: feature release pipeline

This is the canonical multi-server composition pattern — a release pipeline that spans GitHub, Stripe, Vercel, and back to GitHub:

```typescript
import { query } from "@anthropic-ai/claude-code";

interface ReleaseConfig {
  owner: string;
  repo: string;
  prNumber: number;
  vercelProjectId: string;
  stripeProductName?: string;
}

async function featureReleasePipeline(config: ReleaseConfig): Promise<void> {
  // Phase 1: Validate PR and gather context
  const contextMessages: any[] = [];
  for await (const msg of query({
    prompt: `
      Gather release context for PR #${config.prNumber} in ${config.owner}/${config.repo}.

      1. Use mcp__github__list_pull_requests to verify PR #${config.prNumber} exists and is merged
      2. Use mcp__github__list_commits to get all commits in the PR
      3. Read the PR body for any Stripe product changes or deployment notes
      4. Summarize: what changed, does it include pricing changes, is it ready for deploy?

      Output structured JSON:
      { "merged": boolean, "commitCount": number, "hasPricingChanges": boolean,
        "summary": string, "deployReady": boolean }
    `,
    options: {
      maxTurns: 10,
      allowedTools: [
        "mcp__github__list_pull_requests",
        "mcp__github__list_commits",
        "mcp__github__get_file_contents",
      ],
      mcpServers: {
        github: {
          type: "stdio",
          command: "npx",
          args: ["-y", "@anthropic-ai/github-mcp-server@1.2.0"],
          env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN! },
        },
      },
      model: "claude-sonnet-4-6",
    },
  })) {
    contextMessages.push(msg);
  }

  // Phase 2: If pricing changes detected, update Stripe
  // (only runs if Phase 1 found hasPricingChanges: true)
  for await (const msg of query({
    prompt: `
      The merged PR includes pricing changes. Update Stripe products/prices.

      1. Read the pricing spec file that was changed in the PR
      2. Use mcp__stripe__create_product for any new products
      3. Use mcp__stripe__create_price for any new prices
      4. Output the new Stripe IDs for deployment configuration
    `,
    options: {
      maxTurns: 15,
      allowedTools: [
        "Read",
        "mcp__stripe__create_product",
        "mcp__stripe__create_price",
      ],
      mcpServers: {
        stripe: {
          type: "stdio",
          command: "npx",
          args: ["-y", "@anthropic-ai/stripe-mcp-server@2.0.1"],
          env: { STRIPE_SECRET_KEY: process.env.STRIPE_TEST_SECRET_KEY! },
        },
      },
      model: "claude-sonnet-4-6",
    },
  })) {
    // Collect Stripe IDs
  }

  // Phase 3: Deploy to Vercel
  for await (const msg of query({
    prompt: `
      Deploy the merged PR to Vercel production.

      1. Use mcp__vercel__create_deployment for project "${config.vercelProjectId}"
         with target "production"
      2. Poll mcp__vercel__list_deployments every 15 seconds until the deployment
         is READY or ERROR
      3. If ERROR, use mcp__vercel__get_deployment_logs to get failure details
      4. Output: deployment URL or error details
    `,
    options: {
      maxTurns: 15,
      allowedTools: [
        "mcp__vercel__create_deployment",
        "mcp__vercel__list_deployments",
        "mcp__vercel__get_deployment_logs",
      ],
      mcpServers: {
        vercel: {
          type: "stdio",
          command: "npx",
          args: ["-y", "@anthropic-ai/vercel-mcp-server@1.1.0"],
          env: {
            VERCEL_TOKEN: process.env.VERCEL_TOKEN!,
            VERCEL_TEAM_ID: process.env.VERCEL_TEAM_ID!,
          },
        },
      },
      model: "claude-sonnet-4-6",
    },
  })) {
    // Collect deployment URL
  }

  // Phase 4: Post deployment status back to GitHub
  for await (const msg of query({
    prompt: `
      Post a deployment summary comment on PR #${config.prNumber} in ${config.owner}/${config.repo}.

      Use mcp__github__add_issue_comment with a markdown body containing:
      - Deployment status (success/failure)
      - Deployment URL
      - Any Stripe products/prices that were created
      - Commit count and summary from Phase 1
    `,
    options: {
      maxTurns: 5,
      allowedTools: ["mcp__github__add_issue_comment"],
      mcpServers: {
        github: {
          type: "stdio",
          command: "npx",
          args: ["-y", "@anthropic-ai/github-mcp-server@1.2.0"],
          env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN! },
        },
      },
      model: "claude-sonnet-4-6",
    },
  })) {
    // Done
  }
}
```

Key design decisions in this pipeline:
- **Each phase gets its own `query()` call** with isolated context and minimal tools
- **Each phase configures only the MCP server it needs** via `mcpServers` — no cross-phase tool leakage
- **`allowedTools` is scoped per phase** — the deploy agent can't create Stripe products, the Stripe agent can't deploy
- **Model selection per phase** — use Sonnet for routine tasks, reserve Opus for complex reasoning phases

### Inline MCP override vs mcp.json

| Approach | Pros | Cons |
|----------|------|------|
| `mcp.json` (file config) | Simple setup, shared across interactive and SDK use, project-level sharing | All servers loaded for all agents, no per-agent scoping |
| `mcpServers` (inline SDK) | Per-agent server scoping, minimal context overhead, dynamic config | More code, servers duplicated across agents, no interactive use |

**Recommendation**: Use `mcp.json` for interactive Claude Code use and for simple single-agent SDK scripts. Use inline `mcpServers` for multi-agent systems where context budget and tool isolation matter.

---

## 9. Auth Patterns and Credential Management

### Always ${ENV_VAR} in mcp.json

Every secret in mcp.json must use the `${ENV_VAR}` syntax. This reads the value from the process environment at server startup. The mcp.json file itself contains only the variable name, never the value.

```jsonc
// CORRECT: references an environment variable
"env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }

// WRONG: hardcoded secret — NEVER do this
"env": { "GITHUB_TOKEN": "ghp_YOUR_TOKEN_HERE" }
```

Even for user-level `~/.claude/mcp.json` that is never committed to git, hardcoding secrets is wrong because:
- It survives in file system backups and Time Machine snapshots
- It creates a single point of failure for rotation (you must edit the file)
- It makes auditing impossible — you can't grep for credential usage patterns
- It violates the principle of separation between configuration and secrets

### .env.local (never committed) pattern

The simplest credential setup for local development:

```bash
# <project>/.env.local — add to .gitignore
GITHUB_TOKEN=ghp_PLACEHOLDER_REPLACE_ME
FIGMA_ACCESS_TOKEN=figd_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
STRIPE_TEST_SECRET_KEY=sk_test_PLACEHOLDER_REPLACE_ME
VERCEL_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxx
FIREBASE_PROJECT_ID=my-project-dev
GOOGLE_APPLICATION_CREDENTIALS=/Users/you/.config/gcloud/claude-agent-key.json
```

```bash
# .gitignore — MUST include these
.env.local
.env.*.local
*.key.json
```

Load these variables before starting Claude Code:

```bash
# Option 1: source in shell profile (~/.zshrc or ~/.bashrc)
export $(grep -v '^#' ~/projects/myapp/.env.local | xargs)

# Option 2: direnv (recommended for project-specific env)
# Install direnv, add to shell profile, then:
# <project>/.envrc
dotenv .env.local

# Option 3: wrapper script
#!/bin/bash
set -a
source .env.local
set +a
exec claude "$@"
```

### Token rotation strategy with secrets manager

For production agent systems, manual `.env.local` management doesn't scale. Use a secrets manager:

**AWS Secrets Manager / Parameter Store:**

```bash
#!/bin/bash
# rotate-mcp-tokens.sh — run from CI or cron

# Fetch current tokens from secrets manager
export GITHUB_TOKEN=$(aws secretsmanager get-secret-value \
  --secret-id claude-agent/github-token \
  --query SecretString --output text)

export STRIPE_TEST_SECRET_KEY=$(aws secretsmanager get-secret-value \
  --secret-id claude-agent/stripe-test-key \
  --query SecretString --output text)

export VERCEL_TOKEN=$(aws secretsmanager get-secret-value \
  --secret-id claude-agent/vercel-token \
  --query SecretString --output text)

# Launch Claude Code with fresh tokens
exec claude "$@"
```

**1Password CLI:**

```bash
#!/bin/bash
# Uses 1Password CLI to inject secrets at startup

export GITHUB_TOKEN=$(op read "op://Development/GitHub Agent Token/credential")
export STRIPE_TEST_SECRET_KEY=$(op read "op://Development/Stripe Test Key/credential")
export VERCEL_TOKEN=$(op read "op://Development/Vercel Token/credential")
export FIGMA_ACCESS_TOKEN=$(op read "op://Development/Figma Token/credential")

exec claude "$@"
```

**HashiCorp Vault:**

```bash
#!/bin/bash
# Fetch short-lived tokens from Vault

export GITHUB_TOKEN=$(vault kv get -field=token secret/claude-agent/github)
export STRIPE_TEST_SECRET_KEY=$(vault kv get -field=key secret/claude-agent/stripe-test)

exec claude "$@"
```

### Per-environment credentials (test vs production)

Never share credentials between test and production environments. Use separate tokens with different permission levels:

```bash
# .env.local (development)
GITHUB_TOKEN=ghp_dev_xxxxx              # Fine-grained token scoped to dev repos only
STRIPE_TEST_SECRET_KEY=sk_test_xxxxx    # Test-mode key — no real charges
VERCEL_TOKEN=xxxxx                       # Token scoped to preview deployments only
FIREBASE_PROJECT_ID=myapp-dev            # Dev Firebase project
GOOGLE_APPLICATION_CREDENTIALS=./keys/dev-agent.json  # Dev service account

# .env.staging (staging — loaded by CI for staging agents)
GITHUB_TOKEN=ghp_staging_xxxxx          # Fine-grained token scoped to staging repos
STRIPE_TEST_SECRET_KEY=sk_test_xxxxx    # Still test-mode — staging never uses live keys
VERCEL_TOKEN=xxxxx                       # Token scoped to staging team
FIREBASE_PROJECT_ID=myapp-staging        # Staging Firebase project
GOOGLE_APPLICATION_CREDENTIALS=./keys/staging-agent.json

# .env.production (production — loaded by CI for production agents)
# NOTE: Stripe key is STILL a test key for agents. Only the payment server
# (not the agent) should ever hold a live Stripe key.
GITHUB_TOKEN=ghp_prod_xxxxx             # Fine-grained token, read-only on prod repos
STRIPE_TEST_SECRET_KEY=sk_test_xxxxx    # Test key — agents never get live keys
VERCEL_TOKEN=xxxxx                       # Token with production deploy permissions
FIREBASE_PROJECT_ID=myapp-production     # Production Firebase project
GOOGLE_APPLICATION_CREDENTIALS=./keys/prod-agent.json  # Production SA with minimal IAM
```

**Key principle**: the environment controls the blast radius. A dev agent with dev tokens can only damage dev data. A staging agent with staging tokens can only damage staging. Credential boundaries enforce isolation even when code has bugs.

### Short-lived tokens via vault injection on startup

The gold standard for agent credential management: tokens that expire within minutes, injected fresh at process start, and revoked on process exit.

```typescript
import { query, type SDKOptions } from "@anthropic-ai/claude-code";
import { execSync } from "child_process";

interface VaultTokens {
  githubToken: string;
  stripeKey: string;
  vercelToken: string;
}

function fetchShortLivedTokens(): VaultTokens {
  // Vault generates tokens with 15-minute TTL
  const githubToken = execSync(
    "vault read -field=token auth/github/agent-token ttl=15m",
    { encoding: "utf-8" }
  ).trim();

  const stripeKey = execSync(
    "vault kv get -field=key secret/claude-agent/stripe-test",
    { encoding: "utf-8" }
  ).trim();

  const vercelToken = execSync(
    "vault read -field=token auth/vercel/agent-token ttl=15m",
    { encoding: "utf-8" }
  ).trim();

  return { githubToken, stripeKey, vercelToken };
}

async function runAgentWithShortLivedTokens(prompt: string): Promise<void> {
  const tokens = fetchShortLivedTokens();

  const options: SDKOptions = {
    maxTurns: 20,
    mcpServers: {
      github: {
        type: "stdio",
        command: "npx",
        args: ["-y", "@anthropic-ai/github-mcp-server@1.2.0"],
        env: { GITHUB_TOKEN: tokens.githubToken },  // Expires in 15 minutes
      },
      stripe: {
        type: "stdio",
        command: "npx",
        args: ["-y", "@anthropic-ai/stripe-mcp-server@2.0.1"],
        env: { STRIPE_SECRET_KEY: tokens.stripeKey },
      },
      vercel: {
        type: "stdio",
        command: "npx",
        args: ["-y", "@anthropic-ai/vercel-mcp-server@1.1.0"],
        env: { VERCEL_TOKEN: tokens.vercelToken },
      },
    },
    model: "claude-sonnet-4-6",
  };

  const controller = new AbortController();

  // Hard timeout: kill the agent before tokens expire
  const tokenExpiryTimeout = setTimeout(() => {
    console.error("Token expiry approaching — aborting agent.");
    controller.abort();
  }, 14 * 60 * 1000); // 14 minutes (1 minute before 15m TTL)

  try {
    for await (const msg of query({
      prompt,
      abortController: controller,
      options,
    })) {
      // Process messages
    }
  } finally {
    clearTimeout(tokenExpiryTimeout);
    // Tokens auto-expire — no explicit revocation needed with Vault TTL
  }
}
```

**Why short-lived tokens matter:**
- If an agent process crashes or hangs, leaked tokens expire automatically
- Token theft window is minutes, not months
- Audit logs show exactly when each token was issued and for which agent run
- No rotation coordination needed — every run gets fresh tokens
- Revocation is implicit — just don't issue new tokens

### Credential validation hook

Add a startup validation hook that verifies all expected credentials are present before any MCP server starts:

```bash
#!/bin/bash
# .claude/hooks/pre-tool-use/validate-credentials.sh
# Run on first MCP tool call to verify credentials are loaded

REQUIRED_VARS=("GITHUB_TOKEN" "STRIPE_TEST_SECRET_KEY" "VERCEL_TOKEN")
MISSING=()

for var in "${REQUIRED_VARS[@]}"; do
  if [[ -z "${!var}" ]]; then
    MISSING+=("$var")
  fi
done

if [[ ${#MISSING[@]} -gt 0 ]]; then
  echo "ERROR: Missing required credentials: ${MISSING[*]}" >&2
  echo "Set these in .env.local or your environment before starting Claude Code." >&2
  exit 1
fi

# Validate Stripe key is test-mode
if [[ "${STRIPE_TEST_SECRET_KEY}" == sk_live_* ]]; then
  echo "BLOCKED: STRIPE_TEST_SECRET_KEY contains a live key. Use sk_test_* only." >&2
  exit 1
fi

exit 0
```

### Summary: credential management tiers

| Tier | Approach | Security | Effort | Use when |
|------|----------|----------|--------|----------|
| 1 — Basic | `.env.local` file | Low | Minimal | Solo developer, local dev only |
| 2 — Team | 1Password/Bitwarden CLI | Medium | Low | Small team, shared dev environment |
| 3 — Production | AWS Secrets Manager / GCP Secret Manager | High | Medium | Staging/production agents, CI/CD |
| 4 — Enterprise | Vault with short-lived tokens + auto-revocation | Very high | High | Regulated industries, multi-tenant, audited |

Start at Tier 1 for prototyping. Move to Tier 2 when you have teammates. Move to Tier 3 when agents run in CI/CD. Move to Tier 4 when you need compliance or handle sensitive data.

---

## Quick Reference: Server Configuration Cheat Sheet

```jsonc
// ─── Minimal mcp.json with all 5 servers ───
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/github-mcp-server@1.2.0"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "figma": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/figma-mcp-server@1.0.3"],
      "env": { "FIGMA_ACCESS_TOKEN": "${FIGMA_ACCESS_TOKEN}" }
    },
    "stripe": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/stripe-mcp-server@2.0.1"],
      "env": { "STRIPE_SECRET_KEY": "${STRIPE_TEST_SECRET_KEY}" }
    },
    "vercel": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/vercel-mcp-server@1.1.0"],
      "env": { "VERCEL_TOKEN": "${VERCEL_TOKEN}" }
    },
    "firebase": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "firebase-tools@14.2.0", "experimental:mcp"],
      "env": {
        "GOOGLE_APPLICATION_CREDENTIALS": "${GOOGLE_APPLICATION_CREDENTIALS}",
        "FIREBASE_PROJECT_ID": "${FIREBASE_PROJECT_ID}"
      }
    }
  }
}
```

**Required environment variables:**
```bash
GITHUB_TOKEN=ghp_...          # Fine-grained PAT, repo-scoped
FIGMA_ACCESS_TOKEN=figd_...   # Personal access token from Figma settings
STRIPE_TEST_SECRET_KEY=sk_test_...  # Test-mode secret key from Stripe dashboard
VERCEL_TOKEN=...              # API token from vercel.com/account/tokens
FIREBASE_PROJECT_ID=...       # Firebase project ID
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json  # IAM service account key
```
