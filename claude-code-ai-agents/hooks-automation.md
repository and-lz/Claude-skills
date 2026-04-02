# Hooks — 100% Compliance Automation

Complete reference for Claude Code hooks: deterministic enforcement of rules that CLAUDE.md cannot guarantee. This file covers hook mechanics, configuration, exit code semantics, environment variables, and a library of production-ready shell and Python hook scripts. For CLAUDE.md authoring, skills, and general Claude Code customization, see the other reference files in this skill.

---

## 1. What hooks are

Hooks are shell scripts (or any executable) invoked **synchronously** by Claude Code at specific lifecycle points during a session. They are the enforcement mechanism for rules that must be followed 100% of the time, with zero exceptions.

### CLAUDE.md vs. hooks

CLAUDE.md instructions are advisory. The model reads them, weighs them against everything else in context, and follows them approximately 80% of the time under normal conditions. That compliance rate drops further as the instruction count grows past 150. For style preferences and general guidance, 80% is fine. For security policies, compliance requirements, and destructive-action prevention, 80% is unacceptable.

Hooks are deterministic. They run as actual processes on the host machine. If a hook exits with code 2, the tool call is blocked — period. The model cannot override, ignore, or "interpret" a hook. There is no compliance rate because there is no discretion.

**Use CLAUDE.md for**: style preferences, naming conventions, preferred libraries, workflow guidance, documentation standards.

**Use hooks for**: blocking destructive commands, preventing secret leaks, enforcing commit formats, mandatory linting, audit logging, license compliance.

### Hook types

There are two hook points in the Claude Code lifecycle:

**PreToolUse** — fires before a tool executes. The hook receives the tool name and input. It CAN block the tool call by exiting with code 2. This is the enforcement point: if a hook says no, the tool does not run.

**PostToolUse** — fires after a tool completes. The hook receives the tool name, input, AND output. It CANNOT undo the action (the tool already ran), but it can surface errors, trigger follow-up actions (linting, logging, type checking), or add context for the model's next decision.

### Execution model

Hooks are synchronous. When a PreToolUse hook runs, Claude Code blocks until it finishes. The model does not continue generating, no other tools run, nothing happens until the hook returns. This means hook performance directly impacts session responsiveness.

Multiple hooks can match the same tool call. They run in the order they appear in the configuration array. For PreToolUse, if any hook in the chain exits with code 2, subsequent hooks in the chain are skipped and the tool call is blocked.

---

## 2. Exit code semantics

Exit codes are the hook's communication channel back to Claude Code. The semantics differ slightly between PreToolUse and PostToolUse:

### PreToolUse exit codes

| Exit Code | Behavior | stdout handling | Use case |
|-----------|----------|-----------------|----------|
| `0` | Tool call proceeds normally | stdout shown as informational context to the model | Hook approves the action, optionally adding context |
| `2` | **Tool call is BLOCKED** | stdout shown as error message to the model | Hook rejects the action — the tool does not execute |
| Any other (1, 3, etc.) | Tool call proceeds (warning) | stdout shown as warning context to the model | Hook flags a concern but does not block |

### PostToolUse exit codes

| Exit Code | Behavior | stdout handling | Use case |
|-----------|----------|-----------------|----------|
| `0` | Normal completion | stdout shown as context to the model | Hook ran successfully, optionally adding context |
| `2` | Error surfaced to model | stdout shown as error context | Hook detected a problem, but the action already happened |
| Any other | Warning surfaced to model | stdout shown as warning context | Hook flags a concern for model awareness |

**Critical distinction**: In PostToolUse, exit code 2 surfaces an error to the model, but it **cannot undo** the tool's action. If the tool wrote a file, the file is written. If the tool ran a command, the command ran. The error just tells the model something went wrong after the fact.

### stdout vs. stderr

- **stdout**: Everything written to stdout is captured and shown to the model as context. This is how hooks communicate information, warnings, and error messages that the model will see and can act on.
- **stderr**: Written to stderr is shown to the user in the Claude Code UI as diagnostic output but is NOT passed to the model. Use stderr for debug logging and human-readable diagnostics that don't need to influence model behavior.

### Example: exit code patterns

```bash
# Approve with context
echo "File is within project directory, proceeding."
exit 0

# Block with error message
echo "BLOCKED: Cannot delete files outside project directory."
exit 2

# Warn but allow
echo "WARNING: This file is over 500 lines. Consider splitting."
exit 1
```

---

## 3. Environment variables

Claude Code sets several environment variables before invoking a hook. These provide all the context needed to make decisions about the tool call.

### Available variables

| Variable | Available in | Description |
|----------|-------------|-------------|
| `CLAUDE_TOOL_NAME` | PreToolUse, PostToolUse | Name of the tool being called (e.g., `Bash`, `Write`, `Edit`, `Read`) |
| `CLAUDE_TOOL_INPUT` | PreToolUse, PostToolUse | JSON string of the tool's input parameters |
| `CLAUDE_TOOL_OUTPUT` | PostToolUse only | JSON string of the tool's output/result |
| `CLAUDE_SESSION_ID` | PreToolUse, PostToolUse | Unique session identifier (UUID) |
| `CLAUDE_PROJECT_DIR` | PreToolUse, PostToolUse | Absolute path to the project root directory |
| `CLAUDE_USER` | PreToolUse, PostToolUse | Username of the current user |

### Reading tool input with jq

The `CLAUDE_TOOL_INPUT` variable contains a JSON object whose shape depends on the tool. Common patterns:

```bash
# Bash tool — extract the command
COMMAND=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.command // ""')

# Write tool — extract the file path and content
FILE_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // ""')
CONTENT=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.content // ""')

# Edit tool — extract file path and strings
FILE_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // ""')
OLD_STRING=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.old_string // ""')
NEW_STRING=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.new_string // ""')

# Read tool — extract file path
FILE_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // ""')

# Glob tool — extract pattern and path
PATTERN=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.pattern // ""')
SEARCH_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.path // ""')
```

### Stdin alternative

Instead of reading `CLAUDE_TOOL_INPUT` from the environment variable, hooks can read the full hook context from stdin as a JSON object:

```bash
# Read the full hook payload from stdin
INPUT=$(cat)

# Extract fields
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // ""')
TOOL_INPUT=$(echo "$INPUT" | jq -r '.tool_input // ""')
SESSION_ID=$(echo "$INPUT" | jq -r '.session_id // ""')
```

The environment variable approach is generally more convenient for simple hooks. The stdin approach is useful when you need the full structured payload or when working with languages that make stdin easier to parse than environment variables.

### Tool input shapes by tool

For reference, here are the input shapes for the most commonly hooked tools:

```jsonc
// Bash tool
{ "command": "string", "timeout": 120000, "description": "string" }

// Write tool
{ "file_path": "/absolute/path", "content": "file content" }

// Edit tool
{ "file_path": "/absolute/path", "old_string": "...", "new_string": "...", "replace_all": false }

// Read tool
{ "file_path": "/absolute/path", "offset": 0, "limit": 2000 }

// Glob tool
{ "pattern": "**/*.ts", "path": "/search/dir" }

// Grep tool
{ "pattern": "regex", "path": "/search/dir", "glob": "*.ts", "output_mode": "content" }
```

---

## 4. settings.json hook configuration

Hooks are configured in `settings.json`, either at the project level or the user level.

### Configuration locations

| Level | Path | Scope | Version controlled? |
|-------|------|-------|-------------------|
| Project | `<project>/.claude/settings.json` | This project only | Yes (commit it) |
| User | `~/.claude/settings.json` | All projects | No (personal) |

Project-level settings override user-level settings when there are conflicts. Both levels can define hooks, and hooks from both levels will run (project-level hooks run first).

### Full schema

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/hook-script.sh"
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/another-hook.sh"
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

### Matcher patterns

The `matcher` field determines which tool calls trigger the hook. Three pattern types are supported:

**Exact match** — matches a specific tool name exactly:
```json
{ "matcher": "Bash", "hooks": [...] }
{ "matcher": "Write", "hooks": [...] }
{ "matcher": "Edit", "hooks": [...] }
```

**Glob pattern** — matches tool names using glob syntax. Useful for MCP tool namespaces:
```json
{ "matcher": "mcp__stripe__*", "hooks": [...] }
{ "matcher": "mcp__github__*", "hooks": [...] }
{ "matcher": "mcp__*", "hooks": [...] }
```

**Wildcard** — matches ALL tool calls:
```json
{ "matcher": "*", "hooks": [...] }
```

### Multiple hooks per matcher

A single matcher can have multiple hooks. They run in array order. For PreToolUse, the first hook that exits with code 2 stops the chain and blocks the tool call:

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "command": "/hooks/block-dangerous-commands.sh"
    },
    {
      "type": "command",
      "command": "/hooks/enforce-project-boundary.sh"
    },
    {
      "type": "command",
      "command": "/hooks/audit-log.sh"
    }
  ]
}
```

If `block-dangerous-commands.sh` exits with code 2, neither `enforce-project-boundary.sh` nor `audit-log.sh` will run.

### Multiple matchers for the same event

You can have multiple matcher entries in the same event array. Each matcher is evaluated independently:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "/hooks/bash-guard.sh" }]
      },
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "/hooks/write-guard.sh" }]
      },
      {
        "matcher": "*",
        "hooks": [{ "type": "command", "command": "/hooks/audit-all.sh" }]
      }
    ]
  }
}
```

When the Bash tool is called, both `bash-guard.sh` (from the "Bash" matcher) and `audit-all.sh` (from the "*" matcher) will run.

### Disabling all hooks

For debugging or when hooks are causing problems, you can temporarily disable all hooks:

```json
{
  "hooks": {
    "PreToolUse": [...],
    "PostToolUse": [...]
  },
  "disableAllHooks": true
}
```

This keeps the hook configuration in place but prevents any hooks from executing. Remove the flag to re-enable.

### Complete real-world settings.json

Here is a production-ready settings.json with a comprehensive hook setup:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/block-dangerous-commands.sh"
          },
          {
            "type": "command",
            "command": ".claude/hooks/enforce-project-boundary.sh"
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/detect-secrets.py"
          },
          {
            "type": "command",
            "command": ".claude/hooks/enforce-project-boundary.sh"
          }
        ]
      },
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/detect-secrets.py"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/enforce-conventional-commits.sh"
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
            "command": ".claude/hooks/auto-lint.sh"
          }
        ]
      },
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/audit-log.sh"
          }
        ]
      }
    ]
  }
}
```

---

## 5. PreToolUse examples — shell scripts

### 5.1 Block dangerous Bash commands

Prevents destructive shell commands that could damage the system or project.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Only applies to Bash tool
[[ "${CLAUDE_TOOL_NAME:-}" != "Bash" ]] && exit 0

COMMAND=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.command // ""')
[[ -z "$COMMAND" ]] && exit 0

# Normalize: collapse whitespace, lowercase for pattern matching
NORMALIZED=$(echo "$COMMAND" | tr '[:upper:]' '[:lower:]' | tr -s ' ')

# --- Dangerous command patterns ---

# rm -rf with root-like paths
if echo "$COMMAND" | grep -qE 'rm\s+(-[a-zA-Z]*f[a-zA-Z]*\s+|--force\s+)*(\/|~|\.\.)'; then
  echo "BLOCKED: 'rm -rf' targeting root, home, or parent directories is not allowed."
  exit 2
fi

# Fork bomb patterns
if echo "$COMMAND" | grep -qE ':\(\)\{.*\|.*\}|\.\s*\/dev\/stdin|fork.*while.*true'; then
  echo "BLOCKED: Fork bomb detected."
  exit 2
fi

# Piping curl/wget to shell
if echo "$COMMAND" | grep -qE '(curl|wget)\s+.*\|\s*(bash|sh|zsh|dash|source)'; then
  echo "BLOCKED: Piping remote content to shell is not allowed. Download first, review, then execute."
  exit 2
fi

# dd targeting block devices
if echo "$COMMAND" | grep -qE 'dd\s+.*of=\/dev\/(sd|hd|nvme|disk|rdisk)'; then
  echo "BLOCKED: 'dd' targeting block devices is not allowed."
  exit 2
fi

# chmod 777 (world-writable)
if echo "$COMMAND" | grep -qE 'chmod\s+(-[a-zA-Z]*\s+)*777'; then
  echo "BLOCKED: 'chmod 777' sets world-writable permissions. Use specific permissions (e.g., 755, 644)."
  exit 2
fi

# mkfs (format filesystem)
if echo "$COMMAND" | grep -qE 'mkfs\.\w+\s+\/dev\/'; then
  echo "BLOCKED: Filesystem formatting is not allowed."
  exit 2
fi

# > /dev/sda (direct device write)
if echo "$COMMAND" | grep -qE '>\s*\/dev\/(sd|hd|nvme)'; then
  echo "BLOCKED: Direct write to block device is not allowed."
  exit 2
fi

exit 0
```

### 5.2 Block writes outside project directory

Ensures all file operations stay within the project root. Also blocks sensitive file patterns.

```bash
#!/usr/bin/env bash
set -euo pipefail

TOOL="${CLAUDE_TOOL_NAME:-}"
PROJECT_DIR="${CLAUDE_PROJECT_DIR:-}"

# Only applies to Write and Edit tools
[[ "$TOOL" != "Write" && "$TOOL" != "Edit" ]] && exit 0
[[ -z "$PROJECT_DIR" ]] && exit 0

# Extract file path from tool input
FILE_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // ""')
[[ -z "$FILE_PATH" ]] && exit 0

# Resolve to absolute path (handles ../ traversal)
RESOLVED_PATH=$(realpath -m "$FILE_PATH" 2>/dev/null || echo "$FILE_PATH")
RESOLVED_PROJECT=$(realpath -m "$PROJECT_DIR" 2>/dev/null || echo "$PROJECT_DIR")

# Check if file is within project directory
if [[ "$RESOLVED_PATH" != "$RESOLVED_PROJECT"/* ]]; then
  echo "BLOCKED: Cannot write to '$FILE_PATH' — outside project directory '$PROJECT_DIR'."
  echo "All file writes must be within the project root."
  exit 2
fi

# Block sensitive file patterns even within project
FILENAME=$(basename "$FILE_PATH")
case "$FILENAME" in
  .env|.env.local|.env.production|.env.staging)
    echo "BLOCKED: Cannot write to environment file '$FILENAME'. Manage secrets outside Claude Code."
    exit 2
    ;;
  *.pem|*.key|*.p12|*.pfx|*.jks)
    echo "BLOCKED: Cannot write to key/certificate file '$FILENAME'."
    exit 2
    ;;
  id_rsa|id_ed25519|id_ecdsa|authorized_keys|known_hosts)
    echo "BLOCKED: Cannot write to SSH file '$FILENAME'."
    exit 2
    ;;
  credentials.json|service-account*.json|*credentials*.yaml)
    echo "BLOCKED: Cannot write to credentials file '$FILENAME'."
    exit 2
    ;;
esac

# Block hidden config directories that could be attack vectors
if echo "$RESOLVED_PATH" | grep -qE '\/\.(ssh|gnupg|aws|gcloud|azure|kube)\/'; then
  echo "BLOCKED: Cannot write to sensitive config directory."
  exit 2
fi

exit 0
```

### 5.3 Enforce conventional commits

Validates that git commit messages follow the conventional commits format.

```bash
#!/usr/bin/env bash
set -euo pipefail

[[ "${CLAUDE_TOOL_NAME:-}" != "Bash" ]] && exit 0

COMMAND=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.command // ""')

# Only check git commit commands with -m flag
if ! echo "$COMMAND" | grep -qE 'git\s+commit\s+.*-m\s+'; then
  exit 0
fi

# Extract the commit message (handles both single and double quotes)
COMMIT_MSG=$(echo "$COMMAND" | sed -n 's/.*git commit.*-m\s*["'\'']\([^"'\'']*\)["'\''].*/\1/p')

# Also handle heredoc-style commit messages
if [[ -z "$COMMIT_MSG" ]]; then
  COMMIT_MSG=$(echo "$COMMAND" | sed -n 's/.*git commit.*-m\s*"\(.*\)"/\1/p')
fi

[[ -z "$COMMIT_MSG" ]] && exit 0

# Conventional commit regex
# Format: type(optional-scope): description
# Types: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert
PATTERN='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-z0-9_-]+\))?!?:\s.+'

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo "BLOCKED: Commit message does not follow conventional commits format."
  echo ""
  echo "Expected format: type(scope): description"
  echo ""
  echo "Valid types: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert"
  echo "Scope is optional: feat(auth): add login"
  echo "Breaking change: feat!: remove deprecated API"
  echo ""
  echo "Your message: '$COMMIT_MSG'"
  exit 2
fi

# Warn on long subject lines (>72 chars is conventional limit)
SUBJECT_LENGTH=${#COMMIT_MSG}
if [[ $SUBJECT_LENGTH -gt 72 ]]; then
  echo "WARNING: Commit subject is $SUBJECT_LENGTH chars (conventional limit is 72)."
  echo "Consider shortening: '$COMMIT_MSG'"
  exit 1
fi

exit 0
```

### 5.4 Block SQL DDL statements

Prevents destructive SQL operations from being run through the Bash tool.

```bash
#!/usr/bin/env bash
set -euo pipefail

[[ "${CLAUDE_TOOL_NAME:-}" != "Bash" ]] && exit 0

COMMAND=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.command // ""')
[[ -z "$COMMAND" ]] && exit 0

# Normalize for case-insensitive matching
UPPER_CMD=$(echo "$COMMAND" | tr '[:lower:]' '[:upper:]')

# Block destructive DDL operations
BLOCKED_PATTERNS=(
  "DROP TABLE"
  "DROP DATABASE"
  "DROP SCHEMA"
  "TRUNCATE TABLE"
  "TRUNCATE "
  "DELETE FROM .* WHERE 1"
  "DELETE FROM .* WITHOUT"
  "ALTER TABLE .* DROP COLUMN"
  "DROP INDEX"
  "DROP VIEW"
  "DROP FUNCTION"
  "DROP PROCEDURE"
  "DROP TRIGGER"
)

for PATTERN in "${BLOCKED_PATTERNS[@]}"; do
  if echo "$UPPER_CMD" | grep -qE "$PATTERN"; then
    echo "BLOCKED: Destructive SQL DDL detected: '$PATTERN'"
    echo ""
    echo "Destructive SQL operations must be run manually with explicit human approval."
    echo "If this is intentional, run the SQL command directly in your database client."
    exit 2
  fi
done

# Warn on UPDATE without WHERE (potential mass update)
if echo "$UPPER_CMD" | grep -qE 'UPDATE\s+\w+\s+SET\s+' && ! echo "$UPPER_CMD" | grep -qE 'WHERE\s+'; then
  echo "WARNING: UPDATE without WHERE clause detected. This will modify ALL rows in the table."
  exit 1
fi

# Warn on DELETE without WHERE
if echo "$UPPER_CMD" | grep -qE 'DELETE\s+FROM\s+' && ! echo "$UPPER_CMD" | grep -qE 'WHERE\s+'; then
  echo "WARNING: DELETE without WHERE clause detected. This will delete ALL rows in the table."
  exit 1
fi

exit 0
```

### 5.5 Enforce branch naming convention

Ensures git branch names follow a team convention before allowing branch creation.

```bash
#!/usr/bin/env bash
set -euo pipefail

[[ "${CLAUDE_TOOL_NAME:-}" != "Bash" ]] && exit 0

COMMAND=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.command // ""')

# Match git checkout -b or git switch -c (branch creation)
BRANCH_NAME=""

if echo "$COMMAND" | grep -qE 'git\s+(checkout\s+-b|switch\s+-c|branch)\s+'; then
  # Extract branch name — last argument after the flag
  BRANCH_NAME=$(echo "$COMMAND" | sed -n 's/.*git\s\+\(checkout\s\+-b\|switch\s\+-c\|branch\)\s\+\([^ ]*\).*/\2/p')
fi

[[ -z "$BRANCH_NAME" ]] && exit 0

# Skip special branch names
case "$BRANCH_NAME" in
  main|master|develop|staging|production|release)
    exit 0
    ;;
esac

# Enforce naming convention: type/description
# Allowed types: feature, bugfix, hotfix, chore, docs, refactor, test, ci
BRANCH_PATTERN='^(feature|bugfix|hotfix|chore|docs|refactor|test|ci)\/[a-z0-9][a-z0-9._-]+$'

if ! echo "$BRANCH_NAME" | grep -qE "$BRANCH_PATTERN"; then
  echo "BLOCKED: Branch name '$BRANCH_NAME' does not follow naming convention."
  echo ""
  echo "Expected format: type/description-in-kebab-case"
  echo ""
  echo "Valid types: feature, bugfix, hotfix, chore, docs, refactor, test, ci"
  echo ""
  echo "Examples:"
  echo "  feature/user-authentication"
  echo "  bugfix/login-redirect-loop"
  echo "  hotfix/payment-rounding-error"
  echo "  chore/update-dependencies"
  echo "  refactor/extract-auth-module"
  exit 2
fi

exit 0
```

### 5.6 Enforce file size limits

Warns when creating or editing files that exceed a line count threshold. Uses exit 1 (warning) instead of exit 2 (blocking) so the action proceeds but the model is made aware.

```bash
#!/usr/bin/env bash
set -euo pipefail

TOOL="${CLAUDE_TOOL_NAME:-}"

# Only applies to Write tool (we can count lines of new content)
[[ "$TOOL" != "Write" ]] && exit 0

FILE_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // ""')
CONTENT=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.content // ""')

[[ -z "$CONTENT" ]] && exit 0

# Count lines in the content being written
LINE_COUNT=$(echo "$CONTENT" | wc -l | tr -d ' ')

WARN_THRESHOLD=500
ERROR_THRESHOLD=1000

if [[ $LINE_COUNT -gt $ERROR_THRESHOLD ]]; then
  echo "WARNING: File '$FILE_PATH' has $LINE_COUNT lines (threshold: $ERROR_THRESHOLD)."
  echo "This file is very large. Consider splitting into smaller modules."
  echo ""
  echo "Suggestions:"
  echo "  - Extract helper functions into separate files"
  echo "  - Split component + logic into separate files"
  echo "  - Use barrel exports (index.ts) to re-export from smaller files"
  exit 1
elif [[ $LINE_COUNT -gt $WARN_THRESHOLD ]]; then
  echo "WARNING: File '$FILE_PATH' has $LINE_COUNT lines (threshold: $WARN_THRESHOLD)."
  echo "Consider whether this file should be split into smaller modules."
  exit 1
fi

exit 0
```

---

## 6. PostToolUse examples — shell scripts

### 6.1 Auto-lint after Edit

Automatically runs ESLint with auto-fix after any file edit on TypeScript or JavaScript files.

```bash
#!/usr/bin/env bash
set -euo pipefail

[[ "${CLAUDE_TOOL_NAME:-}" != "Edit" ]] && exit 0

FILE_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // ""')
[[ -z "$FILE_PATH" ]] && exit 0

# Only lint TS/JS files
case "$FILE_PATH" in
  *.ts|*.tsx|*.js|*.jsx|*.mjs|*.cjs)
    ;;
  *)
    exit 0
    ;;
esac

# Check if file exists (might have been a failed edit)
[[ ! -f "$FILE_PATH" ]] && exit 0

# Check if eslint is available
PROJECT_DIR="${CLAUDE_PROJECT_DIR:-$(pwd)}"
ESLINT=""

if [[ -x "$PROJECT_DIR/node_modules/.bin/eslint" ]]; then
  ESLINT="$PROJECT_DIR/node_modules/.bin/eslint"
elif command -v npx &>/dev/null; then
  ESLINT="npx eslint"
else
  exit 0
fi

# Run eslint --fix (suppress exit code — we don't want to block on lint errors)
LINT_OUTPUT=$($ESLINT --fix "$FILE_PATH" 2>&1) || true

if [[ -n "$LINT_OUTPUT" ]]; then
  echo "ESLint auto-fix applied to $(basename "$FILE_PATH"):"
  echo "$LINT_OUTPUT"
fi

exit 0
```

### 6.2 Audit log all tool uses

Writes a JSONL audit log of every tool invocation with timestamp, session, tool name, and project context.

```bash
#!/usr/bin/env bash
set -euo pipefail

# This hook matches "*" — runs on every tool call
PROJECT_DIR="${CLAUDE_PROJECT_DIR:-unknown}"
SESSION_ID="${CLAUDE_SESSION_ID:-unknown}"
TOOL_NAME="${CLAUDE_TOOL_NAME:-unknown}"
USER="${CLAUDE_USER:-unknown}"

# Determine log directory
LOG_DIR="${PROJECT_DIR}/.claude/audit-logs"
mkdir -p "$LOG_DIR"

# Log file named by date
LOG_FILE="${LOG_DIR}/$(date +%Y-%m-%d).jsonl"

# Build JSON log entry using jq for proper escaping
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Truncate tool input to avoid massive log entries (keep first 2000 chars)
TOOL_INPUT_TRUNCATED=$(echo "${CLAUDE_TOOL_INPUT:-"{}"}" | head -c 2000)

LOG_ENTRY=$(jq -n \
  --arg ts "$TIMESTAMP" \
  --arg sid "$SESSION_ID" \
  --arg tool "$TOOL_NAME" \
  --arg user "$USER" \
  --arg project "$PROJECT_DIR" \
  --arg input "$TOOL_INPUT_TRUNCATED" \
  '{timestamp: $ts, session_id: $sid, tool: $tool, user: $user, project: $project, input: $input}')

# Append to log (atomic-ish write)
echo "$LOG_ENTRY" >> "$LOG_FILE"

# Don't pollute model context — exit silently
exit 0
```

### 6.3 Run type check after TypeScript changes

Triggers a TypeScript type check after `.ts` or `.tsx` files are modified. Uses a background process to avoid blocking.

```bash
#!/usr/bin/env bash
set -euo pipefail

TOOL="${CLAUDE_TOOL_NAME:-}"

# Only run after Edit or Write
[[ "$TOOL" != "Edit" && "$TOOL" != "Write" ]] && exit 0

FILE_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // ""')
[[ -z "$FILE_PATH" ]] && exit 0

# Only for TypeScript files
case "$FILE_PATH" in
  *.ts|*.tsx)
    ;;
  *)
    exit 0
    ;;
esac

PROJECT_DIR="${CLAUDE_PROJECT_DIR:-$(pwd)}"
TSC=""

if [[ -x "$PROJECT_DIR/node_modules/.bin/tsc" ]]; then
  TSC="$PROJECT_DIR/node_modules/.bin/tsc"
elif command -v npx &>/dev/null; then
  TSC="npx tsc"
else
  echo "TypeScript compiler not found. Skipping type check."
  exit 0
fi

# Run tsc in background to avoid blocking the session
# Results are written to a temp file that subsequent hooks or the user can check
TYPECHECK_LOG="$PROJECT_DIR/.claude/typecheck-latest.log"
mkdir -p "$(dirname "$TYPECHECK_LOG")"

(
  cd "$PROJECT_DIR"
  $TSC --noEmit --pretty 2>&1 | head -50 > "$TYPECHECK_LOG"
) &
disown

echo "Type check started in background. Results will be in .claude/typecheck-latest.log"
exit 0
```

---

## 7. Python hook examples

Python hooks are useful for complex logic, regex-heavy validation, and cases where shell scripting becomes unwieldy. Mark them executable (`chmod +x`) and use a shebang.

### 7.1 Detect secret patterns before Write

Scans file content for common secret patterns (API keys, tokens, private keys) and blocks the write if any are found.

```python
#!/usr/bin/env python3
"""PreToolUse hook: Block writes that contain secret patterns."""

import json
import os
import re
import sys

TOOL_NAME = os.environ.get("CLAUDE_TOOL_NAME", "")
if TOOL_NAME not in ("Write", "Edit"):
    sys.exit(0)

try:
    tool_input = json.loads(os.environ.get("CLAUDE_TOOL_INPUT", "{}"))
except json.JSONDecodeError:
    sys.exit(0)

# For Write, check content; for Edit, check new_string
if TOOL_NAME == "Write":
    content = tool_input.get("content", "")
elif TOOL_NAME == "Edit":
    content = tool_input.get("new_string", "")
else:
    sys.exit(0)

if not content:
    sys.exit(0)

file_path = tool_input.get("file_path", "")

# --- Secret patterns ---
# Each tuple: (pattern_name, regex, description)
SECRET_PATTERNS = [
    (
        "Stripe Secret Key",
        r"sk_(live|test)_[a-zA-Z0-9]{20,}",
        "Stripe secret API key detected",
    ),
    (
        "Stripe Restricted Key",
        r"rk_(live|test)_[a-zA-Z0-9]{20,}",
        "Stripe restricted API key detected",
    ),
    (
        "GitHub Personal Access Token",
        r"ghp_[a-zA-Z0-9]{36,}",
        "GitHub personal access token detected",
    ),
    (
        "GitHub OAuth Token",
        r"gho_[a-zA-Z0-9]{36,}",
        "GitHub OAuth access token detected",
    ),
    (
        "GitHub App Token",
        r"(ghu|ghs|ghr)_[a-zA-Z0-9]{36,}",
        "GitHub app token detected",
    ),
    (
        "AWS Access Key ID",
        r"AKIA[0-9A-Z]{16}",
        "AWS access key ID detected",
    ),
    (
        "AWS Secret Access Key",
        r"(?i)aws_secret_access_key\s*[=:]\s*['\"]?[a-zA-Z0-9/+=]{40}",
        "AWS secret access key detected",
    ),
    (
        "Generic JWT",
        r"eyJ[a-zA-Z0-9_-]{10,}\.[a-zA-Z0-9_-]{10,}\.[a-zA-Z0-9_-]{10,}",
        "JWT token detected (may be a real token, not a test fixture)",
    ),
    (
        "RSA Private Key",
        r"-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----",
        "Private key detected",
    ),
    (
        "Hardcoded Password",
        r"""(?i)(password|passwd|pwd)\s*[=:]\s*['"][^'"]{8,}['"]""",
        "Hardcoded password string detected",
    ),
    (
        "Slack Webhook",
        r"https://hooks\.slack\.com/services/T[a-zA-Z0-9_]+/B[a-zA-Z0-9_]+/[a-zA-Z0-9_]+",
        "Slack webhook URL detected",
    ),
    (
        "Google API Key",
        r"AIza[0-9A-Za-z_-]{35}",
        "Google API key detected",
    ),
    (
        "Twilio Auth Token",
        r"(?i)twilio.*['\"][a-f0-9]{32}['\"]",
        "Twilio auth token detected",
    ),
    (
        "SendGrid API Key",
        r"SG\.[a-zA-Z0-9_-]{22}\.[a-zA-Z0-9_-]{43}",
        "SendGrid API key detected",
    ),
]

# Files to skip (test fixtures, examples, documentation)
SKIP_PATTERNS = [
    r"\.test\.(ts|js|py)$",
    r"\.spec\.(ts|js|py)$",
    r"__tests__/",
    r"test_.*\.py$",
    r"\.md$",
    r"CLAUDE\.md$",
    r"\.example$",
    r"\.sample$",
]

# Check if file should be skipped
for skip_pattern in SKIP_PATTERNS:
    if re.search(skip_pattern, file_path):
        sys.exit(0)

# Scan content for secrets
findings = []
for name, pattern, description in SECRET_PATTERNS:
    matches = re.findall(pattern, content)
    if matches:
        findings.append(f"  - {name}: {description} ({len(matches)} occurrence(s))")

if findings:
    print(f"BLOCKED: Potential secrets detected in '{file_path}':")
    print()
    for finding in findings:
        print(finding)
    print()
    print("If these are test fixtures or examples, move them to a test file.")
    print("If these are real secrets, use environment variables or a secrets manager.")
    sys.exit(2)

sys.exit(0)
```

### 7.2 Dependency license check

Blocks `npm install` or `pnpm add` commands that would add GPL/AGPL-licensed packages to the project.

```python
#!/usr/bin/env python3
"""PreToolUse hook: Block installation of packages with restrictive licenses."""

import json
import os
import re
import subprocess
import sys

TOOL_NAME = os.environ.get("CLAUDE_TOOL_NAME", "")
if TOOL_NAME != "Bash":
    sys.exit(0)

try:
    tool_input = json.loads(os.environ.get("CLAUDE_TOOL_INPUT", "{}"))
except json.JSONDecodeError:
    sys.exit(0)

command = tool_input.get("command", "")
if not command:
    sys.exit(0)

# Match npm install / pnpm add / yarn add with package names
install_match = re.search(
    r"(?:npm\s+install|npm\s+i|pnpm\s+add|yarn\s+add)\s+(.+)", command
)

if not install_match:
    sys.exit(0)

# Extract package names (filter out flags)
raw_packages = install_match.group(1).split()
packages = [p for p in raw_packages if not p.startswith("-")]

if not packages:
    sys.exit(0)

# Blocked license patterns (case-insensitive)
BLOCKED_LICENSES = [
    r"GPL-2\.0",
    r"GPL-3\.0",
    r"AGPL-.*",
    r"SSPL-.*",
    r"EUPL-.*",
    r"OSL-3\.0",
    r"^GPL$",
    r"^AGPL$",
]

BLOCKED_RE = re.compile("|".join(BLOCKED_LICENSES), re.IGNORECASE)

blocked_packages = []

for package in packages:
    # Strip version specifier for lookup
    pkg_name = re.sub(r"@[\^~]?[\d.]+.*$", "", package)
    if not pkg_name:
        continue

    try:
        result = subprocess.run(
            ["npm", "view", pkg_name, "license", "--json"],
            capture_output=True,
            text=True,
            timeout=10,
        )
        if result.returncode != 0:
            continue

        license_info = json.loads(result.stdout)
        # npm view returns a string or a list
        if isinstance(license_info, str):
            licenses = [license_info]
        elif isinstance(license_info, list):
            licenses = license_info
        else:
            continue

        for lic in licenses:
            if BLOCKED_RE.search(str(lic)):
                blocked_packages.append(f"  - {pkg_name}: {lic}")
                break

    except (subprocess.TimeoutExpired, json.JSONDecodeError, FileNotFoundError):
        # If we can't check, allow it (fail open for availability)
        continue

if blocked_packages:
    print("BLOCKED: Packages with restrictive licenses detected:")
    print()
    for bp in blocked_packages:
        print(bp)
    print()
    print("GPL/AGPL/SSPL licenses are not compatible with this project's license.")
    print("Choose an alternative package with MIT, Apache-2.0, BSD, or ISC license.")
    sys.exit(2)

sys.exit(0)
```

### 7.3 Conventional commit validator (Python)

A Python implementation of conventional commit validation with richer error messages and support for multi-line commit messages.

```python
#!/usr/bin/env python3
"""PreToolUse hook: Validate conventional commit messages (Python version)."""

import json
import os
import re
import sys

TOOL_NAME = os.environ.get("CLAUDE_TOOL_NAME", "")
if TOOL_NAME != "Bash":
    sys.exit(0)

try:
    tool_input = json.loads(os.environ.get("CLAUDE_TOOL_INPUT", "{}"))
except json.JSONDecodeError:
    sys.exit(0)

command = tool_input.get("command", "")
if not command:
    sys.exit(0)

# Match git commit -m "message" patterns
# Handle single quotes, double quotes, and heredoc-style
commit_patterns = [
    r'git\s+commit\s+.*-m\s+"([^"]+)"',
    r"git\s+commit\s+.*-m\s+'([^']+)'",
    r'git\s+commit\s+.*-m\s+"([^"]+)"',
    # Heredoc pattern
    r"git\s+commit\s+-m\s+\"\$\(cat\s+<<['\"]?EOF['\"]?\n(.*?)EOF",
]

commit_msg = None
for pattern in commit_patterns:
    match = re.search(pattern, command, re.DOTALL)
    if match:
        commit_msg = match.group(1).strip()
        break

if commit_msg is None:
    # Not a git commit command with -m
    sys.exit(0)

# Extract subject line (first line of commit message)
subject = commit_msg.split("\n")[0].strip()

# Valid conventional commit types
VALID_TYPES = [
    "feat",     # New feature
    "fix",      # Bug fix
    "docs",     # Documentation only
    "style",    # Formatting, missing semicolons, etc. (no code change)
    "refactor", # Code change that neither fixes a bug nor adds a feature
    "perf",     # Performance improvement
    "test",     # Adding or correcting tests
    "build",    # Build system or external dependencies
    "ci",       # CI configuration files and scripts
    "chore",    # Other changes that don't modify src or test files
    "revert",   # Reverts a previous commit
]

# Conventional commit regex
# type(optional-scope)!: description
CC_PATTERN = re.compile(
    r"^(?P<type>" + "|".join(VALID_TYPES) + r")"
    r"(?:\((?P<scope>[a-z0-9_.\-/]+)\))?"
    r"(?P<breaking>!)?"
    r":\s+"
    r"(?P<description>.+)$"
)

match = CC_PATTERN.match(subject)

if not match:
    print("BLOCKED: Commit message does not follow conventional commits format.")
    print()
    print(f"  Your subject: \"{subject}\"")
    print()
    print("  Expected format: type(scope): description")
    print()
    print("  Valid types:")
    for t in VALID_TYPES:
        print(f"    - {t}")
    print()
    print("  Examples:")
    print("    feat(auth): add OAuth2 login flow")
    print("    fix: resolve null pointer in user lookup")
    print("    docs(readme): update installation instructions")
    print("    refactor!: redesign database schema (breaking change)")
    sys.exit(2)

# Additional quality checks (warnings, not blocks)
warnings = []

description = match.group("description")

# Subject too long
if len(subject) > 72:
    warnings.append(
        f"Subject line is {len(subject)} chars (recommended max: 72)."
    )

# Description starts with uppercase
if description and description[0].isupper():
    warnings.append(
        "Description should start with lowercase (e.g., 'add feature' not 'Add feature')."
    )

# Description ends with period
if description and description.endswith("."):
    warnings.append(
        "Description should not end with a period."
    )

if warnings:
    print("Conventional commit format OK, but with suggestions:")
    for w in warnings:
        print(f"  - {w}")
    # Exit 1 = warning, proceed but show to model
    sys.exit(1)

sys.exit(0)
```

---

## 8. Performance rules

Hooks are synchronous. Every PreToolUse hook blocks the tool call until it completes. Every PostToolUse hook blocks the model from seeing the tool result. This makes hook performance critical to session responsiveness.

### Performance budget

**Target: < 200ms per hook.** This is the threshold where hooks feel invisible. Above 200ms, the user starts noticing lag. Above 500ms, the experience degrades noticeably. Above 1 second, hooks become a serious productivity drag.

Most well-written hooks finish in 5-50ms. File path checks, regex matching, and jq parsing are all sub-millisecond operations. The performance budget is generous — the risk comes from accidentally including slow operations.

### Fast patterns

Things that are fast and safe to do in hooks:

- **String matching**: `grep -qE`, regex, case statements — microseconds
- **jq parsing**: Parsing `CLAUDE_TOOL_INPUT` with jq — single-digit milliseconds
- **File existence checks**: `[[ -f "$PATH" ]]` — sub-millisecond
- **Path resolution**: `realpath`, `dirname`, `basename` — sub-millisecond
- **Environment variable reads**: Free
- **Line counting**: `wc -l` on content already in memory — sub-millisecond

### Async pattern for expensive PostToolUse hooks

When a PostToolUse hook needs to do something expensive (lint, type check, test run), use the fire-and-forget pattern to avoid blocking:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Quick checks first (< 1ms)
[[ "${CLAUDE_TOOL_NAME:-}" != "Edit" ]] && exit 0
FILE_PATH=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.file_path // ""')
case "$FILE_PATH" in *.ts|*.tsx) ;; *) exit 0 ;; esac

# Expensive operation in background
(
  cd "${CLAUDE_PROJECT_DIR:-.}"
  npx eslint --fix "$FILE_PATH" 2>&1 > /tmp/eslint-last-run.log
) &
disown

# Return immediately — model is not blocked
echo "Lint started in background."
exit 0
```

The key elements:
1. Wrap the expensive operation in `( ... ) &` (subshell in background)
2. Call `disown` so the shell doesn't wait for it
3. Exit immediately with a brief message

### Anti-patterns

**HTTP requests in hooks** — Network calls add 100ms-2000ms of latency. Never make HTTP requests in PreToolUse hooks. If you need external data, cache it locally and read the cache.

```bash
# BAD: HTTP request in a hook (200-2000ms)
LICENSE=$(curl -s "https://registry.npmjs.org/$PACKAGE" | jq -r '.license')

# GOOD: Use cached local data or npm's local cache
LICENSE=$(npm view "$PACKAGE" license 2>/dev/null)
```

**Full test suite in PostToolUse** — Running the full test suite after every edit will grind the session to a halt. Run targeted tests for changed files only, or use the async pattern.

```bash
# BAD: Full test suite on every edit (10-60 seconds)
npm test

# GOOD: Targeted test in background
(npx jest --findRelatedTests "$FILE_PATH" 2>&1 > /tmp/test-latest.log) &
disown
```

**Recursive directory scans** — Scanning the entire project tree is slow on large codebases.

```bash
# BAD: Scan entire tree (100ms-10s depending on size)
find "$PROJECT_DIR" -name "*.ts" -exec grep -l "TODO" {} \;

# GOOD: Check only the file being operated on
grep -c "TODO" "$FILE_PATH" 2>/dev/null || true
```

**Spawning heavy interpreters** — Starting Python, Node, or Ruby interpreters has overhead (30-100ms). For simple checks, use bash. Reserve Python/Node for hooks where you need their libraries.

### Optimization techniques

**Check file extension first** — Most hooks only apply to certain file types. Exit early before doing any work:

```bash
case "$FILE_PATH" in
  *.ts|*.tsx) ;; # Continue
  *) exit 0 ;;   # Skip immediately
esac
```

**Cache tool availability** — Don't check `command -v` or `which` on every invocation. Check once and store the result:

```bash
# Cache eslint path in a temp file
ESLINT_CACHE="/tmp/claude-hook-eslint-path"
if [[ -f "$ESLINT_CACHE" ]]; then
  ESLINT=$(cat "$ESLINT_CACHE")
else
  ESLINT=$(command -v eslint 2>/dev/null || echo "")
  echo "$ESLINT" > "$ESLINT_CACHE"
fi
```

**Use jq -e for boolean checks** — `jq -e` exits with code 1 when the expression evaluates to false or null, avoiding string comparison:

```bash
# Instead of parsing and comparing strings:
VALUE=$(echo "$CLAUDE_TOOL_INPUT" | jq -r '.replace_all // "false"')
if [[ "$VALUE" == "true" ]]; then ...

# Use jq -e directly:
if echo "$CLAUDE_TOOL_INPUT" | jq -e '.replace_all' &>/dev/null; then
  # replace_all is true
fi
```

---

## 9. Debugging hooks

### Simulating hook execution

You can test hooks outside of Claude Code by exporting the environment variables and running the script directly:

```bash
# Simulate a PreToolUse hook for a Bash command
export CLAUDE_TOOL_NAME="Bash"
export CLAUDE_TOOL_INPUT='{"command": "rm -rf /"}'
export CLAUDE_SESSION_ID="test-session-123"
export CLAUDE_PROJECT_DIR="/Users/you/project"
export CLAUDE_USER="you"

# Run the hook directly
bash .claude/hooks/block-dangerous-commands.sh
echo "Exit code: $?"

# Simulate a Write tool with secret content
export CLAUDE_TOOL_NAME="Write"
export CLAUDE_TOOL_INPUT='{"file_path": "/Users/you/project/config.ts", "content": "const key = \"sk_live_YOUR_KEY_HERE\""}'

python3 .claude/hooks/detect-secrets.py
echo "Exit code: $?"
```

### DRY_RUN pattern

Add a dry-run mode to hooks for testing without side effects:

```bash
#!/usr/bin/env bash
set -euo pipefail

DRY_RUN="${DRY_RUN:-0}"

# ... validation logic ...

if [[ "$DRY_RUN" == "1" ]]; then
  echo "[DRY RUN] Would block: $REASON"
  exit 0  # Don't actually block in dry-run mode
fi

echo "BLOCKED: $REASON"
exit 2
```

Test with:

```bash
export CLAUDE_TOOL_NAME="Bash"
export CLAUDE_TOOL_INPUT='{"command": "rm -rf /"}'
DRY_RUN=1 bash .claude/hooks/block-dangerous-commands.sh
```

### stdout vs. stderr for debugging

During development, use stderr for debug output that should appear in the terminal but not be passed to the model:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Debug output — visible to user in Claude Code UI, NOT passed to model
echo "DEBUG: TOOL_NAME=$CLAUDE_TOOL_NAME" >&2
echo "DEBUG: TOOL_INPUT=$CLAUDE_TOOL_INPUT" >&2

# Model-facing output — this IS passed to the model as context
echo "File path is within project boundary."

exit 0
```

### Diagnosing hook failures

When a hook isn't working as expected:

1. **Check permissions**: The hook file must be executable.
   ```bash
   chmod +x .claude/hooks/my-hook.sh
   ```

2. **Check the shebang**: Ensure the script starts with the correct interpreter.
   ```bash
   #!/usr/bin/env bash    # for shell scripts
   #!/usr/bin/env python3  # for Python scripts
   ```

3. **Check jq is installed**: Many hooks depend on jq for JSON parsing.
   ```bash
   command -v jq || echo "jq not found — install with: brew install jq"
   ```

4. **Check settings.json syntax**: A malformed settings.json silently disables all hooks. Validate it:
   ```bash
   python3 -m json.tool .claude/settings.json > /dev/null
   ```

5. **Check matcher patterns**: Ensure the matcher matches the tool name exactly. Common mistakes:
   - `"bash"` (lowercase) does NOT match `"Bash"` (tool names are PascalCase)
   - `"write"` does NOT match `"Write"`
   - `"MCP__stripe__*"` does NOT match `"mcp__stripe__*"` (MCP tool names are lowercase-prefixed)

6. **Test in isolation**: Export the expected environment variables and run the hook directly (see Simulating hook execution above).

### Temporary disable with disableAllHooks

When hooks are causing problems during a session and you need to bypass them quickly:

```json
{
  "hooks": { "...": "..." },
  "disableAllHooks": true
}
```

Add this to your project's `.claude/settings.json` to disable all hooks without removing the configuration. Remove the line to re-enable. This is useful for debugging or when a hook has a bug that blocks all work.

You can also use the user-level `~/.claude/settings.json` to disable hooks globally across all projects.

### Common debugging workflow

```bash
# 1. Identify which hook is failing
# Check Claude Code output for hook error messages

# 2. Find the hook script
cat .claude/settings.json | jq '.hooks'

# 3. Simulate the failing tool call
export CLAUDE_TOOL_NAME="Bash"
export CLAUDE_TOOL_INPUT='{"command": "the command that was blocked"}'
export CLAUDE_PROJECT_DIR="$(pwd)"

# 4. Run with debug output
bash -x .claude/hooks/the-hook.sh

# 5. Check exit code
echo "Exit: $?"

# 6. Fix the issue and test again
# 7. Remove disableAllHooks from settings.json
```

---

## Quick reference: hook checklist

When creating a new hook:

1. **Shebang line**: `#!/usr/bin/env bash` or `#!/usr/bin/env python3`
2. **Strict mode**: `set -euo pipefail` (bash) or proper error handling (Python)
3. **Guard clause**: Exit 0 immediately if the tool name doesn't match
4. **Parse input**: Use jq (bash) or `json.loads(os.environ[...])` (Python)
5. **Validate early**: Check for empty/missing values before processing
6. **Clear messages**: stdout messages should tell the model exactly what went wrong and what to do instead
7. **Correct exit code**: 0 = proceed, 2 = block, 1 = warn
8. **Performance**: Target < 200ms, use async for expensive PostToolUse operations
9. **Make executable**: `chmod +x .claude/hooks/my-hook.sh`
10. **Test in isolation**: Export env vars, run directly, check exit code and output
