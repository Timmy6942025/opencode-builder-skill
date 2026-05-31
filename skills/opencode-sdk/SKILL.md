---
name: opencode-sdk
description: |
  Use this skill when using the OpenCode JavaScript/TypeScript SDK (@opencode-ai/sdk) to programmatically control OpenCode, create sessions, send prompts, manage files, control the TUI, stream events, or build external integrations. Covers both full lifecycle (server+client) and client-only modes, all API methods, structured output, and type safety.
---

# OpenCode JavaScript/TypeScript SDK (`@opencode-ai/sdk`)

Type-safe JavaScript/TypeScript client for the OpenCode server. Use it to build integrations, automate workflows, and control OpenCode programmatically.

All types are auto-generated from the server's OpenAPI 3.1 specification and available in the [types file](https://github.com/anomalyco/opencode/blob/dev/packages/sdk/js/src/gen/types.gen.ts).

---

## Installation

```bash
npm install @opencode-ai/sdk
# or
bun add @opencode-ai/sdk
```

---

## Two Client Modes

### Full Lifecycle (`createOpencode`)

Starts an OpenCode server and returns a connected client. Use this when you own the server lifecycle.

```typescript
import { createOpencode } from "@opencode-ai/sdk"

const { client, server } = await createOpencode({
  hostname: "127.0.0.1",
  port: 4096,
  timeout: 5000,
  config: {
    model: "anthropic/claude-3-5-sonnet-20241022",
  },
})

console.log(`Server running at ${server.url}`)

// Use the client...

server.close()
```

**Options:**

| Option | Type | Description | Default |
|---|---|---|---|
| `hostname` | `string` | Server hostname | `127.0.0.1` |
| `port` | `number` | Server port | `4096` |
| `signal` | `AbortSignal` | Abort signal for cancellation | `undefined` |
| `timeout` | `number` | Timeout in ms for server start | `5000` |
| `config` | `Config` | Configuration object | `{}` |

The instance still picks up your `opencode.json`, but inline config overrides or adds to it. The returned `server` object has a `.url` property and a `.close()` method to shut down the server process.

### Client Only (`createOpencodeClient`)

Connects to an already-running OpenCode server. Use this when the server is managed externally (e.g., `opencode serve` or a TUI instance).

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({
  baseUrl: "http://localhost:4096",
  throwOnError: true,
  responseStyle: "data",
})
```

**Options:**

| Option | Type | Description | Default |
|---|---|---|---|
| `baseUrl` | `string` | URL of the server | `http://localhost:4096` |
| `fetch` | `function` | Custom fetch implementation | `globalThis.fetch` |
| `parseAs` | `string` | Response parsing method | `auto` |
| `responseStyle` | `string` | Return style: `"data"` or `"fields"` | `"fields"` |
| `throwOnError` | `boolean` | Throw errors instead of returning them | `false` |

---

## Response Styles

The `responseStyle` option controls how API responses are returned:

- **`"fields"`** (default): Returns `{ data, error, response }` — you check `error` manually.
- **`"data"`**: Returns just the data payload directly. When combined with `throwOnError: true`, throws on error.

```typescript
// "fields" style — check error manually
const result = await client.session.get({ path: { id: "abc" } })
if (result.error) {
  console.error("Error:", result.error)
} else {
  console.log(result.data)
}

// "data" style — returns payload directly
const session = await client.session.get({ path: { id: "abc" } })
// session is the Session object directly
```

---

## Error Handling

### throwOnError

When `throwOnError` is `false` (default), errors are returned in the `error` field of the response object. When `true`, they throw and should be caught:

```typescript
try {
  const session = await client.session.get({ path: { id: "invalid-id" } })
} catch (error) {
  console.error("Failed to get session:", (error as Error).message)
}
```

### Error Types

The SDK can return the following error types:

| Error Name | Description |
|---|---|
| `ProviderAuthError` | Authentication failure with a provider. Contains `providerID` and `message`. |
| `UnknownError` | An unexpected error occurred. Contains `message`. |
| `MessageOutputLengthError` | Model output exceeded the maximum length. |
| `MessageAbortedError` | The message generation was aborted. Contains `message`. |
| `ApiError` | An API-level error. Contains `message`, `statusCode`, `isRetryable`, `responseHeaders`, `responseBody`. |
| `BadRequest` | Invalid request parameters. Contains `message` and `kind` (`Params`, `Headers`, `Query`, `Body`, `Payload`). |
| `NotFoundError` | Resource not found. Contains `message`. |

### StructuredOutputError

If the model fails to produce valid structured output after all retries, the response will include a `StructuredOutputError`:

```typescript
if (result.data.info.error?.name === "StructuredOutputError") {
  console.error("Failed to produce structured output:", result.data.info.error.message)
  console.error("Attempts:", result.data.info.error.retries)
}
```

---

## Types

Import TypeScript definitions directly from the SDK:

```typescript
import type {
  Session,
  Message,
  AssistantMessage,
  UserMessage,
  Part,
  TextPart,
  FilePart,
  ToolPart,
  ReasoningPart,
  Config,
  Project,
  Provider,
  Agent,
  Symbol,
  FileNode,
  FileContent,
  File,
  Command,
  Todo,
  AgentConfig,
  ProviderConfig,
  Auth,
  Event,
  GlobalEvent,
} from "@opencode-ai/sdk"
```

All types are generated from the server's OpenAPI specification.

---

## Structured Output

Request validated JSON from the model by specifying a `format` with a JSON schema. The model uses a `StructuredOutput` tool to return matching JSON.

### Basic Usage

```typescript
const result = await client.session.prompt({
  path: { id: sessionId },
  body: {
    parts: [{ type: "text", text: "Research Anthropic and provide company info" }],
    format: {
      type: "json_schema",
      schema: {
        type: "object",
        properties: {
          company: { type: "string", description: "Company name" },
          founded: { type: "number", description: "Year founded" },
          products: {
            type: "array",
            items: { type: "string" },
            description: "Main products",
          },
        },
        required: ["company", "founded"],
      },
    },
  },
})

console.log(result.data.info.structured_output)
// { company: "Anthropic", founded: 2021, products: ["Claude", "Claude API"] }
```

### Output Format Types

| Type | Description |
|---|---|
| `"text"` | Default. Standard text response (no structured output) |
| `"json_schema"` | Returns validated JSON matching the provided schema |

### JSON Schema Format

When using `type: "json_schema"`, provide:

| Field | Type | Description |
|---|---|---|
| `type` | `"json_schema"` | Required. Specifies JSON schema mode |
| `schema` | `object` | Required. JSON Schema object defining the output structure |
| `retryCount` | `number` | Optional. Number of validation retries (default: 2) |

### Error Handling

If the model fails to produce valid structured output after all retries, the response will include a `StructuredOutputError`:

```typescript
if (result.data.info.error?.name === "StructuredOutputError") {
  console.error("Failed to produce structured output:", result.data.info.error.message)
  console.error("Attempts:", result.data.info.error.retries)
}
```

### Best Practices

1. **Provide clear descriptions** in your schema properties to help the model understand what data to extract.
2. **Use `required`** to specify which fields must be present.
3. **Keep schemas focused** — complex nested schemas may be harder for the model to fill correctly.
4. **Set appropriate `retryCount`** — increase for complex schemas, decrease for simple ones.

---

## Core Workflows

### Session Management

```typescript
import { createOpencode } from "@opencode-ai/sdk"

const { client, server } = await createOpencode()

// Create a session
const session = await client.session.create({
  body: { title: "My session" },
})
console.log("Session ID:", session.id)

// Create a child session
const childSession = await client.session.create({
  body: { title: "Child session", parentID: session.id },
})

// List all sessions
const sessions = await client.session.list()

// Get a session by ID
const retrieved = await client.session.get({ path: { id: session.id } })

// Update session title
await client.session.update({
  path: { id: session.id },
  body: { title: "Updated Title" },
})

// List child sessions
const children = await client.session.children({ path: { id: session.id } })

// Get todo list for a session
const todos = await client.session.todo({ path: { id: session.id } })

// Delete a session
await client.session.delete({ path: { id: childSession.id } })
```

### Prompting

```typescript
// Send a basic prompt
const result = await client.session.prompt({
  path: { id: session.id },
  body: {
    parts: [{ type: "text", text: "Hello!" }],
  },
})
console.log(result.data.info) // AssistantMessage

// Send prompt with a specific model
const result2 = await client.session.prompt({
  path: { id: session.id },
  body: {
    model: { providerID: "anthropic", modelID: "claude-3-5-sonnet-20241022" },
    parts: [{ type: "text", text: "Explain this codebase" }],
  },
})

// Send prompt with a specific agent
const result3 = await client.session.prompt({
  path: { id: session.id },
  body: {
    agent: "plan",
    parts: [{ type: "text", text: "Create a plan for this feature" }],
  },
})

// Send prompt with system override
const result4 = await client.session.prompt({
  path: { id: session.id },
  body: {
    system: "You are a senior Go developer. Be concise.",
    parts: [{ type: "text", text: "Review this function" }],
  },
})

// Send prompt with tool control
const result5 = await client.session.prompt({
  path: { id: session.id },
  body: {
    tools: { read: true, write: false, bash: false },
    parts: [{ type: "text", text: "Read and analyze the code" }],
  },
})

// Inject context without triggering AI response (noReply)
await client.session.prompt({
  path: { id: session.id },
  body: {
    noReply: true,
    parts: [{ type: "text", text: "You are a helpful assistant." }],
  },
})

// Send prompt with file parts
const result6 = await client.session.prompt({
  path: { id: session.id },
  body: {
    parts: [
      { type: "text", text: "Review this file" },
      {
        type: "file",
        mime: "text/plain",
        url: "file:///path/to/file.ts",
      },
    ],
  },
})

// Send prompt with agent invocation part
const result7 = await client.session.prompt({
  path: { id: session.id },
  body: {
    parts: [
      {
        type: "agent",
        name: "explore",
        source: { value: "Find all usages of createOpencode", start: 0, end: 30 },
      },
    ],
  },
})

// Send prompt asynchronously (fire and forget)
await client.session.prompt_async({
  path: { id: session.id },
  body: {
    parts: [{ type: "text", text: "Process this in the background" }],
  },
})
```

### Commands

```typescript
// Execute a slash command
const cmdResult = await client.session.command({
  path: { id: session.id },
  body: {
    command: "compact",
    arguments: "",
  },
})

// List available commands
const commands = await client.command.list()
```

### Shell

```typescript
// Run a shell command
const shellResult = await client.session.shell({
  path: { id: session.id },
  body: {
    agent: "build",
    command: "ls -la",
  },
})

// Run shell with specific model
const shellResult2 = await client.session.shell({
  path: { id: session.id },
  body: {
    agent: "build",
    model: { providerID: "anthropic", modelID: "claude-3-5-sonnet-20241022" },
    command: "git status",
  },
})
```

### Messages

```typescript
// List messages in a session
const messages = await client.session.messages({ path: { id: session.id } })
for (const msg of messages) {
  console.log(msg.info.role, msg.parts.map(p => p.type))
}

// List messages with limit
const recentMessages = await client.session.messages({
  path: { id: session.id },
  query: { limit: 10 },
})

// Get a single message
const msg = await client.session.message({
  path: { id: session.id, messageID: "msg-123" },
})
```

### Abort

```typescript
// Abort a running session
await client.session.abort({ path: { id: session.id } })
```

### Share / Unshare

```typescript
// Share a session (generates a shareable URL)
const shared = await client.session.share({ path: { id: session.id } })
console.log("Share URL:", shared.share?.url)

// Unshare a session
const unshared = await client.session.unshare({ path: { id: session.id } })
```

### Diff

```typescript
// Get the diff for a session
const diffs = await client.session.diff({ path: { id: session.id } })
for (const diff of diffs) {
  console.log(`${diff.file}: +${diff.additions} -${diff.deletions}`)
}

// Get diff for a specific message
const msgDiffs = await client.session.diff({
  path: { id: session.id },
  query: { messageID: "msg-123" },
})
```

### Summarize

```typescript
// Summarize a session
await client.session.summarize({
  path: { id: session.id },
  body: {
    providerID: "anthropic",
    modelID: "claude-3-5-sonnet-20241022",
  },
})
```

### Revert / Unrevert

```typescript
// Revert a specific message
await client.session.revert({
  path: { id: session.id },
  body: {
    messageID: "msg-123",
    partID: "part-456", // optional — revert a specific part
  },
})

// Restore all reverted messages
await client.session.unrevert({ path: { id: session.id } })
```

### Permissions Response

```typescript
// Respond to a permission request
await client.postSessionByIdPermissionsByPermissionId({
  path: {
    id: session.id,
    permissionID: "perm-123",
  },
  body: {
    response: "always", // "once" | "always" | "reject"
  },
})
```

### Fork

```typescript
// Fork a session at a specific message
const forked = await client.session.fork({
  path: { id: session.id },
  body: {
    messageID: "msg-123", // optional — fork from the beginning if omitted
  },
})
console.log("Forked session:", forked.id)
```

### Init

```typescript
// Analyze the current app and create AGENTS.md
await client.session.init({
  path: { id: session.id },
  body: {
    messageID: "msg-123",
    providerID: "anthropic",
    modelID: "claude-3-5-sonnet-20241022",
  },
})
```

### Session Status

```typescript
// Get status for all sessions
const statuses = await client.session.status()
// Returns: { [sessionID: string]: SessionStatus }
// SessionStatus is: { type: "idle" } | { type: "busy" } | { type: "retry", attempt: number, message: string, next: number }
```

---

### File Operations

```typescript
// Read a file
const content = await client.file.read({
  query: { path: "src/index.ts" },
})
console.log(content.content) // file content string

// List files and directories at a path
const fileList = await client.file.list({
  query: { path: "src" },
})
// Returns FileNode[] with { name, path, absolute, type, ignored }

// Search for text in files (ripgrep-style)
const textResults = await client.find.text({
  query: { pattern: "function.*opencode" },
})
for (const match of textResults) {
  console.log(`${match.path.text}:${match.line_number}: ${match.lines.text}`)
}

// Find files by name (fuzzy match)
const files = await client.find.files({
  query: { query: "*.ts" },
})

// Find directories
const dirs = await client.find.files({
  query: { query: "packages", type: "directory", limit: 20 },
})

// Find files with directory override
const customDirFiles = await client.find.files({
  query: { query: "*.json", directory: "/some/other/path" },
})

// Find workspace symbols
const symbols = await client.find.symbols({
  query: { query: "createOpencode" },
})

// Get file status (tracked files with git status)
const status = await client.file.status()
// Returns File[] with { path, added, removed, status: "added" | "deleted" | "modified" }
```

**`find.files` query parameters:**

| Parameter | Type | Description |
|---|---|---|
| `query` | `string` | Required. Search string (fuzzy match) |
| `type` | `"file"` or `"directory"` | Optional. Limit results to files or directories |
| `directory` | `string` | Optional. Override the project root for the search |
| `limit` | `number` | Optional. Max results (1–200) |
| `dirs` | `"true"` or `"false"` | Optional. Legacy flag (`"false"` returns only files) |

---

### TUI Control

```typescript
// Build a prompt incrementally
await client.tui.appendPrompt({ body: { text: "Fix the bug in" } })
await client.tui.appendPrompt({ body: { text: " src/index.ts" } })
await client.tui.submitPrompt()

// Clear the prompt
await client.tui.clearPrompt()

// Show toast notifications
await client.tui.showToast({
  body: { message: "Task completed", variant: "success" },
})

// Toast with title and duration
await client.tui.showToast({
  body: {
    title: "Build",
    message: "Compilation finished",
    variant: "info",
    duration: 3000,
  },
})

// Toast variants: "info" | "success" | "warning" | "error"
await client.tui.showToast({
  body: { message: "Something went wrong", variant: "error" },
})

// Open UI panels
await client.tui.openModels()
await client.tui.openThemes()
await client.tui.openSessions()
await client.tui.openHelp()

// Execute a command
await client.tui.executeCommand({ body: { command: "/compact" } })

// TUI control (for advanced use — wait for control requests and respond)
const control = await client.tui.control.next()
// control = { path: string, body: unknown }
await client.tui.control.response({ body: control.body })
```

---

### Event Streaming

```typescript
// Subscribe to server-sent events
const events = await client.event.subscribe()

for await (const event of events.stream) {
  console.log("Event:", event.type, event.properties)
}
```

### Filtering Events

```typescript
const events = await client.event.subscribe()

for await (const event of events.stream) {
  switch (event.type) {
    case "session.created":
      console.log("New session:", event.properties.info.id)
      break
    case "session.updated":
      console.log("Session updated:", event.properties.info.title)
      break
    case "session.deleted":
      console.log("Session deleted:", event.properties.info.id)
      break
    case "message.updated":
      console.log("Message:", event.properties.info.role)
      break
    case "message.part.updated":
      if (event.properties.part.type === "text") {
        process.stdout.write(event.properties.delta ?? event.properties.part.text)
      }
      break
    case "message.removed":
      console.log("Message removed:", event.properties.messageID)
      break
    case "permission.updated":
      console.log("Permission request:", event.properties.title)
      break
    case "session.status":
      console.log("Session status:", event.properties.status.type)
      break
    case "session.idle":
      console.log("Session idle:", event.properties.sessionID)
      break
    case "session.error":
      console.error("Session error:", event.properties.error)
      break
    case "file.edited":
      console.log("File edited:", event.properties.file)
      break
    case "todo.updated":
      console.log("Todos:", event.properties.todos)
      break
    case "session.diff":
      console.log("Diff:", event.properties.diff)
      break
    case "session.compacted":
      console.log("Session compacted:", event.properties.sessionID)
      break
    case "vcs.branch.updated":
      console.log("Branch changed:", event.properties.branch)
      break
    case "server.connected":
      console.log("Server connected")
      break
    case "installation.updated":
      console.log("Installed version:", event.properties.version)
      break
    case "installation.update-available":
      console.log("Update available:", event.properties.version)
      break
    default:
      console.log("Unknown event:", event.type)
  }
}
```

### Complete Event Type List

| Event Type | Description |
|---|---|
| `server.connected` | Server connection established |
| `server.instance.disposed` | Server instance disposed |
| `installation.updated` | Version installed |
| `installation.update-available` | Update available |
| `session.created` | Session created |
| `session.updated` | Session updated |
| `session.deleted` | Session deleted |
| `session.status` | Session status changed (`idle`, `busy`, `retry`) |
| `session.idle` | Session became idle |
| `session.compacted` | Session was compacted |
| `session.diff` | Session diff available |
| `session.error` | Session error occurred |
| `message.updated` | Message created or updated |
| `message.removed` | Message removed |
| `message.part.updated` | Message part updated (includes `delta` for streaming) |
| `message.part.removed` | Message part removed |
| `permission.updated` | Permission request pending |
| `permission.replied` | Permission request replied |
| `file.edited` | File was edited |
| `file.watcher.updated` | File system change detected |
| `todo.updated` | Todo list updated |
| `command.executed` | Command executed |
| `vcs.branch.updated` | Git branch changed |
| `tui.prompt.append` | Prompt text appended |
| `tui.command.execute` | TUI command executed |
| `tui.toast.show` | Toast notification shown |
| `pty.created` | PTY session created |
| `pty.updated` | PTY session updated |
| `pty.exited` | PTY session exited |
| `pty.deleted` | PTY session deleted |
| `lsp.client.diagnostics` | LSP diagnostics received |
| `lsp.updated` | LSP status updated |

---

### Authentication

```typescript
// Set API key for a provider
await client.auth.set({
  path: { id: "anthropic" },
  body: { type: "api", key: "your-api-key" },
})

// Set API key with metadata
await client.auth.set({
  path: { id: "openai" },
  body: {
    type: "api",
    key: "sk-...",
    metadata: { org: "org-..." },
  },
})

// Set OAuth credentials
await client.auth.set({
  path: { id: "github-copilot" },
  body: {
    type: "oauth",
    refresh: "refresh-token",
    access: "access-token",
    expires: Date.now() + 3600000,
  },
})

// Set well-known auth
await client.auth.set({
  path: { id: "some-provider" },
  body: {
    type: "wellknown",
    key: "some-key",
    token: "some-token",
  },
})
```

---

### Logging

```typescript
// Write a log entry
await client.app.log({
  body: {
    service: "my-app",
    level: "info",
    message: "Operation completed",
  },
})

// Log with extra metadata
await client.app.log({
  body: {
    service: "my-app",
    level: "error",
    message: "Request failed",
    extra: {
      statusCode: 500,
      url: "/api/users",
      retryCount: 3,
    },
  },
})

// Log levels: "debug" | "info" | "warn" | "error"
await client.app.log({
  body: {
    service: "my-app",
    level: "debug",
    message: "Debug details",
    extra: { userId: "123", action: "login" },
  },
})
```

---

## Complete API Reference

### Global

| Method | Description | Response |
|---|---|---|
| `client.global.health()` | Check server health and version | `{ healthy: true, version: string }` |

```typescript
const health = await client.global.health()
console.log(`Server v${health.data.version} is healthy: ${health.data.healthy}`)
```

### Global Event (Global SSE Stream)

| Method | Description | Response |
|---|---|---|
| `client.global.event.subscribe()` | Subscribe to global event stream (cross-project) | `GlobalEvent` stream |

```typescript
const globalEvents = await client.global.event.subscribe()
for await (const event of globalEvents.stream) {
  console.log(`[${event.directory}]`, event.payload.type, event.payload.properties)
}
```

### App

| Method | Description | Response |
|---|---|---|
| `client.app.log()` | Write a log entry | `boolean` |
| `client.app.agents()` | List all available agents | `Agent[]` |

```typescript
await client.app.log({
  body: {
    service: "my-app",
    level: "info",
    message: "Operation completed",
  },
})

const agents = await client.app.agents()
for (const agent of agents) {
  console.log(agent.name, agent.mode)
}
```

### Project

| Method | Description | Response |
|---|---|---|
| `client.project.list()` | List all projects | `Project[]` |
| `client.project.current()` | Get current project | `Project` |

```typescript
const projects = await client.project.list()
for (const p of projects) {
  console.log(`${p.id}: ${p.worktree}`)
}

const current = await client.project.current()
console.log("Current project:", current.worktree)
```

### Path

| Method | Description | Response |
|---|---|---|
| `client.path.get()` | Get current path info | `Path` |

```typescript
const pathInfo = await client.path.get()
console.log("State:", pathInfo.state)
console.log("Config:", pathInfo.config)
console.log("Worktree:", pathInfo.worktree)
console.log("Directory:", pathInfo.directory)
```

### Config

| Method | Description | Response |
|---|---|---|
| `client.config.get()` | Get config | `Config` |
| `client.config.update()` | Update config (PATCH) | `Config` |
| `client.config.providers()` | List providers and default models | `{ providers: Provider[], default: { [key: string]: string } }` |

```typescript
// Get config
const config = await client.config.get()
console.log("Model:", config.model)

// Update config (partial patch)
const updated = await client.config.update({
  body: { model: "anthropic/claude-3-5-sonnet-20241022" },
})

// List providers and default models
const { providers, default: defaults } = await client.config.providers()
for (const provider of providers) {
  console.log(`${provider.id}: ${Object.keys(provider.models).length} models`)
}
```

### Provider

| Method | Description | Response |
|---|---|---|
| `client.provider.list()` | List all providers with full details | `{ all: Provider[], default: {...}, connected: string[] }` |
| `client.provider.auth()` | Get provider auth methods | `{ [providerID: string]: ProviderAuthMethod[] }` |
| `client.provider.oauth.authorize()` | Authorize via OAuth | `ProviderAuthAuthorization` |
| `client.provider.oauth.callback()` | Handle OAuth callback | `boolean` |

```typescript
// List all providers
const { all, connected } = await client.provider.list()
console.log("Connected providers:", connected)

// Get auth methods
const authMethods = await client.provider.auth()
console.log("Anthropic auth methods:", authMethods["anthropic"])

// OAuth authorize
const auth = await client.provider.oauth.authorize({
  path: { id: "github-copilot" },
  body: { method: 0 },
})
console.log("Open this URL:", auth.url)
```

### Sessions

| Method | Description | Response |
|---|---|---|
| `client.session.list()` | List all sessions | `Session[]` |
| `client.session.create({ body })` | Create a session | `Session` |
| `client.session.get({ path })` | Get session by ID | `Session` |
| `client.session.delete({ path })` | Delete a session | `boolean` |
| `client.session.update({ path, body })` | Update session properties | `Session` |
| `client.session.children({ path })` | List child sessions | `Session[]` |
| `client.session.todo({ path })` | Get todo list for a session | `Todo[]` |
| `client.session.status()` | Get status for all sessions | `{ [sessionID: string]: SessionStatus }` |
| `client.session.init({ path, body })` | Analyze app, create `AGENTS.md` | `boolean` |
| `client.session.fork({ path, body })` | Fork a session | `Session` |
| `client.session.abort({ path })` | Abort a running session | `boolean` |
| `client.session.share({ path })` | Share a session | `Session` |
| `client.session.unshare({ path })` | Unshare a session | `Session` |
| `client.session.diff({ path, query })` | Get diff for a session | `FileDiff[]` |
| `client.session.summarize({ path, body })` | Summarize a session | `boolean` |
| `client.session.revert({ path, body })` | Revert a message | `Session` |
| `client.session.unrevert({ path })` | Restore reverted messages | `Session` |
| `client.postSessionByIdPermissionsByPermissionId()` | Respond to permission request | `boolean` |

### Messages

| Method | Description | Response |
|---|---|---|
| `client.session.messages({ path, query })` | List messages in a session | `{ info: Message, parts: Part[] }[]` |
| `client.session.message({ path })` | Get message details | `{ info: Message, parts: Part[] }` |
| `client.session.prompt({ path, body })` | Send a prompt (waits for response) | `{ info: AssistantMessage, parts: Part[] }` |
| `client.session.prompt_async({ path, body })` | Send prompt asynchronously (no wait) | `void` (204) |
| `client.session.command({ path, body })` | Execute a slash command | `{ info: AssistantMessage, parts: Part[] }` |
| `client.session.shell({ path, body })` | Run a shell command | `AssistantMessage` |

**`session.prompt` body fields:**

| Field | Type | Description |
|---|---|---|
| `parts` | `Array<TextPartInput \| FilePartInput \| AgentPartInput \| SubtaskPartInput>` | Required. Message parts |
| `model` | `{ providerID: string, modelID: string }` | Optional. Model to use |
| `agent` | `string` | Optional. Agent to use |
| `noReply` | `boolean` | Optional. Inject context without triggering AI response |
| `system` | `string` | Optional. System prompt override |
| `tools` | `{ [key: string]: boolean }` | Optional. Enable/disable specific tools |
| `messageID` | `string` | Optional. Message ID for updates |
| `format` | `Format` | Optional. Structured output format |

**`session.shell` body fields:**

| Field | Type | Description |
|---|---|---|
| `command` | `string` | Required. Shell command to execute |
| `agent` | `string` | Required. Agent to use |
| `model` | `{ providerID: string, modelID: string }` | Optional. Model to use |

**`session.command` body fields:**

| Field | Type | Description |
|---|---|---|
| `command` | `string` | Required. Command name |
| `arguments` | `string` | Required. Command arguments |
| `agent` | `string` | Optional. Agent to use |
| `model` | `string` | Optional. Model to use |
| `messageID` | `string` | Optional. Message ID |

### Files

| Method | Description | Response |
|---|---|---|
| `client.file.read({ query })` | Read a file | `FileContent` |
| `client.file.list({ query })` | List files/directories at a path | `FileNode[]` |
| `client.file.status({ query })` | Get status for tracked files | `File[]` |
| `client.find.text({ query })` | Search for text in files | Match objects with `path`, `lines`, `line_number`, `absolute_offset`, `submatches` |
| `client.find.files({ query })` | Find files and directories by name | `string[]` (paths) |
| `client.find.symbols({ query })` | Find workspace symbols | `Symbol[]` |

**`file.read` response (`FileContent`):**

```typescript
{
  type: "text" | "binary"
  content: string
  diff?: string
  patch?: {
    oldFileName: string
    newFileName: string
    oldHeader?: string
    newHeader?: string
    hunks: Array<{
      oldStart: number
      oldLines: number
      newStart: number
      newLines: number
      lines: string[]
    }>
    index?: string
  }
  encoding?: "base64"
  mimeType?: string
}
```

**`file.list` response (`FileNode`):**

```typescript
{
  name: string
  path: string
  absolute: string
  type: "file" | "directory"
  ignored: boolean
}
```

**`file.status` response (`File`):**

```typescript
{
  path: string
  added: number
  removed: number
  status: "added" | "deleted" | "modified"
}
```

**`find.symbols` response (`Symbol`):**

```typescript
{
  name: string
  kind: number
  location: {
    uri: string
    range: {
      start: { line: number, character: number }
      end: { line: number, character: number }
    }
  }
}
```

### TUI

| Method | Description | Response |
|---|---|---|
| `client.tui.appendPrompt({ body })` | Append text to the prompt | `boolean` |
| `client.tui.submitPrompt()` | Submit the current prompt | `boolean` |
| `client.tui.clearPrompt()` | Clear the prompt | `boolean` |
| `client.tui.openHelp()` | Open the help dialog | `boolean` |
| `client.tui.openSessions()` | Open the session selector | `boolean` |
| `client.tui.openModels()` | Open the model selector | `boolean` |
| `client.tui.openThemes()` | Open the theme selector | `boolean` |
| `client.tui.executeCommand({ body })` | Execute a command | `boolean` |
| `client.tui.showToast({ body })` | Show a toast notification | `boolean` |
| `client.tui.control.next()` | Wait for the next control request | `{ path: string, body: unknown }` |
| `client.tui.control.response({ body })` | Respond to a control request | `boolean` |

**`showToast` body:**

| Field | Type | Description |
|---|---|---|
| `title` | `string` | Optional. Toast title |
| `message` | `string` | Required. Toast message |
| `variant` | `"info" \| "success" \| "warning" \| "error"` | Required. Toast variant |
| `duration` | `number` | Optional. Duration in milliseconds |

**Available TUI commands (for `executeCommand`):**

| Command | Description |
|---|---|
| `session.list` | List sessions |
| `session.new` | New session |
| `session.share` | Share session |
| `session.interrupt` | Interrupt session |
| `session.compact` | Compact session |
| `session.page.up` | Scroll up one page |
| `session.page.down` | Scroll down one page |
| `session.half.page.up` | Scroll up half page |
| `session.half.page.down` | Scroll down half page |
| `session.first` | Navigate to first message |
| `session.last` | Navigate to last message |
| `prompt.clear` | Clear prompt |
| `prompt.submit` | Submit prompt |
| `agent.cycle` | Cycle through agents |
| (custom string) | Any custom command |

### Auth

| Method | Description | Response |
|---|---|---|
| `client.auth.set({ path, body })` | Set authentication credentials | `boolean` |

**Auth body types:**

```typescript
// API key auth
{ type: "api", key: string, metadata?: { [key: string]: string } }

// OAuth auth
{ type: "oauth", refresh: string, access: string, expires: number, enterpriseUrl?: string }

// Well-known auth
{ type: "wellknown", key: string, token: string }
```

### Events

| Method | Description | Response |
|---|---|---|
| `client.event.subscribe()` | Subscribe to server-sent events | Event stream |

---

## Type Safety

All API methods are fully typed. The SDK uses the OpenAPI spec to generate TypeScript types for every request and response.

### Key Types

```typescript
import type {
  // Session types
  Session,
  SessionStatus,

  // Message types
  Message,
  UserMessage,
  AssistantMessage,

  // Part types (message components)
  Part,
  TextPart,
  FilePart,
  ToolPart,
  ReasoningPart,
  StepStartPart,
  StepFinishPart,
  SnapshotPart,
  PatchPart,
  AgentPart,
  RetryPart,
  CompactionPart,

  // Input types
  TextPartInput,
  FilePartInput,
  AgentPartInput,
  SubtaskPartInput,

  // Config types
  Config,
  AgentConfig,
  ProviderConfig,
  KeybindsConfig,

  // Provider types
  Provider,
  Model,

  // File types
  FileNode,
  FileContent,
  File,
  FileDiff,

  // Project types
  Project,

  // Other types
  Symbol,
  Command,
  Agent,
  Todo,
  Auth,
  Path,

  // Event types
  Event,
  GlobalEvent,
} from "@opencode-ai/sdk"
```

### Tool State Types

```typescript
import type {
  ToolState,
  ToolStatePending,
  ToolStateRunning,
  ToolStateCompleted,
  ToolStateError,
} from "@opencode-ai/sdk"

// ToolState is a union of:
// ToolStatePending   { status: "pending", input, raw }
// ToolStateRunning   { status: "running", input, title?, metadata?, time }
// ToolStateCompleted { status: "completed", input, output, title, metadata, time, attachments? }
// ToolStateError     { status: "error", input, error, metadata?, time }
```

---

## Complete End-to-End Example

```typescript
import { createOpencode } from "@opencode-ai/sdk"

async function main() {
  const { client, server } = await createOpencode({
    port: 4096,
    config: { model: "anthropic/claude-3-5-sonnet-20241022" },
  })

  try {
    // 1. Health check
    const health = await client.global.health()
    console.log(`Server v${health.data.version} is healthy`)

    // 2. Configure auth
    await client.auth.set({
      path: { id: "anthropic" },
      body: { type: "api", key: process.env.ANTHROPIC_API_KEY! },
    })

    // 3. Check available providers
    const { providers } = await client.config.providers()
    console.log("Available providers:", providers.map(p => p.id))

    // 4. Create a session
    const session = await client.session.create({
      body: { title: "Code Review" },
    })
    console.log("Created session:", session.id)

    // 5. Subscribe to events in the background
    const eventPromise = (async () => {
      const events = await client.event.subscribe()
      for await (const event of events.stream) {
        if (event.type === "message.part.updated" && event.properties.part.type === "text") {
          process.stdout.write(event.properties.delta ?? event.properties.part.text)
        }
      }
    })()

    // 6. Send a prompt with structured output
    const result = await client.session.prompt({
      path: { id: session.id },
      body: {
        parts: [{ type: "text", text: "Review this codebase for issues" }],
        format: {
          type: "json_schema",
          schema: {
            type: "object",
            properties: {
              issues: {
                type: "array",
                items: {
                  type: "object",
                  properties: {
                    file: { type: "string" },
                    severity: { type: "string" },
                    description: { type: "string" },
                  },
                },
              },
            },
          },
        },
      },
    })

    console.log("\nStructured output:", result.data.info.structured_output)

    // 7. Search files
    const tsFiles = await client.find.files({ query: { query: "*.ts", type: "file" } })
    console.log("TypeScript files:", tsFiles.length)

    // 8. Search for text
    const matches = await client.find.text({ query: { pattern: "function" } })
    console.log("Function matches:", matches.length)

    // 9. Read a file
    if (tsFiles.length > 0) {
      const fileContent = await client.file.read({ query: { path: tsFiles[0] } })
      console.log(`First file (${tsFiles[0]}): ${fileContent.content.length} chars`)
    }

    // 10. Run a shell command
    const shell = await client.session.shell({
      path: { id: session.id },
      body: { agent: "build", command: "git log --oneline -5" },
    })
    console.log("Shell output:", shell)

    // 11. Get diff
    const diffs = await client.session.diff({ path: { id: session.id } })
    console.log("Diffs:", diffs)

    // 12. Share the session
    const shared = await client.session.share({ path: { id: session.id } })
    console.log("Share URL:", shared.share?.url)

    // 13. Get session status
    const statuses = await client.session.status()
    console.log("Session statuses:", statuses)

    // 14. Update session title
    await client.session.update({
      path: { id: session.id },
      body: { title: "Code Review - Completed" },
    })

    // 15. Log the operation
    await client.app.log({
      body: {
        service: "review-bot",
        level: "info",
        message: "Code review completed",
        extra: { sessionID: session.id, issuesFound: 5 },
      },
    })

    // 16. Show a success toast
    await client.tui.showToast({
      body: { message: "Code review completed!", variant: "success" },
    })

  } finally {
    // Always close the server
    server.close()
  }
}

main()
```

---

## Additional Examples

### Build Agent — Multi-turn Conversation

```typescript
const { client, server } = await createOpencode()

const session = await client.session.create({
  body: { title: "Multi-turn" },
})

// First message
await client.session.prompt({
  path: { id: session.id },
  body: {
    parts: [{ type: "text", text: "Create a simple HTTP server in Go" }],
  },
})

// Follow-up message in same session (context preserved)
await client.session.prompt({
  path: { id: session.id },
  body: {
    parts: [{ type: "text", text: "Now add error handling" }],
  },
})

// List all messages
const messages = await client.session.messages({ path: { id: session.id } })
for (const msg of messages) {
  console.log(`[${msg.info.role}]`, msg.parts.length, "parts")
}

server.close()
```

### Background Prompting with Async

```typescript
const { client, server } = await createOpencode()

const session = await client.session.create({
  body: { title: "Background Task" },
})

// Fire and forget — returns immediately
await client.session.prompt_async({
  path: { id: session.id },
  body: {
    parts: [{ type: "text", text: "Analyze this codebase" }],
  },
})

// Do other work...
console.log("Prompt submitted in background")

// Monitor via events
const events = await client.event.subscribe()
for await (const event of events.stream) {
  if (event.type === "session.idle" && event.properties.sessionID === session.id) {
    console.log("Background task completed!")
    break
  }
}

// Now fetch the results
const messages = await client.session.messages({ path: { id: session.id } })
console.log("Messages:", messages.length)

server.close()
```

### Permission Auto-Approval

```typescript
const events = await client.event.subscribe()

for await (const event of events.stream) {
  if (event.type === "permission.updated") {
    const perm = event.properties
    console.log(`Permission request: ${perm.title} (${perm.type})`)

    // Auto-approve all permissions
    await client.postSessionByIdPermissionsByPermissionId({
      path: {
        id: perm.sessionID,
        permissionID: perm.id,
      },
      body: {
        response: "always",
      },
    })
  }
}
```

### Config Update at Runtime

```typescript
// Get current config
const config = await client.config.get()
console.log("Current model:", config.model)

// Update config
const updated = await client.config.update({
  body: {
    model: "anthropic/claude-sonnet-4-20250514",
    agent: {
      build: {
        model: "anthropic/claude-sonnet-4-20250514",
        maxSteps: 20,
      },
      plan: {
        model: "anthropic/claude-sonnet-4-20250514",
      },
    },
  },
})
console.log("Updated model:", updated.model)
```

### Revert a Specific Message

```typescript
// Revert a message and its changes
await client.session.revert({
  path: { id: session.id },
  body: {
    messageID: "msg-123",
    partID: "part-456", // optional — if omitted, reverts the entire message
  },
})

// Check the revert state
const session = await client.session.get({ path: { id: session.id } })
console.log("Revert info:", session.revert)

// Undo the revert
await client.session.unrevert({ path: { id: session.id } })
```

### Fork from a Specific Point

```typescript
// Fork a session at a specific message to create a branch
const forkedSession = await client.session.fork({
  path: { id: originalSession.id },
  body: {
    messageID: "msg-42", // fork from this message
  },
})

console.log("Forked session:", forkedSession.id)

// The forked session has all messages up to the fork point
const messages = await client.session.messages({
  path: { id: forkedSession.id },
})
console.log("Messages in fork:", messages.length)
```

### MCP Server Management

```typescript
// Check MCP server status
const mcpStatus = await client.mcp.status()
console.log("MCP servers:", mcpStatus)

// Add a local MCP server
await client.mcp.add({
  body: {
    name: "my-tools",
    config: {
      type: "local",
      command: ["node", "my-mcp-server.js"],
      environment: { API_KEY: "..." },
      enabled: true,
      timeout: 10000,
    },
  },
})

// Add a remote MCP server
await client.mcp.add({
  body: {
    name: "remote-tools",
    config: {
      type: "remote",
      url: "https://mcp.example.com/sse",
      headers: { Authorization: "Bearer token" },
      enabled: true,
    },
  },
})

// Connect / disconnect
await client.mcp.connect({ path: { name: "my-tools" } })
await client.mcp.disconnect({ path: { name: "my-tools" } })
```

### LSP and Formatter Status

```typescript
// Check LSP server status
const lsp = await client.lsp.status()
for (const server of lsp) {
  console.log(`${server.name}: ${server.status}`)
}

// Check formatter status
const formatters = await client.formatter.status()
for (const fmt of formatters) {
  console.log(`${fmt.name}: ${fmt.enabled ? "enabled" : "disabled"}`)
}
```

---

## Key Points

- **Two modes**: `createOpencode` (starts server + client) or `createOpencodeClient` (connects to existing server).
- **Response styles**: `"fields"` (default, returns `{ data, error }`) or `"data"` (returns payload directly).
- **Structured output**: Use `format: { type: "json_schema", schema: {...} }` in prompt body for validated JSON responses.
- **`noReply: true`**: Inject context into a session without triggering an AI response.
- **Events**: Use `client.event.subscribe()` for real-time streaming of session, message, and system events.
- **All types** are auto-generated from the OpenAPI spec — import them from `@opencode-ai/sdk`.
- **Permission handling**: Listen for `permission.updated` events and respond via `postSessionByIdPermissionsByPermissionId`.
- **TUI control**: Use `client.tui.*` methods to drive the TUI from external code (used by IDE plugins).
- **Always close**: Call `server.close()` when using `createOpencode` to clean up resources.
- **OpenAPI spec**: View at `http://<hostname>:<port>/doc` for the full API definition.
