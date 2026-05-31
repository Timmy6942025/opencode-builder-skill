---
name: opencode-plugins
description: |
  Use this skill when building OpenCode plugins, hooking into OpenCode's lifecycle events, intercepting tool execution, injecting environment variables, adding custom tools via plugins, understanding plugin loading order and dependencies, or debugging plugin issues. Covers all plugin hooks, event types, TypeScript support, npm/local loading, custom tool creation, and real-world plugin patterns.
---

# OpenCode Plugins

Write your own plugins to extend OpenCode. Plugins hook into events, customize behavior, add custom tools, and integrate with external services.

---

## Plugin Loading

OpenCode loads plugins from multiple sources. Understanding the loading mechanism is critical for correct plugin placement and debugging.

### Local Files

Place JavaScript or TypeScript files in the plugin directory:

- `.opencode/plugins/` — **Project-level plugins** (scoped to the current project)
- `~/.config/opencode/plugins/` — **Global plugins** (available across all projects)

Files in these directories are automatically loaded at startup. Each file must export one or more plugin functions (see [Plugin Structure](#plugin-structure)).

### npm Packages

Specify npm packages in your config file's `"plugin"` array:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": [
    "opencode-helicone-session",
    "opencode-wakatime",
    "@my-org/custom-plugin"
  ]
}
```

Both **regular** (`package-name`) and **scoped** (`@scope/package-name`) npm packages are supported. Browse available plugins in the [ecosystem](https://opencode.ai/docs/ecosystem#plugins).

### How Plugins Are Installed

**npm plugins** are installed automatically using Bun at startup. Packages and their dependencies are cached in `~/.cache/opencode/node_modules/`. No manual `bun install` is needed — OpenCode handles it transparently.

**Local plugins** are loaded directly from the plugin directory. To use external npm packages in local plugins, you must create a `package.json` within your `.opencode/` directory (see [Dependencies](#dependencies)), or publish the plugin to npm and add it to the config.

### Load Order

Plugins are loaded from all sources and all hooks run in sequence. The load order is:

1. **Global config** (`~/.config/opencode/opencode.json`) — plugins listed in the global config's `"plugin"` array
2. **Project config** (`opencode.json`) — plugins listed in the project config's `"plugin"` array
3. **Global plugin directory** (`~/.config/opencode/plugins/`) — local JS/TS files
4. **Project plugin directory** (`.opencode/plugins/`) — local JS/TS files

### Deduplication Rules

- **Same npm package, same version**: Loaded only once. If the same `name@version` appears in both global and project config, it loads once.
- **Different versions**: Both versions are loaded separately.
- **Local + npm with similar names**: Both are loaded separately. A local file `.opencode/plugins/foo.ts` and an npm package `foo` in the config are distinct plugins that both run.

---

## Plugin Structure

A plugin is a **JavaScript/TypeScript module** that exports one or more plugin functions. Each function receives a context object and returns a hooks object.

### Basic Structure

```js
// .opencode/plugins/example.js
export const MyPlugin = async ({ project, client, $, directory, worktree }) => {
  console.log("Plugin initialized!")

  return {
    // Hook implementations go here
  }
}
```

### Context Object

The plugin function receives a context object with the following properties:

| Property | Type | Description |
|----------|------|-------------|
| `project` | `Project` | The current project information (name, path, etc.) |
| `directory` | `string` | The current working directory |
| `worktree` | `string` | The git worktree path (root of the git repository) |
| `client` | `Client` | An OpenCode SDK client for interacting with the AI server |
| `$` | `Shell` | Bun's [shell API](https://bun.sh/docs/runtime/shell) for executing system commands |

### Returns

The plugin function must return an object where keys are hook names and values are hook handler functions. It can also return a `tool` key for custom tool definitions.

### TypeScript Support

For type-safe plugins, import the `Plugin` type from the plugin package:

```ts
// .opencode/plugins/example.ts
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async ({ project, client, $, directory, worktree }) => {
  return {
    // Type-safe hook implementations
  }
}
```

The `Plugin` type provides full type annotations for all hooks, the context object, and tool definitions.

### Multiple Exports

A single file can export multiple plugin functions. Each export is loaded independently:

```ts
// .opencode/plugins/multi.ts
import type { Plugin } from "@opencode-ai/plugin"

export const PluginA: Plugin = async (ctx) => {
  return {
    "session.idle": async ({ event }) => {
      // Handle idle
    },
  }
}

export const PluginB: Plugin = async (ctx) => {
  return {
    "tool.execute.before": async (input, output) => {
      // Intercept tools
    },
  }
}
```

---

## Dependencies

Local plugins can use external npm packages. Add a `package.json` to your `.opencode/` config directory with the dependencies you need.

### Setup

1. Create `.opencode/package.json`:

```json
{
  "dependencies": {
    "shescape": "^2.1.0"
  }
}
```

2. OpenCode runs `bun install` at startup to install these dependencies.

3. Your plugins can then import them:

```ts
// .opencode/plugins/my-plugin.ts
import { escape } from "shescape"

export const MyPlugin = async (ctx) => {
  return {
    "tool.execute.before": async (input, output) => {
      if (input.tool === "bash") {
        output.args.command = escape(output.args.command)
      }
    },
  }
}
```

### Notes

- Dependencies are installed into `.opencode/node_modules/` (project-level) or `~/.config/opencode/node_modules/` (global-level).
- Bun is the package manager — use Bun-compatible packages.
- If you publish your plugin to npm, dependencies are resolved from the npm package's own `package.json` instead.

---

## Plugin Hooks

All hooks follow the pattern `(input, output) => {}` or `({ event }) => {}` depending on the hook type. Hooks run in the order plugins are loaded (see [Load Order](#load-order)).

### Tool Hooks

#### `tool.execute.before`

Fires before a tool executes. You can mutate `output.args` to modify the tool's arguments, or throw an error to block execution entirely.

```ts
"tool.execute.before": async (input, output) => {
  // input.tool — name of the tool (e.g. "bash", "read", "write", "glob")
  // output.args — mutable arguments object

  // Block dangerous commands
  if (input.tool === "bash" && output.args.command.includes("rm -rf /")) {
    throw new Error("Blocked dangerous command")
  }

  // Modify arguments
  if (input.tool === "bash") {
    output.args.command = `set -euo pipefail && ${output.args.command}`
  }
}
```

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `input.tool` | The tool name being executed (`"bash"`, `"read"`, `"write"`, `"glob"`, `"grep"`, etc.) |
| `output.args` | Mutable object containing the tool's arguments. Shape varies by tool. |

**Throwing:** Throw any `Error` to prevent the tool from executing. The error message is shown to the user.

#### `tool.execute.after`

Fires after a tool executes. Inspect the tool's return value or error state.

```ts
"tool.execute.after": async (input, output) => {
  // input.tool — name of the tool
  // output.result — the tool's return value (may be undefined on error)
  // output.error — error if the tool failed (may be undefined on success)

  if (input.tool === "write") {
    console.log(`File written: ${output.result?.filePath}`)
  }
}
```

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `input.tool` | The tool name that was executed |
| `output.result` | The tool's return value (type varies by tool) |
| `output.error` | Error object if the tool failed, otherwise `undefined` |

### Shell Hooks

#### `shell.env`

Inject or modify environment variables for **all** shell executions — this includes AI tool calls (the `bash` tool) and user terminal sessions.

```ts
"shell.env": async (input, output) => {
  // input.cwd — current working directory
  output.env.MY_API_KEY = "secret"
  output.env.PROJECT_ROOT = input.cwd
  output.env.NODE_ENV = "development"
}
```

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `input.cwd` | The current working directory for the shell execution |
| `output.env` | Mutable object of environment variables. Set keys to inject values. |

**Use cases:** Inject API keys, set project-specific env vars, configure PATH, override defaults.

### Session Hooks

#### `session.created`

Fires when a new session is created.

```ts
"session.created": async ({ event }) => {
  // event.properties — session details
  console.log("New session:", event.properties?.sessionID)
}
```

#### `session.updated`

Fires when a session is updated (e.g., title change, model change).

```ts
"session.updated": async ({ event }) => {
  console.log("Session updated:", event.properties?.sessionID)
}
```

#### `session.idle`

Fires when the AI finishes responding and the session becomes idle. Commonly used for notifications.

```ts
"session.idle": async ({ event }) => {
  await $`osascript -e 'display notification "Done!" with title "opencode"'`
}
```

#### `session.error`

Fires when a session encounters an error.

```ts
"session.error": async ({ event }) => {
  await client.app.log({
    body: {
      service: "my-plugin",
      level: "error",
      message: `Error: ${event.properties?.error}`,
    },
  })
}
```

#### `session.compacted`

Fires after session context has been compacted (context window management).

```ts
"session.compacted": async ({ event }) => {
  console.log("Session compacted:", event.properties?.sessionID)
}
```

#### `session.deleted`

Fires when a session is deleted.

```ts
"session.deleted": async ({ event }) => {
  console.log("Session deleted:", event.properties?.sessionID)
}
```

#### `session.diff`

Fires when a diff is computed (file changes detected).

```ts
"session.diff": async ({ event }) => {
  console.log("Diff available:", event.properties)
}
```

#### `session.status`

Fires when session status changes.

```ts
"session.status": async ({ event }) => {
  console.log("Status changed:", event.properties)
}
```

### Message Hooks

#### `message.updated`

Fires when a message is updated.

```ts
"message.updated": async ({ event }) => {
  console.log("Message updated:", event.properties?.messageID)
}
```

#### `message.removed`

Fires when a message is removed.

```ts
"message.removed": async ({ event }) => {
  console.log("Message removed:", event.properties?.messageID)
}
```

#### `message.part.updated`

Fires when a message part (text chunk, tool call result) is updated.

```ts
"message.part.updated": async ({ event }) => {
  console.log("Part updated:", event.properties?.partID)
}
```

#### `message.part.removed`

Fires when a message part is removed.

```ts
"message.part.removed": async ({ event }) => {
  console.log("Part removed:", event.properties?.partID)
}
```

### TUI Hooks

#### `tui.prompt.append`

Append text to the prompt input before submission.

```ts
"tui.prompt.append": async (input, output) => {
  // output.text — mutable string, prepend or modify
  output.text = `Branch: main\n${output.text}`
}
```

#### `tui.command.execute`

Execute a custom TUI command.

```ts
"tui.command.execute": async (input, output) => {
  // input.command — the command string
  // output.result — the command result
  console.log(`Command: ${input.command}`)
}
```

#### `tui.toast.show`

Show a toast notification in the TUI.

```ts
"tui.toast.show": async (input, output) => {
  // output.message — toast message
  // output.level — severity level
  output.message = "Custom plugin loaded successfully"
  output.level = "info"
}
```

### File Hooks

#### `file.edited`

Fires when a file is edited.

```ts
"file.edited": async ({ event }) => {
  console.log("File edited:", event.properties?.filePath)
}
```

#### `file.watcher.updated`

Fires when the file watcher detects a change on disk.

```ts
"file.watcher.updated": async ({ event }) => {
  console.log("File changed on disk:", event.properties?.filePath)
}
```

### Permission Hooks

#### `permission.asked`

Fires when OpenCode asks for a permission (e.g., file write, shell execution).

```ts
"permission.asked": async ({ event }) => {
  console.log("Permission requested:", event.properties?.permission)
}
```

#### `permission.replied`

Fires when a permission request is replied to (approved or denied).

```ts
"permission.replied": async ({ event }) => {
  console.log("Permission replied:", event.properties?.permission)
}
```

### LSP Hooks

#### `lsp.client.diagnostics`

Fires when LSP diagnostics are received (errors, warnings from language servers).

```ts
"lsp.client.diagnostics": async ({ event }) => {
  console.log("Diagnostics:", event.properties?.diagnostics)
}
```

#### `lsp.updated`

Fires when LSP state is updated (server started, stopped, or reconnected).

```ts
"lsp.updated": async ({ event }) => {
  console.log("LSP state:", event.properties?.language)
}
```

### Other Hooks

#### `command.executed`

Fires when a command is executed.

```ts
"command.executed": async ({ event }) => {
  console.log("Command executed:", event.properties)
}
```

#### `installation.updated`

Fires when installation state changes.

```ts
"installation.updated": async ({ event }) => {
  console.log("Installation state:", event.properties)
}
```

#### `todo.updated`

Fires when a todo item is updated.

```ts
"todo.updated": async ({ event }) => {
  console.log("Todo updated:", event.properties)
}
```

#### `server.connected`

Fires when the server connects.

```ts
"server.connected": async ({ event }) => {
  console.log("Server connected:", event.properties)
}
```

### Experimental Hooks

#### `experimental.session.compacting`

Fires **before** the LLM generates a continuation summary during compaction. This hook allows you to inject domain-specific context or replace the compaction prompt entirely.

**Inject additional context:**

```ts
"experimental.session.compacting": async (input, output) => {
  output.context.push(`## Custom Context
Include any state that should persist across compaction:
- Current task status
- Important decisions made
- Files being actively worked on`)
}
```

**Replace the entire compaction prompt:**

```ts
"experimental.session.compacting": async (input, output) => {
  output.prompt = `You are generating a continuation prompt for a multi-agent swarm session.
Summarize:
1. The current task and its status
2. Which files are being modified and by whom
3. Any blockers or dependencies between agents
4. The next steps to complete the work
Format as a structured prompt that a new agent can use to resume work.`
}
```

**Key behaviors:**
- `output.context` — An array of strings. Push additional context blocks to include in the compaction prompt.
- `output.prompt` — A string. When set, it **completely replaces** the default compaction prompt. When `output.prompt` is set, `output.context` is ignored.

### Event Hook Pattern

For event-based hooks (`session.*`, `message.*`, `file.*`, `permission.*`, `lsp.*`, `command.*`, `installation.*`, `todo.*`, `server.*`), you can use a **generic `event` handler** that listens to all events:

```ts
export const MyPlugin = async ({ client }) => {
  return {
    event: async ({ event }) => {
      // event.type — the event name (e.g. "session.idle", "file.edited")
      // event.properties — event-specific data

      switch (event.type) {
        case "session.idle":
          await client.app.log({
            body: { service: "my-plugin", level: "info", message: "Session idle" },
          })
          break
        case "file.edited":
          console.log("File:", event.properties?.filePath)
          break
        case "session.error":
          console.error("Error:", event.properties?.error)
          break
      }
    },
  }
}
```

This pattern is useful when you want to handle multiple event types in a single handler, or when you want a catch-all for logging/monitoring.

**Important:** Named hooks (like `"session.idle"`) and the generic `event` handler both fire. Named hooks fire first, then the generic handler. If both a named hook and the generic handler exist for the same event, both run.

---

## Custom Tools via Plugins

Plugins can add custom tools that OpenCode's AI can call during conversations. These tools work alongside built-in tools like `read`, `write`, and `bash`.

### The `tool()` Helper

Use the `tool()` helper from `@opencode-ai/plugin` for type-safe tool definitions with Zod validation:

```ts
import { type Plugin, tool } from "@opencode-ai/plugin"

export const CustomToolsPlugin: Plugin = async (ctx) => {
  return {
    tool: {
      mytool: tool({
        description: "This is a custom tool",
        args: {
          foo: tool.schema.string(),
        },
        async execute(args, context) {
          const { directory, worktree } = context
          return `Hello ${args.foo} from ${directory} (worktree: ${worktree})`
        },
      }),
    },
  }
}
```

### Tool Definition Properties

| Property | Type | Description |
|----------|------|-------------|
| `description` | `string` | What the tool does. Shown to the LLM. |
| `args` | `Record<string, ZodType>` | Zod schema defining the tool's arguments. Use `tool.schema.*` helpers. |
| `execute` | `async (args, context) => string` | Function that runs when the tool is called. Must return a string. |

### Schema Helpers

`tool.schema` provides the following Zod helpers:

- `tool.schema.string()` — String argument
- `tool.schema.number()` — Number argument
- `tool.schema.boolean()` — Boolean argument
- `tool.schema.array(schema)` — Array argument
- `tool.schema.object(schema)` — Object argument

Chain with `.describe("...")` to add descriptions visible to the LLM:

```ts
args: {
  query: tool.schema.string().describe("SQL query to execute"),
  limit: tool.schema.number().describe("Max rows to return").optional(),
}
```

### Multiple Tools Per Plugin

A single plugin can define multiple tools:

```ts
import { type Plugin, tool } from "@opencode-ai/plugin"

export const CustomToolsPlugin: Plugin = async (ctx) => {
  return {
    tool: {
      deploy: tool({
        description: "Deploy the application to the configured environment",
        args: {
          environment: tool.schema.string().describe("Target environment (staging, production)"),
        },
        async execute(args, context) {
          const { $ } = ctx
          await $`npm run deploy:${args.environment}`
          return `Deployed to ${args.environment}`
        },
      }),
      dbquery: tool({
        description: "Run a read-only SQL query against the local database",
        args: {
          query: tool.schema.string().describe("SQL query to execute"),
        },
        async execute(args) {
          return `Query result for: ${args.query}`
        },
      }),
    },
  }
}
```

### Tool Naming and Precedence

- **Plugin tools** are named by the key in the `tool` object (e.g., `deploy`, `dbquery`).
- **Custom tools** defined in `.opencode/tools/` are named by the filename (e.g., `database.ts` → tool name `database`).
- **Built-in tools** include `bash`, `read`, `write`, `glob`, `grep`, etc.

**If a plugin tool shares a name with a built-in tool, the plugin tool takes precedence.** This allows you to override built-in behavior:

```ts
// This replaces the built-in `bash` tool
export const MyPlugin: Plugin = async (ctx) => {
  return {
    tool: {
      bash: tool({
        description: "Restricted bash wrapper",
        args: {
          command: tool.schema.string(),
        },
        async execute(args) {
          if (args.command.includes("rm -rf /")) {
            throw new Error("Blocked dangerous command")
          }
          const result = await ctx.$`${args.command}`
          return result.stdout
        },
      }),
    },
  }
}
```

> Prefer unique tool names unless you intentionally want to replace a built-in tool. To disable a built-in tool without overriding it, use [permissions](https://opencode.ai/docs/permissions).

### Tool Execute Context

The `execute` function's second argument provides context about the current session:

| Property | Type | Description |
|----------|------|-------------|
| `context.agent` | `string` | Current agent name |
| `context.sessionID` | `string` | Current session ID |
| `context.messageID` | `string` | Current message ID |
| `context.directory` | `string` | Session working directory |
| `context.worktree` | `string` | Git worktree root |

### Alternative: Plain Zod Import

You can also import Zod directly instead of using `tool.schema`:

```ts
import { z } from "zod"
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Tool using raw Zod",
  args: {
    param: z.string().describe("Parameter description"),
  },
  async execute(args, context) {
    return `Result: ${args.param}`
  },
})
```

---

## Logging from Plugins

Use `client.app.log()` for structured logging instead of `console.log`. Logs are integrated into OpenCode's logging system and can be viewed in the app.

```ts
export const MyPlugin = async ({ client }) => {
  await client.app.log({
    body: {
      service: "my-plugin",
      level: "info",
      message: "Plugin initialized",
      extra: { foo: "bar" },
    },
  })
}
```

### Log Levels

| Level | Description |
|-------|-------------|
| `debug` | Detailed diagnostic information |
| `info` | General informational messages |
| `warn` | Warning messages |
| `error` | Error messages |

### Log Entry Structure

```ts
{
  body: {
    service: string    // Plugin identifier (e.g. "my-plugin")
    level: string      // "debug" | "info" | "warn" | "error"
    message: string    // Log message
    extra?: Record<string, any>  // Optional structured data
  }
}
```

### Best Practices

- Always set `service` to your plugin's name for easy filtering.
- Use `extra` for structured data instead of embedding it in the message string.
- Use `debug` for verbose output, `info` for normal operations, `warn` for recoverable issues, `error` for failures.
- Log initialization in the plugin function body (runs once at startup).
- Log events in hook handlers (runs per event).

---

## Complete Real-World Examples

### Notification Plugin

Send macOS notifications when the AI finishes responding or encounters an error:

```ts
// .opencode/plugins/notification.ts
import type { Plugin } from "@opencode-ai/plugin"

export const NotificationPlugin: Plugin = async ({ $ }) => {
  return {
    "session.idle": async () => {
      await $`osascript -e 'display notification "Session completed!" with title "opencode"'`
    },
    "session.error": async () => {
      await $`osascript -e 'display notification "Session error!" with title "opencode" sound name "Basso"'`
    },
  }
}
```

For the desktop app, OpenCode can send system notifications automatically — check your app settings.

### .env Protection Plugin

Prevent OpenCode from reading `.env` files containing secrets:

```ts
// .opencode/plugins/env-protection.ts
import type { Plugin } from "@opencode-ai/plugin"

export const EnvProtection: Plugin = async () => {
  return {
    "tool.execute.before": async (input, output) => {
      if (input.tool === "read" && output.args.filePath.includes(".env")) {
        throw new Error("Do not read .env files — they contain secrets")
      }
    },
  }
}
```

### Environment Variable Injection Plugin

Inject environment variables into all shell executions:

```ts
// .opencode/plugins/inject-env.ts
import type { Plugin } from "@opencode-ai/plugin"

export const InjectEnvPlugin: Plugin = async () => {
  return {
    "shell.env": async (input, output) => {
      output.env.MY_API_KEY = "secret"
      output.env.PROJECT_ROOT = input.cwd
      output.env.NODE_ENV = "development"
    },
  }
}
```

### Custom Tools Plugin

Define a deploy tool and a database query tool:

```ts
// .opencode/plugins/custom-tools.ts
import { type Plugin, tool } from "@opencode-ai/plugin"

export const CustomToolsPlugin: Plugin = async (ctx) => {
  return {
    tool: {
      deploy_to_vercel: tool({
        description: "Deploy the application to Vercel",
        args: {
          environment: tool.schema.string().describe("Target environment (preview, production)"),
        },
        async execute(args, context) {
          const { $ } = ctx
          const result = await $`vercel deploy --${args.environment}`
          return `Deployed: ${result.stdout}`
        },
      }),
      dbquery: tool({
        description: "Run a read-only SQL query against the local database",
        args: {
          query: tool.schema.string().describe("SQL query to execute"),
        },
        async execute(args) {
          return `Query result for: ${args.query}`
        },
      }),
    },
  }
}
```

### Compaction Customization Plugin

Inject project-specific context into compaction and log when it happens:

```ts
// .opencode/plugins/compaction.ts
import type { Plugin } from "@opencode-ai/plugin"

export const CompactionPlugin: Plugin = async ({ client }) => {
  return {
    "experimental.session.compacting": async (input, output) => {
      await client.app.log({
        body: {
          service: "compaction-plugin",
          level: "info",
          message: "Compacting session with custom context",
        },
      })

      output.context.push(`## Project State
- Active branch: main
- Last deploy: 2024-01-15
- Open PRs: #142, #145
- Key files: src/index.ts, src/config.ts`)
    },
  }
}
```

### Tool Execution Logger Plugin

Log all tool executions with timing information:

```ts
// .opencode/plugins/tool-logger.ts
import type { Plugin } from "@opencode-ai/plugin"

export const ToolLoggerPlugin: Plugin = async ({ client }) => {
  const startTimes = new Map<string, number>()

  return {
    "tool.execute.before": async (input) => {
      startTimes.set(input.tool, Date.now())
    },
    "tool.execute.after": async (input, output) => {
      const start = startTimes.get(input.tool)
      const duration = start ? Date.now() - start : 0
      startTimes.delete(input.tool)

      await client.app.log({
        body: {
          service: "tool-logger",
          level: "debug",
          message: `Tool "${input.tool}" executed in ${duration}ms`,
          extra: {
            tool: input.tool,
            duration,
            success: !output.error,
          },
        },
      })
    },
  }
}
```

### Generic Event Logger Plugin

Log all events using the generic event handler:

```ts
// .opencode/plugins/event-logger.ts
import type { Plugin } from "@opencode-ai/plugin"

export const EventLoggerPlugin: Plugin = async ({ client }) => {
  return {
    event: async ({ event }) => {
      await client.app.log({
        body: {
          service: "event-logger",
          level: "debug",
          message: `Event: ${event.type}`,
          extra: event.properties,
        },
      })
    },
  }
}
```

### Bash Safety Wrapper Plugin

Add safety checks to all bash commands:

```ts
// .opencode/plugins/bash-safety.ts
import type { Plugin } from "@opencode-ai/plugin"

const BLOCKED_PATTERNS = [
  /rm\s+-rf\s+\//,
  /mkfs/,
  /dd\s+if=/,
  />\s+\/dev\/sd/,
  /chmod\s+-R\s+777/,
]

export const BashSafetyPlugin: Plugin = async () => {
  return {
    "tool.execute.before": async (input, output) => {
      if (input.tool !== "bash") return

      const command = output.args.command as string
      for (const pattern of BLOCKED_PATTERNS) {
        if (pattern.test(command)) {
          throw new Error(`Blocked dangerous command matching pattern: ${pattern}`)
        }
      }
    },
  }
}
```

### Multi-Hook Plugin Example

A comprehensive plugin that uses multiple hook types:

```ts
// .opencode/plugins/comprehensive.ts
import type { Plugin } from "@opencode-ai/plugin"

export const ComprehensivePlugin: Plugin = async ({ client, $, directory }) => {
  await client.app.log({
    body: {
      service: "comprehensive",
      level: "info",
      message: `Plugin loaded in ${directory}`,
    },
  })

  return {
    // Inject env vars for all shell commands
    "shell.env": async (input, output) => {
      output.env.PROJECT_DIR = directory
    },

    // Log all tool executions
    "tool.execute.before": async (input) => {
      await client.app.log({
        body: {
          service: "comprehensive",
          level: "debug",
          message: `Tool starting: ${input.tool}`,
        },
      })
    },

    // Notify on errors
    "session.error": async ({ event }) => {
      await client.app.log({
        body: {
          service: "comprehensive",
          level: "error",
          message: `Session error: ${event.properties?.error}`,
        },
      })
      await $`osascript -e 'display notification "Error occurred!" with title "opencode" sound name "Basso"'`
    },

    // Custom tool
    tool: {
      project_info: (await import("@opencode-ai/plugin")).tool({
        description: "Get information about the current project",
        args: {},
        async execute(args, context) {
          return `Project directory: ${context.directory}, Worktree: ${context.worktree}`
        },
      }),
    },
  }
}
```

---

## Troubleshooting

### Plugin Not Loading

1. **Check file location**: Ensure the file is in `.opencode/plugins/` (project) or `~/.config/opencode/plugins/` (global).
2. **Check file extension**: Must be `.js` or `.ts`.
3. **Check exports**: The file must export a function using `export const MyPlugin = async (ctx) => { ... }`.
4. **Check npm config**: If using npm packages, ensure the package name is in `opencode.json`'s `"plugin"` array.
5. **Check syntax errors**: A syntax error in any plugin file can prevent loading. Open the file in an editor with error checking.
6. **Check Bun availability**: npm plugins require Bun. Ensure Bun is installed (`bun --version`).

### Hooks Not Firing

1. **Check hook name**: Ensure the hook name matches exactly (e.g., `"session.idle"` not `"sessionIdle"`).
2. **Check return type**: The plugin function must return an object with hook names as keys.
3. **Check async**: Hook handlers should be `async` functions (or return a Promise).
4. **Check event type filtering**: If using the generic `event` handler, check `event.type` matches what you expect.
5. **Multiple plugins**: Hooks run in load order. If another plugin throws, subsequent hooks may not run.

### TypeScript Errors

1. **Install types**: Run `bun add -D @opencode-ai/plugin` in your project (or add to `.opencode/package.json`).
2. **Import type**: Use `import type { Plugin } from "@opencode-ai/plugin"` (note the `type` keyword for type-only imports).
3. **Check Bun types**: Bun's shell API types are included with Bun. Ensure your editor supports Bun's type definitions.
4. **Zod version**: Ensure Zod version is compatible with the `tool.schema` helpers (bundled with `@opencode-ai/plugin`).

### Custom Tools Not Appearing

1. **Check tool key**: The tool must be in the `tool` object of the returned hooks.
2. **Check tool name**: Tool names are the keys in the `tool` object. Avoid spaces or special characters.
3. **Check execute return**: The `execute` function must return a string.
4. **Check args schema**: Args must be a valid Zod schema object. Use `tool.schema.*` helpers.

### Dependencies Not Installing

1. **Check package.json location**: Must be in `.opencode/package.json` (project) or `~/.config/opencode/package.json` (global).
2. **Check Bun**: `bun install` runs automatically at startup. Ensure Bun is installed.
3. **Check network**: npm packages require network access during installation.
4. **Check cache**: Cached in `~/.cache/opencode/node_modules/`. Try clearing the cache if issues persist.

---

## Summary of All Hooks

| Category | Hook | Signature | When It Fires |
|----------|------|-----------|---------------|
| Tool | `tool.execute.before` | `(input, output) => {}` | Before a tool runs. Mutate `output.args` or throw to block. |
| Tool | `tool.execute.after` | `(input, output) => {}` | After a tool runs. Inspect `output.result` or `output.error`. |
| Shell | `shell.env` | `(input, output) => {}` | Before shell execution. Set `output.env.*` to inject vars. |
| Session | `session.created` | `({ event }) => {}` | New session created. |
| Session | `session.updated` | `({ event }) => {}` | Session updated (title, model, etc.). |
| Session | `session.idle` | `({ event }) => {}` | AI finishes responding. |
| Session | `session.error` | `({ event }) => {}` | Session encounters an error. |
| Session | `session.compacted` | `({ event }) => {}` | Session context is compacted. |
| Session | `session.deleted` | `({ event }) => {}` | Session deleted. |
| Session | `session.diff` | `({ event }) => {}` | Diff computed. |
| Session | `session.status` | `({ event }) => {}` | Status changes. |
| Message | `message.updated` | `({ event }) => {}` | Message updated. |
| Message | `message.removed` | `({ event }) => {}` | Message removed. |
| Message | `message.part.updated` | `({ event }) => {}` | Message part updated. |
| Message | `message.part.removed` | `({ event }) => {}` | Message part removed. |
| TUI | `tui.prompt.append` | `(input, output) => {}` | Append text to prompt (`output.text`). |
| TUI | `tui.command.execute` | `(input, output) => {}` | Execute a custom command. |
| TUI | `tui.toast.show` | `(input, output) => {}` | Show toast notification (`output.message`, `output.level`). |
| File | `file.edited` | `({ event }) => {}` | File edited. |
| File | `file.watcher.updated` | `({ event }) => {}` | File watcher detects change. |
| Permission | `permission.asked` | `({ event }) => {}` | Permission requested. |
| Permission | `permission.replied` | `({ event }) => {}` | Permission replied to. |
| LSP | `lsp.client.diagnostics` | `({ event }) => {}` | LSP diagnostics received. |
| LSP | `lsp.updated` | `({ event }) => {}` | LSP state updated. |
| Command | `command.executed` | `({ event }) => {}` | Command executed. |
| Installation | `installation.updated` | `({ event }) => {}` | Installation state changes. |
| Todo | `todo.updated` | `({ event }) => {}` | Todo item updated. |
| Server | `server.connected` | `({ event }) => {}` | Server connects. |
| Experimental | `experimental.session.compacting` | `(input, output) => {}` | Before compaction. Set `output.context` or `output.prompt`. |
| Generic | `event` | `({ event }) => {}` | All events. Check `event.type` for filtering. |
