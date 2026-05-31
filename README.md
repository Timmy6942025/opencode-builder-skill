# OpenCode Agent Skills

A comprehensive collection of **14 agent skills** for building on top of [OpenCode](https://opencode.ai) — the open-source AI coding agent. These skills cover everything from plugin development and SDK usage to configuration, providers, MCP servers, agents, tools, and more.

Compatible with OpenCode, Kilo, Claude Code, Cursor, Codex, and any tool that supports the open [Agent Skills](https://agentskills.io/) standard.

## What's Included

| Skill | Description |
|-------|-------------|
| `opencode-builder` | Core skill for building extensions, plugins, and integrations on OpenCode |
| `opencode-config` | Complete configuration guide: opencode.json, tui.json, precedence, variables, managed settings |
| `opencode-providers` | All 75+ LLM providers, custom providers, OpenCode Zen, OpenCode Go, local models |
| `opencode-mcp` | Model Context Protocol servers: local/remote, OAuth, tool management |
| `opencode-agents` | Agent system: primary/subagents, configuration, permissions, markdown agents |
| `opencode-tools` | Built-in tools, custom tools, permission system, tool internals |
| `opencode-plugins` | Plugin hooks, events, loading, dependencies, real-world patterns |
| `opencode-sdk` | JavaScript/TypeScript SDK: sessions, files, TUI, events, structured output |
| `opencode-server` | HTTP server, API endpoints, web interface, mDNS discovery |
| `opencode-github` | GitHub Actions integration: issue triage, PR review, CI/CD workflows |
| `opencode-commands` | Custom slash commands: markdown/JSON, arguments, shell output injection |
| `opencode-permissions` | Granular permission system: wildcards, bash rules, external directories |
| `opencode-troubleshooting` | Logs, storage, common issues, debugging tools |
| `opencode-skills` | Agent Skills system: SKILL.md format, discovery, writing, publishing |

## Installation

### For OpenCode / Kilo

Copy the skills you want to your `.agents/skills/` or `.opencode/skills/` directory:

```bash
# Example: install the builder skill
cp -r skills/opencode-builder ~/.agents/skills/
```

### For Claude Code

Copy to your `.claude/skills/` directory:

```bash
cp -r skills/opencode-builder .claude/skills/
```

### For Cursor / Codex / Other Agents

Copy to your agent's skills directory. Each skill is independent — install only what you need.

## Skill Structure

Each skill follows the standard Agent Skills format:

```
skills/
├── opencode-builder/
│   └── SKILL.md
├── opencode-config/
│   └── SKILL.md
├── opencode-providers/
│   └── SKILL.md
└── ...
```

Every `SKILL.md` includes:
- YAML frontmatter with name and description
- Comprehensive documentation
- Practical code examples
- Configuration references
- Best practices and troubleshooting

## Source

All documentation is sourced from the official [OpenCode docs](https://opencode.ai/docs/) and verified against the latest version.

## License

MIT
