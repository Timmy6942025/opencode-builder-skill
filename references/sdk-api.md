# SDK API Reference

Complete reference for `@opencode-ai/sdk`. All types are generated from OpenCode's OpenAPI spec.

## Table of Contents
- [Global](#global)
- [App](#app)
- [Project](#project)
- [Path](#path)
- [Config](#config)
- [Session](#session)
- [File](#file)
- [TUI](#tui)
- [Auth](#auth)
- [Events](#events)

---

## Global

### `global.health()`
Check server health and version.

**Response:** `{ healthy: true, version: string }`

```typescript
const health = await client.global.health()
console.log(health.data.version)
```

---

## App

### `app.log({ body })`
Write a structured log entry.

**Body fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `service` | `string` | Yes | Service/component name |
| `level` | `"debug" \| "info" \| "warn" \| "error"` | Yes | Log level |
| `message` | `string` | Yes | Log message |
| `extra` | `Record<string, any>` | No | Additional structured data |

**Response:** `boolean`

```typescript
await client.app.log({
  body: {
    service: "my-integration",
    level: "info",
    message: "Operation completed",
    extra: { duration: 1200, status: "success" },
  },
})
```

### `app.agents()`
List all available agents.

**Response:** `Agent[]`

---

## Project

### `project.list()`
List all workspace projects.

**Response:** `Project[]`

### `project.current()`
Get the current active project.

**Response:** `Project`

---

## Path

### `path.get()`
Get information about the current path.

**Response:** `Path`

---

## Config

### `config.get()`
Get the current system configuration.

**Response:** `Config`

### `config.providers()`
List available providers and their default models.

**Response:** `{ providers: Provider[], default: { [key: string]: string } }`

---

## Session

Sessions are the primary interaction unit in OpenCode.

### `session.list()`
List all sessions.

**Response:** `Session[]`

### `session.get({ path: { id } })`
Get a specific session.

**Response:** `Session`

### `session.children({ path: { id } })`
List child sessions (thread replies, branches).

**Response:** `Session[]`

### `session.create({ body })`
Create a new session.

**Body fields:**
| Field | Type | Required |
|-------|------|----------|
| `title` | `string` | No |
| `parent` | `string` (session ID) | No |

**Response:** `Session`

```typescript
const session = await client.session.create({
  body: { title: "Refactor auth module" },
})
```

### `session.delete({ path: { id } })`
Delete a session.

**Response:** `boolean`

### `session.update({ path: { id }, body })`
Update session properties.

**Response:** `Session`

### `session.init({ path: { id }, body? })`
Analyze the current app/project and create an `AGENTS.md` context file.

**Response:** `boolean`

### `session.abort({ path: { id } })`
Abort a running session (stops current AI generation).

**Response:** `boolean`

### `session.share({ path: { id } })`
Share a session (generates a public link).

**Response:** `Session`

### `session.unshare({ path: { id } })`
Revoke sharing for a session.

**Response:** `Session`

### `session.summarize({ path: { id }, body? })`
Generate a summary of the session.

**Response:** `boolean`

### `session.messages({ path: { id } })`
List all messages in a session.

**Response:** `Array<{ info: Message, parts: Part[] }>`

### `session.message({ path: { id, messageId } })`
Get a specific message's details.

**Response:** `{ info: Message, parts: Part[] }`

### `session.prompt({ path: { id }, body })`
Send a prompt to the session. This is the primary way to interact with the AI.

**Body fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parts` | `Part[]` | Yes | Message parts (text, images, etc.) |
| `model` | `{ providerID: string, modelID: string }` | No | Override default model |
| `noReply` | `boolean` | No | If `true`, injects context without triggering AI response |
| `format` | `OutputFormat` | No | Structured output format |

**Response:** `AssistantMessage` (if `noReply: false`) or `UserMessage` (if `noReply: true`)

**Example — standard prompt:**
```typescript
const result = await client.session.prompt({
  path: { id: session.id },
  body: {
    parts: [{ type: "text", text: "Refactor this function" }],
  },
})
```

**Example — context injection without reply:**
```typescript
await client.session.prompt({
  path: { id: session.id },
  body: {
    noReply: true,
    parts: [{ type: "text", text: "Project uses TypeScript strict mode." }],
  },
})
```

**Example — structured output:**
```typescript
const result = await client.session.prompt({
  path: { id: sessionId },
  body: {
    parts: [{ type: "text", text: "Extract info" }],
    format: {
      type: "json_schema",
      schema: {
        type: "object",
        properties: { name: { type: "string" } },
        required: ["name"],
      },
      retryCount: 2,
    },
  },
})
```

### `session.command({ path: { id }, body })`
Send a slash command to the session.

**Response:** `{ info: AssistantMessage, parts: Part[] }`

### `session.shell({ path: { id }, body })`
Execute a shell command within the session context.

**Body fields:**
| Field | Type | Required |
|-------|------|----------|
| `command` | `string` | Yes |

**Response:** `AssistantMessage`

### `session.revert({ path: { id }, body })`
Revert a message (undo to a previous state).

**Response:** `Session`

### `session.unrevert({ path: { id } })`
Restore reverted messages.

**Response:** `Session`

### `postSessionByIdPermissionsByPermissionId({ path, body })`
Respond to a permission request.

**Response:** `boolean`

---

## File

### `find.text({ query })`
Search for text across files in the project.

**Query fields:**
| Field | Type | Description |
|-------|------|-------------|
| `pattern` | `string` | Regex or literal pattern |
| `path` | `string` | Restrict to specific path |
| `limit` | `number` | Max results (1–200) |

**Response:** `Array<{ path: string, lines: { text: string }[], line_number: number, absolute_offset: number, submatches: { match: string, start: number, end: number }[] }>`

```typescript
const results = await client.find.text({
  query: { pattern: "function.*handler", limit: 50 },
})
```

### `find.files({ query })`
Find files and directories by name.

**Query fields:**
| Field | Type | Description |
|-------|------|-------------|
| `query` | `string` | Search query (glob or name) |
| `type` | `"file" \| "directory"` | Filter by type |
| `directory` | `string` | Override project root |
| `limit` | `number` | Max results (1–200) |

**Response:** `string[]` (paths)

```typescript
const files = await client.find.files({
  query: { query: "*.ts", type: "file", limit: 100 },
})
```

### `find.symbols({ query })`
Find workspace symbols (LSP-based).

**Response:** `Symbol[]`

### `file.read({ query })`
Read a file's contents.

**Query fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | `string` | Yes | File path relative to project root |

**Response:** `{ type: "raw" | "patch", content: string }`

```typescript
const file = await client.file.read({
  query: { path: "src/index.ts" },
})
console.log(file.data.content)
```

### `file.status({ query? })`
Get git status for tracked files.

**Response:** `File[]`

---

## TUI

Control the Terminal User Interface programmatically.

### `tui.appendPrompt({ body })`
Append text to the current prompt input.

**Body:** `{ text: string }`

**Response:** `boolean`

### `tui.submitPrompt()`
Submit the current prompt (press "enter").

**Response:** `boolean`

### `tui.clearPrompt()`
Clear the current prompt input.

**Response:** `boolean`

### `tui.openHelp()`
Open the help dialog.

**Response:** `boolean`

### `tui.openSessions()`
Open the session selector.

**Response:** `boolean`

### `tui.openThemes()`
Open the theme selector.

**Response:** `boolean`

### `tui.openModels()`
Open the model selector.

**Response:** `boolean`

### `tui.executeCommand({ body })`
Execute a slash command in the TUI.

**Body:** `{ command: string }`

**Response:** `boolean`

### `tui.showToast({ body })`
Show a toast notification.

**Body:** `{ message: string, variant?: "default" | "success" | "error" | "warning" }`

**Response:** `boolean`

---

## Auth

### `auth.set({ path: { id }, body })`
Set authentication credentials for a provider.

**Path:** `{ id: string }` — provider ID (e.g., `"anthropic"`, `"openai"`)

**Body fields:**
| Field | Type | Description |
|-------|------|-------------|
| `type` | `"api"` | Authentication type |
| `key` | `string` | API key |

**Response:** `boolean`

```typescript
await client.auth.set({
  path: { id: "anthropic" },
  body: { type: "api", key: "sk-ant-api03-..." },
})
```

---

## Events

### `event.subscribe()`
Subscribe to real-time server-sent events.

**Response:** `{ stream: AsyncIterable<Event> }`

```typescript
const events = await client.event.subscribe()
for await (const event of events.stream) {
  console.log(event.type, event.properties)
}
```

**Common event types:**
- `session.created` — New session started
- `session.updated` — Session metadata changed
- `session.idle` — AI finished responding
- `session.error` — Error occurred in session
- `message.updated` — Message content changed
- `message.part.updated` — Streaming message part update
- `tool.execute.before` — Tool about to execute
- `tool.execute.after` — Tool finished executing
- `tui.toast.show` — Toast notification shown
- `server.connected` — Server connection established

---

## Types

Import all types from the SDK:

```typescript
import type {
  Session,
  Message,
  Part,
  AssistantMessage,
  UserMessage,
  Config,
  Project,
  Provider,
  File,
  OutputFormat,
} from "@opencode-ai/sdk"
```

### Key Type Interfaces

**Part:**
```typescript
type Part =
  | { type: "text"; text: string }
  | { type: "image"; source: { type: "base64"; media_type: string; data: string } }
  | { type: "tool_use"; name: string; input: Record<string, any> }
  | { type: "tool_result"; tool_use_id: string; content: string }
```

**Session:**
```typescript
interface Session {
  id: string
  title: string
  created_at: string
  updated_at: string
  status: "active" | "idle" | "error"
  // ... additional fields
}
```

**OutputFormat:**
```typescript
type OutputFormat =
  | { type: "text" }
  | {
      type: "json_schema"
      schema: Record<string, any>  // JSON Schema object
      retryCount?: number           // default: 2
    }
```
