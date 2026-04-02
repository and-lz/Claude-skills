# Building Custom MCP Servers

A dense reference for senior engineers building production MCP (Model Context Protocol) servers from scratch in 2026. Covers the protocol wire format, complete TypeScript and Python scaffolds, transport selection, resource serving, tool design principles, security hardening, testing strategies, and deployment patterns.

---

## 1. MCP Protocol Fundamentals

### What MCP Is

MCP is a standardized protocol that lets AI models discover and invoke external tools, read resources, and use prompt templates — all through a single, consistent JSON-RPC 2.0 interface. Instead of each AI platform inventing its own tool-calling format, MCP provides one protocol that any client (Claude Code, IDEs, custom agents) can use to communicate with any server (database tools, file tools, API wrappers, internal company tools).

The protocol is transport-agnostic: the same JSON-RPC messages flow identically over stdio pipes, Server-Sent Events, or HTTP streams. The server implements capabilities; the client discovers and invokes them. Neither side needs to know implementation details of the other.

### JSON-RPC 2.0 Wire Format

Every MCP message is a JSON-RPC 2.0 object. Three message types exist:

**Request** — client or server asks for something:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_files",
    "arguments": { "query": "TODO", "path": "./src" }
  }
}
```

**Response** — answer to a request:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{ "type": "text", "text": "Found 3 matches..." }],
    "isError": false
  }
}
```

**Notification** — one-way message, no `id` field, no response expected:
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": { "uri": "file:///project/config.json" }
}
```

### Message Flow

A complete MCP session follows this lifecycle:

```
Client                                          Server
  |                                               |
  |  ---- initialize (capabilities, version) ---> |
  |  <--- initialize result (server caps) ------- |
  |  ---- notifications/initialized ------------> |
  |                                               |
  |  ---- tools/list ---------------------------> |
  |  <--- tool definitions [ ] ------------------- |
  |                                               |
  |  ---- resources/list -----------------------> |
  |  <--- resource definitions [ ] --------------- |
  |                                               |
  |  ---- prompts/list -------------------------> |
  |  <--- prompt definitions [ ] ----------------- |
  |                                               |
  |  ==== Session active — tool calls begin ===== |
  |                                               |
  |  ---- tools/call { name, arguments } -------> |
  |  <--- result { content[], isError } ---------- |
  |                                               |
  |  ---- resources/read { uri } ---------------> |
  |  <--- resource contents ---------------------- |
  |                                               |
  |  <--- notifications/resources/updated -------- |  (server push)
  |                                               |
  |  ---- tools/call { name, arguments } -------> |
  |  <--- result { content[], isError } ---------- |
  |                                               |
  |  ==== Session ends ========================== |
```

### Key Methods

| Method | Direction | Purpose |
|--------|-----------|---------|
| `initialize` | Client → Server | Exchange protocol version and capabilities |
| `notifications/initialized` | Client → Server | Client confirms initialization complete |
| `tools/list` | Client → Server | Discover available tools and their input schemas |
| `tools/call` | Client → Server | Invoke a tool with arguments |
| `resources/list` | Client → Server | Discover available resources |
| `resources/read` | Client → Server | Read a specific resource by URI |
| `resources/templates/list` | Client → Server | Discover URI templates for dynamic resources |
| `prompts/list` | Client → Server | Discover available prompt templates |
| `prompts/get` | Client → Server | Retrieve a specific prompt with arguments |
| `notifications/resources/updated` | Server → Client | Notify that a resource has changed |
| `notifications/tools/list_changed` | Server → Client | Notify that available tools have changed |

### Capabilities Negotiation

During `initialize`, both sides declare what they support:

```json
{
  "capabilities": {
    "tools": {},
    "resources": { "subscribe": true },
    "prompts": {},
    "logging": {}
  }
}
```

A server that only exposes tools omits `resources` and `prompts` from its capabilities. Clients only call methods for capabilities the server declared.

### 2026 Protocol Additions

**Tasks Primitive**: Long-running operations that report progress. Instead of blocking on a single `tools/call` response, the server creates a task, returns a task ID immediately, and the client polls or subscribes for progress updates. Useful for operations that take seconds to minutes (CI builds, large file processing, database migrations).

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "tasks/create",
  "params": {
    "name": "run_migration",
    "arguments": { "migration": "20260401_add_users_table" }
  }
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "taskId": "task_abc123",
    "status": "running",
    "progress": { "percent": 0, "message": "Starting migration..." }
  }
}
```

**StreamableHTTP Transport**: Replaces SSE for new deployments. Bidirectional streaming over standard HTTP — the server can push notifications and progress updates without the SSE reconnection overhead. Supports horizontal scaling behind load balancers because session state is not tied to a specific connection. All new enterprise deployments in 2026 should prefer StreamableHTTP over SSE.

**Enterprise Audit Events**: Servers can emit structured audit events that enterprise MCP clients capture for compliance logging. Every tool invocation, resource access, and authentication event can be tagged with organizational metadata (team, project, cost center) and routed to centralized audit infrastructure.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/audit",
  "params": {
    "event": "tool_invoked",
    "tool": "query_database",
    "actor": "agent:build-pipeline-01",
    "timestamp": "2026-04-01T14:30:00Z",
    "metadata": { "team": "platform", "costCenter": "eng-tools" }
  }
}
```

---

## 2. TypeScript MCP Server — Complete Scaffold

### Setup

```bash
mkdir my-mcp-server && cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node
npx tsc --init --target ES2022 --module NodeNext --moduleResolution NodeNext --outDir dist --rootDir src
```

Add to `package.json`:
```json
{
  "type": "module",
  "bin": {
    "my-mcp-server": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

### Complete Working Server (~120 lines)

This server exposes three tools: `search_files` (grep through a directory), `read_file` (return file contents), and `query_database` (run a read-only SQL query). It includes Zod input validation, path traversal protection, and structured error handling.

```typescript
#!/usr/bin/env node

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { z } from "zod";
import { readFile, readdir, stat } from "node:fs/promises";
import { resolve, join } from "node:path";

// --- Configuration ---
const ALLOWED_DIR = resolve(process.env.ALLOWED_DIR || process.cwd());
const MAX_FILE_SIZE = 1024 * 1024; // 1 MB

// --- Path security ---
function safePath(userPath: string): string {
  const resolved = resolve(ALLOWED_DIR, userPath);
  if (!resolved.startsWith(ALLOWED_DIR + "/") && resolved !== ALLOWED_DIR) {
    throw new Error(
      `Path traversal blocked: "${userPath}" resolves outside allowed directory "${ALLOWED_DIR}". ` +
      `Provide a relative path within the project.`
    );
  }
  return resolved;
}

// --- Input schemas (Zod) ---
const SearchFilesInput = z.object({
  query: z.string().min(1).describe("Regex pattern to search for"),
  path: z.string().default(".").describe("Directory to search within (relative to project root)"),
  glob: z.string().default("*").describe("File glob filter, e.g. '*.ts'"),
});

const ReadFileInput = z.object({
  path: z.string().min(1).describe("File path relative to project root"),
  maxLines: z.number().int().positive().max(5000).default(500).describe("Max lines to return"),
});

const QueryDatabaseInput = z.object({
  sql: z.string().min(1).describe("Read-only SQL query (SELECT only)"),
  database: z.string().default("main").describe("Database name"),
});

// --- Server setup ---
const server = new Server(
  { name: "my-mcp-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// --- Tool definitions ---
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "search_files",
      description:
        "Search file contents using regex. Returns matching lines with file paths and line numbers. " +
        "Use this to find function definitions, imports, error messages, or any text pattern across the codebase.",
      inputSchema: {
        type: "object" as const,
        properties: {
          query: { type: "string", description: "Regex pattern to search for" },
          path: { type: "string", description: "Directory to search (relative to project root)", default: "." },
          glob: { type: "string", description: "File glob filter, e.g. '*.ts'", default: "*" },
        },
        required: ["query"],
      },
    },
    {
      name: "read_file",
      description:
        "Read the contents of a single file. Returns the text content with line numbers. " +
        "Use this after search_files to examine a specific file, or when you know the exact path.",
      inputSchema: {
        type: "object" as const,
        properties: {
          path: { type: "string", description: "File path relative to project root" },
          maxLines: { type: "number", description: "Max lines to return (default 500, max 5000)", default: 500 },
        },
        required: ["path"],
      },
    },
    {
      name: "query_database",
      description:
        "Execute a read-only SQL query against the project database. Only SELECT statements are allowed. " +
        "Returns results as a JSON array of row objects. Use this to inspect data, check counts, or verify state.",
      inputSchema: {
        type: "object" as const,
        properties: {
          sql: { type: "string", description: "SQL SELECT query" },
          database: { type: "string", description: "Database name (default: 'main')", default: "main" },
        },
        required: ["sql"],
      },
    },
  ],
}));

// --- Tool execution ---
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    switch (name) {
      case "search_files": {
        const input = SearchFilesInput.parse(args);
        const dir = safePath(input.path);
        const regex = new RegExp(input.query, "gi");

        // Simple recursive search (production: use ripgrep child process)
        const results: string[] = [];
        const entries = await readdir(dir, { recursive: true, withFileTypes: true });

        for (const entry of entries) {
          if (!entry.isFile()) continue;
          const filePath = join(entry.parentPath ?? dir, entry.name);
          const fileStat = await stat(filePath);
          if (fileStat.size > MAX_FILE_SIZE) continue;

          try {
            const content = await readFile(filePath, "utf-8");
            const lines = content.split("\n");
            lines.forEach((line, i) => {
              if (regex.test(line)) {
                const relPath = filePath.slice(ALLOWED_DIR.length + 1);
                results.push(`${relPath}:${i + 1}: ${line.trimEnd()}`);
              }
              regex.lastIndex = 0; // Reset regex state
            });
          } catch {
            // Skip unreadable files
          }

          if (results.length >= 200) {
            results.push(`\n... truncated at 200 matches. Narrow your query or path.`);
            break;
          }
        }

        return {
          content: [{ type: "text", text: results.length > 0 ? results.join("\n") : "No matches found." }],
        };
      }

      case "read_file": {
        const input = ReadFileInput.parse(args);
        const filePath = safePath(input.path);
        const fileStat = await stat(filePath);

        if (fileStat.size > MAX_FILE_SIZE) {
          return {
            content: [{
              type: "text",
              text: `File is ${(fileStat.size / 1024).toFixed(0)} KB, exceeding the 1 MB limit. ` +
                    `Use search_files to find the specific section you need.`,
            }],
            isError: true,
          };
        }

        const content = await readFile(filePath, "utf-8");
        const lines = content.split("\n");
        const truncated = lines.slice(0, input.maxLines);
        const numbered = truncated.map((line, i) => `${i + 1}\t${line}`).join("\n");

        const suffix = lines.length > input.maxLines
          ? `\n\n--- Showing ${input.maxLines} of ${lines.length} lines. Pass a higher maxLines to see more. ---`
          : "";

        return {
          content: [{ type: "text", text: numbered + suffix }],
        };
      }

      case "query_database": {
        const input = QueryDatabaseInput.parse(args);

        // Reject non-SELECT queries
        const normalized = input.sql.trim().toUpperCase();
        if (!normalized.startsWith("SELECT") && !normalized.startsWith("WITH") && !normalized.startsWith("EXPLAIN")) {
          return {
            content: [{
              type: "text",
              text: `Only SELECT, WITH, and EXPLAIN queries are allowed. ` +
                    `Received: "${input.sql.slice(0, 50)}...". Rewrite as a SELECT query.`,
            }],
            isError: true,
          };
        }

        // Placeholder: replace with your actual database connection
        // import Database from "better-sqlite3";
        // const db = new Database(`./data/${input.database}.db`, { readonly: true });
        // const rows = db.prepare(input.sql).all();
        const rows = [{ placeholder: "Replace with real database connection" }];

        return {
          content: [{ type: "text", text: JSON.stringify(rows, null, 2) }],
        };
      }

      default:
        return {
          content: [{
            type: "text",
            text: `Unknown tool: "${name}". Available tools: search_files, read_file, query_database.`,
          }],
          isError: true,
        };
    }
  } catch (error) {
    // Zod validation errors → actionable message
    if (error instanceof z.ZodError) {
      const issues = error.issues.map(
        (i) => `  - ${i.path.join(".")}: ${i.message}`
      ).join("\n");
      return {
        content: [{
          type: "text",
          text: `Invalid input for tool "${name}":\n${issues}\n\nCheck the tool's inputSchema and retry.`,
        }],
        isError: true,
      };
    }

    const msg = error instanceof Error ? error.message : String(error);
    return {
      content: [{ type: "text", text: `Tool "${name}" failed: ${msg}` }],
      isError: true,
    };
  }
});

// --- Start server ---
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error(`MCP server "${server.serverInfo.name}" running on stdio`);
}

main().catch((err) => {
  console.error("Fatal:", err);
  process.exit(1);
});
```

### Key Implementation Details

**`isError` flag**: When a tool call fails, set `isError: true` in the response. This tells the MCP client that the result is an error message, not valid tool output. The model uses this signal to understand it should retry or take a different approach rather than treating the error text as successful output.

**Zod validation → actionable messages**: Instead of letting Zod throw generic errors, catch `ZodError` and format the issues list with field paths and messages. The model can read these and self-correct its next tool call.

**Switch-based handler**: One `setRequestHandler(CallToolRequestSchema, ...)` with a `switch(name)` over tool names. This is the standard pattern — each case validates input, executes logic, and returns `{ content[], isError? }`.

**`safePath()` guard**: Every tool that touches the filesystem must call `safePath()` before any I/O. The guard resolves the path, checks that it's within `ALLOWED_DIR`, and throws an actionable error if not. This is the primary defense against path traversal attacks from hallucinated or injected tool arguments.

---

## 3. Python MCP Server — Complete Scaffold

### Setup

```bash
mkdir my-mcp-server && cd my-mcp-server
python -m venv .venv && source .venv/bin/activate
pip install mcp pydantic
```

### Complete Working Server

This server exposes two tools: `search_files` and `read_file`. It uses Pydantic for input validation with custom validators for path security, and runs over stdio.

```python
#!/usr/bin/env python3
"""MCP server with file search and read tools."""

import os
import re
from pathlib import Path
from typing import Any

from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent, CallToolResult
from pydantic import BaseModel, field_validator

# --- Configuration ---
ALLOWED_DIR = Path(os.environ.get("ALLOWED_DIR", os.getcwd())).resolve()
MAX_FILE_SIZE = 1024 * 1024  # 1 MB


# --- Path security ---
def safe_path(user_path: str) -> Path:
    """Resolve a user-provided path and verify it's within ALLOWED_DIR."""
    resolved = (ALLOWED_DIR / user_path).resolve()
    if not (str(resolved).startswith(str(ALLOWED_DIR) + os.sep) or resolved == ALLOWED_DIR):
        raise ValueError(
            f'Path traversal blocked: "{user_path}" resolves outside '
            f'allowed directory "{ALLOWED_DIR}". '
            f"Provide a relative path within the project."
        )
    return resolved


# --- Input models (Pydantic) ---
class SearchFilesInput(BaseModel):
    query: str
    path: str = "."
    max_results: int = 200

    @field_validator("query")
    @classmethod
    def query_not_empty(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("query must not be empty — provide a regex pattern to search for")
        return v

    @field_validator("path")
    @classmethod
    def path_is_safe(cls, v: str) -> str:
        safe_path(v)  # Raises ValueError if unsafe
        return v


class ReadFileInput(BaseModel):
    path: str
    max_lines: int = 500

    @field_validator("path")
    @classmethod
    def path_is_safe(cls, v: str) -> str:
        safe_path(v)  # Raises ValueError if unsafe
        return v

    @field_validator("max_lines")
    @classmethod
    def max_lines_in_range(cls, v: int) -> int:
        if v < 1 or v > 5000:
            raise ValueError("max_lines must be between 1 and 5000")
        return v


# --- Server setup ---
app = Server("my-mcp-server")


@app.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="search_files",
            description=(
                "Search file contents using regex. Returns matching lines with "
                "file paths and line numbers. Use this to find function definitions, "
                "imports, error messages, or any text pattern across the codebase."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Regex pattern to search for",
                    },
                    "path": {
                        "type": "string",
                        "description": "Directory to search (relative to project root)",
                        "default": ".",
                    },
                    "max_results": {
                        "type": "integer",
                        "description": "Maximum matches to return (default 200)",
                        "default": 200,
                    },
                },
                "required": ["query"],
            },
        ),
        Tool(
            name="read_file",
            description=(
                "Read the contents of a single file. Returns text content with "
                "line numbers. Use this after search_files to examine a specific "
                "file, or when you know the exact path."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "File path relative to project root",
                    },
                    "max_lines": {
                        "type": "integer",
                        "description": "Max lines to return (default 500, max 5000)",
                        "default": 500,
                    },
                },
                "required": ["path"],
            },
        ),
    ]


@app.call_tool()
async def call_tool(name: str, arguments: dict[str, Any]) -> list[TextContent]:
    try:
        if name == "search_files":
            return await _search_files(SearchFilesInput(**arguments))
        elif name == "read_file":
            return await _read_file(ReadFileInput(**arguments))
        else:
            raise ValueError(
                f'Unknown tool: "{name}". Available tools: search_files, read_file.'
            )
    except Exception as e:
        # Return error as content with isError semantics
        # The MCP Python SDK surfaces this as an error response
        raise


async def _search_files(input: SearchFilesInput) -> list[TextContent]:
    """Recursively search files for a regex pattern."""
    search_dir = safe_path(input.path)
    pattern = re.compile(input.query, re.IGNORECASE)
    results: list[str] = []

    for file_path in search_dir.rglob("*"):
        if not file_path.is_file():
            continue
        if file_path.stat().st_size > MAX_FILE_SIZE:
            continue

        try:
            content = file_path.read_text(encoding="utf-8", errors="replace")
        except (PermissionError, OSError):
            continue

        for i, line in enumerate(content.splitlines(), start=1):
            if pattern.search(line):
                rel = file_path.relative_to(ALLOWED_DIR)
                results.append(f"{rel}:{i}: {line.rstrip()}")

                if len(results) >= input.max_results:
                    results.append(
                        f"\n... truncated at {input.max_results} matches. "
                        f"Narrow your query or path."
                    )
                    return [TextContent(type="text", text="\n".join(results))]

    text = "\n".join(results) if results else "No matches found."
    return [TextContent(type="text", text=text)]


async def _read_file(input: ReadFileInput) -> list[TextContent]:
    """Read and return file contents with line numbers."""
    file_path = safe_path(input.path)

    if not file_path.is_file():
        raise FileNotFoundError(
            f'File not found: "{input.path}". Check the path and try again.'
        )

    size = file_path.stat().st_size
    if size > MAX_FILE_SIZE:
        raise ValueError(
            f"File is {size // 1024} KB, exceeding the 1 MB limit. "
            f"Use search_files to find the specific section you need."
        )

    content = file_path.read_text(encoding="utf-8", errors="replace")
    lines = content.splitlines()
    truncated = lines[: input.max_lines]
    numbered = "\n".join(f"{i}\t{line}" for i, line in enumerate(truncated, start=1))

    if len(lines) > input.max_lines:
        numbered += (
            f"\n\n--- Showing {input.max_lines} of {len(lines)} lines. "
            f"Pass a higher max_lines to see more. ---"
        )

    return [TextContent(type="text", text=numbered)]


# --- Entry point ---
async def main():
    async with stdio_server() as (read_stream, write_stream):
        await app.run(read_stream, write_stream, app.create_initialization_options())


if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### Key Implementation Details

**`stdio_server()` context manager**: The MCP Python SDK provides `stdio_server()` as an async context manager that handles the stdin/stdout transport setup. It yields `(read_stream, write_stream)` which you pass to `app.run()`. The context manager handles cleanup on exit.

**Pydantic `field_validator` for path security**: Path validation runs inside the model class itself, before any tool logic executes. This means every path is validated at parse time — you cannot accidentally skip the security check. The `@classmethod` decorator is required for Pydantic v2 validators.

**`ALLOWED_DIR` from environment**: The allowed directory is configurable via the `ALLOWED_DIR` environment variable, falling back to `cwd`. This lets the same server binary serve different project directories depending on how it's invoked. Set it in the MCP client configuration (see Transport Options below).

**Async handler pattern**: All tool handlers are `async` functions. Even though file I/O in this example uses synchronous `pathlib` calls (which is fine for local files), the async signature allows you to use `aiohttp`, `asyncpg`, or other async libraries for network-bound operations without blocking the event loop.

---

## 4. Transport Options

### stdio — Local Child Process

The simplest transport. The MCP client spawns the server as a child process. Messages flow over stdin/stdout. The server inherits the client's environment variables.

**Configuration** (in `.claude/mcp.json` or `~/.claude/mcp.json`):
```json
{
  "mcpServers": {
    "my-tools": {
      "type": "stdio",
      "command": "node",
      "args": ["./dist/index.js"],
      "env": {
        "ALLOWED_DIR": "/Users/me/projects/myapp"
      }
    }
  }
}
```

For a Python server:
```json
{
  "mcpServers": {
    "my-python-tools": {
      "type": "stdio",
      "command": "python",
      "args": ["./server.py"],
      "env": {
        "ALLOWED_DIR": "/Users/me/projects/myapp"
      }
    }
  }
}
```

**When to use**: Personal developer tools. Local project tooling. Prototyping. Any case where the server runs on the same machine as the client, latency must be minimal, and no other clients need to connect simultaneously.

**Advantages**: Zero network setup. Environment variable inheritance. Lowest possible latency. Server lifecycle tied to client — starts and stops automatically.

**Limitations**: Single client only. Cannot be shared across machines. Server must be installed locally.

### SSE — Server-Sent Events over HTTP

The server runs as an HTTP service. The client connects over HTTP to discover tools, and the server uses Server-Sent Events to stream responses and notifications.

**Server code** (TypeScript with Express):
```typescript
import express from "express";
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";

const app = express();
const server = new Server(
  { name: "my-sse-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// ... register tool handlers same as stdio example ...

// SSE endpoint — one transport per client connection
app.get("/sse", async (req, res) => {
  const transport = new SSEServerTransport("/messages", res);
  await server.connect(transport);
});

// Message endpoint — client posts JSON-RPC messages here
app.post("/messages", async (req, res) => {
  // The SSEServerTransport handles routing internally
  // This endpoint is referenced in the SSE stream's initial message
});

app.listen(3100, () => {
  console.log("MCP SSE server on http://localhost:3100");
});
```

**Client configuration**:
```json
{
  "mcpServers": {
    "team-tools": {
      "type": "sse",
      "url": "https://mcp.internal.company.com/sse",
      "headers": {
        "Authorization": "Bearer ${MCP_AUTH_TOKEN}"
      }
    }
  }
}
```

**When to use**: Team-shared servers. Hosted internal tools. Any case where multiple developers need access to the same MCP server from different machines. Services that need to persist between client sessions.

**Advantages**: Multi-client support. Server runs independently of any client. Can be hosted behind a reverse proxy with authentication. Server state persists across client connections.

**Limitations**: SSE is unidirectional (server → client only); the client posts messages via a separate HTTP endpoint. Connection drops require reconnection logic. Horizontal scaling is harder because SSE connections are stateful.

### StreamableHTTP — Bidirectional HTTP Streams (2026)

The newest transport. Uses standard HTTP with streaming request and response bodies for bidirectional communication. Designed for enterprise deployments that need horizontal scaling, load balancer compatibility, and server-initiated messages.

**Client configuration**:
```json
{
  "mcpServers": {
    "enterprise-tools": {
      "type": "streamable-http",
      "url": "https://mcp.prod.company.com/v1/stream",
      "headers": {
        "Authorization": "Bearer ${MCP_AUTH_TOKEN}",
        "X-Org-Id": "org_12345"
      }
    }
  }
}
```

**When to use**: Enterprise deployments requiring horizontal scaling. Production systems behind load balancers. Servers that need to push notifications proactively (e.g., resource change events, task progress). Any deployment where SSE's unidirectional limitation is a constraint.

**Advantages**: True bidirectional streaming. Stateless from the load balancer's perspective — any server instance can handle any request. Native support for server-initiated notifications without polling. Built-in session resumption after network interruptions.

**Limitations**: Newer protocol — library support is still maturing. More complex server implementation than stdio or SSE. Requires HTTP/2 or HTTP/3 for optimal performance.

### Transport Decision Matrix

| Factor | stdio | SSE | StreamableHTTP |
|--------|-------|-----|----------------|
| Latency | Lowest (IPC) | Low (HTTP) | Low (HTTP) |
| Clients | Single | Multiple | Multiple |
| Scaling | N/A | Vertical only | Horizontal |
| Auth | Env vars (inherited) | HTTP headers | HTTP headers |
| Setup complexity | Minimal | Moderate | Higher |
| Server lifecycle | Tied to client | Independent | Independent |
| Network required | No | Yes | Yes |
| Load balancer support | N/A | Stateful (sticky sessions) | Stateless (any backend) |
| Server push | N/A | Yes (SSE stream) | Yes (bidirectional) |
| Best for | Personal dev tools | Team-shared tools | Enterprise production |

**Rule of thumb**: Start with stdio. Move to SSE when you need team sharing. Move to StreamableHTTP when you need enterprise-grade scaling and reliability.

---

## 5. Resources — Serving Files and Data

Resources let an MCP server expose read-only data that clients can browse and retrieve — similar to a filesystem or API endpoint, but discoverable through the MCP protocol.

### When to Use Resources vs Tools

**Resources** are for data retrieval with known URIs — the client knows what it wants to read. Think of them like GET endpoints in a REST API. Examples: configuration files, database schemas, API documentation, log files.

**Tools** are for operations that require computation, have side effects, or where the client doesn't know what data exists until it searches. Think of them like POST endpoints. Examples: searching files, running queries, creating records.

If the model needs to browse or read data: resource. If the model needs to do something: tool.

### Static Resources

Resources with fixed URIs that are always available:

```typescript
import {
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

server.setRequestHandler(ListResourcesRequestSchema, async () => ({
  resources: [
    {
      uri: "config://app/database",
      name: "Database Configuration",
      description: "Current database connection settings (credentials redacted)",
      mimeType: "application/json",
    },
    {
      uri: "schema://app/users",
      name: "Users Table Schema",
      description: "Column definitions and constraints for the users table",
      mimeType: "application/json",
    },
    {
      uri: "docs://api/endpoints",
      name: "API Endpoint Reference",
      description: "All REST API endpoints with methods, paths, and response schemas",
      mimeType: "text/markdown",
    },
  ],
}));

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params;

  switch (uri) {
    case "config://app/database":
      return {
        contents: [{
          uri,
          mimeType: "application/json",
          text: JSON.stringify({
            host: process.env.DB_HOST,
            port: 5432,
            database: "myapp",
            ssl: true,
            // Credentials intentionally omitted
          }, null, 2),
        }],
      };

    case "schema://app/users":
      // Read from actual database metadata
      return {
        contents: [{
          uri,
          mimeType: "application/json",
          text: JSON.stringify({
            table: "users",
            columns: [
              { name: "id", type: "uuid", primaryKey: true },
              { name: "email", type: "text", unique: true, nullable: false },
              { name: "created_at", type: "timestamptz", default: "now()" },
            ],
          }, null, 2),
        }],
      };

    case "docs://api/endpoints":
      return {
        contents: [{
          uri,
          mimeType: "text/markdown",
          text: "# API Endpoints\n\n## GET /users\nReturns paginated user list...",
        }],
      };

    default:
      throw new Error(`Unknown resource: ${uri}`);
  }
});
```

### Dynamic Resources with URI Templates

For resources where the URI includes variable parts (like a file path), use URI templates. The client fills in the template parameters based on context.

```typescript
import {
  ListResourceTemplatesRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

server.setRequestHandler(ListResourceTemplatesRequestSchema, async () => ({
  resourceTemplates: [
    {
      uriTemplate: "file://project/{path}",
      name: "Project File",
      description: "Read any file in the project directory by its relative path",
      mimeType: "text/plain",
    },
    {
      uriTemplate: "log://app/{date}",
      name: "Application Log",
      description: "Application log for a specific date (YYYY-MM-DD format)",
      mimeType: "text/plain",
    },
  ],
}));

// The ReadResourceRequestSchema handler processes both static and template-based URIs
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params;

  // Match file://project/{path}
  const fileMatch = uri.match(/^file:\/\/project\/(.+)$/);
  if (fileMatch) {
    const filePath = safePath(decodeURIComponent(fileMatch[1]));
    const content = await readFile(filePath, "utf-8");
    return {
      contents: [{ uri, mimeType: "text/plain", text: content }],
    };
  }

  // Match log://app/{date}
  const logMatch = uri.match(/^log:\/\/app\/(\d{4}-\d{2}-\d{2})$/);
  if (logMatch) {
    const date = logMatch[1];
    const logPath = safePath(`logs/${date}.log`);
    const content = await readFile(logPath, "utf-8");
    return {
      contents: [{ uri, mimeType: "text/plain", text: content }],
    };
  }

  throw new Error(`Unknown resource URI: ${uri}`);
});
```

### Resource Subscriptions

Servers can notify clients when a resource has changed. The client subscribes to specific URIs, and the server sends `notifications/resources/updated` when those resources change.

```typescript
import {
  SubscribeRequestSchema,
  UnsubscribeRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { watch } from "node:fs";

const subscriptions = new Map<string, Set<string>>(); // uri → set of subscription IDs

server.setRequestHandler(SubscribeRequestSchema, async (request) => {
  const { uri } = request.params;

  // Start watching the underlying file
  const fileMatch = uri.match(/^file:\/\/project\/(.+)$/);
  if (fileMatch) {
    const filePath = safePath(decodeURIComponent(fileMatch[1]));

    const watcher = watch(filePath, () => {
      // Notify the client that this resource has changed
      server.notification({
        method: "notifications/resources/updated",
        params: { uri },
      });
    });

    // Store watcher for cleanup on unsubscribe
    if (!subscriptions.has(uri)) {
      subscriptions.set(uri, new Set());
    }
  }

  return {};
});
```

When a resource changes, the client receives the notification and can re-read the resource to get updated content. This enables real-time awareness of configuration changes, log updates, and data mutations without polling.

---

## 6. Tool Design Principles

These seven rules make the difference between MCP tools that work reliably with AI models and tools that cause constant frustration, hallucinated arguments, and cascading errors.

### Rule 1: One Tool, One Action

Each tool should do exactly one thing. Broad tools with mode parameters or multi-action semantics confuse the model about when to use what.

**Bad** — one tool doing three things:
```json
{
  "name": "manage_database",
  "description": "Manage the database. Set action to 'query', 'insert', or 'delete'.",
  "inputSchema": {
    "properties": {
      "action": { "type": "string", "enum": ["query", "insert", "delete"] },
      "sql": { "type": "string" },
      "table": { "type": "string" },
      "data": { "type": "object" }
    }
  }
}
```

**Good** — three focused tools:
```json
[
  {
    "name": "query_database",
    "description": "Execute a read-only SQL SELECT query. Returns results as JSON rows.",
    "inputSchema": {
      "properties": { "sql": { "type": "string" } },
      "required": ["sql"]
    }
  },
  {
    "name": "insert_record",
    "description": "Insert a single row into a table. Returns the inserted row with generated ID.",
    "inputSchema": {
      "properties": {
        "table": { "type": "string" },
        "data": { "type": "object" }
      },
      "required": ["table", "data"]
    }
  },
  {
    "name": "delete_record",
    "description": "Delete a single row by ID. Returns the deleted row for confirmation.",
    "inputSchema": {
      "properties": {
        "table": { "type": "string" },
        "id": { "type": "string" }
      },
      "required": ["table", "id"]
    }
  }
]
```

Granular tools can be individually included or excluded from `allowedTools`. A `manage_database` tool is all-or-nothing — you cannot give the agent read access without also giving it delete access.

### Rule 2: Descriptions Are for the LLM, Not the Human

The model reads tool descriptions to decide when and how to use each tool. Write descriptions that tell the model specifically when to reach for this tool and what it returns.

**Bad**:
```
"description": "Searches files."
```

**Good**:
```
"description": "Search file contents using regex across the project directory. Returns matching lines with file paths and line numbers. Use this to find function definitions, variable usages, import statements, error messages, or any text pattern. Returns up to 200 matches. If you need the full content of a specific file, use read_file instead."
```

Include in every description:
- What the tool does (one sentence)
- What it returns (format and content)
- When to use it (positive guidance)
- When NOT to use it and what to use instead (negative guidance, if ambiguous)

### Rule 3: Return Actionable Errors

When a tool fails, the error message should tell the model exactly what went wrong and how to fix it. The model cannot debug opaque errors.

**Bad**:
```json
{ "content": [{ "type": "text", "text": "Error: ENOENT" }], "isError": true }
```

**Good**:
```json
{
  "content": [{
    "type": "text",
    "text": "File not found: \"src/utils/helpers.ts\". The file does not exist at this path. Use search_files with query \"helpers\" to locate the correct file path."
  }],
  "isError": true
}
```

Pattern for actionable errors: **what failed** + **why it failed** + **what to do instead**. The third part is what makes errors actionable rather than just informational.

### Rule 4: Never Silently Truncate — Paginate

If a tool might return more data than fits in the context window, implement explicit pagination with a cursor. Never silently drop results — the model cannot reason about data it does not know exists.

```typescript
const PagedSearchInput = z.object({
  query: z.string(),
  path: z.string().default("."),
  cursor: z.string().optional().describe("Pagination cursor from a previous search result"),
  pageSize: z.number().default(50).describe("Results per page (default 50)"),
});

// In the handler:
const allResults = /* ... search logic ... */;
const startIndex = input.cursor ? parseInt(input.cursor, 10) : 0;
const page = allResults.slice(startIndex, startIndex + input.pageSize);
const nextCursor = startIndex + input.pageSize < allResults.length
  ? String(startIndex + input.pageSize)
  : null;

return {
  content: [{
    type: "text",
    text: JSON.stringify({
      results: page,
      totalCount: allResults.length,
      returnedCount: page.length,
      nextCursor,  // null means no more pages
    }),
  }],
};
```

When there are more results, the response includes `nextCursor`. The model passes it back in the next call to get the next page. When `nextCursor` is `null`, all results have been returned.

### Rule 5: Idempotency — Prefer create_or_update

Tools that create resources should be safe to call multiple times with the same input. If the resource already exists, update it rather than failing or creating a duplicate.

```typescript
case "create_or_update_config": {
  const input = ConfigInput.parse(args);
  const configPath = safePath(`config/${input.name}.json`);

  // Read existing config if it exists
  let existing: Record<string, unknown> = {};
  try {
    existing = JSON.parse(await readFile(configPath, "utf-8"));
  } catch {
    // File doesn't exist — will create
  }

  // Merge new values over existing
  const merged = { ...existing, ...input.values, updatedAt: new Date().toISOString() };
  await writeFile(configPath, JSON.stringify(merged, null, 2));

  return {
    content: [{
      type: "text",
      text: JSON.stringify({
        action: Object.keys(existing).length > 0 ? "updated" : "created",
        path: `config/${input.name}.json`,
        config: merged,
      }),
    }],
  };
}
```

Idempotent tools are safer because the model may retry a tool call after a timeout or ambiguous result. A non-idempotent `create_config` tool would fail on retry ("config already exists"), forcing the model to figure out the recovery path. A `create_or_update_config` tool handles retries transparently.

### Rule 6: Structured JSON Output Over Prose

Return structured JSON objects, not free-text prose. Structured output is easier for the model to parse, reference specific fields from, and use as input to subsequent tool calls.

**Bad**:
```
"The users table has 3 columns: id (uuid, primary key), email (text, unique, not null), and created_at (timestamptz, defaults to now)."
```

**Good**:
```json
{
  "table": "users",
  "columns": [
    { "name": "id", "type": "uuid", "primaryKey": true },
    { "name": "email", "type": "text", "unique": true, "nullable": false },
    { "name": "created_at", "type": "timestamptz", "default": "now()" }
  ],
  "rowCount": 15234
}
```

The model can extract specific fields from JSON (`columns[1].type`) but has to pattern-match and hope with prose. Structured output also compresses better in the context window — less tokens for more information.

### Rule 7: Input Validation Errors Must Be Specific and Fixable

When input validation fails, tell the model exactly which field is wrong, what the constraint is, and what a valid value looks like.

**Bad**:
```
"Invalid input"
```

**Good**:
```
Invalid input for tool "search_files":
  - query: String must contain at least 1 character(s). Provide a non-empty regex pattern.
  - max_results: Number must be less than or equal to 500. You provided 10000. Use a value between 1 and 500.
```

This is why Zod and Pydantic are critical — they produce structured error messages that include the field path, the constraint, and the violating value. Format these into a single error message and return with `isError: true`.

---

## 7. Security

Every MCP tool is a privilege boundary. The model's tool call arguments come from LLM reasoning, which can be influenced by adversarial content in files, web pages, or other tool results. Treat every input to every tool as untrusted.

### Path Traversal Protection

The most common vulnerability in file-serving MCP tools. The model might generate `../../etc/passwd` as a file path — either through hallucination or because adversarial content in a processed file suggested it.

**TypeScript guard** (use this in every file-touching tool):
```typescript
import { resolve } from "node:path";

const ALLOWED_DIR = resolve(process.env.ALLOWED_DIR || process.cwd());

function safePath(userPath: string): string {
  const resolved = resolve(ALLOWED_DIR, userPath);
  // Must start with ALLOWED_DIR + separator (not just prefix match)
  if (!resolved.startsWith(ALLOWED_DIR + "/") && resolved !== ALLOWED_DIR) {
    throw new Error(
      `Path traversal blocked: "${userPath}" resolves to "${resolved}" ` +
      `which is outside the allowed directory "${ALLOWED_DIR}". ` +
      `Provide a path relative to the project root.`
    );
  }
  return resolved;
}
```

**Why `resolve` + `startsWith`**: `path.resolve()` collapses `..` segments, resolves symlinks implicitly, and produces an absolute path. Then `startsWith(ALLOWED_DIR + "/")` ensures the result is genuinely inside the allowed directory. The `+ "/"` prevents a subtle bug where `/allowed` would pass the check for an allowed dir of `/allow`. The `=== ALLOWED_DIR` check handles the case where the user requests the root directory itself.

### Never Pass LLM Output to Shell

LLMs can hallucinate plausible-looking but dangerous shell commands, especially when processing adversarial content. The pattern `exec(modelOutput)` is always wrong.

**Bad** — shell injection vector:
```typescript
import { exec } from "node:child_process";

// NEVER DO THIS — model output goes directly to shell
exec(`grep -r "${input.query}" ${input.path}`);
```

If `input.query` is `"; rm -rf / #`, the shell executes the destructive command.

**Good** — argument array, no shell:
```typescript
import { execFile } from "node:child_process";

// Safe: arguments are passed as an array, never interpreted by shell
execFile("grep", ["-r", input.query, safePath(input.path)], (err, stdout) => {
  // ...
});
```

`execFile` passes each argument directly to the process without shell interpretation. There is no shell to inject into. Semicolons, backticks, pipes, and redirects are treated as literal strings.

For more complex cases, use a library that builds commands from validated parts:

```typescript
import { execa } from "execa";

const result = await execa("rg", [
  "--json",
  "--max-count", "200",
  input.query,
  safePath(input.path),
]);
```

### Output Sanitization

MCP tool responses go back into the model's context. If your tool reads files or database records that contain secrets, those secrets end up in the context window — visible in logs, potentially echoed in model output, and stored in conversation history.

```typescript
const SECRET_PATTERNS = [
  /(?:token|key|secret|password|credential|auth)[\s]*[=:]\s*["']?[^\s"']{8,}/gi,
  /(?:ghp|gho|ghu|ghs|ghr)_[A-Za-z0-9_]{36,}/g,  // GitHub tokens
  /sk-[A-Za-z0-9]{20,}/g,                           // OpenAI/Anthropic keys
  /eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}/g,   // JWTs
  /AKIA[A-Z0-9]{16}/g,                              // AWS access keys
];

function sanitizeOutput(text: string): string {
  let result = text;
  for (const pattern of SECRET_PATTERNS) {
    result = result.replace(pattern, "[REDACTED]");
  }
  return result;
}

// Use in every tool that returns external data:
return {
  content: [{ type: "text", text: sanitizeOutput(rawOutput) }],
};
```

This is defense in depth — it catches secrets that shouldn't be in files but are. It does not replace proper secrets management (which prevents secrets from being in files in the first place).

### Rate Limiting

For SSE and StreamableHTTP servers that are network-accessible, rate limiting prevents both accidental runaway loops and deliberate abuse.

```typescript
import rateLimit from "express-rate-limit";

// Apply to the SSE server
const limiter = rateLimit({
  windowMs: 60 * 1000,   // 1-minute window
  max: 100,               // 100 requests per minute per client
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    jsonrpc: "2.0",
    error: {
      code: -32000,
      message: "Rate limit exceeded. Maximum 100 tool calls per minute. Wait and retry.",
    },
  },
});

app.use("/messages", limiter);
```

For stdio servers (single client, local), rate limiting is typically unnecessary — but consider implementing a per-tool call budget if the server accesses external paid APIs.

For in-memory rate limiting without Express:

```typescript
const callCounts = new Map<string, { count: number; resetAt: number }>();
const RATE_LIMIT = 100;
const WINDOW_MS = 60_000;

function checkRateLimit(toolName: string): void {
  const now = Date.now();
  const entry = callCounts.get(toolName);

  if (!entry || now > entry.resetAt) {
    callCounts.set(toolName, { count: 1, resetAt: now + WINDOW_MS });
    return;
  }

  entry.count++;
  if (entry.count > RATE_LIMIT) {
    const waitSec = Math.ceil((entry.resetAt - now) / 1000);
    throw new Error(
      `Rate limit exceeded for "${toolName}": ${RATE_LIMIT} calls/min. ` +
      `Wait ${waitSec}s and retry.`
    );
  }
}
```

### OAuth for User-Scoped Access

When your MCP server wraps an API that requires user-specific authentication (e.g., a user's GitHub repos, their Jira tickets), implement the OAuth redirect pattern. The MCP client initiates the auth flow, the user approves in their browser, and the server receives and stores the token.

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";

// During initialization, check for stored credentials
server.setRequestHandler(InitializeRequestSchema, async (request) => {
  const token = await getStoredToken(request.params.clientInfo?.name);

  if (!token) {
    // Return an auth-required response
    // The client will present the auth URL to the user
    return {
      serverInfo: { name: "github-tools", version: "1.0.0" },
      capabilities: { tools: {} },
      instructions: "Authentication required. Visit the URL provided to authorize.",
      _meta: {
        authUrl: `https://github.com/login/oauth/authorize?client_id=${CLIENT_ID}&redirect_uri=${REDIRECT_URI}&scope=repo`,
      },
    };
  }

  return {
    serverInfo: { name: "github-tools", version: "1.0.0" },
    capabilities: { tools: {} },
  };
});

// OAuth callback handler (for SSE/HTTP transport)
app.get("/oauth/callback", async (req, res) => {
  const { code, state } = req.query;
  const tokenResponse = await exchangeCodeForToken(code as string);
  await storeToken(state as string, tokenResponse.access_token);
  res.send("Authorization complete. You can close this window.");
});
```

**Never store OAuth tokens in plaintext files.** Use the system keychain (`keytar` on Node), encrypted storage, or a secrets vault. Tokens in plaintext files are readable by any process on the machine.

### Audit Logging Every Tool Call

Every tool invocation should be logged with enough detail to reconstruct what happened, when, and why. This is non-negotiable for production servers.

```typescript
import { appendFile } from "node:fs/promises";

interface AuditEntry {
  timestamp: string;
  tool: string;
  arguments: Record<string, unknown>;
  resultType: "success" | "error";
  durationMs: number;
  clientInfo?: string;
}

async function auditLog(entry: AuditEntry): Promise<void> {
  const line = JSON.stringify(entry) + "\n";
  await appendFile(
    process.env.AUDIT_LOG || "./audit.jsonl",
    line,
    "utf-8"
  );
}

// Wrap every tool call:
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const start = Date.now();
  const { name, arguments: args } = request.params;

  try {
    const result = await executeToolCall(name, args);
    await auditLog({
      timestamp: new Date().toISOString(),
      tool: name,
      arguments: sanitizeForLog(args),  // Redact sensitive fields
      resultType: "success",
      durationMs: Date.now() - start,
    });
    return result;
  } catch (error) {
    await auditLog({
      timestamp: new Date().toISOString(),
      tool: name,
      arguments: sanitizeForLog(args),
      resultType: "error",
      durationMs: Date.now() - start,
    });
    throw error;
  }
});
```

Use JSONL (one JSON object per line) for audit logs — it is appendable, streamable, and parseable with standard tools (`jq`, `grep`). Rotate logs daily or by size. Ship to centralized logging in production.

---

## 8. Testing MCP Servers

### MCP Inspector — Interactive UI

The MCP SDK ships an inspector tool that connects to your server and lets you interactively browse tools, call them with arguments, and see raw JSON-RPC messages.

```bash
npx @modelcontextprotocol/inspector
```

This opens a web UI where you can:
- Connect to any MCP server (stdio or network)
- Browse discovered tools, resources, and prompts
- Call tools with custom arguments and see formatted results
- View raw JSON-RPC message traffic for debugging
- Test error handling by sending invalid inputs

Use the inspector as your first testing tool during development. It gives immediate feedback on whether your tool definitions, input schemas, and error messages look correct before any automated testing.

### Unit Testing with vitest

Test each tool handler in isolation. Mock the filesystem or database, validate inputs, and check both happy paths and error paths.

```typescript
import { describe, it, expect, beforeEach, vi } from "vitest";
import { createTestServer } from "./test-helpers.js"; // Your helper that sets up the server

describe("search_files tool", () => {
  let server: ReturnType<typeof createTestServer>;

  beforeEach(() => {
    server = createTestServer({
      allowedDir: "/tmp/test-project",
    });
  });

  it("returns matching lines with file paths and line numbers", async () => {
    // Arrange: create test files
    await writeTestFile("/tmp/test-project/src/app.ts", [
      'import { useState } from "react";',
      "// TODO: refactor this",
      'const name = "test";',
    ].join("\n"));

    // Act
    const result = await server.callTool("search_files", {
      query: "TODO",
      path: "src",
    });

    // Assert
    expect(result.isError).toBeUndefined();
    const text = result.content[0].text;
    expect(text).toContain("src/app.ts:2:");
    expect(text).toContain("TODO: refactor this");
  });

  it("blocks path traversal attempts", async () => {
    const result = await server.callTool("search_files", {
      query: "password",
      path: "../../etc",
    });

    expect(result.isError).toBe(true);
    expect(result.content[0].text).toContain("Path traversal blocked");
    expect(result.content[0].text).toContain("outside allowed directory");
  });

  it("returns actionable error for empty query", async () => {
    const result = await server.callTool("search_files", {
      query: "",
      path: ".",
    });

    expect(result.isError).toBe(true);
    expect(result.content[0].text).toContain("query");
    expect(result.content[0].text).toContain("at least 1 character");
  });

  it("truncates results with a message when limit is reached", async () => {
    // Arrange: create files with many matches
    const lines = Array.from({ length: 300 }, (_, i) => `match_${i}`).join("\n");
    await writeTestFile("/tmp/test-project/big.txt", lines);

    const result = await server.callTool("search_files", {
      query: "match_",
      path: ".",
    });

    const text = result.content[0].text;
    expect(text).toContain("truncated at 200 matches");
    expect(text).toContain("Narrow your query");
  });
});

describe("read_file tool", () => {
  it("returns file with line numbers", async () => {
    await writeTestFile("/tmp/test-project/readme.md", "# Title\n\nContent here.");

    const result = await server.callTool("read_file", { path: "readme.md" });

    expect(result.content[0].text).toContain("1\t# Title");
    expect(result.content[0].text).toContain("3\tContent here.");
  });

  it("rejects files outside allowed directory", async () => {
    const result = await server.callTool("read_file", {
      path: "../../../etc/passwd",
    });

    expect(result.isError).toBe(true);
    expect(result.content[0].text).toContain("Path traversal blocked");
  });

  it("returns error for non-existent file with suggestion", async () => {
    const result = await server.callTool("read_file", {
      path: "nonexistent.ts",
    });

    expect(result.isError).toBe(true);
    expect(result.content[0].text).toContain("not found");
  });

  it("rejects files exceeding size limit", async () => {
    // Create a file larger than MAX_FILE_SIZE
    const bigContent = "x".repeat(2 * 1024 * 1024);
    await writeTestFile("/tmp/test-project/huge.bin", bigContent);

    const result = await server.callTool("read_file", { path: "huge.bin" });

    expect(result.isError).toBe(true);
    expect(result.content[0].text).toContain("exceeding the 1 MB limit");
    expect(result.content[0].text).toContain("search_files");
  });
});
```

**Test helper** — a minimal wrapper that creates a server instance and exposes a `callTool` method without needing a transport:

```typescript
// test-helpers.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";

export function createTestServer(config: { allowedDir: string }) {
  process.env.ALLOWED_DIR = config.allowedDir;

  // Import and initialize your server module
  // (structure your server so the handler logic is importable separately from the transport startup)
  const server = createServer(); // Your factory function

  return {
    async callTool(name: string, args: Record<string, unknown>) {
      // Directly invoke the CallToolRequest handler
      const handler = server.getRequestHandler("tools/call");
      return handler({
        method: "tools/call",
        params: { name, arguments: args },
      });
    },
  };
}
```

### Integration Testing with Claude Code SDK

Test the full loop: a real Claude Code agent connects to your MCP server, discovers tools, and uses them to complete a task. This catches schema mismatches, description quality issues, and tool interaction patterns.

```typescript
import { query } from "@anthropic-ai/claude-code";
import { describe, it, expect } from "vitest";

describe("MCP server integration with Claude Code", () => {
  it("agent can search and read files using MCP tools", async () => {
    const messages: string[] = [];

    for await (const msg of query({
      prompt: 'Find all TODO comments in the project and read the first file that contains one. Report the file path and line number.',
      options: {
        maxTurns: 10,
        cwd: "/tmp/test-project",
        allowedTools: [
          "mcp__my-tools__search_files",
          "mcp__my-tools__read_file",
        ],
        mcpServers: [{
          name: "my-tools",
          type: "stdio",
          command: "node",
          args: ["./dist/index.js"],
          env: { ALLOWED_DIR: "/tmp/test-project" },
        }],
        model: "claude-haiku-4-5",  // Use Haiku for fast, cheap integration tests
      },
    })) {
      if (msg.type === "assistant") {
        const text = msg.message.content
          .filter((b: any) => b.type === "text")
          .map((b: any) => b.text)
          .join("");
        if (text) messages.push(text);
      }
    }

    const output = messages.join("\n");
    expect(output).toContain("TODO");
    // Verify the agent actually found and read a file
    expect(output).toMatch(/\.ts:\d+/);
  }, { timeout: 60_000 }); // Agent tests need longer timeouts
});
```

**Use `claude-haiku-4-5` for integration tests.** It is fast and cheap — ideal for testing tool interaction patterns without the cost of Opus. Only use Opus for eval suites where reasoning quality matters.

### Record/Replay Pattern for Regression Testing

Record the JSON-RPC messages from a successful test run and replay them to detect regressions without calling the LLM.

```typescript
import { writeFile, readFile } from "node:fs/promises";

// --- Recording mode ---
const recording: Array<{ request: unknown; response: unknown }> = [];

// Wrap the tool handler to record calls
const originalHandler = server.getRequestHandler("tools/call");
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const response = await originalHandler(request);
  recording.push({ request: request.params, response });
  return response;
});

// After the test run, save the recording
await writeFile("fixtures/search-and-read.jsonl",
  recording.map(r => JSON.stringify(r)).join("\n")
);

// --- Replay mode ---
async function replayTest(fixturePath: string) {
  const lines = (await readFile(fixturePath, "utf-8")).trim().split("\n");

  for (const line of lines) {
    const { request, response: expectedResponse } = JSON.parse(line);
    const actualResponse = await server.callTool(request.name, request.arguments);

    // Compare structure (ignoring timestamps, etc.)
    expect(normalizeResponse(actualResponse)).toEqual(
      normalizeResponse(expectedResponse)
    );
  }
}
```

Record once manually (or during a CI golden run). Replay on every code change. When a replay fails, either the change is a regression (fix it) or the expected output has legitimately changed (update the fixture).

---

## 9. Deployment and Distribution

### npm Package with bin Field

The standard way to distribute a TypeScript MCP server. Users install it globally or run it via `npx`.

**`package.json`**:
```json
{
  "name": "@myorg/mcp-project-tools",
  "version": "1.0.0",
  "description": "MCP server for project file search and database queries",
  "type": "module",
  "bin": {
    "mcp-project-tools": "./dist/index.js"
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "npm run build"
  },
  "engines": {
    "node": ">=18"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "zod": "^3.23.0"
  }
}
```

Make sure `dist/index.js` starts with the shebang `#!/usr/bin/env node` so it's executable when installed via `bin`.

**User installation** — in their `mcp.json`:
```json
{
  "mcpServers": {
    "project-tools": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@myorg/mcp-project-tools"],
      "env": {
        "ALLOWED_DIR": "${workspaceFolder}"
      }
    }
  }
}
```

The `-y` flag auto-confirms the npx install prompt. The server is downloaded and cached on first use, then reused on subsequent runs. Users do not need to manually `npm install`.

### Docker Deployment for SSE/HTTP Servers

For network-accessible servers (SSE or StreamableHTTP), Docker provides reproducible builds and easy deployment.

**`Dockerfile`**:
```dockerfile
FROM node:22-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Non-root user for security
RUN adduser --system --no-create-home mcp
USER mcp

EXPOSE 3100
ENV NODE_ENV=production
CMD ["node", "dist/index.js"]
```

**`docker-compose.yml`** (for local team use):
```yaml
services:
  mcp-server:
    build: .
    ports:
      - "3100:3100"
    environment:
      - ALLOWED_DIR=/data/project
      - AUDIT_LOG=/logs/audit.jsonl
      - DB_HOST=postgres
      - DB_PASSWORD_FILE=/run/secrets/db_password
    volumes:
      - ./project:/data/project:ro   # Read-only project files
      - ./logs:/logs                   # Audit logs
    secrets:
      - db_password
    restart: unless-stopped

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**Key Docker practices for MCP servers**:
- Mount project files as read-only (`:ro`) unless the server explicitly needs write access
- Use Docker secrets for credentials, not environment variables (env vars appear in `docker inspect`)
- Run as a non-root user
- Use multi-stage builds to keep the final image small (no TypeScript compiler, no dev dependencies)
- Set `restart: unless-stopped` for reliability

### Versioning Rules

MCP servers are APIs consumed by AI agents. Apply the same versioning discipline as any other API.

**Semver compliance**:
- **Patch** (1.0.1): Bug fixes, performance improvements, documentation. No schema changes.
- **Minor** (1.1.0): New tools added, new optional fields on existing tools, new resources. All existing tool calls continue to work unchanged.
- **Major** (2.0.0): Tool removed, tool renamed, required field added to existing tool, output format changed, breaking behavior change.

**Additive tool changes are always minor**:
Adding a new tool is backward-compatible — existing clients that don't know about the tool simply never call it. Adding an optional field to an existing tool's inputSchema is backward-compatible — existing calls that omit the field use the default.

**Deprecation period**:
Before removing a tool in a major version:
1. Mark the tool as deprecated in its description: `"[DEPRECATED — use query_database_v2 instead] ..."`
2. Keep the deprecated tool working for at least one minor version cycle
3. Log a warning when the deprecated tool is called
4. Remove in the next major version

```typescript
// Deprecation warning in tool handler
case "old_tool_name": {
  console.warn(`[DEPRECATED] Tool "old_tool_name" called — will be removed in v3.0.0. Use "new_tool_name" instead.`);
  await auditLog({
    ...baseEntry,
    metadata: { deprecated: true, replacement: "new_tool_name" },
  });
  // ... existing implementation continues to work ...
}
```

**Version in server info**: Always include the version in the `initialize` response so clients can detect and adapt to server versions:

```typescript
const server = new Server(
  { name: "my-mcp-server", version: "1.2.0" },
  { capabilities: { tools: {} } }
);
```

### Distribution Checklist

Before publishing an MCP server:

1. **README documents every tool** — name, description, input schema, example call, example response
2. **All tools have Zod/Pydantic validation** — no unvalidated inputs
3. **Path traversal tests pass** — every file-touching tool tested with `../` inputs
4. **Error messages are actionable** — every error tells the model what to do next
5. **Secret patterns are redacted** — output sanitization is in place
6. **Audit logging works** — every tool call produces a log entry
7. **`npx -y` installation tested** — clean install on a fresh machine works
8. **License file present** — `LICENSE` in the package root
9. **Changelog maintained** — `CHANGELOG.md` documents every version
10. **CI runs tests on every push** — unit tests, path traversal tests, and at least one integration test
