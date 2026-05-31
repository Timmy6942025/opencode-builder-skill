---
name: opencode-tools
description: |
  Use this skill when working with OpenCode's built-in tools (bash, read, write, edit, grep, glob, lsp, apply_patch, skill, todowrite, webfetch, websearch, question), creating custom tools in TypeScript/JavaScript, configuring tool permissions, understanding MCP tool integration, or understanding tool internals like ripgrep integration, .ignore files, and tool precedence.
---

# OpenCode Tools

> **📚 Official Docs:** For the latest information, always refer to the official documentation:
> [https://opencode.ai/docs/tools/](https://opencode.ai/docs/tools/) and [https://opencode.ai/docs/custom-tools/](https://opencode.ai/docs/custom-tools/)

OpenCode provides a set of built-in tools that let LLMs interact with your codebase. You can extend functionality with [custom tools](#custom-tools) or [MCP server integrations](#mcp-tools). All tools are **enabled** by default and don't require permission unless configured otherwise.

Tools allow the LLM to perform actions: reading files, editing code, running shell commands, searching content, fetching web pages, asking questions, and more. Permissions control which actions require approval.

---

## Table of Contents

- [Built-in Tools](#built-in-tools)
  - [bash](#bash)
  - [edit](#edit)
  - [write](#write)
  - [read](#read)
  - [grep](#grep)
  - [glob](#glob)
  - [lsp (experimental)](#lsp-experimental)
  - [apply_patch](#apply_patch)
  - [skill](#skill)
  - [todowrite](#todowrite)
  - [webfetch](#webfetch)
  - [websearch](#websearch)
  - [question](#question)
- [Permission Configuration](#permission-configuration)
  - [Actions](#actions)
  - [Global Configuration](#global-configuration)
  - [Granular Rules (Object Syntax)](#granular-rules-object-syntax)
  - [Wildcards](#wildcards)
  - [Home Directory Expansion](#home-directory-expansion)
  - [External Directories](#external-directories)
  - [Doom Loop Detection](#doom-loop-detection)
  - [Available Permission Keys](#available-permission-keys)
  - [Defaults](#defaults)
  - [What "Ask" Does](#what-ask-does)
  - [Agent Permissions](#agent-permissions)
- [Custom Tools](#custom-tools)
  - [Location](#location)
  - [Structure](#structure)
  - [Multiple Tools Per File](#multiple-tools-per-file)
  - [Name Collisions with Built-in Tools](#name-collisions-with-built-in-tools)
  - [Arguments (Zod Schema)](#arguments-zod-schema)
  - [Context Object](#context-object)
  - [Invoking Scripts in Any Language](#invoking-scripts-in-any-language)
  - [Complete Examples](#complete-examples)
- [MCP Tools](#mcp-tools)
- [Internals](#internals)
  - [Ripgrep Integration](#ripgrep-integration)
  - [Ignore Patterns](#ignore-patterns)
- [Tool Precedence](#tool-precedence)
- [Defaults Summary](#defaults-summary)

---

## Built-in Tools

### `bash`

**Permission key:** `bash`
**Matches against:** Parsed command string

Execute shell commands in your project environment. This tool allows the LLM to run terminal commands like `npm install`, `git status`, `python3 script.py`, or any other shell command.

Permission rules match against the parsed command string. For example, `"git status --porcelain"` is matched against the pattern.

```json
{ "permission": { "bash": "allow" } }
```

**Granular example** — allow git and npm, deny rm, ask for everything else:

```json
{
  "permission": {
    "bash": {
      "*": "ask",
      "git *": "allow",
      "npm *": "allow",
      "rm *": "deny",
      "grep *": "allow"
    }
  }
}
```

**Important:** Commands with arguments require explicit patterns. `"grep *"` allows `grep pattern file.txt`, while `"grep"` alone (without wildcard) would block commands with arguments. Commands like `git status` work for default behavior but need `"git status *"` when arguments are passed.

---

### `edit`

**Permission key:** `edit`
**Matches against:** File path

Modify existing files using exact string replacements. This is the primary way the LLM modifies code — it finds an exact string in a file and replaces it with new content.

The `edit` permission also controls `write` and `apply_patch`. All file modifications share the `edit` permission key.

```json
{ "permission": { "edit": "allow" } }
```

**Granular example** — deny edits globally, allow only docs:

```json
{
  "permission": {
    "edit": {
      "*": "deny",
      "packages/web/src/content/docs/*.mdx": "allow"
    }
  }
}
```

---

### `write`

**Permission key:** `edit` (shared with `edit` and `apply_patch`)
**Matches against:** File path

Create new files or overwrite existing ones. This is separate from `edit` in functionality (full file creation/overwrite vs. string replacement) but shares the same permission key.

There is no separate `write` permission key — it is controlled entirely by the `edit` permission.

```json
{ "permission": { "edit": "allow" } }
```

---

### `read`

**Permission key:** `read`
**Matches against:** File path

Read file contents from your codebase. Supports reading specific line ranges for large files using offset and limit parameters. Returns file contents with line numbers.

```json
{ "permission": { "read": "allow" } }
```

**`.env` files are denied by default.** The default read permission is:

```json
{
  "permission": {
    "read": {
      "*": "allow",
      "*.env": "deny",
      "*.env.*": "deny",
      "*.env.example": "allow"
    }
  }
}
```

---

### `grep`

**Permission key:** `grep`
**Matches against:** Regex pattern

Search file contents using regular expressions. Uses [ripgrep](https://github.com/BurntSushi/ripgrep) under the hood for fast content search across your codebase. Supports full regex syntax and file pattern filtering (include/exclude).

```json
{ "permission": { "grep": "allow" } }
```

---

### `glob`

**Permission key:** `glob`
**Matches against:** Glob pattern

Find files by pattern matching using glob patterns like `**/*.js` or `src/**/*.ts`. Uses [ripgrep](https://github.com/BurntSushi/ripgrep) under the hood. Returns matching file paths sorted by modification time (most recent first).

```json
{ "permission": { "glob": "allow" } }
```

---

### `lsp` (experimental)

**Permission key:** `lsp`
**Matches against:** Non-granular

Interact with configured LSP servers for code intelligence features. Provides operations like finding definitions, references, hover information, call hierarchy, and symbol search.

**Requires** the environment variable `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` (or the broader `OPENCODE_EXPERIMENTAL=true`). This tool is not available by default.

```json
{ "permission": { "lsp": "allow" } }
```

**Supported operations:**

| Operation | Description |
|---|---|
| `goToDefinition` | Jump to the definition of a symbol |
| `findReferences` | Find all references to a symbol |
| `hover` | Get hover information for a symbol |
| `documentSymbol` | List symbols in the current document |
| `workspaceSymbol` | Search for symbols across the workspace |
| `goToImplementation` | Jump to the implementation of an interface/trait |
| `prepareCallHierarchy` | Prepare call hierarchy for a symbol |
| `incomingCalls` | Get incoming calls to a function |
| `outgoingCalls` | Get outgoing calls from a function |

To configure which LSP servers are available, see [LSP Servers](https://opencode.ai/docs/lsp/).

---

### `apply_patch`

**Permission key:** `edit` (shared with `edit` and `write`)
**Matches against:** Embedded path markers in patch text

Apply patches and diffs to files. Useful for applying changes from various sources. Unlike `edit` which uses `filePath`, `apply_patch` uses `patchText` with embedded marker lines that specify paths relative to the project root.

```json
{ "permission": { "edit": "allow" } }
```

**Patch format — paths are embedded as marker lines in `patchText`:**

```
*** Add File: src/new-file.ts
*** Update File: src/existing.ts
*** Move to: src/renamed.ts
*** Delete File: src/obsolete.ts
```

**Hook integration:** When handling `tool.execute.before` or `tool.execute.after` hooks, check `input.tool === "apply_patch"` (not `"patch"`). Use `output.args.patchText` to access the patch content, not `output.args.filePath`.

The `apply_patch` tool is controlled by the `edit` permission, which covers all file modifications (`edit`, `write`, `apply_patch`).

---

### `skill`

**Permission key:** `skill`
**Matches against:** Skill name

Load a [skill](https://opencode.ai/docs/skills/) (a `SKILL.md` file) and return its content in the conversation. Permission rules match against the skill name.

```json
{ "permission": { "skill": "allow" } }
```

---

### `todowrite`

**Permission key:** `todowrite`
**Matches against:** Non-granular

Manage todo lists during coding sessions. Creates and updates task lists to track progress during complex, multi-step operations. The LLM uses this tool to organize work, track completed items, and manage pending tasks.

```json
{ "permission": { "todowrite": "allow" } }
```

**Note:** This tool is **disabled for subagents by default**, but you can enable it manually. See [Agent Permissions](#agent-permissions).

---

### `webfetch`

**Permission key:** `webfetch`
**Matches against:** URL

Fetch and read web pages. Allows the LLM to retrieve content from specific URLs. Useful for looking up documentation, researching online resources, or reading API responses.

Permission rules match against the full URL.

```json
{ "permission": { "webfetch": "allow" } }
```

---

### `websearch`

**Permission key:** `websearch`
**Matches against:** Query string

Search the web using Exa AI. Performs web searches to find relevant information online. Useful for researching topics, finding current events, or gathering information beyond the training data cutoff.

**Availability:** Only available when using the OpenCode provider or when the `OPENCODE_ENABLE_EXA` environment variable is set to any truthy value (`true`, `1`).

```json
{ "permission": { "websearch": "allow" } }
```

**Enabling:**

```bash
OPENCODE_ENABLE_EXA=1 opencode
```

No API key is required — the tool connects directly to Exa AI's hosted MCP service without authentication.

**Tip:** Use `websearch` when you need to find information (discovery), and `webfetch` when you need to retrieve content from a specific URL (retrieval).

---

### `question`

**Permission key:** `question`
**Matches against:** Non-granular

Ask the user questions during execution. This tool allows the LLM to interactively gather input from the user mid-task. Each question includes a header, question text, and a list of options. Users can select from the provided options or type a custom answer.

```json
{ "permission": { "question": "allow" } }
```

**Use cases:**
- Gathering user preferences or requirements
- Clarifying ambiguous instructions
- Getting decisions on implementation choices
- Offering choices about what direction to take

When there are multiple questions, users can navigate between them before submitting all answers.

---

## Permission Configuration

### Actions

Each permission rule resolves to one of three actions:

| Action | Behavior |
|---|---|
| `"allow"` | Run without approval |
| `"ask"` | Prompt for approval before running |
| `"deny"` | Block the action entirely |

### Global Configuration

Set a single action for all tools:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": "allow"
}
```

Set a default with per-tool overrides:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "*": "ask",
    "bash": "allow",
    "edit": "deny"
  }
}
```

### Granular Rules (Object Syntax)

For most permissions, you can use an object to apply different actions based on the tool input. Rules are evaluated by pattern match, with the **last matching rule winning**. A common pattern is to put the catch-all `"*"` rule first, and more specific rules after it.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "bash": {
      "*": "ask",
      "git *": "allow",
      "npm *": "allow",
      "rm *": "deny",
      "grep *": "allow"
    },
    "edit": {
      "*": "deny",
      "packages/web/src/content/docs/*.mdx": "allow"
    }
  }
}
```

**Rule evaluation order:** The last matching pattern wins. This means you can layer broad catch-all rules with increasingly specific overrides.

### Wildcards

Permission patterns use simple wildcard matching:

| Character | Meaning |
|---|---|
| `*` | Matches zero or more of any character |
| `?` | Matches exactly one character |
| All other characters | Match literally |

**Examples:**
- `"git *"` — matches `git status`, `git commit -m "msg"`, `git push origin main`
- `"git ?*"` — matches `git status` but not `git` alone
- `"*.md"` — matches `README.md`, `docs/guide.md`
- `"rm *"` — matches `rm file.txt`, `rm -rf dir/`

### Home Directory Expansion

Use `~` or `$HOME` at the start of a pattern to reference your home directory. This is particularly useful for `external_directory` rules.

| Pattern | Expands To |
|---|---|
| `~/projects/*` | `/Users/username/projects/*` |
| `$HOME/projects/*` | `/Users/username/projects/*` |
| `~` | `/Users/username` |

### External Directories

Use `external_directory` to allow tool calls that touch paths outside the working directory where OpenCode was started. This applies to any tool that takes a path as input (e.g., `read`, `edit`, `glob`, `grep`, and many `bash` commands).

Home expansion (`~/...`) only affects how a pattern is written — it does not make an external path part of the current workspace. Paths outside the working directory must still be allowed via `external_directory`.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "external_directory": {
      "~/projects/personal/**": "allow"
    }
  }
}
```

Any directory allowed here inherits the same defaults as the current workspace. Since `read` defaults to `"allow"`, reads are also allowed for entries under `external_directory` unless overridden. Layer extra rules when tools should be restricted:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "external_directory": {
      "~/projects/personal/**": "allow"
    },
    "edit": {
      "~/projects/personal/**": "deny"
    }
  }
}
```

Keep the list focused on trusted paths, and layer extra allow or deny rules as needed for other tools (e.g., `bash`).

### Doom Loop Detection

The `doom_loop` permission triggers when the same tool call repeats **3 times with identical input**. This prevents infinite loops where the LLM repeatedly attempts the same failing operation.

Default: `"ask"` — prompts for approval when a doom loop is detected.

### Available Permission Keys

| Key | Matches Against | Description |
|---|---|---|
| `read` | File path | Reading file contents |
| `edit` | File path | All file modifications (covers `edit`, `write`, `apply_patch`) |
| `glob` | Glob pattern | File pattern matching |
| `grep` | Regex pattern | Content search |
| `bash` | Parsed command string | Shell command execution |
| `task` | Subagent type | Launching subagents |
| `skill` | Skill name | Loading SKILL.md files |
| `lsp` | Non-granular | LSP server queries |
| `question` | Non-granular | Asking user questions |
| `webfetch` | URL | Fetching web content |
| `websearch` | Query string | Web search via Exa AI |
| `external_directory` | Path outside working directory | Access to paths outside the project |
| `doom_loop` | Repeated identical input | Infinite loop detection |

### Defaults

If you don't specify anything, OpenCode starts from permissive defaults:

| Permission | Default |
|---|---|
| Most tools | `"allow"` |
| `read` | `"allow"` (`.env` files denied by default) |
| `doom_loop` | `"ask"` |
| `external_directory` | `"ask"` |
| `todowrite` (subagents) | Disabled |
| `lsp` | Requires `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` |
| `websearch` | Requires OpenCode provider or `OPENCODE_ENABLE_EXA=true` |

**`.env` file defaults:**

```json
{
  "permission": {
    "read": {
      "*": "allow",
      "*.env": "deny",
      "*.env.*": "deny",
      "*.env.example": "allow"
    }
  }
}
```

### What "Ask" Does

When OpenCode prompts for approval, the UI offers three outcomes:

| Option | Behavior |
|---|---|
| `once` | Approve just this request |
| `always` | Approve future requests matching the suggested patterns (for the rest of the current session) |
| `reject` | Deny the request |

The set of patterns that `always` would approve is provided by the tool. For example, bash approvals typically whitelist a safe command prefix like `git status*`.

### Agent Permissions

You can override permissions per agent. Agent rules merge with the global config and take precedence. Agent-specific patterns override global patterns for that agent only.

**In `opencode.json`:**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "bash": {
      "*": "ask",
      "git *": "allow",
      "git commit *": "deny",
      "git push *": "deny",
      "grep *": "allow"
    }
  },
  "agent": {
    "build": {
      "permission": {
        "bash": {
          "*": "ask",
          "git *": "allow",
          "git commit *": "ask",
          "git push *": "deny",
          "grep *": "allow"
        }
      }
    }
  }
}
```

**In Markdown agent files** (`~/.config/opencode/agents/review.md`):

```markdown
---
description: Code review without edits
mode: subagent
permission:
  edit: deny
  bash: ask
  webfetch: deny
---
Only analyze code and suggest changes.
```

---

## Custom Tools

Custom tools are functions you create that the LLM can call during conversations. They work alongside OpenCode's built-in tools. Tool definitions are written in **TypeScript** or **JavaScript**, but can invoke scripts in **any language**.

### Location

| Scope | Path |
|---|---|
| Project-level | `.opencode/tools/` in your project root |
| Global | `~/.config/opencode/tools/` |

Place TypeScript or JavaScript files in either location. The filename (without extension) becomes the tool name.

### Structure

Use the `tool()` helper from `@opencode-ai/plugin` for type-safety and validation:

```typescript
// .opencode/tools/database.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Query the project database",
  args: {
    query: tool.schema.string().describe("SQL query to execute"),
  },
  async execute(args) {
    return `Executed query: ${args.query}`
  },
})
```

The **filename** becomes the **tool name** — this creates a `database` tool.

You can also import Zod directly and export a plain object:

```typescript
import { z } from "zod"

export default {
  description: "Tool description",
  args: {
    param: z.string().describe("Parameter description"),
  },
  async execute(args, context) {
    return "result"
  },
}
```

### Multiple Tools Per File

Each named export becomes a separate tool with the name `<filename>_<exportname>`:

```typescript
// .opencode/tools/math.ts
import { tool } from "@opencode-ai/plugin"

export const add = tool({
  description: "Add two numbers",
  args: {
    a: tool.schema.number().describe("First number"),
    b: tool.schema.number().describe("Second number"),
  },
  async execute(args) {
    return (args.a + args.b).toString()
  },
})

export const multiply = tool({
  description: "Multiply two numbers",
  args: {
    a: tool.schema.number().describe("First number"),
    b: tool.schema.number().describe("Second number"),
  },
  async execute(args) {
    return (args.a * args.b).toString()
  },
})
```

This creates two tools: `math_add` and `math_multiply`.

### Name Collisions with Built-in Tools

Custom tools are keyed by tool name. If a custom tool uses the same name as a built-in tool, the **custom tool takes precedence** and replaces the built-in.

For example, this file replaces the built-in `bash` tool:

```typescript
// .opencode/tools/bash.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Restricted bash wrapper",
  args: {
    command: tool.schema.string(),
  },
  async execute(args) {
    return `blocked: ${args.command}`
  },
})
```

**Prefer unique names** unless you intentionally want to replace a built-in tool. If you want to disable a built-in without overriding it, use [permissions](#permission-configuration) instead.

### Arguments (Zod Schema)

Use `tool.schema` (which wraps [Zod](https://zod.dev)) to define argument types with validation:

```typescript
args: {
  // String
  query: tool.schema.string().describe("SQL query to execute"),

  // Number
  limit: tool.schema.number().describe("Max results").optional(),

  // Boolean
  verbose: tool.schema.boolean().describe("Enable verbose output"),

  // Enum
  format: tool.schema.enum(["json", "text", "csv"]).describe("Output format"),

  // Array
  tags: tool.schema.array(tool.schema.string()).describe("Filter tags"),
}
```

**Available schema types:**

| Method | Type | Description |
|---|---|---|
| `tool.schema.string()` | `string` | Text values |
| `tool.schema.number()` | `number` | Numeric values |
| `tool.schema.boolean()` | `boolean` | True/false values |
| `tool.schema.enum([...])` | `enum` | One of a set of string values |
| `tool.schema.array(inner)` | `array` | List of values |
| `.optional()` | (any) | Makes the argument optional |
| `.describe(str)` | (any) | Adds a description for the LLM |

You can also import Zod directly:

```typescript
import { z } from "zod"

export default {
  description: "Tool description",
  args: {
    param: z.string().describe("Parameter description"),
  },
  async execute(args, context) {
    return "result"
  },
}
```

### Context Object

Tools receive session context as the second argument to `execute`:

```typescript
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Get project information",
  args: {},
  async execute(args, context) {
    const { agent, sessionID, messageID, directory, worktree } = context
    return `Agent: ${agent}, Dir: ${directory}, Worktree: ${worktree}`
  },
})
```

| Property | Type | Description |
|---|---|---|
| `context.agent` | `string` | Current agent name |
| `context.sessionID` | `string` | Active session identifier |
| `context.messageID` | `string` | Current message identifier |
| `context.directory` | `string` | Session working directory |
| `context.worktree` | `string` | Git worktree root |

Use `context.directory` for the session working directory. Use `context.worktree` for the git worktree root (important when working in git worktrees).

### Invoking Scripts in Any Language

Tool definitions must be TypeScript/JavaScript, but can invoke scripts in **any language** using `Bun.$`:

```python
# .opencode/tools/add.py
import sys
a = int(sys.argv[1])
b = int(sys.argv[2])
print(a + b)
```

```typescript
// .opencode/tools/python-add.ts
import { tool } from "@opencode-ai/plugin"
import path from "path"

export default tool({
  description: "Add two numbers using Python",
  args: {
    a: tool.schema.number().describe("First number"),
    b: tool.schema.number().describe("Second number"),
  },
  async execute(args, context) {
    const script = path.join(context.worktree, ".opencode/tools/add.py")
    const result = await Bun.$`python3 ${script} ${args.a} ${args.b}`.text()
    return result.trim()
  },
})
```

The [`Bun.$`](https://bun.sh/docs/runtime/shell) utility runs the external script and captures its output.

### Complete Examples

**Database query tool:**

```typescript
// .opencode/tools/database.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Query the project SQLite database",
  args: {
    query: tool.schema.string().describe("SQL query to execute"),
    database: tool.schema.string().describe("Database file path").optional(),
  },
  async execute(args, context) {
    const db = args.database || "data.db"
    const result = await Bun.$`sqlite3 ${db} ${args.query}`.text()
    return result.trim() || "(no results)"
  },
})
```

**Python integration tool:**

```typescript
// .opencode/tools/lint-python.ts
import { tool } from "@opencode-ai/plugin"
import path from "path"

export default tool({
  description: "Run Python linter on a file",
  args: {
    filePath: tool.schema.string().describe("Python file to lint"),
  },
  async execute(args, context) {
    const fullPath = path.resolve(context.worktree, args.filePath)
    const result = await Bun.$`python3 -m py_compile ${fullPath}`.text()
    return result.trim() || "No syntax errors"
  },
})
```

**File analysis tool with multiple exports:**

```typescript
// .opencode/tools/analyze.ts
import { tool } from "@opencode-ai/plugin"
import fs from "fs"
import path from "path"

export const lines = tool({
  description: "Count lines in a file",
  args: {
    path: tool.schema.string().describe("File path"),
  },
  async execute(args, context) {
    const content = fs.readFileSync(path.resolve(context.worktree, args.path), "utf8")
    const count = content.split("\n").length
    return `${count} lines`
  },
})

export const size = tool({
  description: "Get file size in bytes",
  args: {
    path: tool.schema.string().describe("File path"),
  },
  async execute(args, context) {
    const stat = fs.statSync(path.resolve(context.worktree, args.path))
    return `${stat.size} bytes`
  },
})
```

Creates tools: `analyze_lines` and `analyze_size`.

---

## MCP Tools

MCP (Model Context Protocol) servers allow you to integrate external tools and services. Once configured, MCP tools appear **alongside built-in tools** in the tool list available to the LLM.

MCP tools are registered with the **server name as prefix**, following the naming convention `<servername>_<toolname>`. For example, a Sentry MCP server might provide tools like `sentry_list_issues`, `sentry_get_issue`, etc.

**Permission example** — require approval for all tools from an MCP server:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "mymcp_*": "ask"
  }
}
```

**Caveats:**
- MCP servers add to the context window. Be careful with which servers you enable — too many tools can quickly exceed context limits.
- Certain MCP servers (like the GitHub MCP server) tend to add a lot of tokens and can easily exceed the context limit.
- MCP server tools can be enabled/disabled globally or per-agent using glob patterns in the `tools` config.

**Global enable/disable:**

```json
{
  "mcp": {
    "my-mcp-server": {
      "type": "local",
      "command": ["npx", "-y", "my-mcp-command"]
    }
  },
  "tools": {
    "my-mcp-server": false
  }
}
```

**Per-agent enable:**

```json
{
  "mcp": {
    "my-mcp": {
      "type": "local",
      "command": ["npx", "-y", "my-mcp-command"],
      "enabled": true
    }
  },
  "tools": {
    "my-mcp*": false
  },
  "agent": {
    "my-agent": {
      "tools": {
        "my-mcp*": true
      }
    }
  }
}
```

See [MCP Servers](https://opencode.ai/docs/mcp-servers/) for full configuration details including local servers, remote servers, and OAuth authentication.

---

## Internals

### Ripgrep Integration

The `grep` and `glob` tools use [ripgrep](https://github.com/BurntSushi/ripgrep) under the hood. Ripgrep is a fast search tool that provides:

- **Regex search** for `grep` — full regex syntax with file pattern filtering
- **Glob matching** for `glob` — pattern-based file discovery
- **`.gitignore` respect** — by default, files and directories listed in `.gitignore` are excluded from searches and listings

This means `node_modules/`, `dist/`, `build/`, and other gitignored directories are automatically skipped during searches unless explicitly included.

### Ignore Patterns

Create a `.ignore` file in your project root to override `.gitignore` exclusions. The `.ignore` file follows `.gitignore` syntax.

To **include** files that would normally be ignored, use the `!` prefix:

```
!node_modules/
!dist/
!build/
```

This allows ripgrep to search within `node_modules/`, `dist/`, and `build/` directories even if they're listed in `.gitignore`.

The `.ignore` file provides fine-grained control over what gets searched, independent of git tracking.

---

## Tool Precedence

When multiple tools share the same name, OpenCode resolves them in this order (highest priority first):

1. **Custom tools** (`.opencode/tools/` or `~/.config/opencode/tools/`) — highest priority
2. **Plugin tools** (from OpenCode plugins)
3. **Built-in tools** — lowest priority, can be overridden or replaced

This means a custom tool with the same name as a built-in tool will replace it. To disable a built-in tool without replacing it, use [permissions](#permission-configuration) rather than creating an override tool.

---

## Defaults Summary

| Permission | Default |
|---|---|
| Most tools | `"allow"` |
| `read` | `"allow"` (`.env` files denied) |
| `doom_loop` | `"ask"` |
| `external_directory` | `"ask"` |
| `todowrite` (subagents) | Disabled |
| `lsp` | Requires `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` |
| `websearch` | Requires OpenCode provider or `OPENCODE_ENABLE_EXA=true` |
