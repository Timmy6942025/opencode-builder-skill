---
name: opencode-builder
description: |
  Use this skill when the user is building extensions, plugins, integrations, or custom developer tools on top of OpenCode (opencode.ai). This includes: creating plugins with custom AI tools and lifecycle hooks, using the SDK for programmatic session control and multi-agent workflows, building model providers or MCP integrations, configuring opencode.json, and implementing advanced orchestration like checkpoint/resume, parallel execution, or adversarial review. Trigger for any development work that extends or automates OpenCode. Do NOT trigger for general usage questions, troubleshooting, UI theming, or tool comparisons.
---

# OpenCode Builder

A comprehensive skill for building on top of [OpenCode](https://opencode.ai) — an AI-powered coding assistant. This skill covers both the **JavaScript/TypeScript SDK** for programmatic control and the **plugin architecture** for extending OpenCode's behavior.

---

## Table of Contents

1. [Quick Start: Choose Your Path](#quick-start-choose-your-path)
2. [SDK Path: Programmatic Control](#sdk-path-programmatic-control)
3. [Plugin Path: Extending OpenCode](#plugin-path-extending-opencode)
4. [Configuration Deep-Dive](#configuration-deep-dive)
5. [Common Patterns](#common-patterns)
6. [Advanced Orchestration](#advanced-orchestration)
7. [Agent Skills](#agent-skills)
8. [Custom Commands](#custom-commands)
9. [Debugging & Troubleshooting](#debugging--troubleshooting)
10. [Best Practices](#best-practices)

---

## Quick Start: Choose Your Path

OpenCode offers two primary extension surfaces:

| Path | Use When | Key Packages |
|------|----------|-------------|
| **SDK** (`@opencode-ai/sdk`) | You want to control OpenCode from an external app, script, or service. Build integrations, automate workflows, or create UIs that interact with OpenCode. | `@opencode-ai/sdk` |
| **Plugin** (`@opencode-ai/plugin`) | You want to run code *inside* OpenCode, hook into its lifecycle, add custom AI tools, or modify behavior. | `@opencode-ai/plugin` |

**You can use both in the same project.** A plugin can internally use the SDK client to call back into OpenCode.

### Decision Table

| Scenario | Recommended Path |
|----------|-----------------|
| Build a CI/CD integration | SDK |
| Add a custom tool the AI can call | Plugin |
| Create a webhook listener that triggers sessions | SDK |
| Intercept and modify tool calls | Plugin |
| Build a dashboard showing session status | SDK |
| Protect sensitive files from being read | Plugin |
| Automate multi-agent workflows | SDK + Plugin |
| Add notifications on session completion | Plugin |
| Create an external UI for OpenCode | SDK |
| Inject environment variables into all shell commands | Plugin |

---

## SDK Path: Programmatic Control

### Installation

```bash
npm install @opencode-ai/sdk
# or
bun add @opencode-ai/sdk
```

### Two Client Modes

**Mode 1: Full Lifecycle** — starts a local server + client

```typescript
import { createOpencode } from "@opencode-ai/sdk"

const { client, server } = await createOpencode({
  hostname: "127.0.0.1",
  port: 4096,
  timeout: 5000,          // ms to wait for server start
  config: {               // overrides / merges with opencode.json
    model: "anthropic/claude-sonnet-4-5",
  },
})

// Use client...
await server.close()
```

**Mode 2: Client Only** — connects to an already-running OpenCode server

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({
  baseUrl: "http://localhost:4096",
  throwOnError: false,     // return error objects instead of throwing
  responseStyle: "fields", // or "data"
})
```

### Response Styles

- `"fields"` (default) — Each field is a separate result property
- `"data"` — Nested data object

### Error Handling

```typescript
// With throwOnError: false (default)
const result = await client.session.create({ body: {} })
if (result.error) {
  console.error("Failed:", result.error)
}

// With throwOnError: true
const session = await client.session.create({
  body: {},
  throwOnError: true, // throws on error
})
```

### Core Workflows

#### 1. Session Management

Sessions are the primary unit of interaction in OpenCode.

```typescript
// Create a session
const session = await client.session.create({
  body: { title: "My automation task" },
})

// Initialize (analyzes project, creates AGENTS.md context)
await client.session.init({ path: { id: session.id } })

// Send a prompt and get AI response
const result = await client.session.prompt({
  path: { id: session.id },
  body: {
    parts: [{ type: "text", text: "Refactor the auth module" }],
  },
})
// result.data.info — assistant message metadata
// result.data.parts — message parts (text, tool calls, etc.)

// Inject context WITHOUT triggering a response
await client.session.prompt({
  path: { id: session.id },
  body: {
    noReply: true,
    parts: [{ type: "text", text: "You are working in a Next.js project." }],
  },
})

// Run a shell command through the session
const shellResult = await client.session.shell({
  path: { id: session.id },
  body: { command: "npm test" },
})

// List all messages
const messages = await client.session.messages({ path: { id: session.id } })

// Get session children (subagent sessions)
const children = await client.session.children({ path: { id: session.id } })

// Get todo list
const todos = await client.session.todo({ path: { id: session.id } })

// Abort a running session
await client.session.abort({ path: { id: session.id } })

// Fork a session
const forked = await client.session.fork({
  path: { id: session.id },
  body: { messageID: "optional-msg-id" },
})

// Share / unshare
await client.session.share({ path: { id: session.id } })
await client.session.unshare({ path: { id: session.id } })

// Get diff
const diff = await client.session.diff({ path: { id: session.id } })

// Summarize session
await client.session.summarize({
  path: { id: session.id },
  body: { providerID: "anthropic", modelID: "claude-sonnet-4-5" },
})

// Revert a message
await client.session.revert({
  path: { id: session.id },
  body: { messageID: "msg-123" },
})

// Delete session
await client.session.delete({ path: { id: session.id } })
```

#### 2. Structured Output

Request validated JSON from the model using JSON Schema:

```typescript
const result = await client.session.prompt({
  path: { id: sessionId },
  body: {
    parts: [{ type: "text", text: "Analyze this codebase" }],
    format: {
      type: "json_schema",
      schema: {
        type: "object",
        properties: {
          summary: { type: "string" },
          techStack: { type: "array", items: { type: "string" } },
          entryPoints: { type: "array", items: { type: "string" } },
        },
        required: ["summary", "techStack"],
      },
      retryCount: 2,
    },
  },
})

console.log(result.data.info.structured_output)
// { summary: "...", techStack: ["React", "TypeScript"], entryPoints: [...] }

if (result.data.info.error?.name === "StructuredOutputError") {
  console.error("Failed after", result.data.info.error.retries, "attempts")
}
```

**Best practices:**
- Provide clear `description` fields on schema properties
- Keep schemas focused; deeply nested schemas are harder to fill
- Increase `retryCount` for complex schemas
- Always handle `StructuredOutputError` gracefully

#### 3. File Operations

```typescript
// Read a file
const file = await client.file.read({ query: { path: "src/index.ts" } })

// Search for text across files
const matches = await client.find.text({
  query: { pattern: "function.*handler" },
})

// Find files by name
const files = await client.find.files({
  query: { query: "*.ts", type: "file", limit: 50 },
})

// Find directories
const dirs = await client.find.files({
  query: { query: "components", type: "directory" },
})

// Find workspace symbols
const symbols = await client.find.symbols({
  query: { query: "UserService" },
})

// Get git status for tracked files
const status = await client.file.status()
```

#### 4. TUI Control

```typescript
await client.tui.appendPrompt({ body: { text: "npm install lodash" } })
await client.tui.submitPrompt()
await client.tui.clearPrompt()
await client.tui.showToast({
  body: { message: "Done!", variant: "success" },
})
await client.tui.openHelp()
await client.tui.openSessions()
await client.tui.openModels()
await client.tui.openThemes()
await client.tui.executeCommand({ body: { command: "/clear" } })
```

#### 5. Event Streaming

```typescript
const events = await client.event.subscribe()
for await (const event of events.stream) {
  console.log(event.type, event.properties)
  // Types: "session.updated", "message.part.updated", "tool.execute.after", etc.
}
```

#### 6. Authentication

```typescript
await client.auth.set({
  path: { id: "anthropic" },
  body: { type: "api", key: "sk-ant-api03-..." },
})
```

#### 7. Logging

```typescript
await client.app.log({
  body: {
    service: "my-integration",
    level: "info",
    message: "Deployment started",
    extra: { env: "production", version: "1.2.3" },
  },
})
```

### SDK Type Safety

```typescript
import type { Session, Message, Part, Config, Project } from "@opencode-ai/sdk"
```

---

## Plugin Path: Extending OpenCode

### Installation

```bash
npm install @opencode-ai/plugin
# or
bun add @opencode-ai/plugin
```

### Plugin Basics

A plugin is a JS/TS module that exports a function receiving a context object and returning hooks:

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async ({ project, client, $, directory, worktree }) => {
  console.log("Plugin loaded for project:", project?.name)

  return {
    // Hook implementations go here
  }
}
```

**Context object properties:**

| Property | Type | Description |
|----------|------|-------------|
| `project` | `Project \| null` | Current project metadata |
| `directory` | `string` | Current working directory |
| `worktree` | `string` | Git worktree path |
| `client` | `OpencodeClient` | SDK client for calling back into OpenCode |
| `$` | `BunShell` | Bun's shell API for executing commands |

### Loading Plugins

**1. Local files** (auto-loaded at startup)
- `.opencode/plugins/` — project-level
- `~/.config/opencode/plugins/` — global

**2. npm packages** (installed automatically via Bun)
```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["opencode-helicone-session", "@my-org/custom-plugin"]
}
```

**Load order:** Global config → Project config → Global plugin dir → Project plugin dir

**Dependencies for local plugins:** Add a `package.json` to `.opencode/`:
```json
{
  "dependencies": {
    "shescape": "^2.1.0"
  }
}
```

### All Plugin Hooks

#### Tool Hooks

```typescript
return {
  // BEFORE any tool executes — modify args or block actions
  "tool.execute.before": async (input, output) => {
    if (input.tool === "read" && output.args.filePath.includes(".env")) {
      throw new Error("Blocked: do not read .env files")
    }
    if (input.tool === "bash") {
      const { escape } = await import("shescape")
      output.args.command = escape(output.args.command)
    }
  },

  // AFTER tool execution — inspect or modify results
  "tool.execute.after": async (input, output) => {
    console.log(`Tool ${input.tool} finished:`, output.result)
  },
}
```

#### Shell Hooks

```typescript
return {
  "shell.env": async (input, output) => {
    output.env.MY_API_KEY = process.env.MY_API_KEY
    output.env.PROJECT_ROOT = input.cwd
  },
}
```

#### Session Hooks

```typescript
return {
  "session.created": async ({ event }) => {
    console.log("New session:", event.properties.sessionId)
  },
  "session.updated": async ({ event }) => {},
  "session.idle": async ({ event }) => {
    // AI finished responding — great for notifications
  },
  "session.error": async ({ event }) => {
    console.error("Session error:", event.properties.error)
  },
  "session.compacted": async ({ event }) => {},
  "session.deleted": async ({ event }) => {},
  "session.diff": async ({ event }) => {},
  "session.status": async ({ event }) => {},
}
```

#### Message Hooks

```typescript
return {
  "message.updated": async ({ event }) => {},
  "message.removed": async ({ event }) => {},
  "message.part.updated": async ({ event }) => {},
  "message.part.removed": async ({ event }) => {},
}
```

#### TUI Hooks

```typescript
return {
  "tui.prompt.append": async ({ event }) => {},
  "tui.command.execute": async ({ event }) => {},
  "tui.toast.show": async ({ event }) => {},
}
```

#### File Hooks

```typescript
return {
  "file.edited": async ({ event }) => {},
  "file.watcher.updated": async ({ event }) => {},
}
```

#### Permission Hooks

```typescript
return {
  "permission.asked": async ({ event }) => {},
  "permission.replied": async ({ event }) => {},
}
```

#### LSP Hooks

```typescript
return {
  "lsp.client.diagnostics": async ({ event }) => {},
  "lsp.updated": async ({ event }) => {},
}
```

#### Other Hooks

```typescript
return {
  "command.executed": async ({ event }) => {},
  "installation.updated": async ({ event }) => {},
  "todo.updated": async ({ event }) => {},
  "server.connected": async ({ event }) => {},
}
```

#### Compaction Hooks (Experimental)

```typescript
return {
  "experimental.session.compacting": async (input, output) => {
    // Inject additional context
    output.context.push(`
## Project State
- Currently working on: auth refactor
- Important decisions: using JWT, not sessions
- Active files: src/auth.ts, src/middleware.ts
`)

    // OR replace the entire compaction prompt:
    output.prompt = `
You are generating a continuation prompt for a multi-agent session.
Summarize:
1. Current task and status
2. Files being modified
3. Blockers or dependencies
4. Next steps
`
  },
}
```

### Custom Tools via Plugins

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"

export const CustomToolsPlugin: Plugin = async (ctx) => {
  return {
    tool: {
      deploy_to_vercel: tool({
        description: "Deploy the current project to Vercel",
        args: {
          preview: tool.schema.boolean().optional(),
          environment: tool.schema.enum(["production", "staging"]).optional(),
        },
        async execute(args, context) {
          const { directory, worktree } = context
          const { $ } = ctx
          const result = await $`cd ${directory} && vercel ${args.preview ? "--preview" : ""}`
          return `Deployed! Output: ${result.stdout}`
        },
      }),
    },
  }
}
```

**Tool schema helpers:**
- `tool.schema.string()`, `.number()`, `.boolean()`, `.enum([...])`, `.array(itemSchema)`
- `.optional()` — make any field optional
- `.describe("help text")` — add field descriptions

**Tool precedence:** Plugin tools override built-in tools with the same name.

### Standalone Custom Tools (`.opencode/tools/`)

You can also define custom tools as standalone TypeScript/JS files in `.opencode/tools/`. These are loaded automatically without needing a plugin wrapper:

```
.opencode/tools/
  my-tool.ts
  another-tool.js
```

```typescript
// .opencode/tools/my-tool.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "A standalone custom tool",
  args: {
    query: tool.schema.string().describe("Input query"),
  },
  async execute(args, context) {
    return `Result for: ${args.query}`
  },
})
```

Standalone tools are simpler for single-function tools where you don't need lifecycle hooks or plugin context. For more complex tools that need access to the plugin context (`$`, `client`, `directory`), use plugin-defined tools instead.

### Event Hook Pattern

Generic event listener for any event type:

```typescript
return {
  event: async ({ event }) => {
    if (event.type === "session.idle") {
      await ctx.client.app.log({
        body: { service: "my-plugin", level: "info", message: "Session completed" },
      })
    }
  },
}
```

### Logging from Plugins

Always use `client.app.log()` instead of `console.log`:

```typescript
await ctx.client.app.log({
  body: {
    service: "my-plugin",
    level: "info",
    message: "Plugin initialized",
    extra: { version: "1.0.0" },
  },
})
```

Levels: `debug`, `info`, `warn`, `error`

---

## Configuration Deep-Dive

### opencode.json Schema

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5",
  "provider": {
    "anthropic": {
      "options": {
        "timeout": 600000,
        "chunkTimeout": 30000
      }
    }
  },
  "plugin": ["opencode-helicone-session"],
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    }
  },
  "tools": {
    "write": false,
    "bash": false
  },
  "permission": {
    "*": "allow",
    "bash": "ask"
  },
  "agent": {
    "code-reviewer": {
      "description": "Reviews code for best practices",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-5",
      "permission": { "edit": "deny" }
    }
  },
  "default_agent": "build",
  "share": "manual",
  "formatter": true,
  "lsp": true,
  "snapshot": true,
  "autoupdate": true,
  "instructions": ["CONTRIBUTING.md", "docs/guidelines.md"],
  "disabled_providers": [],
  "enabled_providers": [],
  "experimental": {},
  "attachment": {
    "image": {
      "auto_resize": true,
      "max_width": 2000,
      "max_height": 2000,
      "max_base64_bytes": 5242880
    }
  },
  "shell": "zsh",
  "server": {
    "port": 4096,
    "hostname": "0.0.0.0",
    "mdns": true,
    "cors": ["http://localhost:5173"]
  },
  "watcher": {
    "ignore": ["node_modules/**", "dist/**", ".git/**"]
  },
  "compaction": {
    "auto": true,
    "prune": true,
    "reserved": 10000
  }
}
```

### Config Precedence (8 tiers, later wins)

1. Remote config (`.well-known/opencode`)
2. Global config (`~/.config/opencode/opencode.json`)
3. Custom config (`OPENCODE_CONFIG` env var)
4. Project config (`opencode.json` in project root)
5. `.opencode` directories
6. Inline config (`OPENCODE_CONFIG_CONTENT` env var)
7. Managed config files (`/Library/Application Support/opencode/` etc.)
8. macOS managed preferences (`.mobileconfig` via MDM)

Configs are **merged**, not replaced. Non-conflicting settings from all configs are preserved.

### Variable Substitution

```json
{
  "model": "{env:OPENCODE_MODEL}",
  "provider": {
    "openai": {
      "options": {
        "apiKey": "{file:~/.secrets/openai-key}"
      }
    }
  }
}
```

- `{env:VARIABLE_NAME}` — environment variable
- `{file:path/to/file}` — file contents (relative to config or absolute)

### tui.json

```json
{
  "$schema": "https://opencode.ai/tui.json",
  "theme": "tokyonight",
  "leader_timeout": 2000,
  "keybinds": {
    "leader": "ctrl+x",
    "command_list": "ctrl+p"
  },
  "scroll_speed": 3,
  "scroll_acceleration": { "enabled": true },
  "diff_style": "auto",
  "mouse": true,
  "attention": {
    "enabled": true,
    "notifications": true,
    "sound": true,
    "volume": 0.4
  }
}
```

---

## Common Patterns

### Pattern 1: Background Service Plugin

```typescript
import { Plugin } from "@opencode-ai/plugin"
import { spawn } from "node:child_process"

const PORT = 18921
const PROXY = `http://localhost:${PORT}`

async function startProxy(): Promise<void> {
  try {
    const res = await fetch(`${PROXY}/health`, { signal: AbortSignal.timeout(1000) })
    if (res.ok) return
  } catch { /* not running */ }

  const child = spawn("node", ["proxy.js", String(PORT)], {
    stdio: "ignore",
    detached: true,
  })
  child.unref()

  for (let i = 0; i < 10; i++) {
    try {
      const res = await fetch(`${PROXY}/health`, { signal: AbortSignal.timeout(1000) })
      if (res.ok) return
    } catch {}
    await new Promise(r => setTimeout(r, 500))
  }
}

export const ServicePlugin: Plugin = async (ctx) => {
  await startProxy()
  return {
    "shell.env": async (_input, output) => {
      output.env.MY_PROXY_URL = PROXY
    },
  }
}
```

### Pattern 2: Notification on Session Completion

```typescript
import { Plugin } from "@opencode-ai/plugin"

export const NotificationPlugin: Plugin = async ({ client }) => {
  return {
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        await client.app.log({
          body: { service: "notify", level: "info", message: "Session completed" },
        })
      }
    },
  }
}
```

### Pattern 3: File Protection Plugin

```typescript
import { Plugin } from "@opencode-ai/plugin"

const BLOCKED_PATTERNS = [/\.env/, /\.ssh/, /secret/i, /token/i]

export const FileProtectionPlugin: Plugin = async () => {
  return {
    "tool.execute.before": async (input, output) => {
      if (input.tool !== "read") return
      const path = output.args.filePath as string
      if (BLOCKED_PATTERNS.some(p => p.test(path))) {
        throw new Error(`Blocked: ${path} is on the sensitive files list`)
      }
    },
  }
}
```

### Pattern 4: MCP Server Integration

```json
{
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    },
    "github": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

MCP servers run as separate processes (stdio). Use MCP for wrapping CLIs, SaaS APIs, community tools. Use plugins for lifecycle hooks, custom logic, intercepting behavior.

### Pattern 5: Custom Model Provider

```json
{
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama Local",
      "options": { "baseURL": "http://localhost:11434/v1" },
      "models": {
        "llama3.2": { "name": "Llama 3.2" }
      }
    }
  },
  "model": "ollama/llama3.2"
}
```

### Pattern 6: SDK + Plugin Combo

```typescript
import { Plugin } from "@opencode-ai/plugin"

export const OrchestratorPlugin: Plugin = async ({ client }) => {
  return {
    "session.created": async ({ event }) => {
      const sessionId = event.properties.sessionId
      await client.session.init({ path: { id: sessionId } })
      await client.session.prompt({
        path: { id: sessionId },
        body: {
          noReply: true,
          parts: [{ type: "text", text: "This is a Next.js 14 App Router project." }],
        },
      })
    },
  }
}
```

### Pattern 7: Usage Tracking

```typescript
import { Plugin } from "@opencode-ai/plugin"
import { readFile, writeFile } from "node:fs/promises"

const USAGE_FILE = "/tmp/opencode-usage.json"

export const UsagePlugin: Plugin = async () => {
  return {
    "tool.execute.after": async (input, output) => {
      const data = JSON.parse(await readFile(USAGE_FILE, "utf-8").catch(() => '{"tools":{}}'))
      data.tools[input.tool] = (data.tools[input.tool] ?? 0) + 1
      await writeFile(USAGE_FILE, JSON.stringify(data))
    },
  }
}
```

### Pattern 8: Environment Variable Injection

```typescript
import { Plugin } from "@opencode-ai/plugin"

export const InjectEnvPlugin: Plugin = async () => {
  return {
    "shell.env": async (input, output) => {
      output.env.MY_API_KEY = "secret"
      output.env.PROJECT_ROOT = input.cwd
    },
  }
}
```

---

## Advanced Orchestration

When building large-scale systems on top of OpenCode — multi-agent workflows, automated refactoring pipelines, deep research systems, or CI/CD integrations — you need patterns for orchestrating many OpenCode sessions as worker agents.

### Key Concepts

| Concept | What It Means on OpenCode |
|---------|--------------------------|
| **Session-as-Agent** | Each `session.create()` spawns an isolated agent with its own context, tools, and message history |
| **External Orchestrator** | A standalone Node.js/Bun app using `@opencode-ai/sdk` to spawn and coordinate sessions |
| **Plugin Orchestrator** | An OpenCode plugin that hooks into session lifecycle and renders TUI dashboards |
| **State Management** | SQLite (`bun:sqlite`) or file-based checkpoints — sessions are ephemeral |
| **Parallel Execution** | `Promise.all` with concurrency limits (3–5 sessions recommended) |
| **Checkpoint/Resume** | Save step, results, and metadata after each phase; resume by reconciling completed agents |
| **Adversarial Review** | Spawn multiple agents from different angles, synthesizer merges findings |
| **Quality Gates** | Validate agent outputs with tests (`session.shell()`) before accepting |

### Quick Example: Parallel Code Review

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })

async function parallelReview(files: string[]): Promise<string[]> {
  const reviews = await Promise.all(
    files.map(async (file) => {
      const session = await client.session.create({ body: { title: `review-${file}` } })
      await client.session.init({ path: { id: session.id } })
      const result = await client.session.prompt({
        path: { id: session.id },
        body: {
          parts: [{ type: "text", text: `Review ${file} for bugs and style issues.` }],
        },
      })
      const text = result.data.parts
        ?.filter((p: any) => p.type === "text")
        .map((p: any) => p.text)
        .join("\n") || ""
      await client.session.delete({ path: { id: session.id } })
      return text
    })
  )
  return reviews
}
```

### When to Use Orchestration

| Scenario | Approach |
|----------|----------|
| Security audit across 200+ files | Parallel review agents + synthesizer |
| Framework migration (React 17 → 18) | Parallel migration agents + test validation gates |
| Deep research with cross-checking | Parallel researchers + adversarial verification + synthesis |
| CI/CD pipeline integration | External orchestrator + webhook triggers |
| Real-time TUI dashboard | Plugin with `session.idle` hook + SQLite state |

---

## Agent Skills

Skills are reusable instruction sets that agents load on-demand. Create a `SKILL.md` file:

```
.opencode/skills/my-skill/SKILL.md
```

```markdown
---
name: my-skill
description: Does something useful for the user
---

## What I do
- Step 1
- Step 2

## When to use me
Use this when the user asks about X.
```

Skills are discovered from:
- `.opencode/skills/<name>/SKILL.md` (project)
- `~/.config/opencode/skills/<name>/SKILL.md` (global)
- `.claude/skills/<name>/SKILL.md` (Claude-compatible)
- `.agents/skills/<name>/SKILL.md` (agent-compatible)

---

## Custom Commands

Create slash commands via markdown files:

```
.opencode/commands/test.md
```

```markdown
---
description: Run tests with coverage
agent: build
model: anthropic/claude-haiku-4-5
---

Run the full test suite with coverage report and show any failures.
Focus on the failing tests and suggest fixes.
```

Placeholders: `$ARGUMENTS`, `$1`, `$2`, `` !`command` ``, `@filename`

---

## Debugging & Troubleshooting

### Plugin Not Loading

1. Check file location: `.opencode/plugins/` or `~/.config/opencode/plugins/`
2. Check for syntax errors — OpenCode silently skips broken plugins
3. Verify exports: `export const MyPlugin = async (ctx) => ({ ... })`
4. Use `client.app.log()` for debugging
5. For npm plugins: ensure listed in `opencode.json`, restart OpenCode

### SDK Connection Issues

1. Verify server is running on expected port (default: 4096)
2. Check `baseUrl` matches actual server URL
3. Use `client.global.health()` to verify connectivity
4. Increase `timeout` if server takes long to start

### TypeScript Errors

1. Ensure `@opencode-ai/plugin` is installed
2. Import types: `import type { Plugin } from "@opencode-ai/plugin"`
3. Use `moduleResolution: "bundler"` in tsconfig

### Tool Not Appearing

1. Plugin must return `tool: { myTool: tool({ ... }) }`
2. Tool names must be unique (plugin tools override built-ins)
3. Restart OpenCode after plugin changes

### Hook Not Firing

1. Verify event name is exactly correct (case-sensitive)
2. Some hooks are experimental: `experimental.session.compacting`
3. Add a log in the plugin body to confirm loading

---

## Best Practices

1. **Use TypeScript** — both packages provide full type definitions
2. **Use `client.app.log()`** — never `console.log` in plugins; logs are structured and actionable
3. **Handle errors gracefully** — plugins that throw can break OpenCode workflows
4. **Keep plugin initialization fast** — OpenCode waits for all plugins to load at startup
5. **Use `noReply: true`** for context injection — don't trigger unnecessary AI responses
6. **Validate tool args** — the `tool.schema.*` helpers do runtime validation
7. **Be careful with `tool.execute.before`** — throwing errors blocks the tool entirely
8. **Use the SDK inside plugins** — the `client` in plugin context is the same SDK client
9. **Respect user config** — read from `opencode.json`, don't hardcode credentials
10. **Document your plugin** — include a README with setup instructions and `opencode.json` example
11. **Use config merging** — don't overwrite entire config objects, let OpenCode merge
12. **Test plugins in isolation** — use a fresh project directory for testing
13. **Use meaningful plugin names** — npm package names should be `opencode-<name>`
14. **Clean up resources** — close file handles, child processes, and connections in plugin cleanup
