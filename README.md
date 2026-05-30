# opencode-builder

A Kilo/agent skill for building extensions, plugins, integrations, and custom developer tools on top of [OpenCode](https://opencode.ai).

## What This Skill Covers

- **SDK Path** — Programmatic control via `@opencode-ai/sdk` for external apps, scripts, and services
- **Plugin Path** — Extending OpenCode's behavior with `@opencode-ai/plugin`
- **Configuration** — `opencode.json` schema and TypeScript config
- **Common Patterns** — Background services, notifications, file protection, MCP, custom model providers
- **Advanced Orchestration** — Multi-agent workflows, parallel execution, checkpoint/resume, adversarial review
- **Debugging** — Troubleshooting plugins, SDK connections, TypeScript errors

## Installation

### For Kilo
Copy the `SKILL.md` and `references/` directory to your `.agents/skills/opencode-builder/` directory.

### For Claude Code / Claude.ai
Copy the `SKILL.md` file to your project's `.claude/skills/` directory.

## Usage

This skill is automatically triggered when you ask about:
- Building OpenCode plugins
- Using the OpenCode SDK
- Creating MCP integrations for OpenCode
- Configuring opencode.json
- Advanced OpenCode orchestration patterns

## Reference Files

| File | Description |
|------|-------------|
| `references/sdk-api.md` | SDK API documentation |
| `references/plugin-reference.md` | Plugin architecture reference |
| `references/examples.md` | Code examples and patterns |
| `references/advanced-orchestration.md` | Multi-agent workflow patterns |

## License

MIT