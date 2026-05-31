---
name: opencode-permissions
description: |
  Use this skill when configuring OpenCode's permission system — controlling which tools require approval, setting granular bash command rules, managing external directory access, configuring per-agent permissions, or understanding the allow/ask/deny permission model. Covers wildcards, pattern matching, doom loop detection, and the complete permission key reference.
---

# OpenCode Permissions

> **📚 Official Docs:** For the latest information, always refer to the official documentation:
> [https://opencode.ai/docs/permissions/](https://opencode.ai/docs/permissions/)

Control which actions require approval to run in OpenCode.

OpenCode uses the `permission` config to decide whether a given action should run automatically, prompt you, or be blocked. Permissions control what tools can do during a session.

---

## Actions

Each permission rule resolves to one of three actions:

| Action | Behavior |
|--------|----------|
| `"allow"` | Run without approval |
| `"ask"` | Prompt for approval before running |
| `"deny"` | Block the action entirely |

---

## Configuration

### Set All Permissions at Once

You can set a global default for all permissions:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": "allow"
}
```

### Set Per-Tool Permissions

Override specific tools while keeping a global default:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "*": "ask",
    "bash": "allow",
    "edit": "deny"
  }
}
```

---

## Granular Rules (Object Syntax)

For most permissions, you can use an object to apply different actions based on the tool input. Rules are evaluated by pattern match, with the **last matching rule winning**.

### Basic Pattern

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "bash": {
      "*": "ask",
      "git *": "allow",
      "npm *": "allow",
      "rm *": "deny",
      "grep *": "allow"
    },
    "edit": {
      "*": "deny",
      "packages/web/src/content/docs/*.mdx": "allow"
    }
  }
}
```

In this example:
- `git status` → last matching rule is `"git *": "allow"` → allowed
- `rm -rf /` → last matching rule is `"rm *": "deny"` → denied
- `npm install` → last matching rule is `"npm *": "allow"` → allowed
- Edit to `.mdx` files → allowed; edit to other files → denied

### Wildcards

Permission patterns use simple wildcard matching:

| Pattern | Matches |
|---------|---------|
| `*` | Zero or more of any character |
| `?` | Exactly one character |
| All other characters | Match literally |

Examples:
- `"git *"` matches `git status`, `git commit -m "msg"`, `git push origin main`
- `"git ?*"` matches `git status` but not `git` alone
- `"rm *"` matches `rm file.txt`, `rm -rf /tmp/dir`

### Home Directory Expansion

You can use `~` or `$HOME` at the start of a pattern to reference your home directory:

| Pattern | Expands To |
|---------|-----------|
| `~/projects/*` | `/Users/username/projects/*` |
| `$HOME/projects/*` | `/Users/username/projects/*` |
| `~` | `/Users/username` |

This is particularly useful for `external_directory` rules.

---

## Available Permissions

OpenCode permissions are keyed by tool name, plus a couple of safety guards:

| Key | Tools It Gates | Matching Input |
|-----|---------------|----------------|
| `read` | `read` | File path |
| `edit` | `write`, `edit`, `apply_patch` | File path |
| `glob` | `glob` | Glob pattern |
| `grep` | `grep` | Regex pattern |
| `bash` | `bash` | Parsed command (e.g., `git status --porcelain`) |
| `task` | `task` | Subagent type |
| `skill` | `skill` | Skill name |
| `lsp` | `lsp` | (non-granular currently) |
| `question` | `question` | (non-granular) |
| `webfetch` | `webfetch` | URL |
| `websearch` | `websearch` | Query |
| `external_directory` | Any tool reading/writing outside project | Path |
| `todowrite` | `todowrite`, `todoread` | (non-granular) |
| `doom_loop` | Recovery when same tool call repeats 3x with identical input | (non-granular) |

The `read`, `edit`, `glob`, `grep`, `list`, `bash`, `task`, `external_directory`, `lsp`, and `skill` keys accept either a shorthand action (`"allow" | "ask" | "deny"`) or an object of glob/pattern → action for fine-grained control. The remaining keys accept the shorthand action only.

---

## Defaults

If you don't specify anything, OpenCode starts from permissive defaults:

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

---

## What "Ask" Does

When OpenCode prompts for approval, the UI offers three outcomes:

| Outcome | Behavior |
|---------|----------|
| `once` | Approve just this request |
| `always` | Approve future requests matching the suggested patterns (for the rest of the current OpenCode session) |
| `reject` | Deny the request |

The set of patterns that `always` would approve is provided by the tool (for example, bash approvals typically whitelist a safe command prefix like `git status*`).

---

## External Directories

Use `external_directory` to allow tool calls that touch paths outside the working directory where OpenCode was started. This applies to any tool that takes a path as input (e.g., `read`, `edit`, `glob`, `grep`, and many `bash` commands).

Home expansion (`~/...`) only affects how a pattern is written. It does not make an external path part of the current workspace, so paths outside the working directory must still be allowed via `external_directory`.

### Allow Access to a Directory

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "external_directory": {
      "~/projects/personal/**": "allow"
    }
  }
}
```

### Layer Additional Rules

Any directory allowed here inherits the same defaults as the current workspace. Since `read` defaults to `allow`, reads are also allowed for entries under `external_directory` unless overridden:

```json
{
  "$schema": "https://opencode.ai/config.json",
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

This allows reading files in `~/projects/personal/` but blocks editing them.

---

## Agent Permissions

You can override permissions per agent. Agent permissions are merged with the global config, and agent rules take precedence.

### JSON Configuration

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "bash": {
      "*": "ask",
      "git *": "allow",
      "git commit *": "deny",
      "git push *": "deny",
      "grep *": "allow"
    }
  },
  "agent": {
    "build": {
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
  }
}
```

### Markdown Agent Configuration

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

### Bash Permission Patterns

Use pattern matching for commands with arguments. `"grep *"` allows `grep pattern file.txt`, while `"grep"` alone would block it. Commands like `git status` work for default behavior but require explicit permission (like `"git status *"`) when arguments are passed.

```json
{
  "permission": {
    "bash": {
      "*": "ask",
      "git status *": "allow",
      "git diff *": "allow",
      "git log *": "allow",
      "npm test *": "allow",
      "npm run *": "allow",
      "rm *": "deny"
    }
  }
}
```

---

## Complete Examples

### Restrictive Setup

Everything requires approval except safe read operations:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "*": "ask",
    "read": "allow",
    "glob": "allow",
    "grep": "allow",
    "skill": "allow",
    "question": "allow"
  }
}
```

### Permissive with Specific Denials

Allow everything except dangerous operations:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "*": "allow",
    "bash": {
      "*": "allow",
      "rm -rf *": "deny",
      "sudo *": "deny",
      "git push --force *": "deny"
    },
    "external_directory": "ask"
  }
}
```

### Development Agent with Git Restrictions

```json
{
  "agent": {
    "build": {
      "permission": {
        "bash": {
          "*": "allow",
          "git push *": "ask",
          "git commit *": "ask",
          "rm *": "ask"
        },
        "edit": {
          "*.env": "deny",
          "*.env.*": "deny",
          "*": "allow"
        }
      }
    }
  }
}
```

### Read-Only Review Agent

```json
{
  "agent": {
    "reviewer": {
      "mode": "subagent",
      "permission": {
        "edit": "deny",
        "bash": {
          "*": "deny",
          "git diff *": "allow",
          "git log *": "allow",
          "grep *": "allow",
          "find *": "allow"
        },
        "webfetch": "allow",
        "skill": "allow"
      }
    }
  }
}
```

---

## MCP Tool Permissions

You can control permissions for MCP server tools using glob patterns on the tool name prefix:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "sentry_*": "allow",
    "context7_*": "ask",
    "github_*": {
      "*": "ask",
      "github_list_repos *": "allow",
      "github_create_issue *": "deny"
    }
  }
}
```

---

## Doom Loop Detection

The `doom_loop` permission triggers when the same tool call repeats 3 times with identical input. This is a safety mechanism to prevent infinite loops. It defaults to `"ask"`, giving you the option to break out of the loop.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "doom_loop": "deny"
  }
}
```

---

## Troubleshooting

### Tool Blocked Unexpectedly

1. Check the global permission config in `opencode.json`
2. Check agent-specific permissions (agent rules take precedence)
3. Check if the tool matches a deny pattern
4. Use `opencode debug config` to see the resolved permission config

### Permission Prompts Too Frequent

1. Use the `always` option in the permission dialog to whitelist patterns
2. Configure explicit `"allow"` rules for safe commands
3. Use object syntax to allow specific command patterns

### .env Files Blocked

This is the default behavior. The `read` permission denies `*.env` and `*.env.*` by default. Override with:

```json
{
  "permission": {
    "read": {
      "*": "allow",
      "*.env": "allow"
    }
  }
}
```

> **Warning:** Allowing `.env` reads means the LLM can see your secrets. Only do this if you understand the security implications.

---

## Key Points

1. **Last match wins** — Put broad rules first, specific exceptions after
2. **Agent overrides global** — Agent-specific permissions take precedence
3. **`.env` denied by default** — Override explicitly if needed
4. **`external_directory` and `doom_loop` default to `"ask"`** — Everything else defaults to `"allow"`
5. **Patterns use simple globs** — `*` matches anything, `?` matches one character
6. **`always` in the UI** — Whitelists patterns for the rest of the session
7. **MCP tools use prefix matching** — `servername_*` matches all tools from that server
