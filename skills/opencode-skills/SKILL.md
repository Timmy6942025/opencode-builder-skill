---
name: opencode-skills
description: Create, configure, and troubleshoot OpenCode agent skills (SKILL.md definitions). Covers file locations, discovery, frontmatter spec, permissions, per-agent overrides, and publishing.
---

## Overview

OpenCode agent skills are reusable instruction sets defined as `SKILL.md` files. Agents see available skills listed in the `skill` tool description and can load full content on demand by calling the skill tool. Skills let you encapsulate domain knowledge, workflows, and best practices that agents can discover and use without bloating every conversation with their full content.

Skills are:
- **On-demand loaded**: agents see only names and descriptions until they explicitly load a skill
- **Project-local or global**: scoped to a repository or shared across all projects
- **Permission-controlled**: restrict which agents can load which skills via pattern-based rules
- **Claude-compatible**: share a directory convention with Claude Code skills

---

## File Locations

OpenCode searches for `SKILL.md` files in these directories. Each skill lives in its own subdirectory named after the skill.

### Project-local (in your repository)

| Path | Description |
|---|---|
| `.opencode/skills/<name>/SKILL.md` | Primary OpenCode project skill |
| `.claude/skills/<name>/SKILL.md` | Claude Code-compatible project skill |
| `.agents/skills/<name>/SKILL.md` | Agent-compatible project skill |

### Global (shared across all projects)

| Path | Description |
|---|---|
| `~/.config/opencode/skills/<name>/SKILL.md` | Primary OpenCode global skill |
| `~/.claude/skills/<name>/SKILL.md` | Claude Code-compatible global skill |
| `~/.agents/skills/<name>/SKILL.md` | Agent-compatible global skill |

**Naming convention**: The directory name `<name>` must match the `name` field in the YAML frontmatter.

---

## Discovery

### Project-local discovery

OpenCode walks up from your current working directory (CWD) until it reaches the git worktree root. During this traversal it loads:

1. `skills/*/SKILL.md` from every `.opencode/` directory found along the path
2. `skills/*/SKILL.md` from every `.claude/` directory found along the path
3. `skills/*/SKILL.md` from every `.agents/` directory found along the path

This means a skill placed in `.opencode/skills/` at the repository root is available to all sessions in that repo, while a skill in a subdirectory is only available when that directory (or a child) is the CWD.

### Global discovery

Global skills are loaded from:
- `~/.config/opencode/skills/*/SKILL.md`
- `~/.claude/skills/*/SKILL.md`
- `~/.agents/skills/*/SKILL.md`

Global skills are available in every project regardless of CWD.

### Deduplication

If a skill name appears in multiple locations, the first match found during discovery wins. Project-local skills take precedence over global skills.

---

## SKILL.md Frontmatter

Every `SKILL.md` **must** begin with YAML frontmatter delimited by `---` lines. Only the following fields are recognized; unknown fields are silently ignored.

### Required fields

| Field | Type | Constraints | Description |
|---|---|---|---|
| `name` | string | 1–64 chars, regex `^[a-z0-9]+(-[a-z0-9]+)*$` | Unique skill identifier. Must match the directory name containing `SKILL.md`. |
| `description` | string | 1–1024 chars | Human-readable summary shown to agents. Used for trigger accuracy—be specific. |

### Optional fields

| Field | Type | Description |
|---|---|---|
| `license` | string | SPDX license identifier (e.g., `MIT`, `Apache-2.0`). Informational only. |
| `compatibility` | string | Target platform (e.g., `opencode`, `claude-code`). Informational only. |
| `metadata` | map\<string, string\> | Arbitrary key-value pairs for categorization (e.g., `audience`, `workflow`, `category`). |

### Name validation rules

- Must be 1 to 64 characters long
- Must be lowercase alphanumeric with single hyphen separators
- Must not start or end with a hyphen
- Must not contain consecutive hyphens (`--`)
- Must match the directory name that contains the `SKILL.md` file

Regex pattern:

```
^[a-z0-9]+(-[a-z0-9]+)*$
```

Valid names: `git-release`, `react-testing`, `api-v2`, `k8s-deploy`
Invalid names: `-git`, `Git-Release`, `api--v2`, `my skill name`

### Description guidelines

The `description` field is what the agent sees in the tool description alongside the skill name. It must be:

- **Specific enough** for the agent to correctly decide when to load the skill
- **Concise** but complete—cover the scope without fluff
- **Action-oriented** where possible (e.g., "Create consistent releases and changelogs" vs "A skill about releases")

---

## Tool Description (How Skills Appear to Agents)

When OpenCode starts, it injects available skills into the `skill` tool's description as an XML block. Agents see this before deciding which skills to load.

Example tool description:

```
Skills provide specialized instructions and workflows for specific tasks.
Use the skill tool to load a skill when a task matches its description.

<available_skills>
  <skill>
    <name>git-release</name>
    <description>Create consistent releases and changelogs</description>
  </skill>
  <skill>
    <name>react-testing</name>
    <description>Write and run React component tests with Vitest</description>
  </skill>
</available_skills>
```

The agent reads this section to decide which skills are relevant to the current conversation, then loads them on demand.

---

## Loading

### How agents load skills

Agents call the skill tool with the skill name:

```javascript
skill({ name: "git-release" })
```

The tool returns the full content of the `SKILL.md` file (after the frontmatter), injected into the conversation context as a `<skill_content>` block:

```xml
<skill_content name="git-release">
## What I do
- Draft release notes from merged PRs
- Propose a version bump
- Provide a copy-pasteable `gh release create` command

## When to use me
Use this when you are preparing a tagged release.
Ask clarifying questions if the target versioning scheme is unclear.
</skill_content>
```

### Loading behavior

- Loading is **synchronous**—the agent receives the full content immediately
- Skills can be loaded **multiple times** if the agent needs a refresh
- Loading a skill **does not** automatically execute anything—it merely provides instructions to the agent
- The agent decides what to do with the loaded instructions based on context

### What happens after loading

After loading, the agent has the skill's instructions in context and should follow them. The skill content persists for the duration of the conversation or until the context window fills up and older messages are pruned.

---

## Permissions

### Global permissions

Control which skills agents can access using pattern-based permissions in `opencode.json`:

```json
{
  "permission": {
    "skill": {
      "*": "allow",
      "internal-docs": "allow",
      "internal-*": "deny",
      "experimental-*": "ask"
    }
  }
}
```

### Permission values

| Value | Behavior |
|---|---|
| `allow` | Skill loads immediately without user interaction |
| `deny` | Skill is hidden from the agent entirely; access is rejected |
| `ask` | User is prompted for approval before the skill loads |

### Pattern matching

- `*` matches any skill name
- `internal-*` matches `internal-docs`, `internal-tools`, etc.
- `experimental-*` matches any skill starting with `experimental-`
- Patterns are evaluated in order; first match wins

### Per-agent overrides

Give specific agents different permissions than the global defaults.

**Custom agents** (in agent frontmatter):

```yaml
---
name: plan-agent
permission:
  skill:
    "documents-*": "allow"
    "internal-*": "allow"
---
```

**Built-in agents** (in `opencode.json`):

```json
{
  "agent": {
    "plan": {
      "permission": {
        "skill": {
          "internal-*": "allow"
        }
      }
    }
  }
}
```

### Disable the skill tool entirely

Prevent specific agents from accessing any skills:

**Custom agents** (in agent frontmatter):

```yaml
---
name: plan-agent
tools:
  skill: false
---
```

**Built-in agents** (in `opencode.json`):

```json
{
  "agent": {
    "plan": {
      "tools": {
        "skill": false
      }
    }
  }
}
```

When disabled, the `<available_skills>` section is omitted entirely from the tool description, and the agent has no awareness of available skills.

---

## Writing Good Skills

### Description quality

The `description` field is the single most important factor in whether an agent correctly loads your skill. Follow these guidelines:

- **Be specific**: "Run pytest with coverage and generate HTML reports" is better than "Testing skill"
- **State the action**: "Create and manage Kubernetes deployments" vs "Kubernetes skill"
- **Scope it**: "Format Python code with Black and isort" vs "Python formatting"
- **Include trigger phrases**: Mention the tools, frameworks, or patterns the agent should associate with the skill

### Progressive disclosure

Structure your `SKILL.md` for progressive disclosure:

1. **First paragraph / overview**: One or two sentences explaining what the skill does. This is what the agent reasons about when deciding to load.
2. **Instructions**: Step-by-step workflows, rules, and constraints. This is the core content the agent follows after loading.
3. **Resources**: File references, code snippets, external links, and examples. These provide concrete implementation details.

### Include practical examples

Skills work best when they include:

- **Code snippets** the agent can adapt
- **File path conventions** (e.g., "Place tests in `tests/<module>/test_<feature>.py`")
- **Command examples** (e.g., `npm run test -- --coverage`)
- **Before/after comparisons** showing expected output

### When to use / when not to use

Include explicit sections like:

```markdown
## When to use this skill
- When the user asks to create, update, or review pull requests
- When working with GitHub Actions CI/CD pipelines

## When NOT to use this skill
- For GitLab merge requests (use `gitlab-mr` instead)
- For simple single-commit changes that don't need CI configuration
```

### Avoid these pitfalls

- **Overly broad descriptions**: "Helps with coding" triggers on everything and nothing
- **Missing context**: Don't assume the agent knows your project structure—state it
- **No examples**: Abstract instructions without concrete examples are hard to follow
- **Giant monoliths**: If a skill covers too much, split it into focused sub-skills

---

## Complete Example

Below is a full `SKILL.md` example for a hypothetical `react-testing` skill:

````markdown
---
name: react-testing
description: Write, run, and debug React component tests using Vitest and Testing Library
license: MIT
compatibility: opencode
metadata:
  audience: frontend-developers
  framework: react
  tools: vitest,testing-library
---

# React Testing Skill

Write reliable React component tests with Vitest and @testing-library/react.

## Setup requirements

- Project must have `vitest` and `@testing-library/react` installed
- Test files must be co-located with components or in a `__tests__/` directory
- Use `.test.tsx` or `.test.jsx` file extension

## Test structure

Every test file should follow this pattern:

```typescript
import { describe, it, expect } from "vitest";
import { render, screen, fireEvent } from "@testing-library/react";
import { ComponentName } from "../ComponentName";

describe("ComponentName", () => {
  it("renders correctly", () => {
    render(<ComponentName />);
    expect(screen.getByText("Expected text")).toBeInTheDocument();
  });
});
```

## Testing patterns

### User interactions
Always use `fireEvent` or `userEvent` from Testing Library—never call component methods directly.

### Async operations
Use `waitFor` or `findBy` queries for async content:
```typescript
await waitFor(() => {
  expect(screen.getByText("Loaded")).toBeInTheDocument();
});
```

### Mocking
Mock modules with `vi.mock()`:
```typescript
vi.mock("../api", () => ({
  fetchData: vi.fn().mockResolvedValue({ data: "test" }),
}));
```

## Running tests

```bash
# Run all tests
npm run test

# Run with coverage
npm run test -- --coverage

# Run specific file
npx vitest run src/components/Button.test.tsx

# Watch mode
npx vitest --watch
```

## When to use this skill
- Writing new component tests for React components
- Debugging failing tests
- Adding test coverage to existing components
- Setting up test utilities and helpers

## When NOT to use this skill
- For end-to-end tests (use Playwright or Cypress instead)
- For non-React code (vanilla JS, Node.js utilities)
- For API integration tests
````

---

## Troubleshooting

### Skill not showing up

If a skill does not appear in the `<available_skills>` section:

1. **Check the filename**: It must be exactly `SKILL.md` (all caps, `.md` extension)
2. **Check frontmatter**: Verify that `name` and `description` are present and non-empty
3. **Validate the name**: Ensure it matches `^[a-z0-9]+(-[a-z0-9]+)*$` and matches the directory name
4. **Check discovery paths**: Confirm the file is in one of the six supported locations
5. **Check permissions**: A skill with `"deny"` permission is hidden from agents entirely
6. **Check for duplicates**: If multiple skills share a name, only the first discovered wins

### Common mistakes

| Problem | Fix |
|---|---|
| `skill.md` or `Skill.md` | Rename to `SKILL.md` (exact case) |
| Name contains uppercase | Change to lowercase: `Git-Release` → `git-release` |
| Name has spaces | Use hyphens: `my skill` → `my-skill` |
| Missing frontmatter | Add `---` delimited YAML at the top |
| Name doesn't match directory | Ensure `name:` matches the folder name exactly |
| Empty description | Add a meaningful 1–1024 char description |
| File in wrong location | Move to one of the six supported paths |

### Permission debugging

If a skill exists but is inaccessible:

1. Check `opencode.json` for `permission.skill` rules
2. Look for `deny` patterns matching the skill name
3. Check agent-specific overrides that might restrict access
4. Try adding an explicit `allow` rule for the skill name

---

## Publishing Skills

### SkillsMP auto-indexing

SkillsMP (Skills Marketplace) auto-indexes skills from GitHub repositories:

- Tag your repository with `claude-skills` on GitHub
- The repository must have at least **2 stars**
- SkillsMP periodically scans for new and updated repositories
- Your skills become discoverable in the SkillsMP directory

### MCPMarket submission

Submit skills to MCPMarket for broader discoverability:

1. Ensure your skill follows the SKILL.md format
2. Submit a PR or issue to the MCPMarket repository
3. Include a description, category, and usage examples

### GitHub topics

Add relevant GitHub topics to your skill repository for discoverability:

- `opencode-skill` — Primary topic for OpenCode skills
- `claude-skills` — For SkillsMP auto-indexing
- `agent-skill` — General agent skill topic
- `opencode` — General OpenCode ecosystem

### Distribution best practices

- **Version your skills**: Use git tags for releases
- **Document dependencies**: List required tools, plugins, or configurations
- **Provide installation instructions**: Show users how to clone or copy the skill
- **Include a README**: Explain the skill alongside the `SKILL.md` file
- **Test with multiple agents**: Verify the skill works across different agent configurations
