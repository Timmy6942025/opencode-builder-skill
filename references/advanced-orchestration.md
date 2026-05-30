# Advanced Orchestration: Building Multi-Agent Workflows on OpenCode

How to build large-scale orchestration systems on top of OpenCode — similar in spirit to Claude Code's Dynamic Workflows, but built on OpenCode's SDK and plugin architecture.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Part 1: Multi-Agent Workflow Primitives](#part-1-multi-agent-workflow-primitives)
- [Part 2: State Management](#part-2-state-management)
- [Part 3: Parallel Execution](#part-3-parallel-execution)
- [Part 4: Checkpoint & Resume](#part-4-checkpoint--resume)
- [Part 5: Adversarial Review & Quality Gates](#part-5-adversarial-review--quality-gates)
- [Part 6: Workflow Scripting Layer](#part-6-workflow-scripting-layer)
- [Part 7: Monitoring & Observability](#part-7-monitoring--observability)
- [Part 8: Putting It All Together](#part-8-putting-it-all-together)

---

## Architecture Overview

At the highest level, an orchestration system on OpenCode looks like this:

```
┌─────────────────────────────────────────────────────────────┐
│                     YOUR ORCHESTRATOR                        │
│  (TypeScript framework using @opencode-ai/sdk)              │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Worker 1   │  │   Worker 2   │  │   Worker N   │      │
│  │  (Session)   │  │  (Session)   │  │  (Session)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                 │                 │              │
│         └─────────────────┼─────────────────┘              │
│                           ▼                                  │
│                    ┌──────────────┐                          │
│                    │  Synthesizer │                          │
│                    │  (Session)   │                          │
│                    └──────────────┘                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              OpenCode Plugin (optional)                    │
│    - Tracks session lifecycle                              │
│    - Renders TUI dashboard                                 │
│    - Logs structured events                                │
└─────────────────────────────────────────────────────────────┘
```

**Key insight:** OpenCode sessions *are* your subagents. You don't need a separate agent runtime. Each `session.create()` spawns an isolated context with its own message history, file access, and tool use. The orchestrator coordinates them via the SDK.

**Two implementation strategies:**

| Strategy | Use When | Components |
|----------|----------|-----------|
| **External Orchestrator** | Complex workflows, CI/CD integration, headless automation | Standalone Node.js/Bun app using `@opencode-ai/sdk` |
| **Plugin Orchestrator** | Tight OpenCode integration, TUI dashboard, real-time feedback | OpenCode plugin using `@opencode-ai/plugin` + SDK client |

Most production systems use **both**: an external orchestrator for the heavy lifting, plus a plugin for TUI integration and lifecycle hooks.

---

## Part 1: Multi-Agent Workflow Primitives

### The Session-as-Agent Pattern

Each OpenCode session is an agent. To spawn a worker:

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })

async function spawnAgent(title: string, context: string): Promise<string> {
  const session = await client.session.create({ body: { title } })
  await client.session.init({ path: { id: session.id } })

  // Inject context without triggering a response
  await client.session.prompt({
    path: { id: session.id },
    body: {
      noReply: true,
      parts: [{ type: "text", text: context }],
    },
  })

  return session.id
}
```

### Sending Tasks and Collecting Results

```typescript
async function runTask(sessionId: string, task: string): Promise<string> {
  const result = await client.session.prompt({
    path: { id: sessionId },
    body: {
      parts: [{ type: "text", text: task }],
    },
  })

  // Extract the assistant's response text
  const assistantMessage = result.data.parts
    ?.filter((p: any) => p.type === "text")
    .map((p: any) => p.text)
    .join("\n") || ""

  return assistantMessage
}

async function getResult(sessionId: string): Promise<string> {
  const messages = await client.session.messages({ path: { id: sessionId } })
  const lastAssistant = [...messages.data].reverse().find(
    (m: any) => m.info.role === "assistant"
  )
  return lastAssistant?.parts
    ?.filter((p: any) => p.type === "text")
    .map((p: any) => p.text)
    .join("\n") || ""
}
```

### Waiting for Completion

```typescript
async function waitForIdle(sessionId: string, timeoutMs = 300_000): Promise<boolean> {
  const start = Date.now()
  while (Date.now() - start < timeoutMs) {
    const session = await client.session.get({ path: { id: sessionId } })
    if (session.data.status === "idle") return true
    if (session.data.status === "error") throw new Error(`Session ${sessionId} errored`)
    await new Promise(r => setTimeout(r, 1000))
  }
  throw new Error(`Timeout waiting for session ${sessionId}`)
}
```

**Alternative: Event-driven waiting**

```typescript
async function waitForIdleEvent(sessionId: string): Promise<void> {
  const events = await client.event.subscribe()
  for await (const event of events.stream) {
    if (event.type === "session.idle" && event.properties.sessionId === sessionId) {
      return
    }
    if (event.type === "session.error" && event.properties.sessionId === sessionId) {
      throw new Error(`Session error: ${event.properties.error}`)
    }
  }
}
```

### Cleanup

```typescript
async function cleanupAgents(sessionIds: string[]): Promise<void> {
  await Promise.all(sessionIds.map(id =>
    client.session.delete({ path: { id } }).catch(() => {})
  ))
}
```

---

## Part 2: State Management

Sessions are ephemeral — their state lives only in OpenCode's memory. For workflows that span minutes, hours, or need resumability, you need external state.

### Option A: SQLite (Recommended for Single-Node)

Bun has built-in SQLite support via `bun:sqlite`:

```typescript
import { Database } from "bun:sqlite"
import { join } from "node:path"
import { homedir } from "node:os"

const DB_PATH = join(homedir(), ".opencode", "workflows.db")

interface WorkflowState {
  id: string
  status: "running" | "paused" | "completed" | "failed"
  created_at: string
  updated_at: string
  checkpoint: string  // JSON blob
}

interface AgentState {
  id: string
  workflow_id: string
  session_id: string
  status: "pending" | "running" | "completed" | "failed"
  task: string
  result: string | null
  created_at: string
  completed_at: string | null
}

class WorkflowStore {
  private db: Database

  constructor() {
    this.db = new Database(DB_PATH)
    this.db.run(`
      CREATE TABLE IF NOT EXISTS workflows (
        id TEXT PRIMARY KEY,
        status TEXT NOT NULL,
        created_at TEXT NOT NULL,
        updated_at TEXT NOT NULL,
        checkpoint TEXT NOT NULL DEFAULT '{}'
      )
    `)
    this.db.run(`
      CREATE TABLE IF NOT EXISTS agents (
        id TEXT PRIMARY KEY,
        workflow_id TEXT NOT NULL,
        session_id TEXT NOT NULL,
        status TEXT NOT NULL,
        task TEXT NOT NULL,
        result TEXT,
        created_at TEXT NOT NULL,
        completed_at TEXT,
        FOREIGN KEY (workflow_id) REFERENCES workflows(id)
      )
    `)
  }

  createWorkflow(id: string): void {
    const now = new Date().toISOString()
    this.db.run(
      "INSERT INTO workflows (id, status, created_at, updated_at, checkpoint) VALUES (?, ?, ?, ?, ?)",
      [id, "running", now, now, "{}"]
    )
  }

  updateCheckpoint(workflowId: string, checkpoint: object): void {
    this.db.run(
      "UPDATE workflows SET checkpoint = ?, updated_at = ? WHERE id = ?",
      [JSON.stringify(checkpoint), new Date().toISOString(), workflowId]
    )
  }

  getCheckpoint(workflowId: string): object {
    const row = this.db.query("SELECT checkpoint FROM workflows WHERE id = ?").get(workflowId) as any
    return row ? JSON.parse(row.checkpoint) : {}
  }

  registerAgent(agent: Omit<AgentState, "created_at" | "completed_at">): void {
    this.db.run(
      "INSERT INTO agents (id, workflow_id, session_id, status, task, result, created_at) VALUES (?, ?, ?, ?, ?, ?, ?)",
      [agent.id, agent.workflow_id, agent.session_id, agent.status, agent.task, agent.result, new Date().toISOString()]
    )
  }

  completeAgent(agentId: string, result: string): void {
    this.db.run(
      "UPDATE agents SET status = ?, result = ?, completed_at = ? WHERE id = ?",
      ["completed", result, new Date().toISOString(), agentId]
    )
  }

  getAgents(workflowId: string): AgentState[] {
    return this.db.query("SELECT * FROM agents WHERE workflow_id = ?").all(workflowId) as AgentState[]
  }

  close(): void {
    this.db.close()
  }
}
```

### Option B: File-Based Checkpoints (Simplest)

For workflows that don't need querying:

```typescript
import { writeFile, readFile, mkdir } from "node:fs/promises"
import { join } from "node:path"
import { homedir } from "node:os"

const CHECKPOINT_DIR = join(homedir(), ".opencode", "checkpoints")

async function saveCheckpoint(workflowId: string, state: object): Promise<void> {
  await mkdir(CHECKPOINT_DIR, { recursive: true })
  const path = join(CHECKPOINT_DIR, `${workflowId}.json`)
  await writeFile(path, JSON.stringify(state, null, 2))
}

async function loadCheckpoint(workflowId: string): Promise<object | null> {
  try {
    const path = join(CHECKPOINT_DIR, `${workflowId}.json`)
    const data = await readFile(path, "utf-8")
    return JSON.parse(data)
  } catch {
    return null
  }
}
```

### Option C: Session-as-State (Clever but Limited)

You can store intermediate state in a dedicated "state" session:

```typescript
const stateSession = await client.session.create({ body: { title: "workflow-state" } })

async function setState(key: string, value: any): Promise<void> {
  await client.session.prompt({
    path: { id: stateSession.id },
    body: {
      noReply: true,
      parts: [{ type: "text", text: `SET ${key} = ${JSON.stringify(value)}` }],
    },
  })
}
```

**Not recommended for production** — sessions can be compacted/summarized, which may lose structured data.

---

## Part 3: Parallel Execution

### Basic Parallel Fan-Out

```typescript
async function parallelTasks<T>(
  items: T[],
  fn: (item: T, index: number) => Promise<string>,
  concurrency = 5
): Promise<string[]> {
  const results: string[] = new Array(items.length)
  const executing: Promise<void>[] = []

  for (let i = 0; i < items.length; i++) {
    const p = fn(items[i], i).then(result => {
      results[i] = result
      const idx = executing.indexOf(p)
      if (idx !== -1) executing.splice(idx, 1)
    })
    executing.push(p)

    if (executing.length >= concurrency) {
      await Promise.race(executing)
    }
  }

  await Promise.all(executing)
  return results
}
```

### Worker Pool with OpenCode Sessions

```typescript
interface Worker {
  sessionId: string
  busy: boolean
}

class SessionPool {
  private workers: Worker[] = []
  private queue: { task: string; resolve: (result: string) => void; reject: (err: Error) => void }[] = []

  constructor(private client: any, private poolSize = 3) {}

  async init(context: string): Promise<void> {
    for (let i = 0; i < this.poolSize; i++) {
      const session = await this.client.session.create({
        body: { title: `worker-${i}` },
      })
      await this.client.session.init({ path: { id: session.id } })
      await this.client.session.prompt({
        path: { id: session.id },
        body: { noReply: true, parts: [{ type: "text", text: context }] },
      })
      this.workers.push({ sessionId: session.id, busy: false })
    }
  }

  async execute(task: string): Promise<string> {
    const worker = this.workers.find(w => !w.busy)
    if (worker) {
      return this.runOnWorker(worker, task)
    }
    return new Promise((resolve, reject) => {
      this.queue.push({ task, resolve, reject })
    })
  }

  private async runOnWorker(worker: Worker, task: string): Promise<string> {
    worker.busy = true
    try {
      const result = await this.client.session.prompt({
        path: { id: worker.sessionId },
        body: { parts: [{ type: "text", text: task }] },
      })
      const text = result.data.parts
        ?.filter((p: any) => p.type === "text")
        .map((p: any) => p.text)
        .join("\n") || ""
      return text
    } finally {
      worker.busy = false
      const next = this.queue.shift()
      if (next) {
        this.runOnWorker(worker, next.task).then(next.resolve).catch(next.reject)
      }
    }
  }

  async destroy(): Promise<void> {
    await Promise.all(this.workers.map(w =>
      this.client.session.delete({ path: { id: w.sessionId } }).catch(() => {})
    ))
  }
}
```

### Backpressure & Rate Limiting

```typescript
class RateLimitedExecutor {
  private lastRequest = 0
  private tokens: number

  constructor(
    private maxConcurrency: number,
    private minIntervalMs: number,
    private tokenBucket = 100
  ) {
    this.tokens = tokenBucket
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // Token bucket for rate limiting
    while (this.tokens < 1) {
      await new Promise(r => setTimeout(r, 1000))
      this.tokens = Math.min(this.tokenBucket, this.tokens + 10)
    }
    this.tokens--

    // Minimum interval between requests
    const now = Date.now()
    const wait = Math.max(0, this.minIntervalMs - (now - this.lastRequest))
    if (wait > 0) await new Promise(r => setTimeout(r, wait))
    this.lastRequest = Date.now()

    return fn()
  }
}
```

---

## Part 4: Checkpoint & Resume

### Saving Checkpoints

```typescript
interface Checkpoint {
  workflowId: string
  step: number
  completedSteps: string[]
  agentResults: Record<string, { task: string; result: string; sessionId: string }>
  metadata: Record<string, any>
  timestamp: string
}

async function saveCheckpoint(
  client: any,
  cp: Checkpoint,
  saveFn: (id: string, state: object) => Promise<void>
): Promise<void> {
  await saveFn(cp.workflowId, cp)
  await client.app.log({
    body: {
      service: "orchestrator",
      level: "info",
      message: `Checkpoint saved at step ${cp.step}`,
      extra: { workflowId: cp.workflowId },
    },
  })
}
```

### Resuming from Checkpoint

```typescript
async function resumeWorkflow(
  client: any,
  workflowId: string,
  loadFn: (id: string) => Promise<object | null>,
  spawnFn: (title: string, context: string) => Promise<string>,
  runFn: (sessionId: string, task: string) => Promise<string>,
  runStepFn: (id: string, step: number, cp: Checkpoint) => Promise<void>
): Promise<void> {
  const cp = await loadFn(workflowId) as Checkpoint | null
  if (!cp) {
    throw new Error(`No checkpoint found for workflow ${workflowId}`)
  }

  console.log(`Resuming workflow ${workflowId} from step ${cp.step}`)

  // Recreate agents that were running but not completed
  for (const [agentId, agentData] of Object.entries(cp.agentResults)) {
    if (!agentData.result) {
      // This agent was interrupted — respawn it
      const sessionId = await spawnFn(`agent-${agentId}`, "")
      // Re-inject the task
      const result = await runFn(sessionId, agentData.task)
      cp.agentResults[agentId] = { ...agentData, result, sessionId }
    }
  }

  // Continue from the next step
  await runStepFn(workflowId, cp.step + 1, cp)
}
```

### Partial Completion Handling

When a workflow is interrupted, some agents may have finished and others may not:

```typescript
async function reconcileState(
  client: any,
  workflowId: string,
  expectedAgents: string[],
  store: WorkflowStore,
  startAgentFn: (wfId: string, agentId: string) => Promise<void>,
  getResultFn: (sessionId: string) => Promise<string>
): Promise<void> {
  const agents = store.getAgents(workflowId)

  for (const agentId of expectedAgents) {
    const agent = agents.find(a => a.id === agentId)
    if (!agent) {
      // Agent was never started — start it
      await startAgentFn(workflowId, agentId)
    } else if (agent.status === "running") {
      // Agent may have been interrupted — check if session is still alive
      try {
        const session = await client.session.get({ path: { id: agent.session_id } })
        if (session.data.status === "idle") {
          // Session finished but we didn't record it — capture result
          const result = await getResultFn(agent.session_id)
          store.completeAgent(agentId, result)
        }
        // If still running, leave it be
      } catch {
        // Session no longer exists — restart
        await startAgentFn(workflowId, agentId)
      }
    }
  }
}
```

---

## Part 5: Adversarial Review & Quality Gates

### Multi-Angle Synthesis

Instead of trusting a single agent, have multiple agents tackle the problem from different angles, then synthesize:

```typescript
async function multiAngleAnalysis(
  task: string,
  angles: string[]
): Promise<string> {
  // Spawn one agent per angle
  const agents = await Promise.all(
    angles.map(async (angle, i) => {
      const sessionId = await spawnAgent(
        `angle-${i}`,
        `You are analyzing this from the angle: ${angle}. Be thorough and specific.`
      )
      return { sessionId, angle }
    })
  )

  // Run all analyses in parallel
  const results = await Promise.all(
    agents.map(({ sessionId, angle }) =>
      runTask(sessionId, task).then(result => ({ angle, result }))
    )
  )

  // Spawn a synthesizer agent to merge findings
  const synthesizerId = await spawnAgent(
    "synthesizer",
    "You are a synthesis expert. You will receive multiple analyses of the same task from different angles. Your job is to identify agreements, contradictions, and produce a unified, high-confidence answer."
  )

  const synthesisPrompt = [
    `Task: ${task}`,
    "",
    ...results.map(r => `--- Analysis from ${r.angle} ---\n${r.result}\n`),
    "",
    "Synthesize these analyses. Highlight areas of agreement, note any contradictions, and provide a final, unified answer with confidence levels.",
  ].join("\n")

  const synthesis = await runTask(synthesizerId, synthesisPrompt)

  await cleanupAgents(agents.map(a => a.sessionId).concat(synthesizerId))
  return synthesis
}
```

### Cross-Verification (Refutation Loop)

Have one agent propose a solution, another try to break it:

```typescript
async function adversarialReview(
  problem: string,
  maxRounds = 3
): Promise<{ solution: string; confidence: number }> {
  let currentSolution = ""
  let lastCritique = ""
  let round = 0

  const proponentId = await spawnAgent("proponent", "You propose solutions.")
  const criticId = await spawnAgent("critic", "You critically review solutions and find flaws.")

  while (round < maxRounds) {
    // Proponent proposes or refines
    const proposePrompt = round === 0
      ? `Propose a solution to: ${problem}`
      : `Previous solution was critiqued as follows:\n${lastCritique}\n\nRefine the solution addressing these criticisms.`

    currentSolution = await runTask(proponentId, proposePrompt)

    // Critic reviews
    lastCritique = await runTask(criticId, `Critique this solution thoroughly. Find all flaws, edge cases, and incorrect assumptions:\n\n${currentSolution}`)

    // Check if critic is satisfied
    const verdict = await runTask(
      criticId,
      `Does the solution have any remaining flaws? Answer ONLY "APPROVED" or "REJECTED: [reason]".`
    )

    if (verdict.includes("APPROVED")) {
      await cleanupAgents([proponentId, criticId])
      return { solution: currentSolution, confidence: 1 - (round / maxRounds) }
    }

    round++
  }

  await cleanupAgents([proponentId, criticId])
  return { solution: currentSolution, confidence: 0.3 }
}
```

### Voting/Consensus Pattern

For binary or categorical decisions:

```typescript
async function consensusVote(
  question: string,
  options: string[],
  numVoters = 3
): Promise<{ winner: string; confidence: number }> {
  const voters = await Promise.all(
    Array.from({ length: numVoters }, (_, i) =>
      spawnAgent(`voter-${i}`, "You are an independent evaluator. Answer concisely.")
    )
  )

  const votes = await Promise.all(
    voters.map(id =>
      runTask(
        id,
        `Question: ${question}\nOptions: ${options.join(", ")}\nAnswer with ONLY the option name. No explanation.`
      )
    )
  )

  // Count votes
  const counts: Record<string, number> = {}
  for (const vote of votes) {
    const clean = vote.trim().toLowerCase()
    for (const opt of options) {
      if (clean.includes(opt.toLowerCase())) {
        counts[opt] = (counts[opt] || 0) + 1
        break
      }
    }
  }

  const winner = Object.entries(counts).sort((a, b) => b[1] - a[1])[0]
  const confidence = winner ? winner[1] / numVoters : 0

  await cleanupAgents(voters)
  return { winner: winner?.[0] || "none", confidence }
}
```

### Quality Gate: Test Validation

Before accepting an agent's code changes, run tests:

```typescript
async function validateWithTests(
  client: any,
  sessionId: string,
  testCommand = "npm test"
): Promise<{ passed: boolean; output: string }> {
  const result = await client.session.shell({
    path: { id: sessionId },
    body: { command: testCommand },
  })

  const output = result.data.parts
    ?.filter((p: any) => p.type === "text")
    .map((p: any) => p.text)
    .join("\n") || ""

  const passed = output.includes("PASS") || output.includes("ok") || !output.includes("FAIL")
  return { passed, output }
}
```

---

## Part 6: Workflow Scripting Layer

### Minimal Workflow DSL

Define workflows as TypeScript functions with a context object:

```typescript
interface WorkflowContext {
  workflowId: string
  client: any
  store: WorkflowStore
  checkpoint: (step: number, extra?: object) => Promise<void>
  spawn: (title: string, context: string) => Promise<string>
  run: (sessionId: string, task: string) => Promise<string>
  parallel: <T>(items: T[], fn: (item: T, index: number) => Promise<string>, concurrency?: number) => Promise<string[]>
  synthesize: (results: string[], prompt: string) => Promise<string>
  vote: (question: string, options: string[]) => Promise<string>
  validate: (sessionId: string, testCmd: string) => Promise<boolean>
}

type Workflow = (ctx: WorkflowContext) => Promise<void>
```

### Example: Security Audit Workflow

```typescript
function chunk<T>(array: T[], size: number): T[][] {
  const chunks: T[][] = []
  for (let i = 0; i < array.length; i += size) {
    chunks.push(array.slice(i, i + size))
  }
  return chunks
}

const securityAuditWorkflow: Workflow = async (ctx) => {
  const { client, workflowId, spawn, run, parallel, synthesize, checkpoint } = ctx

  // Step 1: Discover all source files
  const files = await client.find.files({
    query: { query: "*.{ts,js,tsx,jsx}", type: "file", limit: 200 },
  })
  const fileList = files.data || []
  await checkpoint(1, { fileCount: fileList.length })

  // Step 2: Spawn review agents (one per batch of 10 files)
  const batches = chunk(fileList, 10)
  const reviews = await parallel(batches, async (batch, i) => {
    const agentId = await spawn(`security-reviewer-${i}`,
      "You are a security expert. Review code for vulnerabilities: SQL injection, XSS, path traversal, hardcoded secrets, insecure dependencies."
    )
    const filesText = batch.join("\n")
    return run(agentId, `Review these files for security issues:\n${filesText}`)
  }, 5)
  await checkpoint(2, { reviews })

  // Step 3: Synthesize findings
  const findings = await synthesize(reviews,
    "Consolidate these security reviews. Remove duplicates. Prioritize by severity. Format as a markdown report with file paths and line numbers."
  )
  await checkpoint(3, { findings })

  // Step 4: Adversarial verification
  const verifierId = await spawn("verifier", "You verify security findings. Try to disprove or confirm each finding.")
  const verifiedFindings = await run(verifierId, `Verify these security findings. Mark each as CONFIRMED or FALSE_POSITIVE:\n\n${findings}`)

  // Step 5: Output final report
  console.log("=== SECURITY AUDIT REPORT ===")
  console.log(verifiedFindings)
}
```

### Example: Codebase Migration Workflow

```typescript
function extractFiles(plan: string, complexity: string): string[] {
  // Simple regex-based extraction — replace with robust parsing for production
  const regex = new RegExp(`\\b${complexity}\\b.*?(?:file|path)[\s:]*(\\S+\\.(?:ts|js|tsx|jsx))`, "gi")
  const matches = plan.match(regex) || []
  return matches.map(m => {
    const fileMatch = m.match(/(\S+\.(?:ts|js|tsx|jsx))/)
    return fileMatch ? fileMatch[1] : ""
  }).filter(Boolean)
}

const migrationWorkflow: Workflow = async (ctx) => {
  const { spawn, run, parallel, checkpoint, validate } = ctx

  // Phase 1: Analysis
  const analyzerId = await spawn("analyzer", "You analyze codebases for migration planning.")
  const analysis = await run(analyzerId,
    "Analyze this codebase. List all files that use the old API pattern. Categorize by complexity: trivial, moderate, complex."
  )
  await checkpoint(1, { analysis })

  // Phase 2: Plan generation
  const plannerId = await spawn("planner", "You create detailed migration plans.")
  const plan = await run(plannerId,
    `Create a migration plan based on this analysis. For each file, specify exact changes needed:\n\n${analysis}`
  )
  await checkpoint(2, { plan })

  // Phase 3: Parallel migration (trivial files first)
  const trivialFiles = extractFiles(plan, "trivial")
  const migratedTrivial = await parallel(trivialFiles, async (file) => {
    const agentId = await spawn(`migrator-${file}`, "You apply migration changes precisely.")
    const result = await run(agentId, `Migrate ${file} according to this plan:\n${plan}`)
    const tests = await validate(agentId, "npm test -- --run")
    return { file, result, tests }
  }, 8)
  await checkpoint(3, { migratedTrivial })

  // Phase 4: Handle failures
  const failed = migratedTrivial.filter(m => !m.tests)
  if (failed.length > 0) {
    const fixerId = await spawn("fixer", "You fix migration failures.")
    for (const { file } of failed) {
      await run(fixerId, `The migration of ${file} failed tests. Fix it while preserving the migration intent.`)
    }
  }

  // Phase 5: Final verification
  const finalTests = await validate(analyzerId, "npm run test:ci")
  console.log(finalTests ? "Migration successful!" : "Migration completed with test failures.")
}
```

### Example: Deep Research Workflow

```typescript
const deepResearchWorkflow: Workflow = async (ctx) => {
  const { spawn, run, parallel, synthesize, checkpoint } = ctx

  // Step 1: Generate sub-questions
  const plannerId = await spawn("planner", "You break research topics into sub-questions.")
  const subQuestionsRaw = await run(plannerId,
    `Break this research topic into 5-8 specific sub-questions that, when answered, fully cover the topic. Format as a numbered list.`
  )
  const subQuestions = subQuestionsRaw.split("\n").filter(q => q.match(/^\d+\./))
  await checkpoint(1, { subQuestions })

  // Step 2: Parallel research on each sub-question
  const findings = await parallel(subQuestions, async (question, i) => {
    const researcherId = await spawn(`researcher-${i}`,
      "You are a thorough researcher. Search for evidence, cite sources, and note confidence levels."
    )
    return run(researcherId, `Research this question thoroughly: ${question}`)
  }, 5)
  await checkpoint(2, { findings })

  // Step 3: Cross-check for contradictions
  const checkerId = await spawn("checker", "You identify contradictions across research findings.")
  const contradictions = await run(checkerId,
    `Identify any contradictions or inconsistencies across these findings. For each, note which researchers disagree and why:\n\n${findings.join("\n\n---\n\n")}`
  )
  await checkpoint(3, { contradictions })

  // Step 4: Final synthesis with citations
  const synthesis = await synthesize(findings,
    "Synthesize these research findings into a comprehensive report. Resolve contradictions where possible. Include a confidence assessment for each major claim."
  )

  console.log(synthesis)
}
```

---

## Part 7: Monitoring & Observability

### Structured Logging Across Agents

Use `client.app.log()` to emit structured events that a monitoring system can consume:

```typescript
interface WorkflowEvent {
  service: "orchestrator"
  level: "debug" | "info" | "warn" | "error"
  message: string
  extra: {
    workflowId: string
    step?: number
    agentId?: string
    sessionId?: string
    durationMs?: number
    tokenCount?: number
    event: "agent_spawned" | "agent_started" | "agent_completed" | "agent_failed" | "checkpoint_saved" | "workflow_completed" | "workflow_failed"
  }
}

async function logEvent(client: any, event: WorkflowEvent): Promise<void> {
  await client.app.log({ body: event })
}
```

### Real-Time TUI Dashboard Plugin

A plugin that shows workflow progress in OpenCode's TUI:

```typescript
import { Plugin } from "@opencode-ai/plugin"
import { Database } from "bun:sqlite"

export const WorkflowDashboardPlugin: Plugin = async ({ client }) => {
  const db = new Database("~/.opencode/workflows.db")

  return {
    "session.idle": async ({ event }) => {
      // Check if this session belongs to a workflow
      const agent = db.query(
        "SELECT workflow_id, task FROM agents WHERE session_id = ?"
      ).get(event.properties.sessionId) as any

      if (agent) {
        const workflow = db.query(
          "SELECT * FROM workflows WHERE id = ?"
        ).get(agent.workflow_id) as any

        const completed = db.query(
          "SELECT COUNT(*) as count FROM agents WHERE workflow_id = ? AND status = 'completed'"
        ).get(agent.workflow_id) as any

        const total = db.query(
          "SELECT COUNT(*) as count FROM agents WHERE workflow_id = ?"
        ).get(agent.workflow_id) as any

        await client.tui.showToast({
          body: {
            message: `Workflow ${workflow.id}: ${completed.count}/${total.count} agents complete`,
            variant: "default",
          },
        })
      }
    },
  }
}
```

### Progress Tracking with Checkpoints

```typescript
interface ProgressTracker {
  totalSteps: number
  completedSteps: number
  agentStates: Map<string, "pending" | "running" | "completed" | "failed">
}

function renderProgress(tracker: ProgressTracker): string {
  const pct = Math.round((tracker.completedSteps / tracker.totalSteps) * 100)
  const bar = "█".repeat(pct / 5) + "░".repeat(20 - pct / 5)
  const agents = Array.from(tracker.agentStates.entries())
    .map(([id, status]) => `${id}: ${status}`)
    .join(", ")
  return `[${bar}] ${pct}% | Agents: ${agents}`
}
```

---

## Part 8: Putting It All Together

### Complete Orchestrator Class

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"
import { Database } from "bun:sqlite"
import { randomUUID } from "node:crypto"

export class OpenCodeOrchestrator {
  private client: any
  private store: WorkflowStore
  private activeWorkflows = new Map<string, AbortController>()

  constructor(baseUrl: string) {
    this.client = createOpencodeClient({ baseUrl })
    this.store = new WorkflowStore()
  }

  async runWorkflow(
    workflow: Workflow,
    options: { resumeFrom?: string; onProgress?: (msg: string) => void } = {}
  ): Promise<string> {
    const workflowId = options.resumeFrom || randomUUID()
    const abortController = new AbortController()
    this.activeWorkflows.set(workflowId, abortController)

    if (!options.resumeFrom) {
      this.store.createWorkflow(workflowId)
    }

    const ctx: WorkflowContext = {
      workflowId,
      client: this.client,
      store: this.store,
      checkpoint: async (step, extra = {}) => {
        const cp = this.store.getCheckpoint(workflowId)
        this.store.updateCheckpoint(workflowId, { ...cp, step, ...extra })
        options.onProgress?.(`Step ${step} complete`)
      },
      spawn: async (title, context) => {
        const session = await this.client.session.create({ body: { title } })
        await this.client.session.init({ path: { id: session.id } })
        if (context) {
          await this.client.session.prompt({
            path: { id: session.id },
            body: { noReply: true, parts: [{ type: "text", text: context }] },
          })
        }
        return session.id
      },
      run: async (sessionId, task) => {
        const result = await this.client.session.prompt({
          path: { id: sessionId },
          body: { parts: [{ type: "text", text: task }] },
        })
        return result.data.parts
          ?.filter((p: any) => p.type === "text")
          .map((p: any) => p.text)
          .join("\n") || ""
      },
      parallel: async (items, fn, concurrency = 5) => {
        const results: string[] = new Array(items.length)
        const executing: Promise<void>[] = []
        for (let i = 0; i < items.length; i++) {
          if (abortController.signal.aborted) throw new Error("Workflow aborted")
          const p = fn(items[i], i).then(r => {
            results[i] = r
            const idx = executing.indexOf(p)
            if (idx !== -1) executing.splice(idx, 1)
          })
          executing.push(p)
          if (executing.length >= concurrency) {
            await Promise.race(executing)
          }
        }
        await Promise.all(executing)
        return results
      },
      synthesize: async (results, prompt) => {
        const synthId = await ctx.spawn("synthesizer", "You are a synthesis expert.")
        const fullPrompt = [prompt, "", ...results.map((r, i) => `--- Input ${i + 1} ---\n${r}\n`)].join("\n")
        return ctx.run(synthId, fullPrompt)
      },
      vote: async (question, options) => {
        const voters = await Promise.all(
          Array.from({ length: 3 }, () => ctx.spawn("voter", "You are an independent evaluator."))
        )
        const votes = await Promise.all(
          voters.map(id =>
            ctx.run(id, `Question: ${question}\nOptions: ${options.join(", ")}\nAnswer with ONLY the option name.`)
          )
        )
        const counts: Record<string, number> = {}
        for (const vote of votes) {
          for (const opt of options) {
            if (vote.toLowerCase().includes(opt.toLowerCase())) {
              counts[opt] = (counts[opt] || 0) + 1
              break
            }
          }
        }
        const winner = Object.entries(counts).sort((a, b) => b[1] - a[1])[0]
        return winner?.[0] || options[0]
      },
      validate: async (sessionId, testCmd) => {
        const result = await this.client.session.shell({
          path: { id: sessionId },
          body: { command: testCmd },
        })
        const output = result.data.parts
          ?.filter((p: any) => p.type === "text")
          .map((p: any) => p.text)
          .join("\n") || ""
        return output.includes("PASS") || !output.includes("FAIL")
      },
    }

    try {
      await workflow(ctx)
      this.store.db.run("UPDATE workflows SET status = ? WHERE id = ?", ["completed", workflowId])
      return workflowId
    } catch (err) {
      this.store.db.run("UPDATE workflows SET status = ? WHERE id = ?", ["failed", workflowId])
      throw err
    } finally {
      this.activeWorkflows.delete(workflowId)
    }
  }

  abortWorkflow(workflowId: string): void {
    const controller = this.activeWorkflows.get(workflowId)
    if (controller) {
      controller.abort()
      this.activeWorkflows.delete(workflowId)
    }
  }

  async resumeWorkflow(workflowId: string, workflow: Workflow): Promise<string> {
    return this.runWorkflow(workflow, { resumeFrom: workflowId })
  }

  destroy(): void {
    this.store.close()
  }
}
```

### Usage

```typescript
const orchestrator = new OpenCodeOrchestrator("http://localhost:4096")

const workflowId = await orchestrator.runWorkflow(securityAuditWorkflow, {
  onProgress: (msg) => console.log(msg),
})

console.log(`Workflow completed: ${workflowId}`)

orchestrator.destroy()
```

---

## Best Practices for Large-Scale Orchestration

1. **Always checkpoint before parallel execution** — if the workflow crashes during fan-out, you need to know which agents were spawned
2. **Use structured output for agent results** — JSON Schema ensures you can programmatically process agent outputs
3. **Set timeouts on every agent** — hung sessions waste tokens and block workflows
4. **Clean up sessions aggressively** — each idle session consumes resources; delete them as soon as you extract results
5. **Log everything** — `client.app.log()` is your observability backbone
6. **Model selection per task** — use fast/cheap models for simple tasks, powerful models for synthesis and review
7. **Concurrency limits** — OpenCode's server has limits; don't spawn 100 agents simultaneously. Start with 3-5 concurrent sessions
8. **Handle `session.error`** — agents can crash; have fallback logic
9. **Rate limit your orchestrator** — API providers have rate limits; pace your requests
10. **Test workflows on small inputs first** — a 10-file migration should work before you try 1000 files
