---
name: opencode-server
description: |
  Use this skill when running, configuring, connecting to, or extending the OpenCode HTTP server. Covers opencode serve, opencode web, authentication, mDNS discovery, CORS, the full OpenAPI 3.1 spec, all REST endpoints (sessions, messages, config, providers, files, tools, LSP, MCP, agents, TUI control, events, docs), the SDK generated from the spec, connecting clients, attaching terminals, IDE plugin integration, and real-time SSE event streaming.
---

# OpenCode Server

> **📚 Official Docs:** For the latest information, always refer to the official documentation:
> [https://opencode.ai/docs/server/](https://opencode.ai/docs/server/)

OpenCode exposes a headless HTTP server that implements a full REST API and publishes an OpenAPI 3.1 specification. The TUI is a client that talks to this server. The server enables multiple clients (TUI, web, IDE plugins, custom integrations) to control OpenCode programmatically.

---

## Usage

Start a standalone HTTP server:

```bash
opencode serve [--port <number>] [--hostname <string>] [--mdns] [--mdns-domain <string>] [--cors <origin>]
```

### Flags

| Flag | Description | Default |
|------|-------------|---------|
| `--port` | Port to listen on | `0 (random available port)` |
| `--hostname` | Hostname to listen on | `127.0.0.1` |
| `--mdns` | Enable mDNS discovery | `false` |
| `--mdns-domain` | Custom domain name for mDNS service | `opencode.local` |
| `--cors` | Additional browser origins to allow (repeatable) | `[]` |

The `--cors` flag can be passed multiple times:

```bash
opencode serve --cors http://localhost:5173 --cors https://app.example.com
```

When you run `opencode` normally (without `serve`), it starts both a TUI and an internal server. The TUI is the client that talks to this server. Running `opencode serve` starts a standalone headless server without launching the TUI.

---

## Authentication

Protect the server with HTTP basic auth using environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `OPENCODE_SERVER_PASSWORD` | Password for HTTP basic auth | _(none — unauthenticated)_ |
| `OPENCODE_SERVER_USERNAME` | Username for HTTP basic auth | `opencode` |

```bash
OPENCODE_SERVER_PASSWORD=your-password opencode serve
```

Authentication applies to both `opencode serve` and `opencode web`. If `OPENCODE_SERVER_PASSWORD` is not set, the server is unsecured — acceptable for local use, but should be set for network access.

---

## Architecture

The architecture separates the TUI (client) from the server:

- **Server** — Headless HTTP process that exposes the full OpenCode API.
- **TUI** — A client that connects to the server and renders the terminal UI.
- **Web** — A browser-based client that connects to the server.
- **SDK** — A type-safe JavaScript/TypeScript client generated from the server's OpenAPI spec.

The server publishes an OpenAPI 3.1 specification at `/doc`. The SDK (`@opencode-ai/sdk`) is generated from this spec, providing fully typed request/response objects for every endpoint.

This separation allows multiple clients to attach simultaneously — a terminal TUI, a browser tab, and custom integrations can all share the same sessions and state.

---

## Connect to an Existing Server

When you start the TUI, it randomly assigns a port and hostname. You can instead pass `--hostname` and `--port` flags to control the binding:

```bash
opencode --hostname 127.0.0.1 --port 4096
```

Then connect other clients to the same server:

```bash
# Connect the TUI to an existing server
opencode attach http://localhost:4096

# Or use the SDK
import { createOpencodeClient } from "@opencode-ai/sdk"
const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })
```

### TUI Endpoint for IDE Plugins

The `/tui` endpoint allows IDE plugins to drive the TUI through the server — prefill prompts, submit prompts, execute commands, open dialogs, and show notifications. This is the primary integration point for OpenCode IDE plugins.

---

## OpenAPI Spec

The server publishes an OpenAPI 3.1 spec at:

```
http://<hostname>:<port>/doc
```

For example: `http://localhost:4096/doc`

Use the spec to:
- Generate clients in any language
- Inspect request/response types in a Swagger explorer
- Build custom integrations with full type safety

---

## API Reference

All endpoints are relative to `http://<hostname>:<port>`.

---

### Global

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/global/health` | Get server health and version | `{ healthy: true, version: string }` |
| `GET` | `/global/event` | Get global events as SSE stream | Event stream |

#### `GET /global/health`

Returns server health status and version string. Useful for health checks and monitoring.

```json
{ "healthy": true, "version": "1.0.0" }
```

#### `GET /global/event`

Server-Sent Events stream of global events. First event is `server.connected`, then bus events stream continuously.

---

### Project

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/project` | List all projects | `Project[]` |
| `GET` | `/project/current` | Get the current project | `Project` |

The `Project` type contains project metadata including name, path, and configuration information.

---

### Path & VCS

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/path` | Get the current path | `Path` |
| `GET` | `/vcs` | Get VCS info for the current project | `VcsInfo` |

`/path` returns the current working directory and project root. `/vcs` returns version control system information (branch, status, etc.).

---

### Instance

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `POST` | `/instance/dispose` | Dispose the current instance | `boolean` |

Shuts down the current OpenCode instance. Use with caution — this terminates the server process.

---

### Config

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/config` | Get config info | `Config` |
| `PATCH` | `/config` | Update config | `Config` |
| `GET` | `/config/providers` | List providers and default models | `{ providers: Provider[], default: { [key: string]: string } }` |

#### `GET /config`

Returns the full configuration object for the current instance.

#### `PATCH /config`

Partially updates the configuration. Only the provided fields are modified; omitted fields remain unchanged.

#### `GET /config/providers`

Returns all configured providers and their default model mappings. The `default` map keys are provider IDs and values are model IDs.

---

### Provider

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/provider` | List all providers | `{ all: Provider[], default: {...}, connected: string[] }` |
| `GET` | `/provider/auth` | Get provider authentication methods | `{ [providerID: string]: ProviderAuthMethod[] }` |
| `POST` | `/provider/{id}/oauth/authorize` | Authorize a provider using OAuth | `ProviderAuthAuthorization` |
| `POST` | `/provider/{id}/oauth/callback` | Handle OAuth callback for a provider | `boolean` |

#### `GET /provider`

Returns:
- `all` — All configured providers
- `default` — Default model for each provider
- `connected` — List of connected (authenticated) provider IDs

#### `GET /provider/auth`

Returns available authentication methods for each provider. Useful for building OAuth flows in custom clients.

#### `POST /provider/{id}/oauth/authorize`

Initiates an OAuth authorization flow for the specified provider. Returns authorization URL and state.

#### `POST /provider/{id}/oauth/callback`

Handles the OAuth callback after the user authorizes. Returns `true` if authentication succeeded.

---

### Sessions

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/session` | List all sessions | `Session[]` |
| `POST` | `/session` | Create a new session | `Session` |
| `GET` | `/session/status` | Get session status for all sessions | `{ [sessionID: string]: SessionStatus }` |
| `GET` | `/session/:id` | Get session details | `Session` |
| `DELETE` | `/session/:id` | Delete a session and all its data | `boolean` |
| `PATCH` | `/session/:id` | Update session properties | `Session` |
| `GET` | `/session/:id/children` | Get a session's child sessions | `Session[]` |
| `GET` | `/session/:id/todo` | Get the todo list for a session | `Todo[]` |
| `POST` | `/session/:id/init` | Analyze app and create `AGENTS.md` | `boolean` |
| `POST` | `/session/:id/fork` | Fork an existing session at a message | `Session` |
| `POST` | `/session/:id/abort` | Abort a running session | `boolean` |
| `POST` | `/session/:id/share` | Share a session | `Session` |
| `DELETE` | `/session/:id/share` | Unshare a session | `Session` |
| `GET` | `/session/:id/diff` | Get the diff for this session | `FileDiff[]` |
| `POST` | `/session/:id/summarize` | Summarize the session | `boolean` |
| `POST` | `/session/:id/revert` | Revert a message | `boolean` |
| `POST` | `/session/:id/unrevert` | Restore all reverted messages | `boolean` |
| `POST` | `/session/:id/permissions/:permissionID` | Respond to a permission request | `boolean` |

#### `POST /session`

Create a new session. Body:

```json
{
  "parentID": "optional-parent-session-id",
  "title": "Optional session title"
}
```

Returns the created `Session` object.

#### `PATCH /session/:id`

Update session properties. Body:

```json
{
  "title": "New title"
}
```

#### `POST /session/:id/init`

Analyze the application and generate an `AGENTS.md` file. Body:

```json
{
  "messageID": "message-to-respond-to",
  "providerID": "anthropic",
  "modelID": "claude-3-5-sonnet-20241022"
}
```

#### `POST /session/:id/fork`

Fork an existing session, optionally at a specific message. Body:

```json
{
  "messageID": "optional-message-id-to-fork-at"
}
```

Returns a new `Session` object that is a copy of the original.

#### `POST /session/:id/share`

Makes the session publicly accessible. Returns the updated session with sharing metadata.

#### `DELETE /session/:id/share`

Revokes public sharing for the session.

#### `GET /session/:id/diff`

Get file changes for the session. Optional query parameter `messageID` to get diff up to a specific message.

Returns `FileDiff[]` — an array of file diffs showing added, removed, and modified content.

#### `POST /session/:id/summarize`

Generate a summary of the session. Body:

```json
{
  "providerID": "anthropic",
  "modelID": "claude-3-5-sonnet-20241022"
}
```

#### `POST /session/:id/revert`

Revert a specific message (and optionally a specific part within it). Body:

```json
{
  "messageID": "message-to-revert",
  "partID": "optional-specific-part"
}
```

#### `POST /session/:id/unrevert`

Restores all reverted messages in the session. No body required.

#### `POST /session/:id/permissions/:permissionID`

Respond to a pending permission request. Body:

```json
{
  "response": "allow",
  "remember": true
}
```

---

### Messages

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/session/:id/message` | List messages in a session | `{ info: Message, parts: Part[] }[]` |
| `POST` | `/session/:id/message` | Send a message and wait for response | `{ info: Message, parts: Part[] }` |
| `GET` | `/session/:id/message/:messageID` | Get message details | `{ info: Message, parts: Part[] }` |
| `POST` | `/session/:id/prompt_async` | Send a message asynchronously (no wait) | `204 No Content` |
| `POST` | `/session/:id/command` | Execute a slash command | `{ info: Message, parts: Part[] }` |
| `POST` | `/session/:id/shell` | Run a shell command | `{ info: Message, parts: Part[] }` |
| `DELETE` | `/session/:id/message/:messageID` | Delete a specific message | `boolean` |
| `DELETE` | `/session/:id/message/:messageID/part/:partID` | Delete a specific message part | `boolean` |
| `PATCH` | `/session/:id/message/:messageID/part/:partID` | Update a specific message part | `Part` |

#### `GET /session/:id/message`

List messages in a session. Optional query parameter `limit` to restrict the number of messages returned.

Each message contains:
- `info` — The `Message` object (role, content, metadata)
- `parts` — Array of `Part` objects (text chunks, tool calls, tool results)

#### `POST /session/:id/message`

Send a message and wait for the AI response. Body:

```json
{
  "messageID": "optional-message-id",
  "model": { "providerID": "anthropic", "modelID": "claude-3-5-sonnet-20241022" },
  "agent": "optional-agent-name",
  "noReply": false,
  "system": "optional-system-prompt",
  "tools": ["optional-tool-whitelist"],
  "parts": [
    { "type": "text", "text": "Hello!" }
  ]
}
```

- `noReply: true` — Injects context without triggering an AI response (returns a UserMessage)
- `model` — Optionally override the model for this message
- `agent` — Optionally specify an agent
- `tools` — Optionally restrict which tools the model can use
- `parts` — Message content parts (text, images, etc.)

#### `POST /session/:id/prompt_async`

Same body as `/session/:id/message` but returns immediately with `204 No Content`. The response is processed asynchronously; listen to events to know when it completes.

#### `POST /session/:id/command`

Execute a slash command. Body:

```json
{
  "messageID": "optional-message-id",
  "agent": "optional-agent",
  "model": { "providerID": "...", "modelID": "..." },
  "command": "/compact",
  "arguments": "optional arguments string"
}
```

#### `POST /session/:id/shell`

Run a shell command within the session context. Body:

```json
{
  "agent": "optional-agent",
  "model": { "providerID": "...", "modelID": "..." },
  "command": "ls -la"
}
```

Returns the assistant message with tool call results.

#### `DELETE /session/:id/message/:messageID`

Delete a specific message from a session. This permanently removes the message and all its parts.

#### `DELETE /session/:id/message/:messageID/part/:partID`

Delete a specific part within a message. Removes only the specified part (e.g., a single tool call or text chunk).

#### `PATCH /session/:id/message/:messageID/part/:partID`

Update a specific message part. Body:

```json
{
  "text": "Updated text content"
}
```

Allows modifying the content of a specific part within a message.

---

### Commands

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/command` | List all available commands | `Command[]` |

Returns all registered slash commands (e.g., `/compact`, `/init`, `/clear`).

---

### Files

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/find?pattern=<pat>` | Search for text in files | Match objects |
| `GET` | `/find/file?query=<q>` | Find files and directories by name | `string[]` |
| `GET` | `/find/symbol?query=<q>` | Find workspace symbols | `Symbol[]` |
| `GET` | `/file?path=<path>` | List files and directories | `FileNode[]` |
| `GET` | `/file/content?path=<p>` | Read a file | `FileContent` |
| `GET` | `/file/status` | Get status for tracked files | `File[]` |

#### `GET /find?pattern=<pat>`

Search for text in files using a regex pattern. Returns an array of match objects:

```json
[
  {
    "path": "src/index.ts",
    "lines": ["matching line content"],
    "line_number": 42,
    "absolute_offset": 1234,
    "submatches": [{ "match": "matched text", "start": 0, "end": 12 }]
  }
]
```

#### `GET /find/file?query=<q>`

Find files and directories by name using fuzzy matching.

**Query Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `query` | Yes | Search string (fuzzy match) |
| `type` | No | Limit to `"file"` or `"directory"` |
| `directory` | No | Override the project root for the search |
| `limit` | No | Max results (1–200) |
| `dirs` | No | Legacy flag (`"false"` returns only files) |

Returns `string[]` — array of matching file/directory paths.

#### `GET /find/symbol?query=<q>`

Find workspace symbols (functions, classes, variables, etc.).

#### `GET /file?path=<path>`

List files and directories at the given path. Returns `FileNode[]` with file metadata.

#### `GET /file/content?path=<p>`

Read the content of a file. Returns `FileContent` with the file's raw content.

#### `GET /file/status`

Get status for all tracked (VCS-tracked) files. Returns `File[]` with file status information.

---

### Tools (Experimental)

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/experimental/tool/ids` | List all tool IDs | `ToolIDs` |
| `GET` | `/experimental/tool` | List tools with JSON schemas for a model | `ToolList` |

#### `GET /experimental/tool`

Query parameters:
- `provider` — Provider ID
- `model` — Model ID

Returns the full tool definitions with JSON schemas for the specified model, showing which tools are available and their argument schemas.

---

### LSP, Formatters & MCP

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/lsp` | Get LSP server status | `LSPStatus[]` |
| `GET` | `/formatter` | Get formatter status | `FormatterStatus[]` |
| `GET` | `/mcp` | Get MCP server status | `{ [name: string]: MCPStatus }` |
| `POST` | `/mcp` | Add MCP server dynamically | MCP status object |

#### `GET /lsp`

Returns the status of all configured LSP (Language Server Protocol) servers — running, stopped, error states.

#### `GET /formatter`

Returns the status of all configured code formatters.

#### `GET /mcp`

Returns the status of all configured MCP (Model Context Protocol) servers. Each entry is keyed by server name.

#### `POST /mcp`

Dynamically add an MCP server at runtime. Body:

```json
{
  "name": "my-mcp-server",
  "config": {
    "command": "node",
    "args": ["server.js"]
  }
}
```

---

### Agents

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/agent` | List all available agents | `Agent[]` |

Returns all registered agents (built-in and custom) with their configurations.

---

### Logging

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `POST` | `/log` | Write a log entry | `boolean` |

Body:

```json
{
  "service": "my-app",
  "level": "info",
  "message": "Operation completed",
  "extra": { "key": "value" }
}
```

Log levels: `debug`, `info`, `warn`, `error`.

---

### TUI Control

These endpoints control the Terminal UI through the server. Used primarily by IDE plugins.

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `POST` | `/tui/append-prompt` | Append text to the prompt | `boolean` |
| `POST` | `/tui/submit-prompt` | Submit the current prompt | `boolean` |
| `POST` | `/tui/clear-prompt` | Clear the prompt | `boolean` |
| `POST` | `/tui/open-help` | Open the help dialog | `boolean` |
| `POST` | `/tui/open-sessions` | Open the session selector | `boolean` |
| `POST` | `/tui/open-models` | Open the model selector | `boolean` |
| `POST` | `/tui/open-themes` | Open the theme selector | `boolean` |
| `POST` | `/tui/execute-command` | Execute a command | `boolean` |
| `POST` | `/tui/show-toast` | Show toast notification | `boolean` |
| `GET` | `/tui/control/next` | Wait for the next control request | Control request object |
| `POST` | `/tui/control/response` | Respond to a control request | `boolean` |

#### `POST /tui/append-prompt`

Body: `{ "text": "text to append" }`

Appends text to the current prompt input without submitting.

#### `POST /tui/execute-command`

Body: `{ "command": "/compact" }`

Executes a slash command through the TUI.

#### `POST /tui/show-toast`

Body: `{ "title": "Optional title", "message": "Toast message", "variant": "success" }`

Variants: `info`, `success`, `warning`, `error`.

#### `GET /tui/control/next`

Long-polling endpoint that waits for the next control request from the TUI. Returns a control request object that represents a pending user interaction (e.g., permission prompt, input request).

#### `POST /tui/control/response`

Body: `{ "body": "response content" }`

Responds to a pending control request obtained from `/tui/control/next`.

---

### PTY (Terminal Pseudoterminal)

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/pty` | List all PTY sessions | `PTY[]` |
| `POST` | `/pty` | Create a new PTY session | `PTY` |
| `DELETE` | `/pty` | Close all PTY sessions | `boolean` |
| `GET` | `/pty/shells` | List available shells | `string[]` |
| `GET` | `/pty/:ptyID` | Get PTY session details | `PTY` |
| `PUT` | `/pty/:ptyID` | Update PTY session properties | `PTY` |
| `DELETE` | `/pty/:ptyID` | Close a PTY session | `boolean` |
| `GET` | `/pty/:ptyID/connect` | Connect to a PTY via SSE stream | Event stream |
| `POST` | `/pty/:ptyID/connect-token` | Generate a connection token for PTY | `ConnectToken` |

#### `POST /pty`

Create a new PTY session. Body:

```json
{
  "shell": "/bin/zsh",
  "cols": 80,
  "rows": 24
}
```

#### `GET /pty/:ptyID/connect`

Server-Sent Events stream for bidirectional terminal I/O. Use the connection token from `/pty/:ptyID/connect-token` to authenticate.

#### `POST /pty/:ptyID/connect-token`

Generate a short-lived token for connecting to the PTY session. Returns a token object with expiry.

---

### Permission

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/permission` | List pending permission requests | `Permission[]` |
| `POST` | `/permission/:requestID/reply` | Reply to a permission request | `boolean` |

#### `GET /permission`

Returns all pending permission requests that require user approval (e.g., file writes, shell commands).

#### `POST /permission/:requestID/reply`

Reply to a pending permission request. Body:

```json
{
  "response": "allow",
  "remember": true
}
```

Response values: `allow`, `deny`. The `remember` flag persists the decision for future requests of the same type.

---

### Question

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/question` | List pending questions | `Question[]` |
| `POST` | `/question/:requestID/reply` | Reply to a question | `boolean` |
| `POST` | `/question/:requestID/reject` | Reject/dismiss a question | `boolean` |

#### `GET /question`

Returns all pending questions from the AI that require user input.

#### `POST /question/:requestID/reply`

Reply to a pending question. Body:

```json
{
  "response": "Your answer here"
}
```

#### `POST /question/:requestID/reject`

Reject or dismiss a pending question without answering.

---

### Skill

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/skill` | List all available skills | `Skill[]` |

Returns all registered skills with their metadata and descriptions.

---

### Sync

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `POST` | `/sync/history` | Sync conversation history | `boolean` |
| `POST` | `/sync/replay` | Replay events to restore state | `boolean` |
| `POST` | `/sync/start` | Start sync session | `boolean` |
| `POST` | `/sync/steal` | Steal/takeover a session from another client | `boolean` |

#### `POST /sync/history`

Synchronize conversation history between clients. Used when a new client connects and needs to catch up.

#### `POST /sync/replay`

Replay stored events to bring a client up to date with the current server state.

#### `POST /sync/steal`

Take control of a session from another connected client. The other client receives a disconnect notification.

---

### Workspace (Experimental)

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/experimental/workspace` | List workspaces | `Workspace[]` |
| `POST` | `/experimental/workspace` | Create a workspace | `Workspace` |
| `GET` | `/experimental/workspace/adapter` | Get workspace adapter info | `Adapter` |
| `GET` | `/experimental/workspace/:id/status` | Get workspace status | `WorkspaceStatus` |
| `POST` | `/experimental/workspace/sync-list` | Sync workspace file list | `boolean` |
| `POST` | `/experimental/workspace/warp` | Warp to a workspace location | `boolean` |
| `DELETE` | `/experimental/workspace/:id` | Delete a workspace | `boolean` |

#### `POST /experimental/workspace`

Create a new workspace. Body:

```json
{
  "name": "my-workspace",
  "path": "/path/to/workspace"
}
```

#### `POST /experimental/workspace/warp`

Navigate to a specific location within the workspace. Body:

```json
{
  "path": "src/index.ts",
  "line": 42
}
```

---

### Worktree (Experimental)

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/experimental/worktree` | List git worktrees | `Worktree[]` |
| `POST` | `/experimental/worktree` | Create a worktree | `Worktree` |
| `DELETE` | `/experimental/worktree` | Delete worktrees | `boolean` |
| `POST` | `/experimental/worktree/reset` | Reset a worktree to clean state | `boolean` |

#### `POST /experimental/worktree`

Create a new git worktree. Body:

```json
{
  "branch": "feature/my-branch",
  "path": "/path/to/worktree"
}
```

#### `POST /experimental/worktree/reset`

Reset a worktree to a clean state, discarding all uncommitted changes.

---

### Auth

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `PUT` | `/auth/:id` | Set authentication credentials | `boolean` |
| `DELETE` | `/auth/:providerID` | Remove authentication credentials | `boolean` |

Body must match the provider's auth schema. Example:

```json
{
  "type": "api",
  "key": "your-api-key"
}
```

For OAuth providers, use the `/provider/{id}/oauth/authorize` and `/provider/{id}/oauth/callback` endpoints instead.

---

### Events

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/event` | Server-sent events stream | SSE stream |

The SSE stream sends events in real-time. The first event is `server.connected`, followed by bus events as they occur.

Event types include:
- `server.connected` — Server connection established
- `session.created` — New session created
- `session.updated` — Session properties changed
- `session.idle` — AI finished responding
- `session.error` — Session encountered an error
- `session.compacted` — Context was compacted
- `session.deleted` — Session was deleted
- `message.updated` — Message was updated
- `message.removed` — Message was removed
- `message.part.updated` — Message part (text chunk, tool call) updated
- `message.part.removed` — Message part removed
- `file.edited` — File was edited
- `permission.asked` — Permission request
- `permission.replied` — Permission response
- `lsp.updated` — LSP state change
- `todo.updated` — Todo item changed
- `command.executed` — Command was executed

Connect to the SSE stream:

```bash
curl -N http://localhost:4096/event
```

---

### Docs

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| `GET` | `/doc` | OpenAPI 3.1 specification | HTML page |

Returns an HTML page with the full OpenAPI 3.1 specification rendered in a Swagger UI explorer. Accessible at `http://localhost:4096/doc`.

---

## Web Interface

Start the browser-based web interface:

```bash
opencode web [--port <number>] [--hostname <string>] [--mdns] [--mdns-domain <string>] [--cors <origin>]
```

### Configuration

All `opencode serve` flags apply to `opencode web` as well:

```bash
# Fixed port
opencode web --port 4096

# Accessible on network
opencode web --hostname 0.0.0.0

# With authentication
OPENCODE_SERVER_PASSWORD=secret opencode web

# With mDNS
opencode web --mdns

# With custom mDNS domain
opencode web --mdns --mdns-domain myproject.local
```

When using `0.0.0.0`, OpenCode displays both local and network addresses:

```
Local access:     http://localhost:4096
Network access:   http://192.168.1.100:4096
```

### Config File

Server settings can also be configured in `opencode.json`:

```json
{
  "server": {
    "port": 4096,
    "hostname": "0.0.0.0",
    "mdns": true,
    "cors": ["https://example.com"]
  }
}
```

Command line flags take precedence over config file settings.

### Attaching a Terminal

Attach a terminal TUI to a running web server:

```bash
# Start the web server
opencode web --port 4096

# In another terminal, attach the TUI
opencode attach http://localhost:4096
```

Both the web interface and terminal share the same sessions and state. You can use both simultaneously.

### Web Interface Features

- **Sessions** — View and manage sessions from the homepage. See active sessions and start new ones.
- **Server Status** — Click "See Servers" to view connected servers and their status.

---

## mDNS Discovery

Enable mDNS to advertise the server on the local network:

```bash
opencode serve --mdns
# or
opencode web --mdns
```

This automatically sets the hostname to `0.0.0.0` and advertises the server as `opencode.local`.

### Custom Domain Names

Run multiple instances on the same network by customizing the mDNS domain:

```bash
opencode serve --mdns --mdns-domain project-a.local
opencode serve --mdns --mdns-domain project-b.local
```

Clients can then discover instances by browsing for `_opencode._tcp` services, or connect directly:

```bash
opencode attach http://project-a.local:4096
```

### How mDNS Works

- The server registers a DNS-SD service under `_opencode._tcp`
- Default instance name: `opencode` (becomes `opencode.local`)
- Custom domain names: `--mdns-domain myapp.local` becomes the service address
- Clients on the same network can discover the server without knowing its IP address

---

## SDK Integration

The OpenCode JS/TS SDK (`@opencode-ai/sdk`) is generated from the server's OpenAPI spec. It provides two modes:

### Full Lifecycle (Server + Client)

```typescript
import { createOpencode } from "@opencode-ai/sdk"

const { client, server } = await createOpencode({
  hostname: "127.0.0.1",
  port: 4096,
  timeout: 5000,
  config: { model: "anthropic/claude-3-5-sonnet-20241022" },
})

console.log(`Server running at ${server.url}`)

// Use the client...

server.close()
```

### Client Only

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({
  baseUrl: "http://localhost:4096",
  throwOnError: true,
  responseStyle: "data",
})
```

### Key SDK Methods

| Method | Description |
|--------|-------------|
| `client.global.health()` | Check server health |
| `client.session.create()` | Create a session |
| `client.session.list()` | List sessions |
| `client.session.prompt()` | Send a prompt |
| `client.session.messages()` | List messages |
| `client.tui.appendPrompt()` | Append to prompt |
| `client.tui.submitPrompt()` | Submit prompt |
| `client.event.subscribe()` | Subscribe to SSE events |
| `client.auth.set()` | Set provider credentials |

See the [SDK skill](../opencode-sdk/SKILL.md) for full SDK documentation.

---

## Complete Workflow Example

### 1. Start the server

```bash
opencode serve --port 4096
```

### 2. Check health

```bash
curl http://localhost:4096/global/health
# {"healthy":true,"version":"1.0.0"}
```

### 3. Create a session

```bash
curl -X POST http://localhost:4096/session \
  -H "Content-Type: application/json" \
  -d '{"title": "Code Review"}'
```

### 4. Send a prompt

```bash
curl -X POST http://localhost:4096/session/<session-id>/message \
  -H "Content-Type: application/json" \
  -d '{
    "parts": [{"type": "text", "text": "Review this codebase for issues"}],
    "model": {"providerID": "anthropic", "modelID": "claude-3-5-sonnet-20241022"}
  }'
```

### 5. Stream events

```bash
curl -N http://localhost:4096/event
```

### 6. View the spec

Open `http://localhost:4096/doc` in a browser.

---

## Troubleshooting

### Server won't start

- Check if the port is already in use: `lsof -i :4096`
- Try a different port: `opencode serve --port 5000`

### Authentication fails

- Ensure `OPENCODE_SERVER_PASSWORD` is set before starting the server
- The default username is `opencode`; override with `OPENCODE_SERVER_USERNAME`
- Both `serve` and `web` commands use the same auth environment variables

### CORS errors in browser

- Add the frontend origin to `--cors`: `opencode serve --cors http://localhost:5173`
- Multiple `--cors` flags are allowed
- The server automatically includes localhost origins for development

### mDNS not discoverable

- Ensure `--mdns` flag is passed
- Check that the hostname is `0.0.0.0` (mDNS sets this automatically)
- Verify mDNS is enabled on the client machine
- Try connecting directly by IP instead of `.local` domain

### TUI connection refused

- Verify the server is running: `curl http://localhost:4096/global/health`
- Check the correct port and hostname
- Use `opencode attach http://localhost:4096` to connect the TUI

### SDK types not available

- Install the SDK: `npm install @opencode-ai/sdk`
- Types are auto-generated from the OpenAPI spec at `/doc`
- Import types: `import type { Session, Message } from "@opencode-ai/sdk"`
