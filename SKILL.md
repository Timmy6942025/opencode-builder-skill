---
name: opencode-builder
description: |
  Use this skill when the user is building extensions, plugins, integrations, or custom developer tools on top of OpenCode (opencode.ai). This includes: creating plugins with custom AI tools and lifecycle hooks, using the SDK for programmatic session control and multi-agent workflows, building model providers or MCP integrations, configuring opencode.json, and implementing advanced orchestration like checkpoint/resume, parallel execution, or adversarial review. Trigger for any development work that extends or automates OpenCode. Do NOT trigger for general usage questions, troubleshooting, UI theming, or tool comparisons.
---

# OpenCode Builder

A comprehensive skill for building on top of [OpenCode](https://opencode.ai) — an AI-powered coding assistant. This skill covers both the **JavaScript/TypeScript SDK** for programmatic control and the **plugin architecture** for extending OpenCode's behavior.

## Table of Contents

1. [Quick Start: Choose Your Path](#quick-start-choose-your-path)
2. [SDK Path: Programmatic Control](#sdk-path-programmatic-control)
3. [Plugin Path: Extending OpenCode](#plugin-path-extending-opencode)
4. [Configuration Deep-Dive](#configuration-deep-dive)
5. [Common Patterns](#common-patterns)
6. [Advanced Orchestration](#advanced-orchestration)
7. [Debugging & Troubleshooting](#debugging--troubleshooting)
8. [Reference Files](#reference-files)

---

## Quick Start: Choose Your Path

OpenCode offers two primary extension surfaces:

| Path | Use When | Key Packages |
|------|----------|-------------|
| **SDK** (`@opencode-ai/sdk`) | You want to control OpenCode from an external app, script, or service. Build integrations, automate workflows, or create UIs that interact with OpenCode. | `@opencode-ai/sdk` |
| **Plugin** (`@opencode-ai/plugin`) | You want to run code *inside* OpenCode, hook into its lifecycle, add custom AI tools, or modify behavior. | `@opencode-ai/plugin` |

**You can use both in the same project.** A plugin can internally use the SDK client to call back into OpenCode.

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
    model: "anthropic/claude-3-5-sonnet-20241022",
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

### Core Workflows

#### 1. Session Management

Sessions are the primary unit of interaction in OpenCode. Create them, send prompts, and read messages.

```typescript
// Create a session
const session = await client.session.create({
  body: { title: "My automation task" },
})

// Initialize a session (analyzes project and creates AGENTS.md context)
await client.session.init({ path: { id: session.id } })

// Send a prompt and get AI response
const result = await client.session.prompt({
  path: { id: session.id },
  body: {
    parts: [{ type: "text", text: "Refactor the auth module" }],
  },
})
// result.data.info contains the assistant message metadata
// result.data.parts contains message parts (text, tool calls, etc.)

// Inject context WITHOUT triggering a response (great for setup/context)
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

// List all messages in a session
const messages = await client.session.messages({ path: { id: session.id } })
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
      retryCount: 2,  // retries if validation fails (default: 2)
    },
  },
})

console.log(result.data.info.structured_output)
// { summary: "...", techStack: ["React", "TypeScript"], entryPoints: [...] }

// Check for structured output errors
if (result.data.info.error?.name === "StructuredOutputError") {
  console.error("Failed after", result.data.info.error.retries, "attempts")
}
```

**Best practices for structured output:**
- Provide clear `description` fields on schema properties — the model uses these
- Keep schemas focused; deeply nested schemas are harder to fill correctly
- Increase `retryCount` for complex schemas, decrease for simple ones
- Always handle `StructuredOutputError` gracefully

#### 3. File Operations

```typescript
// Read a file
const file = await client.file.read({ query: { path: "src/index.ts" } })
// file.data.content has the raw text
// file.data.type is "raw" or "patch"

// Search for text across files
const matches = await client.find.text({
  query: { pattern: "function.*handler" },
})
// matches.data is an array of { path, lines, line_number, submatches }

// Find files by name
const files = await client.find.files({
  query: { query: "*.ts", type: "file", limit: 50 },
})

// Find directories
const dirs = await client.find.files({
  query: { query: "components", type: "directory" },
})

// Get git status for tracked files
const status = await client.file.status()
```

#### 4. TUI Control

Control the OpenCode terminal interface programmatically:

```typescript
await client.tui.appendPrompt({ body: { text: "npm install lodash" } })
await client.tui.submitPrompt()          // Press "enter"
await client.tui.showToast({
  body: { message: "Done!", variant: "success" },
})
await client.tui.openHelp()
await client.tui.openSessions()
await client.tui.openModels()
await client.tui.executeCommand({ body: { command: "/clear" } })
```

#### 5. Event Streaming

Subscribe to real-time events via Server-Sent Events:

```typescript
const events = await client.event.subscribe()
for await (const event of events.stream) {
  console.log(event.type, event.properties)
  // event.type examples: "session.updated", "message.part.updated", "tool.execute.after"
}
```

#### 6. Authentication

Set API keys for third-party providers that OpenCode will use:

```typescript
await client.auth.set({
  path: { id: "anthropic" },
  body: { type: "api", key: "sk-ant-api03-..." },
})
```

#### 7. Logging

Write structured log entries that appear in OpenCode's internal logs:

```typescript
await client.app.log({
  body: {
    service: "my-integration",
    level: "info",    // debug | info | warn | error
    message: "Deployment started",
    extra: { env: "production", version: "1.2.3" },
  },
})
```

### SDK Type Safety

All types are generated from OpenCode's OpenAPI spec:

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
  // Initialization code runs here
  console.log("Plugin loaded for project:", project?.name)

  return {
    // Hook implementations go here
  }
}
```

**Context object (`ctx`) properties:**

| Property | Type | Description |
|----------|------|-------------|
| `project` | `Project \| null` | Current project metadata |
| `directory` | `string` | Current working directory |
| `worktree` | `string` | Git worktree path |
| `client` | `OpencodeClient` | SDK client for calling back into OpenCode |
| `$` | `BunShell` | Bun's shell API for executing commands |

### Loading Plugins

Plugins load from two sources:

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
OpenCode runs `bun install` at startup.

### Custom Tools

Add AI-accessible tools that OpenCode can invoke:

```typescript
import { Plugin, tool } from "@opencode-ai/plugin"

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
          // args.preview, args.environment are typed and validated
          const { $ } = ctx
          const result = await $`cd ${directory} && vercel ${args.preview ? "--preview" : ""}`
          return `Deployed! Output: ${result.stdout}`
        },
      }),
    },
  }
}
```

**Tool precedence:** If a plugin tool shares a name with a built-in tool, the **plugin's version wins**.

**Tool schema helpers:**
- `tool.schema.string()`
- `tool.schema.number()`
- `tool.schema.boolean()`
- `tool.schema.enum([...])`
- `tool.schema.array(itemSchema)`
- `.optional()` — make any field optional
- `.describe("help text")` — add field descriptions

### Lifecycle Hooks

#### Shell / Tool Interception

```typescript
return {
  // Runs BEFORE any tool executes — modify args or block actions
  "tool.execute.before": async (input, output) => {
    if (input.tool === "read" && output.args.filePath.includes(".env")) {
      throw new Error("Blocked: do not read .env files")
    }
    // Modify arguments
    if (input.tool === "bash") {
      const { escape } = await import("shescape")
      output.args.command = escape(output.args.command)
    }
  },

  // Runs AFTER tool execution — inspect or modify results
  "tool.execute.after": async (input, output) => {
    console.log(`Tool ${input.tool} finished with:`, output.result)
  },

  // Inject environment variables into ALL shell execution
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
  "session.updated": async ({ event }) => {
    // Session metadata changed
  },
  "session.idle": async ({ event }) => {
    // AI finished responding — great time for notifications
  },
  "session.error": async ({ event }) => {
    console.error("Session error:", event.properties.error)
  },
}
```

#### Message Hooks

```typescript
return {
  "message.updated": async ({ event }) => {
    // A message was modified
  },
  "message.part.updated": async ({ event }) => {
    // Streaming update to a message part
  },
  "message.part.removed": async ({ event }) => {
    // A part was deleted (e.g., user removed a tool call)
  },
}
```

#### TUI Hooks

```typescript
return {
  "tui.prompt.append": async ({ event }) => {
    // User added text to the prompt
  },
  "tui.toast.show": async ({ event }) => {
    // A toast notification was shown
  },
  "tui.command.execute": async ({ event }) => {
    // User executed a slash command
  },
}
```

#### Compaction Hooks (Experimental)

Customize how session context is summarized when it gets too long:

```typescript
return {
  "experimental.session.compacting": async (input, output) => {
    // Inject additional context into the compaction prompt
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

When `output.prompt` is set, it **replaces** the default compaction prompt entirely.

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

Always use `client.app.log()` instead of `console.log` for structured, actionable logging:

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
  "model": "anthropic/claude-3-5-sonnet-20241022",
  "provider": {
    "anthropic": {
      "name": "Anthropic",
      "api": "anthropic",
      "url": "https://api.anthropic.com"
    },
    "openai": {
      "name": "OpenAI",
      "api": "openai",
      "url": "https://api.openai.com/v1"
    },
    "custom": {
      "name": "My Proxy",
      "api": "openai",
      "url": "http://localhost:18921/v1"
    }
  },
  "plugin": [
    "opencode-helicone-session",
    "file:///Users/me/my-plugin/src/index.ts"
  ],
  "mcp": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    }
  }
}
```

**Runtime note:** OpenCode runs on [Bun](https://bun.sh/), so plugins can use Bun-native APIs like `Bun.file()`, `Bun.write()`, `Bun.hash()`, and top-level `await` without bundling.

**Config locations (precedence: top to bottom):**
1. `~/.config/opencode/opencode.json` — global defaults
2. `./opencode.json` — project-specific overrides

**Key fields:**
- `model` — default model ID (format: `provider/model-id`)
- `provider` — custom API endpoints, useful for proxies or self-hosted models
- `plugin` — npm package names or `file://` paths to local plugins
- `mcp` — Model Context Protocol server configurations

### TypeScript Plugin tsconfig

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true
  }
}
```

---

## Common Patterns

### Pattern 1: Plugin That Starts a Background Service

A plugin that auto-starts a proxy server if not already running:

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

  // Wait for startup
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
        // macOS notification via osascript
        await client.app.log({
          body: { service: "notify", level: "info", message: "Session completed" },
        })
        // Or use osascript / notify-send / your preferred notifier
      }
    },
  }
}
```

### Pattern 3: File Protection Plugin

Block OpenCode from reading sensitive files:

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

### Pattern 4: Model Context Protocol (MCP)

Add external tools to OpenCode via MCP servers. Configure them in `opencode.json`:

```json
{
  "mcp": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

MCP servers provide tools that OpenCode can invoke alongside built-in and plugin tools.
Unlike plugins, MCP servers run as separate processes and communicate via stdio.
Use MCP for: wrapping existing CLIs, integrating with SaaS APIs, or reusing community tools.
Use plugins for: lifecycle hooks, custom logic, intercepting OpenCode behavior, or when you
need the full plugin context (`client`, `$`, `directory`, etc.).

### Pattern 5: Custom Model Provider

Add a new LLM provider by configuring it in `opencode.json`:

```json
{
  "provider": {
    "ollama": {
      "name": "Ollama Local",
      "api": "openai",
      "url": "http://localhost:11434/v1"
    }
  },
  "model": "ollama/llama3.2"
}
```

Or wrap an existing service with a proxy plugin that provides an OpenAI-compatible endpoint.

### Pattern 6: SDK + Plugin Combo

A plugin that uses the SDK client to create child sessions or read messages:

```typescript
import { Plugin } from "@opencode-ai/plugin"

export const OrchestratorPlugin: Plugin = async ({ client }) => {
  return {
    "session.created": async ({ event }) => {
      // Auto-initialize new sessions with project context
      const sessionId = event.properties.sessionId
      await client.session.init({ path: { id: sessionId } })

      // Inject project-specific context
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

### Pattern 7: Usage Tracking / Rate Limiting

Track API usage in a plugin:

```typescript
import { Plugin } from "@opencode-ai/plugin"
import { readFile, writeFile } from "node:fs/promises"

const USAGE_FILE = "/tmp/opencode-usage.json"
const DAILY_LIMIT = 100

export const RateLimitPlugin: Plugin = async () => {
  return {
    "tool.execute.before": async (input, output) => {
      if (input.tool !== "session.prompt") return
      // Read usage, check limit, optionally block or warn
    },
  }
}
```

---

## Debugging & Troubleshooting

### Plugin Not Loading

1. Check file location: must be in `.opencode/plugins/` (project) or `~/.config/opencode/plugins/` (global)
2. Check for syntax errors — OpenCode silently skips broken plugins
3. Verify exports: `export const MyPlugin = async (ctx) => ({ ... })`
4. Use `client.app.log()` for debugging — check OpenCode's internal logs
5. For npm plugins: ensure they're listed in `opencode.json` and restart OpenCode

### SDK Connection Issues

1. Verify OpenCode server is running on the expected port (default: 4096)
2. Check `baseUrl` matches the actual server URL
3. Use `client.global.health()` to verify connectivity
4. If using `createOpencode()`, increase `timeout` if the server takes long to start

### TypeScript Errors in Plugins

1. Ensure `@opencode-ai/plugin` is installed
2. Import types: `import type { Plugin } from "@opencode-ai/plugin"`
3. Use `moduleResolution: "bundler"` in tsconfig

### Tool Not Appearing

1. Plugin must return `tool: { myTool: tool({ ... }) }`
2. Tool names must be unique (plugin tools override built-ins with same name)
3. Restart OpenCode after plugin changes

### Hook Not Firing

1. Verify the event name is exactly correct (case-sensitive)
2. Some hooks are experimental: `experimental.session.compacting`
3. Check that the plugin is actually loaded (add a log in the plugin body)

---

## Advanced Orchestration

When building large-scale systems on top of OpenCode — multi-agent workflows, automated refactoring pipelines, deep research systems, or CI/CD integrations — you need patterns for orchestrating many OpenCode sessions as worker agents.

This section is a **high-level overview**. For the complete implementation guide, read:

→ **`references/advanced-orchestration.md`** — Multi-agent workflow architecture, state management, parallel execution, checkpoint/resume, adversarial review, workflow scripting DSL, monitoring, and a complete `OpenCodeOrchestrator` class.

### Key Concepts

| Concept | What It Means on OpenCode |
|---------|--------------------------|
| **Session-as-Agent** | Each `session.create()` spawns an isolated agent with its own context, tools, and message history |
| **External Orchestrator** | A standalone Node.js/Bun app using `@opencode-ai/sdk` to spawn and coordinate sessions |
| **Plugin Orchestrator** | An OpenCode plugin that hooks into session lifecycle and renders TUI dashboards |
| **State Management** | SQLite (`bun:sqlite`) or file-based checkpoints — sessions are ephemeral, so workflow state must persist externally |
| **Parallel Execution** | `Promise.all` with concurrency limits (3–5 sessions at a time recommended) |
| **Checkpoint/Resume** | Save `step`, `agentResults`, and `metadata` to disk after each phase; resume by reconciling which agents completed |
| **Adversarial Review** | Spawn multiple agents from different angles, then have a synthesizer merge findings or a critic refute solutions |
| **Quality Gates** | Validate agent outputs with tests (`session.shell()`) before accepting them |

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

## Reference Files

For detailed API references that are too large for the main skill file:

| File | When to Read |
|------|-------------|
| `references/sdk-api.md` | You need the complete SDK method listing with all parameters and response types |
| `references/plugin-reference.md` | You need the exhaustive list of all plugin hooks, events, and context properties |
| `references/examples.md` | You want to see real-world, complete plugin implementations with package.json, tsconfig, and README |
| `references/advanced-orchestration.md` | You are building multi-agent workflows, CI pipelines, automated audits, or any large-scale orchestration on top of OpenCode |

---

## Best Practices Summary

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
