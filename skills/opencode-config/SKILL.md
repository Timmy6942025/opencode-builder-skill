---
name: opencode-config
description: |
  Use this skill when configuring OpenCode's opencode.json or tui.json files, setting up config precedence, managing provider credentials, using variable substitution, deploying managed settings via MDM, or troubleshooting configuration issues. Covers all config options, schema validation, remote/global/project config layers, environment variables, file references, and enterprise managed preferences.
---

# OpenCode Configuration

OpenCode uses JSON-based configuration files to control runtime behavior, TUI appearance, server settings, tool permissions, agent definitions, and provider connections. This skill covers every aspect of configuration: file format, locations, precedence, merging, all available fields, TUI config, variable substitution, enterprise managed settings, environment variables, and troubleshooting.

---

## Table of Contents

- [Format](#format)
- [Config Locations & Precedence](#config-locations--precedence)
- [Config Merging Behavior](#config-merging-behavior)
- [All Config Fields](#all-config-fields)
- [TUI Config (tui.json)](#tui-config-tuijson)
- [Variable Substitution](#variable-substitution)
- [Managed Settings (Enterprise)](#managed-settings-enterprise)
- [Provider-Specific Options](#provider-specific-options)
- [Environment Variables](#environment-variables)
- [Debugging & Verification](#debugging--verification)
- [Troubleshooting](#troubleshooting)

---

## Format

OpenCode supports both **JSON** and **JSONC** (JSON with Comments) formats for all configuration files.

### Schema

Every config file should include a `$schema` field for editor validation and autocomplete:

- **Server/runtime config**: `"$schema": "https://opencode.ai/config.json"`
- **TUI config**: `"$schema": "https://opencode.ai/tui.json"`

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-5",
  "autoupdate": true,
  "server": {
    "port": 4096
  }
}
```

Schemas enable autocompletion in VS Code, JetBrains editors, and other JSON-aware editors. Omitting the schema is valid but not recommended.

---

## Config Locations & Precedence

Configuration files are **merged together**, not replaced. Settings from multiple locations are combined. Later sources override earlier ones only for conflicting keys. Non-conflicting settings from all configs are preserved.

### Precedence Order (lowest to highest)

Config sources are loaded in this order (later sources override earlier ones):

| Priority | Source | Description |
|----------|--------|-------------|
| 1 (lowest) | **Remote config** | Organizational defaults from `.well-known/opencode` |
| 2 | **Global config** | User preferences at `~/.config/opencode/opencode.json` |
| 3 | **Custom config** | Custom overrides via `OPENCODE_CONFIG` env var |
| 4 | **Project config** | Project-specific settings at `opencode.json` in project root |
| 5 | **`.opencode` directories** | Agents, commands, modes, plugins, skills, tools, themes |
| 6 | **Inline config** | Runtime overrides via `OPENCODE_CONFIG_CONTENT` env var |
| 7 | **Managed config files** | Admin-controlled files in platform-specific directories |
| 8 (highest) | **macOS managed preferences** | `.mobileconfig` via MDM — not user-overridable |

For example, if your global config sets `autoupdate: true` and your project config sets `model: "anthropic/claude-sonnet-4-5"`, the final configuration will include both settings. If both set the same key, the project config wins.

### Remote Config

Organizations can provide default configuration via the `.well-known/opencode` endpoint. This is fetched automatically when you authenticate with a provider that supports it.

Remote config is loaded first, serving as the base layer. All other config sources (global, project, managed) can override these defaults.

```jsonc
// Remote config from .well-known/opencode
{
  "mcp": {
    "jira": {
      "type": "remote",
      "url": "https://jira.example.com/mcp",
      "enabled": false
    }
  }
}
```

Override in your local config:

```jsonc
// opencode.json (project or global)
{
  "mcp": {
    "jira": {
      "type": "remote",
      "url": "https://jira.example.com/mcp",
      "enabled": true
    }
  }
}
```

### Global Config

Place your global OpenCode config in `~/.config/opencode/opencode.json`. Use global config for user-wide server/runtime preferences like providers, models, and permissions.

For TUI-specific settings, use `~/.config/opencode/tui.json`.

Global config overrides remote organizational defaults.

### Project Config

Add `opencode.json` in your project root. Project config has the highest precedence among standard config files — it overrides both global and remote configs.

When OpenCode starts up, it first looks for a config file in the current directory, then traverses up to the nearest Git directory. This file is safe to check into Git and uses the same schema as the global one.

### Custom Config Path (`OPENCODE_CONFIG`)

Specify a custom config file path using the `OPENCODE_CONFIG` environment variable:

```bash
export OPENCODE_CONFIG=/path/to/my/custom-config.json
opencode run "Hello world"
```

Custom config is loaded between global and project configs in the precedence order.

### Custom Config Directory (`OPENCODE_CONFIG_DIR`)

Specify a custom config directory using the `OPENCODE_CONFIG_DIR` environment variable. This directory will be searched for agents, commands, modes, and plugins just like the standard `.opencode` directory:

```bash
export OPENCODE_CONFIG_DIR=/path/to/my/config-directory
opencode run "Hello world"
```

The custom directory is loaded after the global config and `.opencode` directories, so it **can override** their settings.

### `.opencode` Directories

The `.opencode` and `~/.config/opencode` directories use **plural names** for subdirectories: `agents/`, `commands/`, `modes/`, `plugins/`, `skills/`, `tools/`, and `themes/`. Singular names (e.g., `agent/`) are also supported for backwards compatibility.

### Inline Config (`OPENCODE_CONFIG_CONTENT`)

Pass configuration directly as a JSON string via the `OPENCODE_CONFIG_CONTENT` environment variable. Useful for runtime overrides without touching files:

```bash
export OPENCODE_CONFIG_CONTENT='{"model": "anthropic/claude-sonnet-4-5"}'
opencode run "Hello world"
```

---

## Config Merging Behavior

Configs are **merged, not replaced**. When two config sources define the same key, the merge behavior depends on the value type:

- **Scalar values** (strings, numbers, booleans): later source wins
- **Objects**: recursively merged; later source keys override earlier ones
- **Arrays**: later source replaces the entire array (not appended)

Example flow:

1. Global sets `autoupdate: true`
2. Project sets `model: "anthropic/claude-sonnet-4-5"`
3. Final config: both `autoupdate: true` AND `model: "anthropic/claude-sonnet-4-5"` are present

Example with arrays (important caveat):

1. Global sets `instructions: ["CONTRIBUTING.md"]`
2. Project sets `instructions: ["docs/guidelines.md"]`
3. Final config: `instructions: ["docs/guidelines.md"]` — the global array is **replaced**, not merged

---

## All Config Fields

### `$schema`

JSON Schema URL for editor validation and autocomplete.

```jsonc
{
  "$schema": "https://opencode.ai/config.json"
}
```

### `model`

The primary LLM model used for conversations. Format: `provider/model-name`.

```jsonc
{
  "model": "anthropic/claude-sonnet-4-5"
}
```

### `small_model`

A separate model for lightweight tasks like title generation and compaction summaries. By default, OpenCode tries to use a cheaper model from your provider, falling back to the main model.

```jsonc
{
  "small_model": "anthropic/claude-haiku-4-5"
}
```

### `provider`

Provider configuration with API keys, endpoints, and options. Each key is a provider ID (e.g., `anthropic`, `openai`, `amazon-bedrock`, `ollama`).

```jsonc
{
  "provider": {
    "anthropic": {
      "models": {},
      "options": {
        "apiKey": "{env:ANTHROPIC_API_KEY}",
        "timeout": 600000,
        "chunkTimeout": 30000,
        "setCacheKey": true
      }
    }
  }
}
```

**Provider options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `timeout` | number \| false | 300000 | Request timeout in milliseconds. Set to `false` to disable. |
| `chunkTimeout` | number | — | Timeout in ms between streamed response chunks. Aborts if no chunk arrives in time. |
| `setCacheKey` | boolean | — | Ensure a cache key is always set for the provider. |
| `baseURL` | string | — | Custom base URL for the provider (useful for proxies, gateways, local servers). |
| `headers` | object | — | Custom HTTP headers sent with provider requests. |

**Custom provider (npm):**

Use any OpenAI-compatible provider by specifying an npm package:

```jsonc
{
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://localhost:11434/v1"
      },
      "models": {
        "llama2": {
          "name": "Llama 2"
        }
      }
    }
  }
}
```

### `plugin`

Load plugins from npm packages or local paths:

```jsonc
{
  "plugin": ["opencode-helicone-session", "@my-org/custom-plugin"]
}
```

Plugins can also be placed in `.opencode/plugins/` or `~/.config/opencode/plugins/` as JavaScript/TypeScript files, which are loaded automatically at startup.

### `mcp`

MCP (Model Context Protocol) server configurations for adding external tools:

```jsonc
{
  "mcp": {
    "my-local-server": {
      "type": "local",
      "command": ["npx", "-y", "@my-org/mcp-server"],
      "enabled": true,
      "environment": {
        "MY_ENV_VAR": "value"
      }
    },
    "my-remote-server": {
      "type": "remote",
      "url": "https://mcp.example.com",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer {env:MY_API_KEY}"
      }
    }
  }
}
```

**Local MCP options:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `type` | `"local"` | Yes | Must be `"local"` |
| `command` | array | Yes | Command and arguments to run the MCP server |
| `environment` | object | No | Environment variables for the server process |
| `enabled` | boolean | No | Enable/disable the server on startup |
| `timeout` | number | No | Timeout in ms for fetching tools (default: 5000) |

**Remote MCP options:**

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `type` | `"remote"` | Yes | Must be `"remote"` |
| `url` | string | Yes | URL of the remote MCP server |
| `enabled` | boolean | No | Enable/disable the server on startup |
| `headers` | object | No | HTTP headers for the request |
| `oauth` | object \| false | No | OAuth config or `false` to disable OAuth auto-detection |
| `timeout` | number | No | Timeout in ms for fetching tools (default: 5000) |

**Remote MCP OAuth options:**

| Option | Type | Description |
|--------|------|-------------|
| `clientId` | string | OAuth client ID. If not provided, dynamic client registration is attempted. |
| `clientSecret` | string | OAuth client secret, if required. |
| `scope` | string | OAuth scopes to request. |

### `tools`

Enable/disable tools globally. Set a tool to `false` to disable it, or `true` to enable it:

```jsonc
{
  "tools": {
    "write": false,
    "bash": false
  }
}
```

Glob patterns are supported for MCP tool names:

```jsonc
{
  "tools": {
    "my-mcp*": false
  }
}
```

As of v1.1.1, the legacy `tools` boolean config is deprecated and has been merged into `permission`.

### `permission`

Granular permission rules for tool execution. Controls whether actions require user approval, are auto-approved, or are blocked.

```jsonc
{
  "permission": {
    "*": "ask",
    "bash": "ask",
    "edit": "allow",
    "bash": {
      "*": "ask",
      "git *": "allow",
      "rm *": "deny"
    },
    "edit": {
      "*": "deny",
      "packages/web/src/*.mdx": "allow"
    }
  }
}
```

**Permission values:**

| Value | Description |
|-------|-------------|
| `"allow"` | Run without approval |
| `"ask"` | Prompt user for approval |
| `"deny"` | Block the action |

**Available permission keys:**

| Key | Matches |
|-----|---------|
| `read` | File reads (matches file path) |
| `edit` | All file modifications (edit, write, patch) |
| `glob` | File globbing (matches glob pattern) |
| `grep` | Content search (matches regex pattern) |
| `bash` | Shell commands (matches parsed commands) |
| `task` | Subagent launches (matches subagent type) |
| `skill` | Skill loading (matches skill name) |
| `lsp` | LSP queries (currently non-granular) |
| `question` | User questions during execution |
| `webfetch` | URL fetching (matches URL) |
| `websearch` | Web search (matches query) |
| `external_directory` | Tool calls touching paths outside project root |
| `doom_loop` | Triggered when same tool call repeats 3x with identical input |
| `*` | Catch-all for any permission |

**Granular rules (object syntax):**

Rules are evaluated by pattern match, with the **last matching rule winning**. Use `"*"` as catch-all first, then specific patterns after.

- `*` matches zero or more of any character
- `?` matches exactly one character
- All other characters match literally

**Home directory expansion:**

Use `~` or `$HOME` at the start of a pattern to reference your home directory:

- `~/projects/*` → `/Users/username/projects/*`
- `$HOME/projects/*` → `/Users/username/projects/*`

**External directory access:**

Use `external_directory` to allow tool calls that touch paths outside the working directory:

```jsonc
{
  "permission": {
    "external_directory": {
      "~/projects/personal/**": "allow"
    }
  }
}
```

**Defaults (if nothing is specified):**

- Most permissions default to `"allow"`
- `doom_loop` and `external_directory` default to `"ask"`
- `read` defaults to `"allow"` but `.env` files are denied:

```jsonc
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

**What "Ask" does:**

When OpenCode prompts for approval, three outcomes are offered:

- `once` — approve just this request
- `always` — approve future requests matching suggested patterns (rest of session)
- `reject` — deny the request

**Agent-level permissions:**

Permissions can be overridden per agent. Agent permissions merge with global config, and agent rules take precedence:

```jsonc
{
  "permission": {
    "bash": {
      "*": "ask",
      "git *": "allow"
    }
  },
  "agent": {
    "build": {
      "permission": {
        "bash": {
          "*": "ask",
          "git commit *": "ask",
          "git push *": "deny"
        }
      }
    }
  }
}
```

### `agent`

Define specialized agents for specific tasks. Each agent can have its own model, prompt, tools, permissions, and other settings.

```jsonc
{
  "agent": {
    "code-reviewer": {
      "description": "Reviews code for best practices and potential issues",
      "model": "anthropic/claude-sonnet-4-5",
      "prompt": "You are a code reviewer. Focus on security, performance, and maintainability.",
      "tools": {
        "write": false,
        "edit": false
      },
      "permission": {
        "edit": "deny"
      }
    }
  }
}
```

Agents can also be defined as markdown files in `~/.config/opencode/agents/` or `.opencode/agents/`:

```markdown
---
description: Code review without edits
mode: subagent
permission:
  edit: deny
  bash: ask
  webfetch: deny
---
Only analyze code and suggest changes.
```

### `default_agent`

Set the default agent used when none is explicitly specified. Must be a primary agent (not a subagent). Can be a built-in agent (`"build"`, `"plan"`) or a custom agent. Falls back to `"build"` with a warning if the specified agent doesn't exist or is a subagent.

```jsonc
{
  "default_agent": "plan"
}
```

This setting applies across all interfaces: TUI, CLI (`opencode run`), desktop app, and GitHub Action.

### `share`

Configure the conversation sharing feature:

```jsonc
{
  "share": "manual"
}
```

| Value | Description |
|-------|-------------|
| `"manual"` | Allow manual sharing via `/share` command (default) |
| `"auto"` | Automatically share new conversations |
| `"disabled"` | Disable sharing entirely |

### `command`

Define custom slash commands for repetitive tasks:

```jsonc
{
  "command": {
    "test": {
      "template": "Run the full test suite with coverage report and show any failures.\nFocus on the failing tests and suggest fixes.",
      "description": "Run tests with coverage",
      "agent": "build",
      "model": "anthropic/claude-haiku-4-5"
    },
    "component": {
      "template": "Create a new React component named $ARGUMENTS with TypeScript support.\nInclude proper typing and basic structure.",
      "description": "Create a new component"
    }
  }
}
```

Commands can also be defined as markdown files in `~/.config/opencode/commands/` or `.opencode/commands/`.

### `formatter`

Enable and configure code formatters. Omit to keep formatters disabled.

```jsonc
{
  "formatter": true
}
```

Use an object to keep built-ins enabled while configuring overrides or custom formatters:

```jsonc
{
  "formatter": {
    "prettier": {
      "disabled": true
    },
    "custom-prettier": {
      "command": ["npx", "prettier", "--write", "$FILE"],
      "environment": {
        "NODE_ENV": "development"
      },
      "extensions": [".js", ".ts", ".jsx", ".tsx"]
    }
  }
}
```

### `lsp`

Enable and configure LSP (Language Server Protocol) servers. Omit to keep LSP disabled.

```jsonc
{
  "lsp": true
}
```

Use an object to keep built-ins enabled while configuring overrides:

```jsonc
{
  "lsp": {
    "typescript": {
      "disabled": true
    }
  }
}
```

### `snapshot`

OpenCode uses snapshots to track file changes during agent operations, enabling undo/revert within a session. Snapshots are enabled by default.

For large repositories or projects with many submodules, disable snapshots to avoid slow indexing and significant disk usage:

```jsonc
{
  "snapshot": false
}
```

Disabling snapshots means changes made by the agent cannot be rolled back through the UI.

### `autoupdate`

OpenCode automatically downloads new updates on startup.

```jsonc
{
  "autoupdate": false
}
```

| Value | Description |
|-------|-------------|
| `true` | Auto-download updates (default) |
| `false` | Disable auto-updates |
| `"notify"` | Notify without auto-downloading (only works if not installed via package manager like Homebrew) |

### `instructions`

Point to instruction/rule files using paths and glob patterns. These files are loaded as system instructions for the LLM:

```jsonc
{
  "instructions": ["CONTRIBUTING.md", "docs/guidelines.md", ".cursor/rules/*.md"]
}
```

### `disabled_providers`

Disable specific providers that would otherwise be loaded automatically:

```jsonc
{
  "disabled_providers": ["openai", "gemini"]
}
```

When a provider is disabled:

- It won't be loaded even if environment variables are set
- It won't be loaded even if API keys are configured via `/connect`
- The provider's models won't appear in the model selection list

### `enabled_providers`

Allowlist specific providers. When set, only the specified providers will be enabled and all others will be ignored:

```jsonc
{
  "enabled_providers": ["anthropic", "openai"]
}
```

`disabled_providers` takes priority over `enabled_providers`. If a provider appears in both, it is disabled.

### `experimental`

Options under active development. Not stable — may change or be removed without notice.

```jsonc
{
  "experimental": {
    "policies": [
      {
        "effect": "deny",
        "action": "provider.use",
        "resource": "openai"
      }
    ]
  }
}
```

**Policies** allow or deny OpenCode actions on configured resources. Currently, policies can control which providers OpenCode may use.

### `attachment.image`

Configure image attachment limits. OpenCode normalizes images before sending to the model. By default, images are resized when they exceed `2000x2000` pixels or `5242880` base64 bytes.

```jsonc
{
  "attachment": {
    "image": {
      "auto_resize": true,
      "max_width": 2000,
      "max_height": 2000,
      "max_base64_bytes": 5242880
    }
  }
}
```

| Option | Type | Description |
|--------|------|-------------|
| `auto_resize` | boolean | Resize images exceeding limits. Set to `false` to reject oversized images instead. |
| `max_width` | number | Maximum image width in pixels before resizing or rejection. |
| `max_height` | number | Maximum image height in pixels before resizing or rejection. |
| `max_base64_bytes` | number | Maximum encoded image payload size (base64 bytes, not original file size). |

If an image still cannot fit after resizing, OpenCode omits oversized tool-result images or fails oversized user-provided images with an image size error.

### `compaction`

Control context compaction behavior:

```jsonc
{
  "compaction": {
    "auto": true,
    "prune": true,
    "reserved": 10000
  }
}
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `auto` | boolean | `true` | Automatically compact the session when context is full. |
| `prune` | boolean | `true` | Remove old tool outputs to save tokens. |
| `reserved` | number | — | Token buffer for compaction. Leaves enough window to avoid overflow during compaction. |

### `watcher`

Configure file watcher ignore patterns:

```jsonc
{
  "watcher": {
    "ignore": ["node_modules/**", "dist/**", ".git/**"]
  }
}
```

Patterns follow glob syntax. Use this to exclude noisy directories from file watching.

### `shell`

Configure the shell used for the interactive terminal and agent tool calls:

```jsonc
{
  "shell": "pwsh"
}
```

If not specified, OpenCode automatically discovers a sensible default based on your OS (`pwsh` or `cmd.exe` on Windows, `/bin/zsh` or `/bin/bash` on macOS/Linux). You can provide an absolute path or a short name.

### `server`

Configure server settings for `opencode serve` and `opencode web` commands:

```jsonc
{
  "server": {
    "port": 4096,
    "hostname": "0.0.0.0",
    "mdns": true,
    "mdnsDomain": "myproject.local",
    "cors": ["http://localhost:5173"]
  }
}
```

| Option | Type | Description |
|--------|------|-------------|
| `port` | number | Port to listen on. |
| `hostname` | string | Hostname to listen on. When `mdns` is enabled and no hostname is set, defaults to `0.0.0.0`. |
| `mdns` | boolean | Enable mDNS service discovery. Allows other devices on the network to discover your OpenCode server. |
| `mdnsDomain` | string | Custom domain name for mDNS service. Defaults to `opencode.local`. Useful for running multiple instances on the same network. |
| `cors` | array | Additional origins to allow for CORS. Values must be full origins (scheme + host + optional port), e.g., `https://app.example.com`. |

---

## TUI Config (`tui.json`)

Use a dedicated `tui.json` (or `tui.jsonc`) file for TUI-specific settings. This is separate from `opencode.json`, which configures server/runtime behavior.

```jsonc
{
  "$schema": "https://opencode.ai/tui.json",
  "theme": "tokyonight",
  "leader_timeout": 2000,
  "scroll_speed": 3,
  "scroll_acceleration": {
    "enabled": false
  },
  "diff_style": "auto",
  "mouse": true,
  "keybinds": {
    "leader": "ctrl+x",
    "command_list": "ctrl+p"
  },
  "attention": {
    "enabled": true,
    "notifications": true,
    "sound": true,
    "volume": 0.4,
    "sound_pack": "opencode.default",
    "sounds": {
      "error": "./sounds/error.mp3"
    }
  }
}
```

Use `OPENCODE_TUI_CONFIG` to point to a custom TUI config file.

Legacy `theme`, `keybinds`, and `tui` keys in `opencode.json` are deprecated and automatically migrated when possible.

### TUI Options

| Option | Type | Description |
|--------|------|-------------|
| `theme` | string | Sets your UI theme. |
| `keybinds` | object | Customizes keyboard shortcuts. Merged with built-in defaults, so only configure shortcuts you want to change. |
| `leader_timeout` | number | How long OpenCode waits after the leader key. Defaults to `2000` ms. |
| `scroll_speed` | number | How fast the TUI scrolls (minimum: `0.001`, supports decimals). Defaults to `3`. Ignored if `scroll_acceleration.enabled` is `true`. |
| `scroll_acceleration.enabled` | boolean | Enable macOS-style scroll acceleration. When enabled, scroll speed increases with rapid scrolling gestures. Takes precedence over `scroll_speed`. |
| `diff_style` | string | Controls diff rendering. `"auto"` adapts to terminal width, `"stacked"` always shows single-column layout. |
| `mouse` | boolean | Enable or disable mouse capture in the TUI (default: `true`). When disabled, terminal's native mouse selection/scrolling behavior is preserved. |
| `attention` | object | Configures TUI desktop notifications and sounds. Disabled by default. |

### Keybinds

All customizable keybinds with their defaults:

```jsonc
{
  "keybinds": {
    "leader": "ctrl+x",
    "app_exit": "ctrl+c,ctrl+d,<leader>q",
    "app_debug": "none",
    "app_console": "none",
    "app_heap_snapshot": "none",
    "app_toggle_animations": "none",
    "app_toggle_file_context": "none",
    "app_toggle_diffwrap": "none",
    "app_toggle_paste_summary": "none",
    "app_toggle_session_directory_filter": "none",
    "command_list": "ctrl+p",
    "help_show": "none",
    "docs_open": "none",
    "editor_open": "<leader>e",
    "theme_list": "<leader>t",
    "theme_switch_mode": "none",
    "theme_mode_lock": "none",
    "sidebar_toggle": "<leader>b",
    "scrollbar_toggle": "none",
    "status_view": "<leader>s",
    "session_export": "<leader>x",
    "session_copy": "none",
    "session_new": "<leader>n",
    "session_list": "<leader>l",
    "session_timeline": "<leader>g",
    "session_fork": "none",
    "session_rename": "ctrl+r",
    "session_delete": "ctrl+d",
    "session_share": "none",
    "session_unshare": "none",
    "session_interrupt": "escape",
    "session_compact": "<leader>c",
    "session_toggle_timestamps": "none",
    "session_toggle_generic_tool_output": "none",
    "session_child_first": "<leader>down",
    "session_child_cycle": "right",
    "session_child_cycle_reverse": "left",
    "session_parent": "up",
    "stash_delete": "ctrl+d",
    "model_provider_list": "ctrl+a",
    "model_favorite_toggle": "ctrl+f",
    "model_list": "<leader>m",
    "model_cycle_recent": "f2",
    "model_cycle_recent_reverse": "shift+f2",
    "model_cycle_favorite": "none",
    "model_cycle_favorite_reverse": "none",
    "mcp_list": "none",
    "provider_connect": "none",
    "console_org_switch": "none",
    "agent_list": "<leader>a",
    "agent_cycle": "tab",
    "agent_cycle_reverse": "shift+tab",
    "variant_cycle": "ctrl+t",
    "variant_list": "none",
    "messages_page_up": "pageup,ctrl+alt+b",
    "messages_page_down": "pagedown,ctrl+alt+f",
    "messages_line_up": "ctrl+alt+y",
    "messages_line_down": "ctrl+alt+e",
    "messages_half_page_up": "ctrl+alt+u",
    "messages_half_page_down": "ctrl+alt+d",
    "messages_first": "ctrl+g,home",
    "messages_last": "ctrl+alt+g,end",
    "messages_next": "none",
    "messages_previous": "none",
    "messages_last_user": "none",
    "messages_copy": "<leader>y",
    "messages_undo": "<leader>u",
    "messages_redo": "<leader>r",
    "messages_toggle_conceal": "<leader>h",
    "tool_details": "none",
    "display_thinking": "none",
    "prompt_submit": "none",
    "prompt_editor_context_clear": "none",
    "prompt_skills": "none",
    "prompt_stash": "none",
    "prompt_stash_pop": "none",
    "prompt_stash_list": "none",
    "workspace_set": "none",
    "input_clear": "ctrl+c",
    "input_paste": {
      "key": "ctrl+v",
      "preventDefault": false
    },
    "input_submit": "return",
    "input_newline": "shift+return,ctrl+return,alt+return,ctrl+j",
    "input_move_left": "left,ctrl+b",
    "input_move_right": "right,ctrl+f",
    "input_move_up": "up",
    "input_move_down": "down",
    "input_select_left": "shift+left",
    "input_select_right": "shift+right",
    "input_select_up": "shift+up",
    "input_select_down": "shift+down",
    "input_line_home": "ctrl+a",
    "input_line_end": "ctrl+e",
    "input_select_line_home": "ctrl+shift+a",
    "input_select_line_end": "ctrl+shift+e",
    "input_visual_line_home": "alt+a",
    "input_visual_line_end": "alt+e",
    "input_select_visual_line_home": "alt+shift+a",
    "input_select_visual_line_end": "alt+shift+e",
    "input_buffer_home": "home",
    "input_buffer_end": "end",
    "input_select_buffer_home": "shift+home",
    "input_select_buffer_end": "shift+end",
    "input_delete_line": "ctrl+shift+d",
    "input_delete_to_line_end": "ctrl+k",
    "input_delete_to_line_start": "ctrl+u",
    "input_backspace": "backspace,shift+backspace",
    "input_delete": "ctrl+d,delete,shift+delete",
    "input_undo": "ctrl+-,super+z",
    "input_redo": "ctrl+.,super+shift+z",
    "input_word_forward": "alt+f,alt+right,ctrl+right",
    "input_word_backward": "alt+b,alt+left,ctrl+left",
    "input_select_word_forward": "alt+shift+f,alt+shift+right",
    "input_select_word_backward": "alt+shift+b,alt+shift+left",
    "input_delete_word_forward": "alt+d,alt+delete,ctrl+delete",
    "input_delete_word_backward": "ctrl+w,ctrl+backspace,alt+backspace",
    "input_select_all": "super+a",
    "history_previous": "up",
    "history_next": "down",
    "dialog.select.prev": "up,ctrl+p",
    "dialog.select.next": "down,ctrl+n",
    "dialog.select.page_up": "pageup",
    "dialog.select.page_down": "pagedown",
    "dialog.select.home": "home",
    "dialog.select.end": "end",
    "dialog.select.submit": "return",
    "dialog.prompt.submit": "return",
    "dialog.mcp.toggle": "space",
    "prompt.autocomplete.prev": "up,ctrl+p",
    "prompt.autocomplete.next": "down,ctrl+n",
    "prompt.autocomplete.hide": "escape",
    "prompt.autocomplete.select": "return",
    "prompt.autocomplete.complete": "tab",
    "permission.prompt.fullscreen": "ctrl+f",
    "plugins.toggle": "space",
    "dialog.plugins.install": "shift+i",
    "terminal_suspend": "ctrl+z",
    "terminal_title_toggle": "none",
    "tips_toggle": "<leader>h",
    "plugin_manager": "none",
    "plugin_install": "none",
    "which_key_toggle": "ctrl+alt+k",
    "which_key_layout_toggle": "ctrl+alt+shift+k",
    "which_key_pending_toggle": "ctrl+alt+shift+p",
    "which_key_group_previous": "ctrl+alt+left,ctrl+alt+[",
    "which_key_group_next": "ctrl+alt+right,ctrl+alt+]",
    "which_key_scroll_up": "ctrl+alt+up,ctrl+alt+p",
    "which_key_scroll_down": "ctrl+alt+down,ctrl+alt+n",
    "which_key_page_up": "ctrl+alt+pageup",
    "which_key_page_down": "ctrl+alt+pagedown",
    "which_key_home": "ctrl+alt+home",
    "which_key_end": "ctrl+alt+end"
  }
}
```

**Windows defaults differ for:**

- `input_undo` defaults to `ctrl+z,ctrl+-,super+z` (adds `ctrl+z` because Windows terminals don't support POSIX suspend)
- `terminal_suspend` is forced to `none` (native Windows terminals don't support POSIX suspend)

**Binding values:**

A string can contain one shortcut or multiple comma-separated shortcuts. Use an array for multiple shortcuts:

```jsonc
{
  "keybinds": {
    "messages_copy": ["<leader>y", "ctrl+shift+c"]
  }
}
```

For advanced cases, use an object with `key`, `event`, `preventDefault`, or `fallthrough`:

```jsonc
{
  "keybinds": {
    "input_paste": {
      "key": "ctrl+v",
      "preventDefault": false
    }
  }
}
```

**Disable a keybind** by setting it to `"none"` or `false`:

```jsonc
{
  "keybinds": {
    "session_compact": "none"
  }
}
```

### Attention (Notifications & Sound)

The `attention` section controls TUI desktop notifications and sounds. Disabled by default.

```jsonc
{
  "attention": {
    "enabled": true,
    "notifications": true,
    "sound": true,
    "volume": 0.4,
    "sound_pack": "opencode.default",
    "sounds": {
      "default": "./sounds/default.mp3",
      "question": "./sounds/question.mp3",
      "permission": "./sounds/permission.mp3",
      "error": "./sounds/error.mp3",
      "done": "./sounds/done.mp3",
      "subagent_done": "./sounds/subagent_done.mp3"
    }
  }
}
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enabled` | boolean | `false` | Master switch for all attention features. |
| `notifications` | boolean | `true` | Allow terminal-mediated desktop notifications when attention is enabled. |
| `sound` | boolean | `true` | Allow attention sounds when attention is enabled. |
| `volume` | number | `0.4` | Sound volume from `0` to `1`. |
| `sound_pack` | string | `"opencode.default"` | Sound pack ID to use. |
| `sounds` | object | `{}` | Override individual sound files for `default`, `question`, `permission`, `error`, `done`, or `subagent_done`. Paths can be absolute, `file://` URLs, or relative to `tui.json`. |

Built-in events play sounds when triggered, and non-subagent events request desktop notifications only when the terminal is blurred.

---

## Variable Substitution

Use variable substitution in config files to reference environment variables and file contents.

### Environment Variables

Use `{env:VARIABLE_NAME}` to substitute environment variables:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "{env:OPENCODE_MODEL}",
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "{env:ANTHROPIC_API_KEY}"
      }
    }
  }
}
```

If the environment variable is not set, it is replaced with an empty string.

### File Contents

Use `{file:path/to/file}` to substitute the contents of a file:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": ["./custom-instructions.md"],
  "provider": {
    "openai": {
      "options": {
        "apiKey": "{file:~/.secrets/openai-key}"
      }
    }
  }
}
```

File paths can be:

- Relative to the config file directory
- Absolute paths starting with `/` or `~`

Useful for:

- Keeping sensitive data like API keys in separate files
- Including large instruction files without cluttering your config
- Sharing common configuration snippets across multiple config files

---

## Managed Settings (Enterprise)

Organizations can enforce configuration that users cannot override. Managed settings are loaded at the highest priority tier (tier 7-8), overriding everything else.

### File-based

Drop an `opencode.json` or `opencode.jsonc` file in the system managed config directory:

| Platform | Path |
|----------|------|
| macOS | `/Library/Application Support/opencode/` |
| Linux | `/etc/opencode/` |
| Windows | `%ProgramData%\opencode` |

These directories require admin/root access to write, so users cannot modify them.

### macOS Managed Preferences (`.mobileconfig` via MDM)

On macOS, OpenCode reads managed preferences from the `ai.opencode.managed` preference domain. Deploy a `.mobileconfig` via MDM (Jamf, Kandji, FleetDM) and the settings are enforced automatically.

OpenCode checks these paths:

1. `/Library/Managed Preferences/<user>/ai.opencode.managed.plist`
2. `/Library/Managed Preferences/ai.opencode.managed.plist`

The plist keys map directly to `opencode.json` fields. MDM metadata keys (`PayloadUUID`, `PayloadType`, etc.) are stripped automatically.

**Creating a `.mobileconfig`:**

Use the `ai.opencode.managed` PayloadType. The OpenCode config keys go directly in the payload dict:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>PayloadContent</key>
  <array>
    <dict>
      <key>PayloadType</key>
      <string>ai.opencode.managed</string>
      <key>PayloadIdentifier</key>
      <string>com.example.opencode.config</string>
      <key>PayloadUUID</key>
      <string>GENERATE-YOUR-OWN-UUID</string>
      <key>PayloadVersion</key>
      <integer>1</integer>
      <key>share</key>
      <string>disabled</string>
      <key>server</key>
      <dict>
        <key>hostname</key>
        <string>127.0.0.1</string>
      </dict>
      <key>permission</key>
      <dict>
        <key>*</key>
        <string>ask</string>
        <key>bash</key>
        <dict>
          <key>*</key>
          <string>ask</string>
          <key>rm -rf *</key>
          <string>deny</string>
        </dict>
      </dict>
    </dict>
  </array>
  <key>PayloadType</key>
  <string>Configuration</string>
  <key>PayloadIdentifier</key>
  <string>com.example.opencode</string>
  <key>PayloadUUID</key>
  <string>GENERATE-YOUR-OWN-UUID</string>
  <key>PayloadVersion</key>
  <integer>1</integer>
</dict>
</plist>
```

Generate unique UUIDs with `uuidgen`. Customize the settings to match your organization's requirements.

**Deploying via MDM:**

- **Jamf Pro:** Computers > Configuration Profiles > Upload > scope to target devices or smart groups
- **Kandji:** Custom Profiles > upload the `.mobileconfig`
- **FleetDM:** Add the `.mobileconfig` to your gitops repo under `mdm.macos_settings.custom_settings` and run `fleetctl apply`

**Verifying on a device:**

```bash
opencode debug config
```

All managed preference keys appear in the resolved config and cannot be overridden by user or project configuration.

---

## Provider-Specific Options

### Amazon Bedrock

Amazon Bedrock supports AWS-specific configuration:

```jsonc
{
  "provider": {
    "amazon-bedrock": {
      "options": {
        "region": "us-east-1",
        "profile": "my-aws-profile",
        "endpoint": "https://bedrock-runtime.us-east-1.vpce-xxxxx.amazonaws.com"
      }
    }
  }
}
```

| Option | Description |
|--------|-------------|
| `region` | AWS region for Bedrock (defaults to `AWS_REGION` env var or `us-east-1`). |
| `profile` | AWS named profile from `~/.aws/credentials` (defaults to `AWS_PROFILE` env var). |
| `endpoint` | Custom endpoint URL for VPC endpoints. Alias for generic `baseURL` using AWS-specific terminology. If both specified, `endpoint` takes precedence. |

**Authentication methods:**

- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`: IAM user access keys
- `AWS_PROFILE`: Named profiles from `~/.aws/credentials`
- `AWS_BEARER_TOKEN_BEDROCK`: Long-term API keys from the Bedrock console
- `AWS_WEB_IDENTITY_TOKEN_FILE` / `AWS_ROLE_ARN`: EKS IRSA (IAM Roles for Service Accounts)

**Authentication precedence:**

1. **Bearer Token** — `AWS_BEARER_TOKEN_BEDROCK` environment variable or token from `/connect` command
2. **AWS Credential Chain** — Profile, access keys, shared credentials, IAM roles, Web Identity Tokens (EKS IRSA), instance metadata

Bearer tokens take precedence over all AWS credential methods including configured profiles.

**Custom inference profiles:**

```jsonc
{
  "provider": {
    "amazon-bedrock": {
      "models": {
        "anthropic-claude-sonnet-4.5": {
          "id": "arn:aws:bedrock:us-east-1:xxx:application-inference-profile/yyy"
        }
      }
    }
  }
}
```

### Generic Provider Options (All Providers)

| Option | Type | Description |
|--------|------|-------------|
| `baseURL` | string | Custom base URL for the provider. Useful for proxies, gateways, or local servers. |
| `timeout` | number \| false | Request timeout in milliseconds (default: 300000). Set to `false` to disable. |
| `chunkTimeout` | number | Timeout between streamed response chunks. |
| `setCacheKey` | boolean | Ensure a cache key is always set. |
| `headers` | object | Custom HTTP headers for requests. |

---

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `OPENCODE_CONFIG` | Path to custom config file |
| `OPENCODE_CONFIG_DIR` | Path to custom config directory |
| `OPENCODE_CONFIG_CONTENT` | Inline JSON config string |
| `OPENCODE_TUI_CONFIG` | Path to custom TUI config file |
| `OPENCODE_SERVER_PASSWORD` | Password for server authentication |
| `OPENCODE_SERVER_USERNAME` | Username for server authentication |
| `OPENCODE_PORT` | Port for the server |
| `OPENCODE_ENABLE_EXA` | Enable Exa integration |
| `OPENCODE_EXPERIMENTAL_LSP_TOOL` | Enable experimental LSP tool |
| `OPENCODE_EXPERIMENTAL` | Enable experimental features |
| `OPENCODE_DISABLE_LSP_DOWNLOAD` | Disable automatic LSP server downloads |
| `OPENCODE_DISABLE_CLAUDE_CODE` | Disable Claude Code integration |
| `OPENCODE_DISABLE_CLAUDE_CODE_PROMPT` | Disable Claude Code prompt |
| `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS` | Disable Claude Code skills |
| `AUTO_SHARE` | Enable automatic sharing |
| `AWS_REGION` | AWS region for Bedrock |
| `AWS_PROFILE` | AWS profile for Bedrock |
| `AWS_BEARER_TOKEN_BEDROCK` | Bearer token for Bedrock |
| `AWS_ACCESS_KEY_ID` | AWS access key ID |
| `AWS_SECRET_ACCESS_KEY` | AWS secret access key |
| `AWS_WEB_IDENTITY_TOKEN_FILE` | Web identity token file for EKS IRSA |
| `AWS_ROLE_ARN` | Role ARN for EKS IRSA |
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `OPENAI_API_KEY` | OpenAI API key |
| `AZURE_RESOURCE_NAME` | Azure OpenAI resource name |
| `AZURE_COGNITIVE_SERVICES_RESOURCE_NAME` | Azure Cognitive Services resource name |
| `GOOGLE_CLOUD_PROJECT` | Google Cloud project ID for Vertex AI |
| `GOOGLE_APPLICATION_CREDENTIALS` | Path to service account JSON key for Vertex AI |
| `VERTEX_LOCATION` | Region for Vertex AI (defaults to `global`) |
| `GITLAB_INSTANCE_URL` | GitLab instance URL for self-hosted |
| `GITLAB_TOKEN` | GitLab personal access token |
| `GITLAB_AI_GATEWAY_URL` | Custom AI Gateway URL for self-hosted GitLab |
| `GITLAB_OAUTH_CLIENT_ID` | OAuth client ID for self-hosted GitLab |
| `DIGITALOCEAN_ACCESS_TOKEN` | DigitalOcean Model Access Key |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare account ID |
| `CLOUDFLARE_API_KEY` | Cloudflare API key |
| `CLOUDFLARE_GATEWAY_ID` | Cloudflare AI Gateway ID |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token |
| `NVIDIA_API_KEY` | NVIDIA API key |
| `AICORE_SERVICE_KEY` | SAP AI Core service key JSON |
| `AICORE_DEPLOYMENT_ID` | SAP AI Core deployment ID |
| `AICORE_RESOURCE_GROUP` | SAP AI Core resource group |
| `CONTEXT7_API_KEY` | Context7 MCP API key |

---

## Debugging & Verification

Use the debug command to verify your resolved configuration:

```bash
opencode debug config
```

This displays the fully merged configuration from all sources, including managed preferences. Useful for:

- Troubleshooting precedence issues
- Confirming that MDM-deployed settings are active
- Verifying variable substitution resolved correctly
- Checking which providers are enabled/disabled
- Inspecting resolved permission rules

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Config not taking effect** | Run `opencode debug config` to see the resolved configuration and verify precedence. |
| **Managed settings not applying** | Check that the `.mobileconfig` is installed (System Settings > Privacy & Security > Profiles on macOS) and that the `PayloadType` is `ai.opencode.managed`. |
| **Provider not loading** | Check `disabled_providers` and `enabled_providers` — `disabled_providers` takes priority. Also verify API keys are set via `/connect` or environment variables. |
| **Variable not resolving** | Ensure the environment variable is set in the same shell session. `{env:VAR}` replaces with empty string if unset. |
| **File reference failing** | Verify the path is relative to the config file directory, or use an absolute path starting with `/` or `~`. |
| **TUI theme not applying** | Use `tui.json`, not `opencode.json`. Legacy `theme` key in `opencode.json` is deprecated. |
| **MCP server tools not appearing** | Check `tools` config — MCP tools can be disabled globally with glob patterns like `"my-mcp*": false`. |
| **Permission denied for external paths** | Add `external_directory` rules to allow access to paths outside the project root. |
| **Array config not merging as expected** | Arrays are replaced, not appended. Later config sources replace entire arrays from earlier sources. |
| **Plugins not loading** | Check that npm plugins are in the `plugin` array and local plugins are in `.opencode/plugins/` or `~/.config/opencode/plugins/`. |
| **Keybind not working** | Verify the keybind value is not set to `"none"`. Check for conflicts with terminal keybindings. |
| **Agent not showing up** | Ensure it's defined as a primary agent (not a subagent) if using `default_agent`. Check that agent markdown files have valid frontmatter. |
