# OpenCode Custom Commands

Expert skill for building, configuring, and troubleshooting OpenCode's custom command system. Covers markdown command files, JSON configuration, prompt placeholders, command options, built-in commands, keybinds, and overriding built-ins.

## Overview

OpenCode's custom command system lets you define reusable prompts that execute via slash commands in the TUI (e.g., `/test`, `/review`, `/deploy`). Commands are triggered by typing `/` followed by the command name in the prompt input.

Custom commands are **in addition to** the built-in commands like `/init`, `/undo`, `/redo`, `/share`, `/help`. Custom commands can also **override** built-in command names — if you define a custom command with the same name as a built-in, the custom version takes precedence.

Commands support prompt placeholders (`$ARGUMENTS`, `$1`, `$2`, `!`command``, `@filename`) for dynamic prompt construction, and options for agent routing, model override, and subagent invocation.

---

## Create via Markdown

The primary way to create custom commands is by placing markdown files in the `commands/` directory.

### Location

| Scope     | Path                            |
|-----------|----------------------------------|
| Project   | `.opencode/commands/`            |
| Global    | `~/.config/opencode/commands/`   |

The **file name** (minus `.md` extension) becomes the command name:

| File Name               | Command         |
|-------------------------|-----------------|
| `test.md`               | `/test`         |
| `review.md`             | `/review`       |
| `deploy-staging.md`     | `/deploy-staging` |
| `create-component.md`   | `/create-component` |

### Structure

A markdown command file has two parts:

1. **Frontmatter** (YAML between `---` delimiters) — defines command properties
2. **Content** (everything after the closing `---`) — becomes the prompt template sent to the LLM

### Frontmatter Fields

| Field         | Required | Description                                                                 |
|---------------|----------|-----------------------------------------------------------------------------|
| `description` | No       | Short description shown in the TUI command list when typing `/`             |
| `agent`       | No       | Which agent executes this command. If the agent is a subagent, triggers subagent invocation by default |
| `model`       | No       | Override the default model for this command                                  |

### Complete Example

Create `.opencode/commands/test.md`:

```markdown
---
description: Run tests with coverage
agent: build
model: anthropic/claude-3-5-sonnet-20241022
---

Run the full test suite with coverage report and show any failures.
Focus on the failing tests and suggest fixes.
```

Usage in TUI:

```
/test
```

### Multi-line Prompts

The content after the frontmatter is the prompt template. You can use multiple paragraphs, code blocks, and any markdown formatting — everything is sent as-is to the LLM:

```markdown
---
description: Refactor module with best practices
---

You are a senior software engineer. Analyze the following module and refactor it according to these principles:

1. Single Responsibility Principle
2. DRY (Don't Repeat Yourself)
3. Clear naming conventions
4. Proper error handling

Ensure all existing tests still pass after refactoring.
```

### Nested Directories

Commands can be organized in subdirectories. The directory structure maps to a namespace:

```
.opencode/commands/
├── test.md              → /test
├── review.md            → /review
└── db/
    ├── migrate.md       → /db/migrate
    └── seed.md          → /db/seed
```

> **Note:** The exact namespace separator behavior depends on your OpenCode version. Verify with `!ls .opencode/commands/` if unsure.

---

## Create via JSON

You can also define commands in `opencode.json` (or `opencode.jsonc`) using the `command` field.

### JSON Configuration

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "command": {
    // This key becomes the command name
    "test": {
      // Required: the prompt sent to the LLM
      "template": "Run the full test suite with coverage report and show any failures.\nFocus on the failing tests and suggest fixes.",
      // Shown in TUI command list
      "description": "Run tests with coverage",
      // Which agent executes (optional)
      "agent": "build",
      // Override model (optional)
      "model": "anthropic/claude-3-5-sonnet-20241022"
    }
  }
}
```

Usage:

```
/test
```

### JSON vs Markdown

| Aspect              | JSON (`opencode.json`)         | Markdown (`.opencode/commands/`) |
|---------------------|-------------------------------|----------------------------------|
| Location            | Project root config file       | Dedicated directory              |
| Template field      | `template` (string, `\n` for newlines) | Content after frontmatter (natural newlines) |
| Multi-command       | All in one file                | One file per command             |
| Version control     | Changes mixed with other config | Isolated per command file        |
| Discoverability     | Requires reading config        | Browsable via file tree          |

**Recommendation:** Use markdown files for most custom commands. Use JSON for quick one-off commands or when you prefer centralized configuration.

---

## Prompt Placeholders

Prompt placeholders allow dynamic content injection into your command templates. They are evaluated at runtime when the command is executed.

### `$ARGUMENTS` — All Arguments as Single String

The `$ARGUMENTS` placeholder is replaced with **all arguments** passed to the command as a single concatenated string.

**Markdown example:**

```markdown
---
description: Create a new component
---

Create a new React component named $ARGUMENTS with TypeScript support.
Include proper typing and basic structure.
```

**JSON example:**

```json
{
  "command": {
    "component": {
      "template": "Create a new React component named $ARGUMENTS with TypeScript support.\nInclude proper typing and basic structure.",
      "description": "Create a new component"
    }
  }
}
```

**Usage:**

```
/component Button
```

**Resulting prompt:**

```
Create a new component named Button with TypeScript support.
Include proper typing and basic structure.
```

**Usage with multiple words:**

```
/component UserAvatar with image loading
```

**Resulting prompt:**

```
Create a new component named UserAvatar with image loading with TypeScript support.
Include proper typing and basic structure.
```

> **Note:** `$ARGUMENTS` captures everything after the command name as one string, including spaces. It does not split on spaces.

---

### `$1`, `$2`, `$3` — Positional Arguments

Individual arguments are accessed via positional parameters. Arguments are split by whitespace.

**Markdown example:**

```markdown
---
description: Create a new file with content
---

Create a file named $1 in the directory $2
with the following content: $3
```

**JSON example:**

```json
{
  "command": {
    "create-file": {
      "template": "Create a file named $1 in the directory $2\nwith the following content: $3",
      "description": "Create a new file with content"
    }
  }
}
```

**Usage:**

```
/create-file config.json src "{ \"key\": \"value\" }"
```

**Resulting prompt:**

```
Create a file named config.json in the directory src
with the following content: { "key": "value" }
```

**Positional argument mapping:**

| Placeholder | Value        |
|-------------|--------------|
| `$1`        | `config.json`|
| `$2`        | `src`        |
| `$3`        | `{ "key": "value" }` |

**Additional positional arguments:**

- `$4`, `$5`, `$6`, ... — Access arguments by position (up to the number of arguments provided)
- Unreferenced positional parameters are simply ignored in the prompt

**Multi-word arguments:** Positional parameters split on whitespace. To pass a multi-word value as a single positional argument, quote it:

```
/create-file README.md . "Hello World"
```

This maps:
- `$1` → `README.md`
- `$2` → `.`
- `$3` → `Hello World`

---

### `!`command`` — Inject Bash Command Output

Use backtick syntax with `!` prefix to run a bash command and inject its output directly into the prompt. The command runs in your project's root directory.

**Syntax:** `!`command``

**Markdown example — Analyze test coverage:**

```markdown
---
description: Analyze test coverage
---

Here are the current test results:
!`npm test`

Based on these results, suggest improvements to increase coverage.
```

**Markdown example — Review recent changes:**

```markdown
---
description: Review recent changes
---

Recent git commits:
!`git log --oneline -10`

Review these changes and suggest any improvements.
```

**Markdown example — Check project structure:**

```markdown
---
description: Analyze project structure
---

Project file tree:
!`find . -type f -name "*.ts" | head -30`

Analyze the module organization and suggest improvements.
```

**JSON example:**

```json
{
  "command": {
    "review-changes": {
      "template": "Recent git commits:\n!`git log --oneline -10`\n\nReview these changes and suggest any improvements.",
      "description": "Review recent changes"
    }
  }
}
```

**Behavior:**

- The bash command executes in the project's root directory
- stdout is captured and inserted into the prompt at the placeholder position
- stderr is typically not included (or handled silently)
- If the command fails, the output may be empty or contain an error message
- The command runs synchronously — the prompt is not sent until the command completes
- Multiple `!`command`` placeholders can be used in a single prompt

**Advanced examples:**

```markdown
---
description: Check dependency versions
---

Current dependencies:
!`cat package.json | jq '.dependencies'`

Check for outdated or vulnerable packages.
```

```markdown
---
description: Lint and format check
---

Lint output:
!`npm run lint 2>&1`

Format check:
!`npx prettier --check . 2>&1`

Fix any issues found.
```

---

### `@filename` — Include File Content

Use `@` followed by a filename to include the file's content in the prompt. The file path is relative to the project root.

**Syntax:** `@filename` or `@path/to/file`

**Markdown example — Review a component:**

```markdown
---
description: Review component
---

Review the component in @src/components/Button.tsx.
Check for performance issues and suggest improvements.
```

**Markdown example — Compare files:**

```markdown
---
description: Compare implementations
---

Compare these two implementations:
@src/utils/old-helper.ts
@src/utils/new-helper.ts

Identify the differences and explain which approach is better.
```

**JSON example:**

```json
{
  "command": {
    "review-component": {
      "template": "Review the component in @src/components/Button.tsx.\nCheck for performance issues and suggest improvements.",
      "description": "Review component"
    }
  }
}
```

**Behavior:**

- The file content is resolved relative to the project root directory
- The file's content is included verbatim in the prompt
- The LLM receives the file content as context
- If the file does not exist, the placeholder may be included as-is or produce an error depending on the OpenCode version
- Multiple `@` references can be used in a single prompt

**Fuzzy file search:** In the TUI prompt input (outside of commands), typing `@` triggers a fuzzy file search. Inside command templates, `@` references are resolved literally by path.

---

## Command Options

Each custom command supports the following configuration options. These can be set in JSON (`opencode.json`) or via frontmatter in markdown files.

### template (Required for JSON; implicit in Markdown)

The prompt that will be sent to the LLM when the command is executed.

**In JSON:** The `template` field is **required**. Use `\n` for newlines within the string.

```json
{
  "command": {
    "test": {
      "template": "Run the full test suite with coverage report.\nFocus on failures and suggest fixes."
    }
  }
}
```

**In Markdown:** The template is the **content after the frontmatter**. No separate field needed — everything below the closing `---` becomes the template.

```markdown
---
description: Run tests
---

Run the full test suite with coverage report.
Focus on failures and suggest fixes.
```

---

### description

A brief description of what the command does. This is displayed in the TUI command list when you type `/` and is purely informational.

**In JSON:**

```json
{
  "command": {
    "test": {
      "template": "...",
      "description": "Run tests with coverage"
    }
  }
}
```

**In Markdown frontmatter:**

```markdown
---
description: Run tests with coverage
---
```

**In TUI:** When you type `/`, the command list shows each command's description next to its name:

```
/test      Run tests with coverage
/review    Review code changes
/deploy    Deploy to staging
```

---

### agent

Optionally specify which [agent](https://opencode.ai/docs/agents/) should execute this command. If the specified agent is a [subagent](https://opencode.ai/docs/agents/#subagents), the command will trigger a subagent invocation by default (see `subtask` below).

**In JSON:**

```json
{
  "command": {
    "review": {
      "template": "Review the recent code changes for quality and correctness.",
      "agent": "plan"
    }
  }
}
```

**In Markdown frontmatter:**

```markdown
---
description: Review code
agent: plan
---

Review the recent code changes for quality and correctness.
```

**Behavior:**

- If `agent` is not specified, the command uses the current agent (the one active in the session)
- If `agent` specifies a subagent, the command automatically invokes it as a subagent (unless `subtask: false` is set)
- If `agent` specifies the primary agent, the command runs in the primary context (unless `subtask: true` is set)

---

### subtask

A boolean that **forces** the command to trigger a [subagent](https://opencode.ai/docs/agents/#subagents) invocation. This is useful when you want the command to run in an isolated context without polluting your primary conversation.

**In JSON:**

```json
{
  "command": {
    "analyze": {
      "template": "Analyze the codebase for security vulnerabilities.",
      "subtask": true
    }
  }
}
```

**In Markdown frontmatter:**

```markdown
---
description: Security analysis
subtask: true
---

Analyze the codebase for security vulnerabilities.
```

**Behavior:**

- When `subtask: true`, the agent acts as a subagent regardless of its `mode` setting in agent configuration
- When `subtask: false`, the agent runs in its default mode even if it's a subagent
- If not specified, defaults to `false` unless the `agent` is a subagent (in which case it defaults to `true`)

**Use cases for `subtask: true`:**

- Long-running analysis that shouldn't block the main conversation
- Tasks where you want results returned without full conversation context
- Parallel execution of independent tasks
- Keeping the primary context clean for focused work

---

### model

Override the default model for this command. Useful when a specific task benefits from a different model (e.g., a reasoning-heavy task using a more capable model, or a simple task using a faster/cheaper model).

**In JSON:**

```json
{
  "command": {
    "analyze": {
      "template": "Analyze this codebase for architectural issues.",
      "model": "anthropic/claude-3-5-sonnet-20241022"
    }
  }
}
```

**In Markdown frontmatter:**

```markdown
---
description: Architectural analysis
model: anthropic/claude-3-5-sonnet-20241022
---

Analyze this codebase for architectural issues.
```

**Common model override patterns:**

| Use Case                        | Model Strategy                              |
|---------------------------------|---------------------------------------------|
| Quick formatting/linting        | Fast, cheaper model (e.g., `openai/gpt-4o-mini`) |
| Deep code review                | Most capable model available                |
| Simple file generation          | Mid-tier model                              |
| Complex multi-step reasoning    | Top-tier reasoning model                    |

---

## Built-in Commands

OpenCode ships with the following built-in commands. These are always available in the TUI. Custom commands can **override** any of these by using the same name.

### /connect

Add a provider to OpenCode. Allows you to select from available providers and add their API keys.

```
/connect
```

---

### /compact

Compact the current session. This summarizes the conversation to reduce token usage while preserving key context. Useful for long sessions approaching context limits.

**Alias:** `/summarize`

```
/compact
```

**Keybind:** `ctrl+x c`

---

### /details

Toggle the display of tool execution details. When enabled, you can see detailed information about how tools are executed (file reads, bash commands, etc.).

```
/details
```

---

### /editor

Open an external editor for composing messages. Uses the editor set in your `EDITOR` environment variable. This is useful for writing long or complex prompts that benefit from proper text editing.

```
/editor
```

**Keybind:** `ctrl+x e`

**Editor setup:** Set the `EDITOR` environment variable:

```bash
# Nano
export EDITOR=nano

# Vim
export EDITOR=vim

# VS Code (with --wait for blocking mode)
export EDITOR="code --wait"

# Cursor
export EDITOR="cursor --wait"
```

For GUI editors, the `--wait` flag is required so the process blocks until the editor is closed.

---

### /exit

Exit OpenCode.

**Aliases:** `/quit`, `/q`

```
/exit
```

**Keybind:** `ctrl+x q`

---

### /export

Export the current conversation to Markdown and open it in your default editor. Uses the `EDITOR` environment variable. The exported markdown includes all messages, tool results, and file changes.

```
/export
```

**Keybind:** `ctrl+x x`

---

### /help

Show the help dialog with available commands and keybinds.

```
/help
```

---

### /init

Guided setup for creating or updating `AGENTS.md`. This walks you through configuring project-specific rules, conventions, and instructions for the AI agent.

```
/init
```

---

### /models

List available models. Shows all models configured across your providers and allows you to switch the active model.

```
/models
```

**Keybind:** `ctrl+x m`

---

### /new

Start a new session. Clears the current conversation and begins fresh.

**Alias:** `/clear`

```
/new
```

**Keybind:** `ctrl+x n`

---

### /redo

Redo a previously undone message. Only available after using `/undo`. Restores the most recently undone user message, all subsequent responses, and any file changes.

**Important:** Your project **must be a Git repository** for file changes to be restored. Internally, OpenCode uses Git to manage file change reversions.

```
/redo
```

**Keybind:** `ctrl+x r`

---

### /sessions

List and switch between sessions. Shows all saved sessions with timestamps and allows you to resume a previous conversation.

**Aliases:** `/resume`, `/continue`

```
/sessions
```

**Keybind:** `ctrl+x l`

---

### /share

Share the current session. Generates a shareable link to the conversation. [Learn more](https://opencode.ai/docs/share/).

```
/share
```

---

### /themes

List available themes. Allows you to switch the TUI color scheme.

```
/themes
```

**Keybind:** `ctrl+x t`

---

### /thinking

Toggle the visibility of thinking/reasoning blocks in the conversation. When enabled, you can see the model's internal reasoning process for models that support extended thinking.

**Note:** This command controls **display only** — it does not enable or disable the model's actual reasoning capabilities. To toggle reasoning capabilities, use `ctrl+t` to cycle through model variants.

```
/thinking
```

---

### /undo

Undo the last message in the conversation. Removes the most recent user message, all subsequent responses, and any file changes made by those responses.

**Important:** Your project **must be a Git repository** for file changes to be reverted. Internally, OpenCode uses Git to manage file change reversions.

```
/undo
```

**Keybind:** `ctrl+x u`

---

### /unshare

Unshare the current session. Revokes the shareable link previously generated with `/share`. [Learn more](https://opencode.ai/docs/share#un-sharing).

```
/unshare
```

---

## Keybinds

OpenCode uses a **leader key** pattern for command shortcuts. The default leader key is `ctrl+x`. To execute a command via keybind, press the leader key followed by the shortcut key.

**Leader key:** `ctrl+x` (configurable in `tui.json` via `keybinds.leader`)

**Leader timeout:** 2000ms (configurable via `leader_timeout` in `tui.json`)

### Complete Keybind Reference

| Command       | Keybind        | Description                              |
|---------------|----------------|------------------------------------------|
| `/compact`    | `ctrl+x c`     | Compact/summarize the current session    |
| `/editor`     | `ctrl+x e`     | Open external editor for message input   |
| `/exit`       | `ctrl+x q`     | Exit OpenCode                            |
| `/export`     | `ctrl+x x`     | Export conversation to Markdown          |
| `/models`     | `ctrl+x m`     | List available models                    |
| `/new`        | `ctrl+x n`     | Start a new session                      |
| `/redo`       | `ctrl+x r`     | Redo last undone message                 |
| `/sessions`   | `ctrl+x l`     | List/switch between sessions             |
| `/themes`     | `ctrl+x t`     | List available themes                    |
| `/undo`       | `ctrl+x u`     | Undo last message                        |

### Other Notable Keybinds (Non-command)

| Keybind               | Action                                |
|-----------------------|---------------------------------------|
| `ctrl+x b`           | Toggle sidebar                        |
| `ctrl+x s`           | Status view                           |
| `ctrl+x a`           | Agent list                            |
| `ctrl+x g`           | Session timeline                      |
| `ctrl+x h`           | Toggle tips / conceals                |
| `ctrl+x y`           | Copy message                          |
| `ctrl+p`             | Command palette                       |
| `ctrl+t`             | Cycle model variants                  |
| `tab` / `shift+tab`  | Cycle agents                          |
| `escape`             | Interrupt session                     |
| `ctrl+c`             | Clear input                           |
| `ctrl+z`             | Suspend terminal (Unix only)          |

### Customizing Keybinds

Keybinds are configured in `tui.json` (separate from `opencode.json`):

```json
{
  "$schema": "https://opencode.ai/tui.json",
  "leader_timeout": 2000,
  "keybinds": {
    "leader": "ctrl+x",
    "session_compact": "<leader>c",
    "session_new": "<leader>n",
    "session_list": "<leader>l"
  }
}
```

To disable a keybind, set it to `"none"`:

```json
{
  "keybinds": {
    "session_compact": "none"
  }
}
```

Multiple keybinds for the same action can be specified as an array:

```json
{
  "keybinds": {
    "messages_copy": ["<leader>y", "ctrl+shift+c"]
  }
}
```

---

## Override Built-ins

Custom commands can **override** built-in commands by using the same command name. If you define a custom command named `help`, `/help` will execute your custom version instead of the built-in help dialog.

### Example: Custom /help

```markdown
---
description: Show project-specific help
---

## Project Commands

- `/test` — Run test suite
- `/lint` — Run linter
- `/deploy` — Deploy to staging

## Project Conventions

- Use TypeScript strict mode
- Follow ESLint airbnb config
- Write tests for all new features
```

Now `/help` shows your custom help content instead of the default OpenCode help.

### Example: Custom /init

```markdown
---
description: Initialize project for AI coding
---

## Setting Up AGENTS.md

Run the following steps to initialize this project:

1. Read the README.md for project overview
2. Examine package.json for dependencies and scripts
3. Check tsconfig.json for TypeScript configuration
4. Create AGENTS.md with project-specific rules
```

### Important Considerations

- **Override scope:** Overrides apply project-wide. If you override `/undo`, you lose access to the built-in undo unless you explicitly implement reversal logic.
- **Naming conflicts:** If a custom command and a built-in share the same name, the custom command always wins. There is no fallback mechanism.
- **Documentation:** When overriding built-ins, document the override clearly so team members understand the behavior change.
- **Global vs project:** Global commands (in `~/.config/opencode/commands/`) can also override built-ins, affecting all projects. Use global overrides cautiously.

---

## Complete Examples

### Example 1: Test Runner with Coverage

**`.opencode/commands/test.md`:**

```markdown
---
description: Run tests with coverage
agent: build
model: anthropic/claude-3-5-sonnet-20241022
---

Run the full test suite with coverage report:

1. Run `!`npm test -- --coverage``
2. Analyze the coverage output
3. Identify any files below 80% coverage
4. For files below threshold, suggest specific test cases to add
5. Check for any flaky or skipped tests

Output a summary:
- Total coverage percentage
- Files below threshold (list each)
- Skipped/flaky tests
- Recommended next steps
```

**JSON equivalent:**

```json
{
  "command": {
    "test": {
      "template": "Run the full test suite with coverage report:\n\n1. Run `!`npm test -- --coverage``\n2. Analyze the coverage output\n3. Identify any files below 80% coverage\n4. For files below threshold, suggest specific test cases to add\n5. Check for any flaky or skipped tests\n\nOutput a summary:\n- Total coverage percentage\n- Files below threshold (list each)\n- Skipped/flaky tests\n- Recommended next steps",
      "description": "Run tests with coverage",
      "agent": "build",
      "model": "anthropic/claude-3-5-sonnet-20241022"
    }
  }
}
```

---

### Example 2: Component Creator with $ARGUMENTS

**`.opencode/commands/component.md`:**

```markdown
---
description: Create a new React component
---

Create a new React component named $ARGUMENTS following these conventions:

## File Structure
Create these files:
- `src/components/$ARGUMENTS/index.tsx` — Main component
- `src/components/$ARGUMENTS/$ARGUMENTS.test.tsx` — Unit tests
- `src/components/$ARGUMENTS/$ARGUMENTS.stories.tsx` — Storybook stories
- `src/components/$ARGUMENTS/styles.module.css` — CSS module styles

## Requirements
- TypeScript with strict typing
- Functional component with hooks
- Props interface exported as `$ARGUMENTSProps`
- Default export of the component
- CSS modules for styling (import as `styles`)
- Basic unit test with React Testing Library
- Storybook story with default and loading variants

## Example Props Interface
```typescript
export interface $ARGUMENTSProps {
  className?: string;
  children?: React.ReactNode;
}
```

Generate all files with complete, production-ready code.
```

**Usage:**

```
/component Button
/component UserAvatar
/component Modal
```

**Resulting prompt (for `/component Button`):**

```
Create a new React component named Button following these conventions:

## File Structure
Create these files:
- `src/components/Button/index.tsx` — Main component
- `src/components/Button/Button.test.tsx` — Unit tests
- `src/components/Button/Button.stories.tsx` — Storybook stories
- `src/components/Button/styles.module.css` — CSS module styles
...
```

---

### Example 3: Git Changelog with !`git log`

**`.opencode/commands/changelog.md`:**

```markdown
---
description: Generate changelog from git history
---

Generate a changelog from recent git history.

## Recent commits:
!`git log --oneline --since="2 weeks ago" --no-merges`

## Full diff summary:
!`git diff --stat HEAD~10..HEAD`

Based on these changes, generate a structured changelog with:

### Categories
- **Features** — New functionality
- **Bug Fixes** — Issue resolutions
- **Refactors** — Code improvements without behavior changes
- **Documentation** — Doc updates
- **Chores** — Maintenance tasks

### Format
Use [Keep a Changelog](https://keepachangelog.com/) format with version header.
Date range: last 2 weeks.
Group related commits together.
Write user-facing descriptions (not developer implementation details).

Output the changelog in markdown format.
```

**JSON equivalent:**

```json
{
  "command": {
    "changelog": {
      "template": "Generate a changelog from recent git history.\n\n## Recent commits:\n!`git log --oneline --since=\"2 weeks ago\" --no-merges`\n\n## Full diff summary:\n!`git diff --stat HEAD~10..HEAD`\n\nBased on these changes, generate a structured changelog with:\n\n### Categories\n- **Features** — New functionality\n- **Bug Fixes** — Issue resolutions\n- **Refactors** — Code improvements without behavior changes\n- **Documentation** — Doc updates\n- **Chores** — Maintenance tasks\n\n### Format\nUse [Keep a Changelog](https://keepachangelog.com/) format with version header.\nDate range: last 2 weeks.\nGroup related commits together.\nWrite user-facing descriptions (not developer implementation details).\n\nOutput the changelog in markdown format.",
      "description": "Generate changelog from git history"
    }
  }
}
```

---

### Example 4: Code Review with @file Reference

**`.opencode/commands/review.md`:**

```markdown
---
description: Review a specific file for issues
agent: plan
---

Perform a thorough code review of @src/api/handlers.ts.

## Review Criteria

### Correctness
- Logic errors and edge cases
- Null/undefined handling
- Race conditions and async issues
- Off-by-one errors

### Security
- Input validation and sanitization
- SQL injection / XSS vulnerabilities
- Authentication/authorization checks
- Secrets or credentials in code

### Performance
- Unnecessary re-renders or computations
- Memory leaks
- N+1 query patterns
- Inefficient algorithms

### Code Quality
- Naming conventions and clarity
- Function length and complexity
- DRY violations
- Error handling completeness

### Testing
- Test coverage gaps
- Missing edge case tests
- Mock quality

## Output Format
For each issue found:
1. **Line number** and code snippet
2. **Severity** (Critical / Warning / Suggestion)
3. **Category** (from above)
4. **Description** of the issue
5. **Recommended fix** with code example

End with an overall assessment and priority-ordered action items.
```

**JSON equivalent:**

```json
{
  "command": {
    "review": {
      "template": "Perform a thorough code review of @src/api/handlers.ts.\n\n## Review Criteria\n\n### Correctness\n- Logic errors and edge cases\n- Null/undefined handling\n- Race conditions and async issues\n- Off-by-one errors\n\n### Security\n- Input validation and sanitization\n- SQL injection / XSS vulnerabilities\n- Authentication/authorization checks\n- Secrets or credentials in code\n\n### Performance\n- Unnecessary re-renders or computations\n- Memory leaks\n- N+1 query patterns\n- Inefficient algorithms\n\n### Code Quality\n- Naming conventions and clarity\n- Function length and complexity\n- DRY violations\n- Error handling completeness\n\n### Testing\n- Test coverage gaps\n- Missing edge case tests\n- Mock quality\n\n## Output Format\nFor each issue found:\n1. **Line number** and code snippet\n2. **Severity** (Critical / Warning / Suggestion)\n3. **Category** (from above)\n4. **Description** of the issue\n5. **Recommended fix** with code example\n\nEnd with an overall assessment and priority-ordered action items.",
      "description": "Review a specific file for issues",
      "agent": "plan"
    }
  }
}
```

---

### Example 5: Subagent Task with Model Override

**`.opencode/commands/deep-analysis.md`:**

```markdown
---
description: Deep codebase analysis via subagent
agent: plan
subtask: true
model: anthropic/claude-3-5-sonnet-20241022
---

Perform a comprehensive analysis of the authentication module:

1. Read all files in @src/auth/
2. Map the authentication flow end-to-end
3. Identify security vulnerabilities
4. Check for compliance with OWASP Top 10
5. Review token handling and session management
6. Assess password hashing implementation
7. Verify CSRF protection

!`find src/auth -type f -name "*.ts" | sort`

Output:
- Architecture diagram (text-based)
- Security audit report
- List of vulnerabilities ranked by severity
- Recommended fixes for each vulnerability
```

**JSON equivalent:**

```json
{
  "command": {
    "deep-analysis": {
      "template": "Perform a comprehensive analysis of the authentication module:\n\n1. Read all files in @src/auth/\n2. Map the authentication flow end-to-end\n3. Identify security vulnerabilities\n4. Check for compliance with OWASP Top 10\n5. Review token handling and session management\n6. Assess password hashing implementation\n7. Verify CSRF protection\n\n!`find src/auth -type f -name \"*.ts\" | sort`\n\nOutput:\n- Architecture diagram (text-based)\n- Security audit report\n- List of vulnerabilities ranked by severity\n- Recommended fixes for each vulnerability",
      "description": "Deep codebase analysis via subagent",
      "agent": "plan",
      "subtask": true,
      "model": "anthropic/claude-3-5-sonnet-20241022"
    }
  }
}
```

---

### Example 6: Combined Placeholders

**`.opencode/commands/scaffold.md`:**

```markdown
---
description: Scaffold a new API endpoint
---

Scaffold a new API endpoint: $1

## Method and Path
- Method: $2
- Path: $3

## Requirements
1. Create route handler in @src/api/routes/
2. Add input validation schema
3. Create service layer function
4. Add unit tests
5. Update API documentation

## Conventions
- Use Zod for request validation
- Return consistent error format: `{ error: string, code: number }`
- Log all errors with structured logging
- All handlers must be async with proper error boundaries

Generate all necessary files.
```

**Usage:**

```
/scaffold users POST /api/v1/users
```

**Resulting prompt:**

```
Scaffold a new API endpoint: users

## Method and Path
- Method: POST
- Path: /api/v1/users

## Requirements
1. Create route handler in @src/api/routes/
2. Add input validation schema
3. Create service layer function
4. Add unit tests
5. Update API documentation
...
```

---

## File Structure Reference

```
.opencode/
├── commands/
│   ├── test.md              → /test
│   ├── review.md            → /review
│   ├── changelog.md         → /changelog
│   ├── component.md         → /component
│   ├── scaffold.md          → /scaffold
│   └── db/
│       ├── migrate.md       → /db/migrate
│       └── seed.md          → /db/seed
└── ...

~/.config/opencode/
└── commands/
    ├── global-review.md     → /global-review (available in all projects)
    └── global-template.md   → /global-template
```

---

## Best Practices

1. **Use descriptive filenames** — The filename becomes the command name. `run-tests.md` is clearer than `rt.md`.

2. **Write clear descriptions** — The `description` field helps users discover commands in the TUI. Be specific about what the command does.

3. **Leverage placeholders wisely** — Use `$ARGUMENTS` for simple single-value inputs. Use `$1`, `$2` for structured multi-argument commands.

4. **Use `!`command`` for dynamic context** — Injecting shell output (git logs, test results, file listings) makes commands much more powerful.

5. **Use `@file` for focused reviews** — Reference specific files to give the LLM precise context without polluting the prompt with unnecessary content.

6. **Set `subtask: true` for heavy analysis** — Long-running tasks should use subagent invocation to avoid blocking the main conversation.

7. **Override built-ins intentionally** — Only override built-in commands when you have a clear reason and have documented the change for your team.

8. **Use global commands sparingly** — Global commands in `~/.config/opencode/commands/` affect all projects. Prefer project-level commands for project-specific workflows.

9. **Test your commands** — Run each command after creation to verify placeholders resolve correctly and the prompt produces the expected output.

10. **Version control your commands** — Commit `.opencode/commands/` to your repository so team members can use the same custom commands.
