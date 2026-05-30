# Real-World Plugin Examples

Complete, production-ready plugin implementations with package.json, tsconfig, and README.

> **Note:** OpenCode runs on Bun, so these examples use Bun-native APIs like `Bun.file()`, `Response.json()`, and top-level `await`. Plugin tool `execute` functions can capture variables (`$`, `directory`) from the outer plugin scope via closure, or accept a second `context` argument. Both patterns are valid.

## Table of Contents
- [Example 1: Model Provider Proxy](#example-1-model-provider-proxy)
- [Example 2: File Protection + Audit Log](#example-2-file-protection--audit-log)
- [Example 3: Session Orchestrator](#example-3-session-orchestrator)
- [Example 4: Custom Tool Suite](#example-4-custom-tool-suite)
- [Example 5: TUI Enhancement](#example-5-tui-enhancement)

---

## Example 1: Model Provider Proxy

Wrap an external API (like Qwen OAuth) and expose it as an OpenAI-compatible endpoint for OpenCode.

**Architecture:**
```
OpenCode ──► Plugin starts proxy ──► Proxy wraps external CLI ──► External API
```

### `src/index.ts` (Plugin)

```typescript
import { Plugin, tool } from "@opencode-ai/plugin"
import { spawn } from "node:child_process"
import { existsSync } from "node:fs"
import { homedir } from "node:os"
import { join } from "node:path"

const PORT = 18921
const PROXY = `http://localhost:${PORT}`
const AUTH_FILE = join(homedir(), ".external-service", "auth.json")

async function startProxy(): Promise<void> {
  // Check if already running
  try {
    const res = await fetch(`${PROXY}/health`, { signal: AbortSignal.timeout(1000) })
    if (res.ok) return
  } catch { /* not running */ }

  const { dirname, fileURLToPath } = await import("node:url")
  const dir = dirname(fileURLToPath(import.meta.url))
  const proxyPath = join(dir, "proxy.js")
  if (!existsSync(proxyPath)) return

  const child = spawn("node", [proxyPath, String(PORT)], {
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

export const ProviderProxyPlugin: Plugin = async (ctx) => {
  const configured = existsSync(AUTH_FILE)

  await ctx.client.app.log({
    body: {
      service: "provider-proxy",
      level: configured ? "info" : "warn",
      message: configured ? "starting proxy" : "not authenticated",
    },
  })

  if (configured) await startProxy()

  return {
    "shell.env": async (_input, output) => {
      output.env.PROVIDER_PROXY_URL = PROXY
    },
    tool: {
      provider_status: tool({
        description: "Check provider proxy status",
        args: {},
        async execute() {
          let status = "stopped"
          try {
            status = (await fetch(`${PROXY}/health`, { signal: AbortSignal.timeout(1000) })).ok
              ? "running"
              : "stopped"
          } catch {}
          return `proxy: ${status}\nendpoint: ${PROXY}/v1`
        },
      }),
    },
  }
}

export default ProviderProxyPlugin
```

### `src/proxy.ts` (Proxy Server)

```typescript
#!/usr/bin/env bun
import { serve } from "bun"
import { spawn } from "node:child_process"
import { existsSync } from "node:fs"

const PORT = parseInt(process.argv[2] || "18921")
const AUTH_FILE = `${process.env.HOME}/.external-service/auth.json`

function buildPrompt(messages: any[]): string {
  const parts: string[] = []
  const last = messages[messages.length - 1]
  for (const msg of messages) {
    const text = typeof msg.content === "string"
      ? msg.content
      : msg.content?.filter((p: any) => p.type === "text").map((p: any) => p.text).join("\n") || ""
    if (!text) continue
    if (msg.role === "system") parts.push(text)
    else if (msg.role === "assistant") parts.push(`Assistant: ${text}`)
    else if (msg.role === "user") parts.push(msg === last ? text : `User: ${text}`)
  }
  return parts.join("\n\n") || (typeof last?.content === "string" ? last.content : "")
}

function runExternal(prompt: string): Promise<{ text: string; usage: any }> {
  return new Promise((resolve, reject) => {
    const child = spawn("external-cli", ["-o", "json", "-p", prompt], { timeout: 120_000 })
    let fullText = ""
    let usage: any = {}

    child.stdout?.on("data", (data: Buffer) => {
      try {
        const event = JSON.parse(data.toString().trim())
        if (event.content) fullText = event.content
        if (event.usage) usage = event.usage
      } catch {}
    })

    child.on("close", (code) => {
      if (code === 0 || code === null) resolve({ text: fullText, usage })
      else reject(new Error(`external-cli exited with ${code}`))
    })
    child.on("error", reject)
  })
}

serve({
  port: PORT,
  async fetch(req) {
    const url = new URL(req.url)

    if (url.pathname === "/health") {
      return Response.json({ status: "ok", auth: existsSync(AUTH_FILE) })
    }

    if (url.pathname === "/v1/chat/completions") {
      if (req.method !== "POST") return new Response("Method Not Allowed", { status: 405 })
      if (!existsSync(AUTH_FILE)) {
        return Response.json({ error: "Not authenticated" }, { status: 503 })
      }

      const body = await req.json()
      const { text, usage } = await runExternal(buildPrompt(body.messages))

      return Response.json({
        id: `chatcmpl-${Date.now()}`,
        object: "chat.completion",
        created: Math.floor(Date.now() / 1000),
        model: "custom-model",
        choices: [{ index: 0, message: { role: "assistant", content: text }, finish_reason: "stop" }],
        usage: {
          prompt_tokens: usage.input || 0,
          completion_tokens: usage.output || 0,
          total_tokens: usage.total || 0,
        },
      })
    }

    if (url.pathname === "/v1/models") {
      return Response.json({
        object: "list",
        data: [{ id: "custom-model", object: "model", owned_by: "custom" }],
      })
    }

    return new Response("Not Found", { status: 404 })
  },
})

console.log(`Proxy running on :${PORT}`)
```

### `package.json`

```json
{
  "name": "opencode-provider-proxy",
  "version": "1.0.0",
  "description": "OpenCode plugin that proxies to an external AI provider",
  "type": "module",
  "main": "src/index.ts",
  "scripts": {
    "build": "bun build src/index.ts --outdir dist --target node",
    "build:proxy": "bun build src/proxy.ts --outdir dist --target bun"
  },
  "dependencies": {
    "@opencode-ai/plugin": "latest"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "typescript": "latest"
  }
}
```

### `tsconfig.json`

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
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### `opencode.json` (User config)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["file:///path/to/opencode-provider-proxy/src/index.ts"],
  "provider": {
    "custom": {
      "name": "Custom Provider",
      "api": "openai",
      "url": "http://localhost:18921/v1"
    }
  },
  "model": "custom/custom-model"
}
```

---

## Example 2: File Protection + Audit Log

Prevent access to sensitive files and log all file operations.

### `src/index.ts`

```typescript
import { Plugin } from "@opencode-ai/plugin"
import { appendFile } from "node:fs/promises"
import { homedir } from "node:os"
import { join } from "node:path"

const AUDIT_LOG = join(homedir(), ".opencode", "audit.log")
const BLOCKED_PATTERNS = [
  /\.env$/,
  /\.env\.local$/,
  /\.env\.production$/,
  /id_rsa/,
  /id_ed25519/,
  /\.ssh\/config/,
  /\.aws\/credentials/,
  /token/i,
  /secret/i,
  /password/i,
  /key\.json$/,
]

function isBlocked(path: string): { blocked: boolean; pattern?: RegExp } {
  for (const pattern of BLOCKED_PATTERNS) {
    if (pattern.test(path)) return { blocked: true, pattern }
  }
  return { blocked: false }
}

async function log(action: string, path: string, result: string): Promise<void> {
  const entry = `[${new Date().toISOString()}] ${action}: ${path} — ${result}\n`
  await appendFile(AUDIT_LOG, entry)
}

export const FileProtectionPlugin: Plugin = async ({ client }) => {
  return {
    "tool.execute.before": async (input, output) => {
      if (input.tool !== "read" && input.tool !== "edit") return

      const filePath = output.args.filePath as string
      const check = isBlocked(filePath)

      if (check.blocked) {
        await log("BLOCKED", filePath, `matched ${check.pattern}`)
        throw new Error(
          `Security policy: ${filePath} is on the sensitive files list. ` +
          `If you need to modify secrets, use environment variables or a secret manager.`
        )
      }
    },

    "tool.execute.after": async (input, output) => {
      if (input.tool === "read" || input.tool === "edit") {
        const filePath = output.args.filePath as string
        await log(input.tool.toUpperCase(), filePath, "completed")
      }
    },
  }
}

export default FileProtectionPlugin
```

---

## Example 3: Session Orchestrator

Auto-initialize sessions with project context and send notifications on completion.

### `src/index.ts`

```typescript
import { Plugin } from "@opencode-ai/plugin"
import { readFile } from "node:fs/promises"
import { join } from "node:path"

interface ProjectContext {
  framework?: string
  language?: string
  importantFiles?: string[]
}

async function detectProjectContext(dir: string): Promise<ProjectContext> {
  const ctx: ProjectContext = {}
  try {
    const pkg = JSON.parse(await readFile(join(dir, "package.json"), "utf-8"))
    ctx.language = "TypeScript/JavaScript"
    if (pkg.dependencies?.next) ctx.framework = "Next.js"
    else if (pkg.dependencies?.["react-scripts"]) ctx.framework = "Create React App"
    else if (pkg.dependencies?.vue) ctx.framework = "Vue"
  } catch {}
  return ctx
}

export const OrchestratorPlugin: Plugin = async ({ client, directory }) => {
  const projectContext = await detectProjectContext(directory)

  return {
    "session.created": async ({ event }) => {
      const sessionId = event.properties.sessionId

      // Initialize session (creates AGENTS.md)
      await client.session.init({ path: { id: sessionId } })

      // Inject project context without triggering AI response
      if (projectContext.framework) {
        await client.session.prompt({
          path: { id: sessionId },
          body: {
            noReply: true,
            parts: [{
              type: "text",
              text: `This project uses ${projectContext.framework} with ${projectContext.language}.`,
            }],
          },
        })
      }

      // Inject important files list
      await client.session.prompt({
        path: { id: sessionId },
        body: {
          noReply: true,
          parts: [{
            type: "text",
            text: "Key files to know about: README.md, package.json, tsconfig.json",
          }],
        },
      })
    },

    "session.idle": async ({ event }) => {
      await client.app.log({
        body: {
          service: "orchestrator",
          level: "info",
          message: `Session ${event.properties.sessionId} completed`,
        },
      })
    },

    "session.error": async ({ event }) => {
      await client.tui.showToast({
        body: {
          message: `Error in session: ${event.properties.error}`,
          variant: "error",
        },
      })
    },
  }
}

export default OrchestratorPlugin
```

---

## Example 4: Custom Tool Suite

A plugin providing multiple custom tools for a specific workflow.

### `src/index.ts`

```typescript
import { Plugin, tool } from "@opencode-ai/plugin"
import { readFile } from "node:fs/promises"
import { join } from "node:path"

export const DevToolsPlugin: Plugin = async ({ $, directory }) => {
  return {
    tool: {
      // Git tools
      git_log: tool({
        description: "Get recent git commit history",
        args: {
          count: tool.schema.number().optional().describe("Number of commits to show"),
          format: tool.schema.enum(["oneline", "full"]).optional(),
        },
        async execute(args) {
          const n = args.count || 5
          const fmt = args.format === "oneline" ? "--oneline" : ""
          const result = await $`cd ${directory} && git log -n ${n} ${fmt}`
          return result.stdout || "No commits found"
        },
      }),

      // Package manager tools
      npm_outdated: tool({
        description: "Check for outdated npm dependencies",
        args: {},
        async execute() {
          const result = await $`cd ${directory} && npm outdated --json`
          const json = result.stdout ? JSON.parse(result.stdout) : {}
          const outdated = Object.entries(json)
            .map(([name, info]: [string, any]) => `${name}: ${info.current} → ${info.latest}`)
            .join("\n")
          return outdated || "All dependencies are up to date"
        },
      }),

      // File analysis tools
      analyze_imports: tool({
        description: "Analyze import statements in a file",
        args: {
          path: tool.schema.string().describe("Relative file path to analyze"),
        },
        async execute(args) {
          const content = await readFile(join(directory, args.path), "utf-8")
          const imports = content.match(/import .+ from ['"].+['"];?/g) || []
          const dynamicImports = content.match(/import\(['"].+['"]\)/g) || []
          return [
            `Static imports (${imports.length}):`,
            ...imports,
            `Dynamic imports (${dynamicImports.length}):`,
            ...dynamicImports,
          ].join("\n")
        },
      }),

      // Environment tools
      check_node_version: tool({
        description: "Check the Node.js version in the current project",
        args: {},
        async execute() {
          const node = await $`node --version`
          try {
            const pkg = JSON.parse(await readFile(join(directory, "package.json"), "utf-8"))
            const engines = pkg.engines?.node || "not specified"
            return `Node: ${node.stdout.trim()}\nRequired: ${engines}`
          } catch {
            return `Node: ${node.stdout.trim()}`
          }
        },
      }),
    },
  }
}

export default DevToolsPlugin
```

---

## Example 5: TUI Enhancement

Enhance the TUI with custom behaviors and notifications.

### `src/index.ts`

```typescript
import { Plugin } from "@opencode-ai/plugin"

export const TUIEnhancementPlugin: Plugin = async ({ client }) => {
  return {
    "tui.prompt.append": async ({ event }) => {
      const text = event.properties.text.toLowerCase()

      // Warn about destructive operations
      if (text.includes("rm -rf") || text.includes("delete everything")) {
        await client.tui.showToast({
          body: {
            message: "⚠️ This looks destructive. Consider using git first.",
            variant: "warning",
          },
        })
      }

      // Auto-suggest based on keywords
      if (text.includes("test") && !text.includes("npm test") && !text.includes("pytest")) {
        await client.tui.showToast({
          body: {
            message: "Tip: Try 'npm test' or 'pytest' depending on your project",
            variant: "default",
          },
        })
      }
    },

    "session.idle": async ({ event }) => {
      // Show completion toast after long sessions
      const sessionId = event.properties.sessionId
      const messages = await client.session.messages({ path: { id: sessionId } })
      const msgCount = messages.data?.length || 0

      if (msgCount > 10) {
        await client.tui.showToast({
          body: {
            message: `Session complete (${msgCount} messages). Consider summarizing with /compact.`,
            variant: "success",
          },
        })
      }
    },

    "lsp.client.diagnostics": async ({ event }) => {
      const diagnostics = event.properties.diagnostics
      const errors = diagnostics.filter((d: any) => d.severity === "error")
      const warnings = diagnostics.filter((d: any) => d.severity === "warning")

      if (errors.length > 0) {
        await client.tui.showToast({
          body: {
            message: `🐛 ${errors.length} error(s), ${warnings.length} warning(s) in ${event.properties.uri}`,
            variant: "error",
          },
        })
      }
    },
  }
}

export default TUIEnhancementPlugin
```

---

## Example 6: Compaction Customization

Customize how session context is summarized for long conversations.

### `src/index.ts`

```typescript
import { Plugin } from "@opencode-ai/plugin"

export const CompactionPlugin: Plugin = async ({ directory }) => {
  return {
    "experimental.session.compacting": async (input, output) => {
      // Replace the default compaction with a domain-specific one
      output.prompt = `
You are generating a continuation prompt for a software development session.

Summarize the following:
1. **Current task**: What is the main goal?
2. **Files modified**: Which files have been changed and how?
3. **Decisions made**: Any architectural or design decisions?
4. **Blockers**: Anything preventing progress?
5. **Next steps**: What should happen next?

Format as a structured prompt that a new developer could use to resume work.
Be specific about file paths, function names, and implementation details.
`
    },
  }
}

export default CompactionPlugin
```

---

## Example 7: SDK + Plugin Combo Project

A project that uses BOTH the SDK (external control) and a plugin (internal extension).

### `plugin/src/index.ts`

```typescript
import { Plugin } from "@opencode-ai/plugin"

export const BridgePlugin: Plugin = async ({ client }) => {
  return {
    "session.idle": async ({ event }) => {
      // When a session completes, notify the external controller
      await client.app.log({
        body: {
          service: "bridge",
          level: "info",
          message: "SESSION_COMPLETE",
          extra: { sessionId: event.properties.sessionId },
        },
      })
    },
  }
}
```

### `controller/src/index.ts`

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })

async function waitForSession(sessionId: string): Promise<void> {
  return new Promise((resolve) => {
    const check = setInterval(async () => {
      const session = await client.session.get({ path: { id: sessionId } })
      if (session.data.status === "idle") {
        clearInterval(check)
        resolve()
      }
    }, 1000)
  })
}

async function runWorkflow() {
  const session = await client.session.create({
    body: { title: "Automated refactor" },
  })

  await client.session.prompt({
    path: { id: session.id },
    body: {
      parts: [{ type: "text", text: "Refactor src/utils.ts to use async/await" }],
    },
  })

  await waitForSession(session.id)

  const messages = await client.session.messages({ path: { id: session.id } })
  const lastMessage = messages.data[messages.data.length - 1]
  console.log("Result:", lastMessage.parts[0].text)
}

runWorkflow().catch(console.error)
```

### `package.json` (monorepo root)

```json
{
  "name": "opencode-integration",
  "private": true,
  "workspaces": ["plugin", "controller"]
}
```
