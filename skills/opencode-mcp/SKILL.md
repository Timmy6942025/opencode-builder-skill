---
name: opencode-mcp
description: |
  Use this skill when adding Model Context Protocol (MCP) servers to OpenCode, configuring local or remote MCP servers, setting up OAuth authentication for MCP, managing MCP server permissions per agent, or troubleshooting MCP connection issues. Covers local/remote server types, OAuth flows, tool management, overriding remote defaults, and common MCP server examples.
---

# OpenCode MCP Servers

Add local and remote MCP tools to OpenCode using the *Model Context Protocol* (MCP). MCP is a standard protocol that lets you connect external tool servers to AI applications. OpenCode supports both local servers (child processes) and remote servers (HTTP). Once added, MCP tools are automatically available to the LLM alongside OpenCode's built-in tools.

---

## Context Caveat

When you use an MCP server, it adds tokens to the context. Every tool the MCP server exposes is described in the context window, which consumes tokens. This can quickly add up if you have a lot of tools from multiple MCP servers. Be careful with which MCP servers you enable.

- Certain MCP servers, like the GitHub MCP server, tend to add a lot of tokens and can easily exceed the context limit.
- Each additional MCP server multiplies the tool definitions injected into context.
- Monitor your context window usage and be selective about which servers you enable.

---

## Enable / Disable

MCP servers are defined in your [OpenCode Config](https://opencode.ai/docs/config/) (`opencode.json` or `opencode.jsonc`) under the `mcp` key. Each server gets a unique name used to reference it in prompts and tool management.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "name-of-mcp-server": {
      // ... server config ...
      "enabled": true
    },
    "name-of-other-mcp-server": {
      // ... server config ...
    }
  }
}
```

- Set `"enabled": true` to explicitly enable a server.
- Set `"enabled": false` to temporarily disable a server without removing its config.
- If `enabled` is omitted, the server is enabled by default.

---

## Local MCP Servers

Local servers run as child processes on your machine. Use `"type": "local"` with a `command` array.

### Full Configuration

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-local-mcp": {
      "type": "local",
      "command": ["npx", "-y", "my-mcp-command"],
      "enabled": true,
      "environment": {
        "MY_ENV_VAR": "my_env_var_value"
      },
      "timeout": 5000
    }
  }
}
```

### Options

| Option | Type | Required | Description |
|---|---|---|---|
| `type` | String | Yes | Must be `"local"`. |
| `command` | Array | Yes | Command and arguments to run the MCP server. Use an array, e.g. `["npx", "-y", "package-name"]` or `["bun", "x", "my-mcp-command"]`. |
| `environment` | Object | No | Environment variables to set when running the server process. |
| `enabled` | Boolean | No | Enable or disable the MCP server on startup. Defaults to `true`. |
| `timeout` | Number | No | Timeout in milliseconds for fetching tools from the MCP server. Defaults to `5000` (5 seconds). |

### Example: `@modelcontextprotocol/server-everything`

The test server from the MCP project — useful for verifying your setup works:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "mcp_everything": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-everything"]
    }
  }
}
```

Then reference it in your prompts:

```
use the mcp_everything tool to add the number 3 and 4
```

### Example: Filesystem Server

Grants the LLM read/write access to specified directories on your filesystem:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    }
  }
}
```

The path argument (`/Users/me/projects`) restricts the server to only that directory.

### Example: GitHub MCP Server

Provides access to GitHub repositories, issues, pull requests, and more:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "github": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-github"],
      "environment": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "{env:GITHUB_TOKEN}"
      }
    }
  }
}
```

**Warning:** The GitHub MCP server adds many tool definitions to context and can easily consume a large portion of your context budget. Consider disabling it globally and enabling it per-agent when needed.

### Running with Bun

If you use Bun instead of Node.js:

```jsonc
{
  "mcp": {
    "my-mcp-foo": {
      "type": "local",
      "command": ["bun", "x", "my-mcp-command-foo"]
    }
  }
}
```

---

## Remote MCP Servers

Remote servers connect over HTTP. Use `"type": "remote"` with a `url`.

### Full Configuration

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-remote-mcp": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer MY_API_KEY"
      },
      "oauth": {},
      "timeout": 5000
    }
  }
}
```

### Options

| Option | Type | Required | Description |
|---|---|---|---|
| `type` | String | Yes | Must be `"remote"`. |
| `url` | String | Yes | URL of the remote MCP server. |
| `enabled` | Boolean | No | Enable or disable the MCP server on startup. |
| `headers` | Object | No | HTTP headers to send with requests (e.g., API keys, custom headers). |
| `oauth` | Object \| false | No | OAuth authentication configuration. See [OAuth](#oauth-authentication) section. Set to `false` to disable auto-detection. |
| `timeout` | Number | No | Timeout in milliseconds for fetching tools. Defaults to `5000` (5 seconds). |

### Using Environment Variables in Headers

Reference environment variables with the `{env:VAR_NAME}` syntax:

```jsonc
{
  "mcp": {
    "my-server": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer {env:MY_API_KEY}"
      }
    }
  }
}
```

This avoids hardcoding secrets in your config file.

---

## OAuth Authentication

OpenCode automatically handles OAuth for remote MCP servers. When a server requires authentication, OpenCode:

1. Detects the `401` response from the server.
2. Initiates the OAuth flow using **Dynamic Client Registration (RFC 7591)** if the server supports it.
3. Opens your browser for the user to authorize.
4. Stores tokens securely for future requests.

### Automatic OAuth

For most OAuth-enabled servers, no special configuration is needed. Just configure the remote server:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-oauth-server": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp"
    }
  }
}
```

If the server requires authentication, OpenCode will prompt you to authenticate when you first try to use it. You can also manually trigger the flow:

```bash
opencode mcp auth my-oauth-server
```

### Pre-registered Credentials

If you have client credentials from the MCP server provider (i.e., you already registered your app), configure them directly:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-oauth-server": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp",
      "oauth": {
        "clientId": "{env:MY_MCP_CLIENT_ID}",
        "clientSecret": "{env:MY_MCP_CLIENT_SECRET}",
        "scope": "tools:read tools:execute"
      }
    }
  }
}
```

When `clientId` and `clientSecret` are provided, OpenCode skips Dynamic Client Registration and uses the pre-registered credentials directly.

### OAuth Options

| Option | Type | Description |
|---|---|---|
| `oauth` | Object \| false | OAuth config object, or `false` to disable OAuth auto-detection entirely. |
| `clientId` | String | OAuth client ID. If omitted, Dynamic Client Registration will be attempted. |
| `clientSecret` | String | OAuth client secret, if required by the authorization server. |
| `scope` | String | OAuth scopes to request during authorization (space-separated). |

### CLI Commands

OpenCode provides several CLI commands for managing MCP authentication:

```bash
# Manually authenticate with a specific server
opencode mcp auth my-oauth-server

# List all MCP servers and their auth/connection status
opencode mcp list

# Remove stored credentials for a server
opencode mcp logout my-oauth-server

# Debug connection and OAuth flow for a specific server
opencode mcp debug my-oauth-server
```

#### `opencode mcp add`

Interactive wizard to add a new MCP server to your configuration. Guides you through choosing local or remote, setting the command or URL, and other options.

#### `opencode mcp auth <server-name>`

Opens your browser to complete the OAuth authorization flow. After you authorize, OpenCode stores the tokens.

#### `opencode mcp list`

Shows all configured MCP servers, their type (local/remote), enabled status, and authentication status.

#### `opencode mcp logout <server-name>`

Removes stored OAuth tokens for the specified server. You will need to re-authenticate on next use.

#### `opencode mcp debug <server-name>`

Shows the current auth status, tests HTTP connectivity to the server, and attempts the OAuth discovery flow. Useful for diagnosing connection and authentication issues.

### Token Storage

OAuth tokens are stored locally at:

```
~/.local/share/opencode/mcp-auth.json
```

This file is managed automatically by OpenCode. Do not edit it manually.

### Disabling OAuth

If a server uses API keys instead of OAuth, disable automatic OAuth detection to prevent unnecessary 401-based OAuth flows:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-api-key-server": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp",
      "oauth": false,
      "headers": {
        "Authorization": "Bearer {env:MY_API_KEY}"
      }
    }
  }
}
```

---

## Tool Management

MCP tools are registered alongside built-in tools. Each MCP tool is named with the pattern:

```
<servername>_<toolname>
```

For example, if your MCP server is named `"github"` and it exposes a tool called `search_repos`, the tool appears as `github_search_repos` in OpenCode.

### Global Enable / Disable

Disable specific MCP tools via the `tools` config:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-mcp-foo": {
      "type": "local",
      "command": ["bun", "x", "my-mcp-command-foo"]
    },
    "my-mcp-bar": {
      "type": "local",
      "command": ["bun", "x", "my-mcp-command-bar"]
    }
  },
  "tools": {
    "my-mcp-foo": false
  }
}
```

In this example, all tools from `my-mcp-foo` are disabled, while `my-mcp-bar` tools remain active.

### Glob Patterns

Use glob patterns to disable all matching tools at once:

```jsonc
{
  "tools": {
    "my-mcp*": false
  }
}
```

Glob syntax:
- `*` — matches zero or more of any character (e.g., `"my-mcp*"` matches `my-mcp_search`, `my-mcp_list`, `my-mcp_fetch`, etc.)
- `?` — matches exactly one character
- All other characters match literally

Since MCP server tools are registered with the server name as a prefix, to disable **all** tools for a specific server:

```jsonc
{
  "tools": {
    "mymcpservername_*": false
  }
}
```

### Per-Agent Enable / Disable

If you have many MCP servers, you may want to disable them globally and enable them selectively per agent. This keeps the default context lean while allowing specific agents to use the tools they need.

**Step 1:** Disable the MCP tools globally:

```jsonc
{
  "tools": {
    "my-mcp*": false
  }
}
```

**Step 2:** Enable them per agent in the `agent` config:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-mcp": {
      "type": "local",
      "command": ["bun", "x", "my-mcp-command"],
      "enabled": true
    }
  },
  "tools": {
    "my-mcp*": false
  },
  "agent": {
    "my-agent": {
      "tools": {
        "my-mcp*": true
      }
    }
  }
}
```

Now `my-mcp` tools are only available to `my-agent`. Other agents won't see these tools in their context.

### MCP Servers in Agent Permissions

MCP tool names follow the same permission model as built-in tools. You can grant or revoke MCP tools in agent permission configs using the same `tools` key with glob patterns.

---

## Overriding Remote Defaults

Organizations can provide default MCP servers via their `.well-known/opencode` endpoint. These servers may be disabled by default, allowing users to opt-in to the ones they need.

To enable a specific server from your organization's remote config, add it to your local config with `"enabled": true`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "jira": {
      "type": "remote",
      "url": "https://jira.example.com/mcp",
      "enabled": true
    }
  }
}
```

**Local config values override remote defaults.** If the organization's remote config defines `jira` with `"enabled": false`, your local `"enabled": true` takes precedence. See [config precedence](https://opencode.ai/docs/config#precedence-order) for more details.

---

## Complete Examples

### Sentry (Remote + OAuth)

Add the [Sentry MCP server](https://mcp.sentry.dev) to interact with your Sentry projects and issues.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/mcp",
      "oauth": {}
    }
  }
}
```

After adding the configuration, authenticate:

```bash
opencode mcp auth sentry
```

This opens a browser window to complete the OAuth flow. Once authenticated, use Sentry tools in your prompts:

```
Show me the latest unresolved issues in my project. use sentry
```

### Context7 (Remote + API Key Header)

Add the [Context7 MCP server](https://github.com/upstash/context7) to search through documentation.

**Without API key (free tier, rate-limited):**

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp"
    }
  }
}
```

**With API key (higher rate limits):**

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp",
      "headers": {
        "CONTEXT7_API_KEY": "{env:CONTEXT7_API_KEY}"
      }
    }
  }
}
```

Assumes the `CONTEXT7_API_KEY` environment variable is set.

Add `use context7` to your prompts:

```
Configure a Cloudflare Worker script to cache JSON API responses for five minutes. use context7
```

Or add to your [AGENTS.md](https://opencode.ai/docs/rules/):

```markdown
When you need to search docs, use `context7` tools.
```

### Grep by Vercel (Remote)

Add the [Grep by Vercel](https://grep.app) MCP server to search through public code snippets on GitHub.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "gh_grep": {
      "type": "remote",
      "url": "https://mcp.grep.app"
    }
  }
}
```

Reference it in prompts with `use the gh_grep tool`:

```
What's the right way to set a custom domain in an SST Astro component? use the gh_grep tool
```

Or add to your AGENTS.md:

```markdown
If you are unsure how to do something, use `gh_grep` to search code examples from GitHub.
```

### Filesystem (Local)

Grants the LLM read/write access to specified directories:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    }
  }
}
```

The path argument restricts filesystem access to the specified directory only.

### GitHub MCP (Local)

Provides access to GitHub repos, issues, pull requests, and more:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "github": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-github"],
      "environment": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "{env:GITHUB_TOKEN}"
      }
    }
  }
}
```

**Context warning:** The GitHub MCP server exposes many tools and adds a large number of tokens to context. Consider disabling it globally and enabling per-agent:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "github": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-github"],
      "environment": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "{env:GITHUB_TOKEN}"
      }
    }
  },
  "tools": {
    "github_*": false
  },
  "agent": {
    "github-agent": {
      "tools": {
        "github_*": true
      }
    }
  }
}
```

### Custom Remote with Headers

A generic remote MCP server with custom authorization and headers:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "my-internal-api": {
      "type": "remote",
      "url": "https://internal-api.example.com/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer {env:INTERNAL_API_TOKEN}",
        "X-Custom-Header": "custom-value"
      },
      "oauth": false,
      "timeout": 10000
    }
  }
}
```

This example:
- Uses an API token from an environment variable
- Adds a custom header
- Disables OAuth (since it uses API key auth)
- Sets a longer timeout (10 seconds) for slower servers

---

## Best Practices

### Be Selective with MCP Servers

Each MCP server adds tool definitions to the context. Only enable servers you actively use. A server with 20 tools adds 20 tool descriptions to every request, consuming significant context budget.

### Watch Context Budget

Servers like GitHub MCP can add thousands of tokens. Monitor your context window usage. If you're hitting context limits, check which MCP servers are enabled and whether you truly need all of them.

### Use Per-Agent Scoping for Large Toolsets

When you have many MCP servers, disable them globally and enable them per agent:

```jsonc
{
  "tools": {
    "github_*": false,
    "sentry_*": false,
    "context7_*": false
  },
  "agent": {
    "code-reviewer": {
      "tools": {
        "github_*": true,
        "sentry_*": true
      }
    },
    "docs-searcher": {
      "tools": {
        "context7_*": true
      }
    }
  }
}
```

This keeps the default agent's context lean while giving specialized agents the tools they need.

### Prefer API Keys Over OAuth When Possible

If a server supports both API key auth and OAuth, prefer API keys — they're simpler and more predictable. Use `"oauth": false` to prevent unnecessary OAuth detection attempts.

### Use `{env:VAR}` for Secrets

Never hardcode API keys, tokens, or secrets in your config files. Always reference environment variables:

```jsonc
{
  "headers": {
    "Authorization": "Bearer {env:MY_SECRET_TOKEN}"
  }
}
```

### Add MCP Usage Hints to Prompts or AGENTS.md

Help the LLM know when to use MCP tools by adding instructions to your prompts or AGENTS.md:

```markdown
When you need to search documentation, use `context7` tools.
When you need to look up Sentry issues, use `sentry` tools.
If you are unsure how to do something, use `gh_grep` to search code examples from GitHub.
```

Or in individual prompts:

```
use the mcp_everything tool to add the number 3 and 4
```

---

## Troubleshooting

### Connection Issues

- **Server not starting:** Check that the `command` array is correct. Test the command manually in your terminal (e.g., `npx -y @modelcontextprotocol/server-everything`).
- **Timeout errors:** The default timeout is 5000ms (5 seconds). If your server takes longer to start, increase the `timeout` value.
- **Remote server unreachable:** Verify the `url` is correct and the server is accessible. Check your network/proxy settings.
- **Environment variables not set:** Ensure any `{env:VAR}` references have the corresponding environment variables set in your shell.

### OAuth Failures

- **Browser doesn't open:** Try running `opencode mcp auth <server-name>` manually.
- **401 loop:** The server may not support Dynamic Client Registration. Use pre-registered credentials with `clientId` and `clientSecret`.
- **Token expired:** Run `opencode mcp logout <server-name>` then `opencode mcp auth <server-name>` to re-authenticate.
- **Check auth status:** Run `opencode mcp list` to see all servers and their auth state.
- **Deep debugging:** Run `opencode mcp debug <server-name>` to see connection details, auth status, and OAuth discovery flow results.

### Tool Not Appearing

- **Server disabled:** Check `"enabled"` is not `false` in your MCP config.
- **Tool disabled in tools config:** Check your `tools` config for glob patterns that might be disabling the tool (e.g., `"my-mcp*": false`).
- **Per-agent restriction:** If tools are disabled globally and enabled per-agent, make sure the agent you're using has the tools enabled.
- **Server failed to start:** Check the server logs. The server process may have crashed during startup.
- **Timeout during tool fetch:** Increase the `timeout` value if the server takes time to initialize.

### Debugging Checklist

1. Run `opencode mcp list` to verify the server is configured and enabled.
2. Run `opencode mcp debug <server-name>` to test connectivity and auth.
3. Check `~/.local/share/opencode/mcp-auth.json` for stored tokens (OAuth servers).
4. Verify environment variables are set: `echo $VARIABLE_NAME`.
5. Test the server command manually outside of OpenCode.
6. Check OpenCode logs for error messages related to MCP.
