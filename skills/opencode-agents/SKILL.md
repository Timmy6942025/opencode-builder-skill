---
name: opencode-agents
description: |
  Complete guide to OpenCode's agent system. Use this skill when configuring, creating, or managing OpenCode agents — including primary agents (Build, Plan), subagents (General, Explore, Scout), hidden agents (Compaction, Title, Summary), custom agents via JSON config or Markdown files, agent permissions (allow/ask/deny with glob patterns), task permissions for subagent invocation, session navigation keybinds, multi-agent workflows, and agent creation via CLI. Covers all agent options: description, mode, model, temperature, steps, disable, prompt, tools (deprecated), permission (with object syntax for bash commands), color, top_p, hidden, and provider-specific pass-through options.
---

# OpenCode Agents — Complete Reference

OpenCode supports a multi-agent architecture where specialized AI assistants handle different aspects of development. Agents come in two types — **primary agents** (direct user interaction, switchable via Tab) and **subagents** (invoked by other agents or via `@mention`/Task tool). There are also **hidden agents** (internal-use only, not user-selectable).

This guide covers every agent type, all built-in agents, every configuration option, the complete permission system, and how to create custom agents.

---

## 1. Agent Types

### Primary Agents

Primary agents are the main assistants you interact with directly. They appear in the agent selector and can be cycled through using the **Tab** key (or your configured `switch_agent` keybind). Each primary agent has its own system prompt, tool permissions, temperature, and behavioral configuration.

OpenCode ships with two built-in primary agents: **Build** and **Plan**.

Primary agents are suitable for:
- Direct user conversation and coding
- Multi-step development tasks
- Planning and architecture analysis
- Read-only exploration

You can switch between primary agents mid-session by pressing **Tab**.

### Subagents

Subagents are specialized assistants that primary agents can invoke for specific tasks. They run in isolated contexts and return results to the invoking agent. You can invoke them:

- **Automatically** — the primary agent decides when to delegate based on subagent descriptions
- **Manually** — by `@mentioning` them in your message (e.g., `@explore find all auth functions`)
- **Programmatically** — via the Task tool from another agent's configuration

OpenCode ships with three built-in subagents: **General**, **Explore**, and **Scout**.

### Hidden Agents

Hidden agents are internal system agents not visible in the UI or `@` autocomplete. They handle behind-the-scenes operations and run automatically when triggered by system events. You cannot select them directly, but they perform critical maintenance functions.

OpenCode ships with three hidden agents: **Compaction**, **Title**, and **Summary**.

---

## 2. Built-in Agents (Complete Reference)

### Build — Primary Agent

| Property | Value |
|----------|-------|
| **Mode** | `primary` |
| **Tools** | All enabled |
| **Default temperature** | Model default (typically 0) |
| **Use case** | General development, file creation, editing, debugging, building features |

Build is the **default** primary agent with all tools enabled. This is the standard agent for development work where you need full access to file operations, system commands, and all available tools. It is the agent you will use for most coding tasks.

**What it can do:**
- Read, write, and edit any file in the project
- Run any shell command
- Search codebases with glob and grep
- Fetch external URLs and web pages
- Invoke subagents via the Task tool
- Load and use skills
- Ask the user questions

**Configuration (built-in defaults):**
```json
{
  "agent": {
    "build": {
      "description": "Default build agent with all tools",
      "mode": "primary"
    }
  }
}
```

### Plan — Primary Agent

| Property | Value |
|----------|-------|
| **Mode** | `primary` |
| **Tools** | All read tools enabled; edit and bash set to `ask` |
| **Default temperature** | Model default (typically 0) |
| **Use case** | Code analysis, planning, architecture review, reconnaissance |

A restricted agent designed for planning and analysis. By default, all file edits and bash commands are set to `ask` mode, meaning they require user confirmation before execution. This agent excels at reading code, analyzing architecture, and producing plans without making unintended changes.

**What it can do:**
- Read any file in the project
- Search with glob and grep
- Suggest changes (but requires approval to apply them)
- Run bash commands (requires approval)
- Fetch external documentation

**What it cannot do without approval:**
- Edit or write files
- Execute bash commands

**Configuration (built-in defaults):**
```json
{
  "agent": {
    "plan": {
      "description": "Restricted agent for planning and analysis",
      "mode": "primary",
      "permission": {
        "edit": "ask",
        "bash": "ask"
      }
    }
  }
}
```

### General — Subagent

| Property | Value |
|----------|-------|
| **Mode** | `subagent` |
| **Tools** | All enabled except todo |
| **Default temperature** | Model default |
| **Use case** | Complex delegated tasks, multi-file refactors, bulk operations |

A general-purpose agent for researching complex questions and executing multi-step tasks. Has full tool access (except todo), so it can make file changes when needed. Use this to run multiple units of work in parallel.

**What it can do:**
- Read, write, and edit files
- Run shell commands
- Search codebases
- Fetch external URLs
- Load skills
- Ask the user questions

**What it cannot do:**
- Write to the todo list

**Invocation:**
```
@general help me refactor the authentication module
```

### Explore — Subagent

| Property | Value |
|----------|-------|
| **Mode** | `subagent` |
| **Tools** | read, grep, glob, webfetch (no edit, write, or bash) |
| **Default temperature** | Model default |
| **Use case** | Fast codebase exploration, finding definitions, understanding architecture |

A fast, read-only agent for exploring codebases. Cannot modify files. Use this when you need to quickly find files by patterns, search code for keywords, or answer questions about the codebase without any risk of changes.

**What it can do:**
- Read files (with line ranges)
- Search with glob patterns (`**/*.ts`, etc.)
- Search content with grep (regex)
- Fetch external URLs for documentation

**What it cannot do:**
- Write or edit any files
- Run shell commands
- Load skills

**Invocation:**
```
@explore find all TypeScript files that handle user authentication
```

### Scout — Subagent

| Property | Value |
|----------|-------|
| **Mode** | `subagent` |
| **Tools** | webfetch, read, grep, glob (no edit, write, or bash) |
| **Default temperature** | Model default |
| **Use case** | External documentation, dependency research, cross-referencing upstream code |

A read-only agent for external docs and dependency research. Use this when you need to clone a dependency repository into OpenCode's managed cache, inspect library source code, or cross-reference local code against upstream implementations without modifying your workspace.

**What it can do:**
- Fetch and read external URLs (documentation, GitHub repos, API references)
- Read files in OpenCode's managed cache
- Search cached content with grep and glob

**What it cannot do:**
- Write or edit files in the project
- Run shell commands
- Load skills

**Invocation:**
```
@scout look up the latest Stripe API docs for payment intents
```

### Compaction — Hidden Primary Agent

| Property | Value |
|----------|-------|
| **Mode** | `primary` (hidden) |
| **Tools** | System-managed |
| **Use case** | Automatic context window compaction |

Handles context window compaction when conversations grow too long. Automatically summarizes older messages to preserve context while staying within token limits. Runs automatically when the context window approaches its limit. Not selectable in the UI.

**Trigger:** Automatic when context window is near capacity.

**Behavior:** Compacts the conversation history by summarizing older messages into a condensed form, preserving the most important context for continued work.

### Title — Hidden Primary Agent

| Property | Value |
|----------|-------|
| **Mode** | `primary` (hidden) |
| **Tools** | System-managed |
| **Use case** | Automatic session title generation |

Generates short session titles based on conversation content. Runs automatically to create descriptive names for chat sessions, making it easier to identify and navigate between sessions. Not selectable in the UI.

**Trigger:** Automatic when a new session starts or after initial messages.

**Behavior:** Analyzes the conversation and produces a concise title (typically 3-7 words) that captures the session's purpose.

### Summary — Hidden Primary Agent

| Property | Value |
|----------|-------|
| **Mode** | `primary` (hidden) |
| **Tools** | System-managed |
| **Use case** | Session persistence and resumption |

Creates session summaries for persistence and retrieval. Used when saving or loading session state. Not selectable in the UI.

**Trigger:** Automatic during session save/persistence operations.

**Behavior:** Generates a structured summary of the session's key points, decisions, and outcomes for later retrieval.

---

## 3. Configuration Methods

Agents can be configured through two methods: JSON configuration in `opencode.json` or Markdown files in agent directories.

### JSON Configuration

Define agents in the `agent` field of `opencode.json`. This method is best for simple agent definitions and quick overrides of built-in agents.

**Project-level config** (`.opencode/config.json`):
```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "build": {
      "description": "Custom build agent",
      "mode": "primary",
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.7,
      "steps": 100
    },
    "plan": {
      "description": "Custom plan agent",
      "mode": "primary",
      "model": "anthropic/claude-haiku-4-20250514",
      "permission": {
        "edit": "ask",
        "bash": "ask"
      }
    }
  }
}
```

**Global config** (`~/.config/opencode/config.json`):
```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "build": {
      "model": "openai/gpt-5"
    }
  }
}
```

### Markdown Files

Create agent definitions as Markdown files in one of these directories:

- **Global:** `~/.config/opencode/agents/` — available in all projects
- **Per-project:** `.opencode/agents/` — available only in the current project

Each Markdown file defines a single agent. The **filename (without extension)** becomes the agent name. For example, `review.md` creates a `review` agent.

**Markdown file structure:**
```markdown
---
description: What this agent does and when to use it
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.1
steps: 50
permission:
  read: "allow"
  edit: "ask"
  bash: "ask"
  glob: "allow"
  grep: "allow"
  webfetch: "allow"
  task: "allow"
color: "#ef4444"
hidden: false
---

You are a specialized agent. Your system prompt goes here.

Instructions and behavioral guidelines for the agent.
```

**Example: `.opencode/agents/security.md`**
```markdown
---
description: Security-focused agent that audits code for vulnerabilities
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.2
steps: 50
permission:
  edit: "deny"
  bash: "ask"
  read: "allow"
  webfetch: "allow"
  glob: "allow"
  grep: "allow"
color: "#ef4444"
---

You are a security auditor. Your role is to:

1. Scan code for common vulnerabilities (SQL injection, XSS, CSRF, etc.)
2. Check for hardcoded secrets and credentials
3. Review authentication and authorization patterns
4. Assess dependency security
5. Report findings with severity ratings

Never modify files. Read-only analysis only.
```

### CLI Creation

Use the `opencode agent create` command to create a new agent interactively:

```bash
opencode agent create
```

This launches an interactive prompt that:
1. Asks where to save the agent (global or project-specific)
2. Asks for a description of what the agent should do
3. Generates an appropriate system prompt and identifier
4. Lets you select which permissions the agent should be allowed (anything not selected is denied)
5. Creates a Markdown file with the agent configuration

The file is saved to either `~/.config/opencode/agents/` (global) or `.opencode/agents/` (project).

---

## 4. All Agent Options (Complete Reference)

### description (required)

A plain-text description of what the agent does and when to use it. This appears in the agent selector UI and is used by the system to understand the agent's purpose (including for automatic subagent selection).

```json
{
  "description": "Reviews code for best practices and potential issues"
}
```

```json
{
  "description": "Database administration agent for migrations and queries"
}
```

**In Markdown files:**
```yaml
---
description: Writes and maintains project documentation
---
```

### mode

Controls how the agent is accessed. Values:

| Value | Behavior |
|-------|----------|
| `"primary"` | Appears in Tab cycle, directly selectable by user |
| `"subagent"` | Invoked via `@mention` or Task tool |
| `"all"` | Available both as primary and subagent (default) |

```json
{
  "mode": "subagent"
}
```

```json
{
  "mode": "primary"
}
```

If no `mode` is specified, it defaults to `"all"`.

### model

Override the model used by this agent. Format: `provider/model-id`.

If you don't specify a model:
- **Primary agents** use the globally configured model
- **Subagents** inherit the model of the primary agent that invoked them

```json
{
  "model": "anthropic/claude-sonnet-4-20250514"
}
```

```json
{
  "model": "openai/gpt-5"
}
```

```json
{
  "model": "opencode/gpt-5.1-codex"
}
```

Use `opencode models` to see available models.

### temperature

Controls randomness and creativity of the LLM's responses. Range: `0.0` to `1.0`.

| Range | Behavior | Best for |
|-------|----------|----------|
| `0.0` - `0.2` | Very focused, deterministic | Code analysis, planning, debugging |
| `0.3` - `0.5` | Balanced with some creativity | General development tasks |
| `0.6` - `1.0` | Creative and varied | Brainstorming, exploration, creative writing |

```json
{
  "temperature": 0.3
}
```

If no temperature is specified, OpenCode uses model-specific defaults (typically `0` for most models, `0.55` for Qwen models).

```json
{
  "agent": {
    "analyze": { "temperature": 0.1 },
    "build": { "temperature": 0.3 },
    "brainstorm": { "temperature": 0.7 }
  }
}
```

### steps

Maximum number of agentic iterations (tool calls) before the agent is forced to produce a text-only response. Prevents infinite loops and controls task scope and cost.

```json
{
  "steps": 100
}
```

When the limit is reached, the agent receives a special system prompt instructing it to respond with a summarization of its work and recommended remaining tasks.

If not set, the agent continues to iterate until the model chooses to stop or the user interrupts.

> **Note:** The legacy `maxSteps` field is deprecated. Use `steps` instead.

### disable

Set to `true` to completely disable an agent. The agent will not appear anywhere in the UI.

```json
{
  "disable": true
}
```

### prompt

Path to a custom system prompt file. Supports the `{file:...}` syntax to reference files relative to the config file location (works for both global and project config).

```json
{
  "prompt": "{file:./prompts/code-review.txt}"
}
```

```json
{
  "prompt": "{file:.opencode/prompts/db-migrator.md}"
}
```

In **Markdown agent files**, the body content after the frontmatter becomes the system prompt directly — no separate file needed.

### tools (deprecated)

> **Deprecated:** Use `permission` instead. The `tools` config is still supported for backwards compatibility but may be removed in a future version.

Enable or disable specific tools. `true` is equivalent to `{"*": "allow"}` permission; `false` is equivalent to `{"*": "deny"}`.

```json
{
  "tools": {
    "write": true,
    "bash": true,
    "webfetch": false
  }
}
```

Agent-specific `tools` override the global config. Supports wildcards:

```json
{
  "agent": {
    "readonly": {
      "tools": {
        "mymcp_*": false,
        "write": false,
        "edit": false
      }
    }
  }
}
```

### permission

Granular permission rules per agent. Controls which tools the agent can use and whether they require confirmation. This is the recommended way to configure agent capabilities.

```json
{
  "permission": {
    "read": "allow",
    "edit": "allow",
    "bash": "ask",
    "glob": "allow",
    "grep": "allow",
    "webfetch": "ask",
    "task": "allow",
    "skill": "allow",
    "question": "allow"
  }
}
```

Each permission key accepts either:
- **Shorthand:** `"allow"`, `"ask"`, or `"deny"`
- **Object syntax:** Glob patterns mapping to actions (for fine-grained control)

See [Section 5: Permission System](#5-permission-system-complete-reference) for the full reference.

### hidden

Hide a subagent from the `@` autocomplete menu. The agent can still be invoked by the model via the Task tool if permissions allow, but it won't appear in the user's autocomplete list.

```json
{
  "hidden": true
}
```

> Only applies to `mode: subagent` agents.

### color

Customize the agent's visual appearance in the UI. Accepts a hex color or a theme color name.

**Hex colors:**
```json
{
  "color": "#3b82f6"
}
```

**Theme colors:** `primary`, `secondary`, `accent`, `success`, `warning`, `error`, `info`

```json
{
  "color": "accent"
}
```

### top_p

Alternative to temperature for controlling randomness. Range: `0.0` to `1.0`. Lower values are more focused; higher values are more diverse.

```json
{
  "top_p": 0.9
}
```

### Additional provider-specific options

Any options not recognized by OpenCode are **passed through directly** to the model provider. This allows you to use provider-specific features.

**OpenAI reasoning models:**
```json
{
  "agent": {
    "deep-thinker": {
      "description": "Agent that uses high reasoning effort for complex problems",
      "model": "openai/gpt-5",
      "reasoningEffort": "high",
      "textVerbosity": "low"
    }
  }
}
```

Other common provider-specific options:
- `maxTokens` — Maximum output tokens
- `stopSequences` — Stop generation at specific tokens
- `topK` — Top-K sampling

Check your provider's documentation for available parameters.

---

## 5. Permission System — Complete Reference

The permission system controls what each agent can do. Permissions can be set globally in `opencode.json` or per-agent in agent configurations. Agent permissions merge with global config, with agent rules taking precedence.

### Permission Actions

| Action | Behavior |
|--------|----------|
| `"allow"` | Tool executes without confirmation |
| `"ask"` | User is prompted before execution (options: `once`, `always`, `reject`) |
| `"deny"` | Tool is blocked entirely |

### Global Permissions

Set default permissions for all agents:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "read": "allow",
    "edit": "allow",
    "bash": "ask",
    "glob": "allow",
    "grep": "allow",
    "webfetch": "ask",
    "task": "allow",
    "skill": "allow"
  }
}
```

Or set all permissions at once:
```json
{
  "permission": "allow"
}
```

### Per-Agent Permissions

Override global permissions for specific agents. Agent rules take precedence over global rules.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "edit": "deny"
  },
  "agent": {
    "build": {
      "permission": {
        "edit": "ask"
      }
    },
    "explorer": {
      "permission": {
        "read": "allow",
        "glob": "allow",
        "grep": "allow",
        "edit": "deny",
        "write": "deny",
        "bash": "deny"
      }
    }
  }
}
```

### Permission Keys — Complete Table

| Key | Tools it gates | Accepts object syntax |
|-----|---------------|----------------------|
| `read` | `read` | Yes (matches file path) |
| `edit` | `write`, `edit`, `apply_patch` | Yes (matches file path) |
| `glob` | `glob` | Yes (matches glob pattern) |
| `grep` | `grep` | Yes (matches regex pattern) |
| `list` | `list` | Yes |
| `bash` | `bash` | Yes (matches parsed commands) |
| `task` | `task` | Yes (matches subagent type) |
| `skill` | `skill` | Yes (matches skill name) |
| `external_directory` | Any tool touching paths outside the project worktree | Yes (matches directory path) |
| `todowrite` | `todowrite`, `todoread` | Shorthand only |
| `webfetch` | `webfetch` | Yes (matches URL) |
| `websearch` | `websearch` | Yes (matches query) |
| `lsp` | `lsp` | Shorthand only |
| `question` | `question` | Shorthand only |
| `doom_loop` | Recovery prompts when an agent appears stuck (same tool call repeated 3x with identical input) | Shorthand only |

### Defaults

If you don't specify anything, OpenCode uses permissive defaults:

- Most permissions default to `"allow"`
- `doom_loop` and `external_directory` default to `"ask"`
- `read` is `"allow"`, but `.env` files are denied by default:

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

### Granular Rules (Object Syntax)

For most permissions, you can use an object to apply different actions based on the tool input. Rules are evaluated by pattern match, with the **last matching rule winning**.

**Basic pattern:**
```json
{
  "permission": {
    "bash": {
      "*": "ask",
      "git status": "allow",
      "git diff": "allow",
      "npm test": "allow",
      "rm *": "deny",
      "sudo *": "deny"
    }
  }
}
```

**With catch-all first, specific rules after (recommended pattern):**
```json
{
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
```

**File-level granularity for edit:**
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

### Wildcards

Permission patterns use simple wildcard matching:

| Pattern | Matches |
|---------|---------|
| `*` | Zero or more of any character |
| `?` | Exactly one character |
| All other characters | Match literally |

Examples:
- `"git *"` — matches `git status`, `git diff --staged`, `git push origin main`
- `"rm ?"` — matches `rm f` but not `rm -rf`
- `"npm test"` — matches exactly `npm test`

### Home Directory Expansion

You can use `~` or `$HOME` at the start of a pattern to reference your home directory:

- `~/projects/*` → `/Users/username/projects/*`
- `$HOME/projects/*` → `/Users/username/projects/*`
- `~` → `/Users/username`

### External Directory Permissions

Use `external_directory` to allow tool calls that touch paths outside the working directory where OpenCode was started. This applies to any tool that takes a path as input (e.g., `read`, `edit`, `glob`, `grep`, and many `bash` commands).

**Allow access to a specific external directory:**
```json
{
  "permission": {
    "external_directory": {
      "~/projects/personal/**": "allow"
    }
  }
}
```

**Allow reads but deny edits in external directories:**
```json
{
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

Any directory allowed via `external_directory` inherits the same defaults as the current workspace. Keep the list focused on trusted paths, and layer extra allow or deny rules as needed.

### Permission in Markdown Agents

Define permissions in the YAML frontmatter of Markdown agent files:

```markdown
---
description: Code review agent
permission:
  read: "allow"
  edit: "deny"
  bash:
    "*": "ask"
    "git diff": "allow"
    "git log": "allow"
    "git log *": "allow"
    "grep *": "allow"
  webfetch: "deny"
---
```

### What "Ask" Does

When OpenCode prompts for approval, the UI offers three outcomes:

| Option | Behavior |
|--------|----------|
| `once` | Approve just this request |
| `always` | Approve future requests matching the suggested patterns (for the rest of the current session) |
| `reject` | Deny the request |

The set of patterns that `always` would approve is provided by the tool (e.g., bash approvals typically whitelist a safe command prefix like `git status*`).

### Pattern Matching for Commands

Use pattern matching for commands with arguments:

- `"grep *"` — allows `grep pattern file.txt`
- `"grep"` alone — would block `grep pattern file.txt` (no arguments match)
- `"git status"` — works for the exact command `git status`
- `"git status *"` — required when arguments are passed

### MCP Tool Permissions

Permission keys are matched as wildcard patterns against the underlying tool name, so the same syntax works for built-ins, custom tools, and MCP tools:

```json
{
  "permission": {
    "bash": {
      "mymcp_*": "deny",
      "mymcp_search": "ask"
    }
  }
}
```

---

## 6. Task Permissions — Controlling Subagent Invocation

Control which subagents an agent can invoke via the Task tool with `permission.task`. Uses glob patterns for flexible matching.

### Basic Configuration

```json
{
  "agent": {
    "orchestrator": {
      "mode": "primary",
      "permission": {
        "task": {
          "*": "deny",
          "orchestrator-*": "allow",
          "code-reviewer": "ask"
        }
      }
    }
  }
}
```

### Pattern Matching Rules

- Rules are evaluated in order
- **Last matching rule wins**
- When set to `deny`, the subagent is removed from the Task tool description entirely, so the model won't attempt to invoke it

### User Override

Users can always invoke any subagent directly via the `@` autocomplete menu, even if the agent's task permissions would deny it. Task permissions only control what the **model** can invoke programmatically.

### Example: Restrictive Orchestrator

```json
{
  "agent": {
    "orchestrator": {
      "mode": "primary",
      "permission": {
        "task": {
          "*": "deny",
          "explore": "allow",
          "scout": "allow",
          "general": "ask",
          "review": "allow",
          "security": "allow"
        }
      }
    }
  }
}
```

This allows the orchestrator to invoke Explore, Scout, Review, and Security subagents freely, requires confirmation for General, and denies all others.

---

## 7. Navigation — Session Keybinds

When subagents create child sessions, OpenCode provides keyboard shortcuts for navigating between them.

### Default Keybinds

| Action | Default Keybind | Description |
|--------|----------------|-------------|
| `session_child_first` | `<Leader> + Down` | Switch to the first child session from the parent |
| `session_child_cycle` | `Right` | Cycle to the next child session |
| `session_child_cycle_reverse` | `Left` | Cycle to the previous child session |
| `session_parent` | `Up` | Return to the parent session |

### Navigation Flow

```
Parent Session (Build)
  ├── Child Session 1 (@explore — investigate auth)
  ├── Child Session 2 (@explore — investigate payments)
  └── Child Session 3 (@scout — Stripe API docs)
```

1. From parent, press `<Leader> + Down` → enters Child Session 1
2. Press `Right` → cycles to Child Session 2
3. Press `Right` → cycles to Child Session 3
4. Press `Left` → cycles back to Child Session 2
5. Press `Up` → returns to Parent Session

### Configuring Custom Keybinds

Keybinds can be customized in your OpenCode config. See the [Keybinds documentation](https://opencode.ai/docs/keybinds/) for details on remapping these actions.

---

## 8. Use Cases

### Build Agent — Full Development Work

The default agent for day-to-day coding with all tools enabled.

```json
{
  "agent": {
    "build": {
      "description": "Primary coding agent with full tool access",
      "mode": "primary",
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.7,
      "steps": 100,
      "permission": {
        "read": "allow",
        "write": "allow",
        "edit": "allow",
        "bash": "allow",
        "glob": "allow",
        "grep": "allow"
      }
    }
  }
}
```

### Plan Agent — Analysis Without Changes

A restricted agent for planning and analysis:

```json
{
  "agent": {
    "plan": {
      "description": "Read-only planning and analysis",
      "mode": "primary",
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.3,
      "steps": 50,
      "permission": {
        "read": "allow",
        "glob": "allow",
        "grep": "allow",
        "edit": "ask",
        "bash": "ask",
        "write": "deny"
      }
    }
  }
}
```

### Review Agent — Code Review

An agent specialized for code review with read-only access:

```json
{
  "agent": {
    "review": {
      "description": "Code review and quality analysis",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.2,
      "steps": 30,
      "permission": {
        "read": "allow",
        "glob": "allow",
        "grep": "allow",
        "bash": {
          "git diff": "allow",
          "git diff *": "allow",
          "git log": "allow",
          "git log *": "allow",
          "*": "deny"
        },
        "edit": "deny",
        "write": "deny"
      }
    }
  }
}
```

### Debug Agent — Investigation

An agent for debugging with read access and approval for changes:

```json
{
  "agent": {
    "debug": {
      "description": "Debugging agent that reads and analyzes but asks before editing",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.4,
      "steps": 60,
      "permission": {
        "read": "allow",
        "glob": "allow",
        "grep": "allow",
        "bash": "ask",
        "edit": "ask",
        "write": "ask"
      }
    }
  }
}
```

### Documentation Agent — Docs Writing

An agent for writing and updating documentation:

```json
{
  "agent": {
    "docs": {
      "description": "Documentation writer with read access and write to markdown files only",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.6,
      "steps": 40,
      "permission": {
        "read": "allow",
        "glob": "allow",
        "grep": "allow",
        "write": "allow",
        "edit": "allow",
        "bash": {
          "npm run docs": "allow",
          "npx typedoc": "allow",
          "*": "ask"
        },
        "webfetch": "allow",
        "skill": "allow"
      }
    }
  }
}
```

---

## 9. Creating Agents

### Via CLI (Recommended for Quick Setup)

```bash
opencode agent create
```

Interactive workflow:
1. Select save location (global or project)
2. Enter agent description
3. System generates a system prompt and identifier
4. Select allowed permissions (unselected = denied)
5. Markdown file is created automatically

### Via Markdown File (Recommended for Complex Agents)

```bash
mkdir -p .opencode/agents
```

Create a `.md` file — the filename becomes the agent name:

**`.opencode/agents/db-admin.md`:**
```markdown
---
description: Database administration agent for migrations and queries
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.3
steps: 50
permission:
  read: "allow"
  write: "ask"
  edit: "ask"
  bash:
    "psql *": "allow"
    "pg_dump *": "allow"
    "rm *": "deny"
    "*": "ask"
  glob: "allow"
  grep: "allow"
  webfetch: "allow"
color: "#8b5cf6"
---

You are a database administration agent. Your responsibilities:

1. Write and review SQL migrations
2. Analyze database schemas
3. Optimize queries
4. Handle backup and restore operations

Always confirm before executing destructive operations.
Never drop tables or databases without explicit user approval.
```

### Via JSON Config (Recommended for Simple Overrides)

**`.opencode/config.json`:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "db-admin": {
      "description": "Database administration agent",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.3,
      "steps": 50,
      "permission": {
        "read": "allow",
        "bash": {
          "psql *": "allow",
          "pg_dump *": "allow",
          "rm *": "deny",
          "*": "ask"
        }
      }
    }
  }
}
```

---

## 10. Complete Examples

### Documentation Agent (Markdown)

**`~/.config/opencode/agents/docs-writer.md`:**
```markdown
---
description: Writes and maintains project documentation
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.6
steps: 80
permission:
  read: "allow"
  write: "allow"
  edit: "allow"
  glob: "allow"
  grep: "allow"
  bash:
    "npm run docs": "allow"
    "npx typedoc": "allow"
    "*": "ask"
  webfetch: "allow"
  skill: "allow"
color: "#10b981"
hidden: false
---

You are a documentation specialist. Your role is to:

1. Write clear, concise documentation for code
2. Maintain README files, API docs, and inline comments
3. Generate documentation from code comments and type annotations
4. Review existing docs for accuracy and completeness
5. Create usage examples and tutorials

Style guidelines:
- Use active voice
- Keep sentences short
- Include code examples for every API endpoint
- Link to related documentation
- Use consistent terminology throughout
```

### Security Auditor Agent (Markdown)

**`~/.config/opencode/agents/security-auditor.md`:**
```markdown
---
description: Performs security audits and identifies vulnerabilities
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.1
steps: 40
permission:
  read: "allow"
  glob: "allow"
  grep: "allow"
  webfetch: "allow"
  edit: "deny"
  bash:
    "git diff": "allow"
    "git diff *": "allow"
    "git log": "allow"
    "git log *": "allow"
    "npm audit": "allow"
    "*": "deny"
color: "#ef4444"
---

You are a security expert. Focus on identifying potential security issues.

Look for:
- Input validation vulnerabilities
- Authentication and authorization flaws
- Data exposure risks
- Dependency vulnerabilities
- Configuration security issues

For each finding, report:
- Severity: Critical / High / Medium / Low / Info
- Location: File and line number
- Description: What the vulnerability is
- Impact: What an attacker could do
- Remediation: How to fix it

Never modify files. Read-only analysis only.
```

### Code Reviewer Agent (JSON Config)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "code-reviewer": {
      "description": "Reviews code for best practices and potential issues",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.1,
      "steps": 30,
      "permission": {
        "read": "allow",
        "glob": "allow",
        "grep": "allow",
        "bash": {
          "git diff": "allow",
          "git diff *": "allow",
          "git log": "allow",
          "git log *": "allow",
          "*": "deny"
        },
        "edit": "deny",
        "write": "deny",
        "webfetch": "allow"
      },
      "color": "#f59e0b"
    }
  }
}
```

### Debug Agent (Markdown)

**`.opencode/agents/debug.md`:**
```markdown
---
description: Debugging agent that reads and analyzes but asks before editing
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.4
steps: 60
permission:
  read: "allow"
  glob: "allow"
  grep: "allow"
  bash: "ask"
  edit: "ask"
  write: "ask"
  webfetch: "allow"
color: "#8b5cf6"
---

You are a debugging specialist. Your approach:

1. Reproduce the issue — understand exactly what's failing
2. Read relevant source code and logs
3. Search for similar patterns in the codebase
4. Identify root cause with evidence
5. Propose a fix with explanation
6. Ask before making any changes

Never guess. Always verify with concrete evidence from the code.
```

---

## 11. Multi-Agent Workflow Patterns

### Orchestrator Pattern

Use a primary agent to delegate tasks to subagents. The orchestrator coordinates work across multiple specialized agents:

```json
{
  "agent": {
    "orchestrator": {
      "description": "Coordinates multi-agent workflows",
      "mode": "primary",
      "model": "anthropic/claude-sonnet-4-20250514",
      "steps": 200,
      "permission": {
        "task": {
          "explore": "allow",
          "scout": "allow",
          "general": "allow",
          "review": "allow"
        },
        "read": "allow",
        "edit": "ask",
        "bash": "ask"
      }
    }
  }
}
```

### Parallel Exploration

The orchestrator can invoke multiple Explore subagents simultaneously:

```
orchestrator → @explore (investigate auth module)
orchestrator → @explore (investigate payment module)
orchestrator → @scout (look up Stripe API docs)
```

### Review Pipeline

A review workflow with specialized agents:

```
Build agent → writes code
    ↓
@review agent → analyzes changes
    ↓
@security agent → audits for vulnerabilities
    ↓
Build agent → incorporates feedback
```

### Research and Implement

```
User → Build agent
  → @explore (understand codebase structure)
  → @scout (look up external documentation)
  → Build agent (implements based on research)
  → @review (validates implementation)
```

---

## 12. Best Practices

1. **Use descriptive agent names** — `db-migrator`, `security-auditor`, `test-runner` are clearer than `agent-1`.

2. **Set appropriate temperature** — Lower (0.1-0.3) for analytical tasks, higher (0.6-0.8) for creative tasks.

3. **Limit steps for subagents** — Subagents should have bounded iteration counts to prevent runaway execution and control costs.

4. **Principle of least privilege** — Grant only the permissions each agent needs. Read-only agents should have `edit`, `write`, and `bash` set to `"deny"`.

5. **Use bash permission patterns** — Use object syntax to allow safe commands (`git status`, `npm test`) while blocking dangerous ones (`rm -rf`, `sudo`).

6. **Prefer Markdown for complex agents** — Markdown files are easier to read, version control, and share than JSON config entries. They also allow embedding the system prompt directly.

7. **Use color coding** — Assign distinct colors by function (blue for build, red for security, green for docs, purple for debug) to improve visual identification.

8. **Test agent permissions** — Verify that agents can perform their intended tasks with the permissions you've set before deploying to production workflows.

9. **Use task permissions for orchestration** — When building multi-agent workflows, restrict which subagents the orchestrator can invoke to prevent unintended delegation.

10. **Use hidden agents for internal helpers** — If you create agents that should only be invoked by other agents (not by users directly), set `hidden: true`.
