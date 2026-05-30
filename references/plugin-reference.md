# Plugin Reference

Exhaustive reference for `@opencode-ai/plugin` hooks, events, context, and tool API.

## Table of Contents
- [Plugin Function Signature](#plugin-function-signature)
- [Context Object](#context-object)
- [Return Value / Hooks](#return-value--hooks)
- [Event Types Reference](#event-types-reference)
- [Tool API](#tool-api)
- [Hook Input/Output Schemas](#hook-inputoutput-schemas)
- [TypeScript Types](#typescript-types)

---

## Plugin Function Signature

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => {
  // Initialization code (runs once at plugin load)
  return {
    // Hooks object
  }
}
```

The plugin function is called once at OpenCode startup. It receives the context object and must return a hooks object (or a Promise resolving to one).

---

## Context Object

The `ctx` object passed to every plugin:

```typescript
interface PluginContext {
  /** Current project metadata (null if not in a project) */
  project: Project | null

  /** Current working directory absolute path */
  directory: string

  /** Git worktree root path */
  worktree: string

  /** SDK client for calling back into OpenCode */
  client: OpencodeClient

  /** Bun shell API for executing commands */
  $: BunShell
}
```

### `project`
Contains metadata about the current workspace project:
- `name` — project name
- `path` — project root directory
- Additional fields depending on project type

### `client`
The same SDK client documented in `sdk-api.md`. Use it from within plugins to:
- Read files (`client.file.read()`)
- Create sessions (`client.session.create()`)
- Inject context (`client.session.prompt({ noReply: true, ... })`)
- Write logs (`client.app.log()`)

### `$`
Bun's shell API. Execute commands, pipe output, capture results:

```typescript
const result = await $`cd ${ctx.directory} && git status`
console.log(result.stdout)
console.log(result.stderr)
console.log(result.exitCode)
```

---

## Return Value / Hooks

The returned object maps hook names to handler functions. Multiple plugins can register the same hook — they run in load order.

### Command Hooks

#### `command.executed`
Fires when a slash command is executed.

```typescript
"command.executed": async ({ event }) => {
  console.log("Command:", event.properties.command)
}
```

### File Hooks

#### `file.edited`
Fires when a file is edited by OpenCode.

```typescript
"file.edited": async ({ event }) => {
  console.log("File edited:", event.properties.path)
}
```

#### `file.watcher.updated`
Fires when the file watcher detects a change (external or internal).

```typescript
"file.watcher.updated": async ({ event }) => {
  // Trigger re-analysis, invalidate caches, etc.
}
```

### Installation Hooks

#### `installation.updated`
Fires when OpenCode installs or updates itself.

### LSP Hooks

#### `lsp.client.diagnostics`
Fires when LSP diagnostics are received.

```typescript
"lsp.client.diagnostics": async ({ event }) => {
  const diagnostics = event.properties.diagnostics
  const hasErrors = diagnostics.some(d => d.severity === "error")
  if (hasErrors) {
    await ctx.client.tui.showToast({
      body: { message: "Build errors detected", variant: "error" },
    })
  }
}
```

#### `lsp.updated`
Fires when LSP server status changes.

### Message Hooks

#### `message.updated`
Fires when a message is modified (content changed, status update).

```typescript
"message.updated": async ({ event }) => {
  console.log("Message updated:", event.properties.messageId)
}
```

#### `message.part.updated`
Fires during streaming — each token update triggers this.

```typescript
"message.part.updated": async ({ event }) => {
  // React to real-time streaming updates
}
```

#### `message.part.removed`
Fires when a message part is removed (e.g., user deletes a tool call).

#### `message.removed`
Fires when an entire message is removed.

### Permission Hooks

#### `permission.asked`
Fires when OpenCode needs user permission for an action.

```typescript
"permission.asked": async ({ event }) => {
  console.log("Permission requested for:", event.properties.action)
}
```

#### `permission.replied`
Fires when the user responds to a permission request.

### Server Hooks

#### `server.connected`
Fires when the OpenCode server finishes starting up.

```typescript
"server.connected": async ({ event }) => {
  console.log("Server ready at:", event.properties.url)
}
```

### Session Hooks

#### `session.created`
New session created.

```typescript
"session.created": async ({ event }) => {
  const sessionId = event.properties.sessionId
  // Initialize session with project context
  await ctx.client.session.init({ path: { id: sessionId } })
}
```

#### `session.updated`
Session metadata changed (title, status, etc.).

#### `session.deleted`
Session deleted.

#### `session.idle`
AI finished responding — session is now idle.

```typescript
"session.idle": async ({ event }) => {
  // Send notification, trigger CI, log completion
  await ctx.client.app.log({
    body: { service: "my-plugin", level: "info", message: "Session completed" },
  })
}
```

#### `session.error`
Error occurred in a session.

```typescript
"session.error": async ({ event }) => {
  console.error("Session error:", event.properties.error)
}
```

#### `session.status`
Session status changed.

#### `session.diff`
Session diff/comparison event.

#### `session.compacted`
Context was summarized/compacted.

#### `experimental.session.compacting`
**Fires BEFORE compaction.** Customize what context survives summarization.

```typescript
"experimental.session.compacting": async (input, output) => {
  // Option 1: Inject additional context
  output.context.push(`
## Critical State
- Current task: refactoring auth
- Decisions made: using OAuth2, not SAML
- Blocked on: waiting for API key
`)

  // Option 2: Replace the entire compaction prompt
  output.prompt = `
You are generating a continuation prompt.
Summarize the current task, files modified, blockers, and next steps.
Format as a structured prompt a new agent could use to resume.
`
}
```

When `output.prompt` is set, `output.context` is **ignored**.

### Shell Hooks

#### `shell.env`
Inject environment variables into ALL shell execution (both AI tools and user terminals).

```typescript
"shell.env": async (input, output) => {
  output.env.MY_API_KEY = process.env.MY_API_KEY
  output.env.PROJECT_ROOT = input.cwd
  output.env.PATH = `/opt/homebrew/bin:${process.env.PATH}`
}
```

### Tool Hooks

#### `tool.execute.before`
Runs BEFORE any tool executes. Use to modify arguments, inject data, or block actions.

```typescript
"tool.execute.before": async (input, output) => {
  // input.tool — the tool name (e.g., "read", "bash", "edit")
  // input.sessionId — current session ID
  // output.args — mutable arguments object

  if (input.tool === "read") {
    const path = output.args.filePath as string
    if (path.includes(".env")) {
      throw new Error("Blocked: cannot read .env files")
    }
  }

  if (input.tool === "bash") {
    // Prepend cd to ensure commands run in correct directory
    const cmd = output.args.command as string
    if (!cmd.startsWith("cd ")) {
      output.args.command = `cd "${input.cwd}" && ${cmd}`
    }
  }
}
```

#### `tool.execute.after`
Runs AFTER tool execution. Inspect or modify results.

```typescript
"tool.execute.after": async (input, output) => {
  // input.tool — tool name
  // input.sessionId — session ID
  // output.result — the tool's return value

  if (input.tool === "bash") {
    console.log("Command output:", output.result)
  }
}
```

### TUI Hooks

#### `tui.prompt.append`
User added text to the prompt input.

```typescript
"tui.prompt.append": async ({ event }) => {
  const text = event.properties.text
  if (text.includes("password")) {
    await ctx.client.tui.showToast({
      body: { message: "⚠️ Avoid sharing passwords in prompts", variant: "warning" },
    })
  }
}
```

#### `tui.command.execute`
User executed a TUI command (not slash command).

#### `tui.toast.show`
Toast notification shown.

### Todo Hooks

#### `todo.updated`
Task list updated.

### Generic Event Hook

#### `event`
Catch-all for any event type. Receives ALL events.

```typescript
event: async ({ event }) => {
  switch (event.type) {
    case "session.idle":
      // Handle completion
      break
    case "session.error":
      // Handle error
      break
    case "tool.execute.before":
      // Handle tool execution
      break
  }
}
```

---

## Event Types Reference

> **Note:** Some property shapes in this table are inferred from the plugin documentation context rather than explicitly documented in the OpenCode API. Core fields like `sessionId`, `messageId`, and `tool` are confirmed; secondary fields may vary by OpenCode version.

All event types and their `properties` payloads:

| Event Type | Properties |
|-----------|------------|
| `command.executed` | `command: string`, `args: string[]` |
| `file.edited` | `path: string`, `content: string` |
| `file.watcher.updated` | `path: string`, `changeType: string` |
| `installation.updated` | `version: string` |
| `lsp.client.diagnostics` | `diagnostics: Diagnostic[]`, `uri: string` |
| `lsp.updated` | `server: string`, `status: string` |
| `message.updated` | `messageId: string`, `sessionId: string` |
| `message.part.updated` | `messageId: string`, `partId: string` |
| `message.part.removed` | `messageId: string`, `partId: string` |
| `message.removed` | `messageId: string`, `sessionId: string` |
| `permission.asked` | `action: string`, `sessionId: string` |
| `permission.replied` | `action: string`, `granted: boolean` |
| `server.connected` | `url: string`, `version: string` |
| `session.created` | `sessionId: string`, `title: string` |
| `session.updated` | `sessionId: string`, `changes: object` |
| `session.deleted` | `sessionId: string` |
| `session.idle` | `sessionId: string` |
| `session.error` | `sessionId: string`, `error: string` |
| `session.status` | `sessionId: string`, `status: string` |
| `session.diff` | `sessionId: string`, `diff: string` |
| `session.compacted` | `sessionId: string` |
| `experimental.session.compacting` | `sessionId: string` |
| `shell.env` | `cwd: string` |
| `tool.execute.before` | `tool: string`, `sessionId: string` |
| `tool.execute.after` | `tool: string`, `sessionId: string` |
| `tui.prompt.append` | `text: string` |
| `tui.command.execute` | `command: string` |
| `tui.toast.show` | `message: string`, `variant: string` |
| `todo.updated` | `todos: Todo[]` |

---

## Tool API

### `tool({ description, args, execute })`

Create a custom AI-accessible tool.

```typescript
import { tool } from "@opencode-ai/plugin"

tool({
  description: string           // What the tool does — the model reads this
  args: Record<string, Schema>  // Argument schema (see below)
  execute: (args, context) => Promise<any> | any
})
```

**Argument schema builders:**

```typescript
import { tool } from "@opencode-ai/plugin"

tool.schema.string()           // String argument
tool.schema.number()           // Number argument
tool.schema.boolean()          // Boolean argument
tool.schema.enum(["a", "b"])   // Enum argument
tool.schema.array(itemSchema)  // Array of items

// Modifiers (chainable):
.optional()                    // Make field optional
tool.schema.string().optional()

tool.schema.string().describe("The file path to read")
```

**Execute function signature:**

```typescript
async execute(args, context) {
  // args — validated and typed arguments
  // context — { directory: string, worktree: string }

  const { directory, worktree } = context
  // Return a string (or any JSON-serializable value)
  return `Result from ${directory}`
}
```

**Full example:**

```typescript
import { Plugin, tool } from "@opencode-ai/plugin"

export const GitToolsPlugin: Plugin = async () => {
  return {
    tool: {
      git_status: tool({
        description: "Get git status for the current repository",
        args: {
          short: tool.schema.boolean().optional().describe("Use short format"),
        },
        async execute(args, context) {
          const { $ } = await import("bun")
          const result = await $`cd ${context.directory} && git status ${args.short ? "-s" : ""}`
          return result.stdout || "No changes"
        },
      }),
    },
  }
}
```

---

## Hook Input/Output Schemas

### `tool.execute.before` / `tool.execute.after`

```typescript
input: {
  tool: string        // Tool name
  sessionId: string  // Current session ID
}

output: {
  args: Record<string, any>   // Mutable arguments (before)
  result: any                 // Tool result (after)
}
```

### `shell.env`

```typescript
input: {
  cwd: string  // Current working directory
}

output: {
  env: Record<string, string>  // Mutable environment variables
}
```

### `experimental.session.compacting`

```typescript
input: {
  sessionId: string
}

output: {
  context: string[]   // Additional context strings to inject
  prompt?: string      // If set, replaces ENTIRE compaction prompt
}
```

---

## TypeScript Types

```typescript
import type {
  Plugin,           // The plugin function type
  PluginContext,    // Context object type
  HookHandler,      // Generic hook handler type
  ToolDefinition,   // Custom tool type
} from "@opencode-ai/plugin"
```

**Plugin type:**
```typescript
type Plugin = (ctx: PluginContext) => Promise<Hooks> | Hooks
```

**Hooks object:**
```typescript
interface Hooks {
  // Shell / Tools
  "tool.execute.before"?: HookHandler
  "tool.execute.after"?: HookHandler
  "shell.env"?: HookHandler

  // Session
  "session.created"?: HookHandler
  "session.updated"?: HookHandler
  "session.deleted"?: HookHandler
  "session.idle"?: HookHandler
  "session.error"?: HookHandler
  "session.status"?: HookHandler
  "session.diff"?: HookHandler
  "session.compacted"?: HookHandler
  "experimental.session.compacting"?: HookHandler

  // Messages
  "message.updated"?: HookHandler
  "message.part.updated"?: HookHandler
  "message.part.removed"?: HookHandler
  "message.removed"?: HookHandler

  // TUI
  "tui.prompt.append"?: HookHandler
  "tui.command.execute"?: HookHandler
  "tui.toast.show"?: HookHandler

  // Commands
  "command.executed"?: HookHandler

  // Files
  "file.edited"?: HookHandler
  "file.watcher.updated"?: HookHandler

  // Permissions
  "permission.asked"?: HookHandler
  "permission.replied"?: HookHandler

  // LSP
  "lsp.client.diagnostics"?: HookHandler
  "lsp.updated"?: HookHandler

  // Server
  "server.connected"?: HookHandler

  // Installation
  "installation.updated"?: HookHandler

  // Todos
  "todo.updated"?: HookHandler

  // Generic
  "event"?: HookHandler

  // Custom tools
  "tool"?: Record<string, ToolDefinition>
}
```
