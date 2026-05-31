---
name: opencode-troubleshooting
description: |
  Use this skill when diagnosing and resolving issues with OpenCode Desktop or CLI. Covers log inspection, storage locations, desktop app troubleshooting (plugins, cache, server connection, Wayland/X11, WebView2, notifications, storage reset), common errors (startup failures, authentication, model not found, ProviderInitError, AI_APICallError, copy/paste on Linux), and CLI debugging tools. Trigger on: troubleshooting, debugging, logs, won't start, error, crash, blank screen, connection failed, provider error, auth error, copy paste linux.
---

# OpenCode Troubleshooting

OpenCode is an AI-powered coding assistant available as a CLI, TUI, web interface, and desktop app. When something goes wrong — the app won't start, models fail to load, authentication breaks, or the UI is unresponsive — this skill provides comprehensive diagnostic and resolution procedures. Start by checking logs and storage, then work through the specific issue category.

---

## Table of Contents

- [Logs](#logs)
- [Storage](#storage)
- [Desktop App Troubleshooting](#desktop-app-troubleshooting)
  - [Quick Checks](#quick-checks)
  - [Disable Plugins](#disable-plugins)
  - [Clear the Cache](#clear-the-cache)
  - [Fix Server Connection Issues](#fix-server-connection-issues)
  - [Linux: Wayland / X11 Issues](#linux-wayland--x11-issues)
  - [Windows: WebView2 Runtime](#windows-webview2-runtime)
  - [Windows: General Performance Issues](#windows-general-performance-issues)
  - [Notifications Not Showing](#notifications-not-showing)
  - [Reset Desktop App Storage (Last Resort)](#reset-desktop-app-storage-last-resort)
- [Common Issues](#common-issues)
  - [OpenCode Won't Start](#opencode-wont-start)
  - [Authentication Issues](#authentication-issues)
  - [Model Not Available](#model-not-available)
  - [ProviderInitError](#provideriniterror)
  - [AI_APICallError and Provider Package Issues](#ai_apicallerror-and-provider-package-issues)
  - [Copy/Paste Not Working on Linux](#copypaste-not-working-on-linux)
- [CLI Debugging Tools](#cli-debugging-tools)
- [Getting Help](#getting-help)

---

## Logs

Log files are the first place to look when diagnosing any issue with OpenCode. They contain detailed information about startup, provider connections, API calls, errors, and internal state transitions.

### Log File Locations

| Platform | Path |
|----------|------|
| macOS    | `~/.local/share/opencode/log/` |
| Linux    | `~/.local/share/opencode/log/` |
| Windows  | Press `WIN+R` and paste `%USERPROFILE%\.local\share\opencode\log` |

### Log File Naming

Log files are named with timestamps in the format `YYYY-MM-DDTHHMMSS.log`. For example:

```
2025-01-09T123456.log
2025-01-09T143022.log
2025-01-10T090015.log
```

OpenCode keeps the **most recent 10 log files** on disk. Older log files are automatically pruned.

### Log Levels

Control log verbosity with the `--log-level` command-line flag:

```bash
opencode --log-level DEBUG
```

Available log levels (from least to most verbose):

- `ERROR` — Only errors
- `WARN` — Warnings and errors
- `INFO` — Informational messages (default)
- `DEBUG` — Full debug output

### Printing Logs to Terminal

Use `--print-logs` to stream log output directly to the terminal instead of writing to files:

```bash
opencode --print-logs
```

This is useful when:
- Log files aren't being written (permissions issue, disk full)
- You want real-time output alongside terminal interaction
- Debugging startup issues where the process exits before writing logs

### Reading Log Files

To view the most recent log file:

```bash
# List log files sorted by modification time (newest first)
ls -lt ~/.local/share/opencode/log/ | head -10

# View the most recent log
cat "$(ls -t ~/.local/share/opencode/log/*.log | head -1)"
```

On Windows (PowerShell):

```powershell
Get-ChildItem "$env:USERPROFILE\.local\share\opencode\log" | Sort-Object LastWriteTime -Descending | Select-Object -First 5
Get-Content (Get-ChildItem "$env:USERPROFILE\.local\share\opencode\log\*.log" | Sort-Object LastWriteTime -Descending | Select-Object -First 1).FullName
```

### What to Look For in Logs

- **Startup errors**: Missing dependencies, port conflicts, configuration parse failures
- **Provider errors**: API key validation, rate limits, model availability, network timeouts
- **Plugin errors**: Failed loads, compatibility issues, unhandled exceptions
- **File system errors**: Permission denied, disk full, missing directories
- **Network errors**: Connection refused, DNS resolution failures, TLS/SSL errors
- **Session errors**: Message processing failures, context window overflow

---

## Storage

OpenCode stores persistent application data on disk, including authentication credentials, session history, and project-specific state.

### Storage Locations

| Platform | Path |
|----------|------|
| macOS    | `~/.local/share/opencode/` |
| Linux    | `~/.local/share/opencode/` |
| Windows  | Press `WIN+R` and paste `%USERPROFILE%\.local\share\opencode` |

### Storage Contents

| Path | Description |
|------|-------------|
| `opencode.db` | SQLite database — primary data store for sessions, messages, and state |
| `snapshot/` | Git-like repo for file change tracking |
| `storage/` | Session data and internal state |
| `log/` | Application log files (see [Logs](#logs)) |

### Additional Data Directories

OpenCode also uses these directories for different purposes:

| Directory | Description |
|-----------|-------------|
| `~/.local/state/opencode/` | Runtime state — `model.json`, `prompt-history.jsonl`, `kv.json` |
| `~/.cache/opencode/` | Provider packages, binaries, and cached data |

### Clearing Storage

To completely reset all OpenCode data (sessions, auth, logs):

```bash
rm -rf ~/.local/share/opencode
```

On Windows (PowerShell):

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.local\share\opencode"
```

**Warning**: This removes all sessions, authentication data, and logs. You will need to re-authenticate with your providers.

---

## Desktop App Troubleshooting

OpenCode Desktop runs a local OpenCode server (the `opencode-cli` sidecar) in the background. Most issues are caused by a misbehaving plugin, a corrupted cache, or a bad server setting.

### Quick Checks

Before diving into detailed troubleshooting, try these rapid fixes:

1. **Fully quit and relaunch the app.** On macOS, right-click the Dock icon and select Quit. On Windows, right-click the system tray icon and select Exit. Then relaunch.
2. **Click Restart.** If the app shows an error screen, click the **Restart** button and copy the error details for debugging.
3. **Reload Webview (macOS only).** Click `OpenCode` menu → **Reload Webview**. This resolves blank or frozen UI without restarting the entire app.

### Disable Plugins

If the desktop app crashes on launch, hangs, or behaves strangely, start by disabling plugins.

#### Check the Global Config

Open your global config file and look for a `plugin` key:

- **macOS/Linux**: `~/.config/opencode/opencode.jsonc` (or `~/.config/opencode/opencode.json`)
- **macOS/Linux (older installs)**: `~/.local/share/opencode/opencode.jsonc`
- **Windows**: Press `WIN+R` and paste `%USERPROFILE%\.config\opencode\opencode.jsonc`

Temporarily disable all plugins by setting `plugin` to an empty array:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": []
}
```

#### Check Plugin Directories

OpenCode loads local plugins from disk. Temporarily move these directories out of the way and restart:

**Global plugins:**
- **macOS/Linux**: `~/.config/opencode/plugins/`
- **Windows**: Press `WIN+R` and paste `%USERPROFILE%\.config\opencode\plugins`

**Project plugins** (per-project config):
- `<your-project>/.opencode/plugins/`

If the app works after disabling, re-enable plugins one at a time to identify the culprit.

### Clear the Cache

If disabling plugins doesn't help (or a plugin install is stuck), clear the cache so OpenCode can rebuild it.

1. Quit OpenCode Desktop completely.
2. Delete the cache directory:

| Platform | Path |
|----------|------|
| macOS    | Finder → `Cmd+Shift+G` → paste `~/.cache/opencode` |
| Linux    | `rm -rf ~/.cache/opencode` |
| Windows  | Press `WIN+R` and paste `%USERPROFILE%\.cache\opencode` |

3. Restart OpenCode Desktop.

OpenCode will recreate the cache directory with fresh data on next launch.

### Fix Server Connection Issues

OpenCode Desktop can either start its own local server (default) or connect to a server URL you configured. If you see a **"Connection Failed"** dialog or the app never gets past the splash screen, check for a custom server URL.

#### Clear the Desktop Default Server URL

From the Home screen, click the server name (with the status dot) to open the Server picker. In the **Default server** section, click **Clear**.

#### Remove `server.port` / `server.hostname` from Config

If your `opencode.json(c)` contains a `server` section, temporarily remove it and restart:

```jsonc
{
  "$schema": "https://opencode.ai/config.json"
  // "server": { "port": 4096, "hostname": "0.0.0.0" }
}
```

#### Check Environment Variables

If `OPENCODE_PORT` is set in your environment, the desktop app will try to use that port for the local server. Unset it (or pick a free port) and restart:

```bash
unset OPENCODE_PORT
```

On Windows (PowerShell):

```powershell
Remove-Item Env:OPENCODE_PORT
```

### Linux: Wayland / X11 Issues

On Linux, some Wayland setups can cause blank windows or compositor errors.

- If you're on Wayland and the app is blank/crashing, try launching with:

```bash
OC_ALLOW_WAYLAND=1 opencode
```

- If that makes things worse, remove it and try launching under an X11 session instead. Most desktop environments allow switching between Wayland and X11 at the login screen.

- To check which display server you're using:

```bash
echo $XDG_SESSION_TYPE
```

### Windows: WebView2 Runtime

On Windows, OpenCode Desktop requires the Microsoft Edge **WebView2 Runtime**. If the app opens to a blank window or won't start:

1. Check if WebView2 is installed: open Settings → Apps → search for "WebView2"
2. If not installed or outdated, download from: https://developer.microsoft.com/en-us/microsoft-edge/webview2/
3. Install and restart OpenCode Desktop

### Windows: General Performance Issues

If you're experiencing slow performance, file access issues, or terminal problems on Windows, try using [WSL (Windows Subsystem for Linux)](https://opencode.ai/docs/windows-wsl). WSL provides a Linux environment that works more seamlessly with OpenCode's features.

To use OpenCode with WSL:

```bash
wsl
# Inside WSL, install and run opencode normally
```

### Notifications Not Showing

OpenCode Desktop only shows system notifications when **both** conditions are met:

1. **Notifications are enabled for OpenCode in your OS settings**
2. **The app window is not focused** — notifications are suppressed when the app is in the foreground to avoid distraction

To enable notifications:

- **macOS**: System Settings → Notifications → OpenCode → toggle on
- **Windows**: Settings → System → Notifications → find OpenCode → toggle on
- **Linux**: Varies by desktop environment (GNOME Settings → Notifications, KDE System Settings → Notifications)

### Reset Desktop App Storage (Last Resort)

If the app won't start and you can't clear settings from inside the UI, reset the desktop app's saved state:

1. Quit OpenCode Desktop completely.
2. Find and delete these files in the OpenCode Desktop app data directory:

| File | Contents |
|------|----------|
| `opencode.settings.dat` | Desktop default server URL |
| `opencode.global.dat` | UI state — recent servers, projects |
| `opencode.workspace.*.dat` | Per-workspace UI state |

**To find the directory:**

| Platform | Path |
|----------|------|
| macOS    | Finder → `Cmd+Shift+G` → `~/Library/Application Support` → search for filenames |
| Linux    | Search under `~/.local/share` for the filenames |
| Windows  | Press `WIN+R` → `%APPDATA%` → search for filenames |

3. Delete the matching files.
4. Restart OpenCode Desktop.

---

## Common Issues

### OpenCode Won't Start

**Symptoms**: App shows a blank screen, immediately exits, or hangs on splash screen.

**Diagnostic steps:**

1. **Check the logs for error messages:**

```bash
ls -lt ~/.local/share/opencode/log/ | head -5
cat "$(ls -t ~/.local/share/opencode/log/*.log | head -1)"
```

2. **Try running with `--print-logs` to see output in the terminal:**

```bash
opencode --print-logs
```

3. **Ensure you have the latest version:**

```bash
opencode upgrade
```

4. **Common causes and fixes:**
   - Corrupted cache → clear `~/.cache/opencode`
   - Misbehaving plugin → disable plugins (see [Disable Plugins](#disable-plugins))
   - Port conflict → check if another process is using the configured port
   - Missing WebView2 (Windows) → install WebView2 runtime
   - Wayland issues (Linux) → try `OC_ALLOW_WAYLAND=1` or switch to X11

### Authentication Issues

**Symptoms**: "Authentication failed", 401/403 errors, models not loading due to missing credentials.

**Diagnostic steps:**

1. **Try re-authenticating with the `/connect` command in the TUI:**

```bash
# Launch the TUI
opencode
# Then type: /connect
```

2. **Check that your API keys are valid** — log in to your provider's dashboard (OpenAI, Anthropic, etc.) and verify:
   - The API key hasn't been revoked
   - Your account is in good standing
   - Usage limits haven't been exceeded

3. **Ensure your network allows connections to the provider's API:**
   - Check firewall/proxy settings
   - Try `curl https://api.openai.com/v1/models` (or equivalent for your provider)
   - Verify DNS resolution works

4. **Verify stored credentials:**

```bash
opencode providers list
```

### Model Not Available

**Symptoms**: `ProviderModelNotFoundError`, model doesn't appear in model selection.

**Diagnostic steps:**

1. **Check that you've authenticated with the provider** — see [Authentication Issues](#authentication-issues).

2. **Verify the model name in your config is correct.** Models must be referenced as `<providerId>/<modelId>`:

| Correct | Incorrect |
|---------|-----------|
| `openai/gpt-4.1` | `gpt-4.1` |
| `anthropic/claude-sonnet-4-5` | `claude-sonnet-4-5` |
| `openrouter/google/gemini-2.5-flash` | `gemini-2.5-flash` |
| `opencode/kimi-k2` | `kimi-k2` |

3. **Some models may require specific access or subscriptions** — check your provider's dashboard for model availability.

4. **List all models you have access to:**

```bash
opencode models
```

### ProviderInitError

**Symptoms**: `ProviderInitError` on startup or when switching models.

**This usually indicates an invalid or corrupted configuration.**

**Resolution steps:**

1. **Verify your provider is set up correctly** — follow the [providers guide](https://opencode.ai/docs/providers/).

2. **Clear your stored configuration:**

```bash
rm -rf ~/.local/share/opencode
```

On Windows:

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.local\share\opencode"
```

3. **Re-authenticate** with your provider using the `/connect` command in the TUI.

4. **If using environment variables**, verify they're set correctly:

```bash
echo $ANTHROPIC_API_KEY  # or OPENAI_API_KEY, etc.
```

### AI_APICallError and Provider Package Issues

**Symptoms**: API call errors, provider fails to respond, unexpected API response formats.

OpenCode dynamically installs provider packages (OpenAI, Anthropic, Google, etc.) as needed and caches them locally. Outdated or corrupted packages can cause API errors.

**Resolution steps:**

1. **Clear the provider package cache:**

```bash
rm -rf ~/.cache/opencode
```

On Windows:

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.cache\opencode"
```

2. **Restart OpenCode** to reinstall the latest provider packages.

3. This forces OpenCode to download the most recent versions of provider packages, which often resolves compatibility issues with model parameters and API changes.

### Copy/Paste Not Working on Linux

**Symptoms**: Cannot copy text from OpenCode TUI or paste into it.

Linux users need a clipboard utility installed for copy/paste functionality. OpenCode auto-detects your display server and uses the appropriate tool.

**For X11 systems:**

```bash
apt install -y xclip
# or
apt install -y xsel
```

**For Wayland systems:**

```bash
apt install -y wl-clipboard
```

**For headless environments (no display server):**

```bash
apt install -y xvfb
Xvfb :99 -screen 0 1024x768x24 > /dev/null 2>&1 &
export DISPLAY=:99.0
```

**Clipboard tool detection order:**

OpenCode detects Wayland and prefers `wl-clipboard`. Otherwise, it searches in order: `xclip`, `xsel`.

---

## CLI Debugging Tools

OpenCode provides several CLI commands for diagnosing issues without needing to inspect files manually.

### Verify Configuration

```bash
opencode debug config
```

Displays the fully merged configuration from all sources (remote, global, project, managed). Useful for:
- Troubleshooting precedence issues
- Confirming MDM-deployed settings are active
- Verifying variable substitution resolved correctly
- Checking which providers are enabled/disabled
- Inspecting resolved permission rules

### Check Credentials

```bash
opencode providers list
```

Lists all stored authentication credentials. Shows which providers have active API keys or tokens. (`auth` is an alias for `providers`.)

### List Available Models

```bash
opencode models
```

Lists all models available across all configured providers. Useful for verifying model names and access.

### Enable Debug Logging

```bash
opencode --log-level DEBUG
```

Runs OpenCode with maximum log verbosity. Combine with `--print-logs` for real-time terminal output:

```bash
opencode --log-level DEBUG --print-logs
```

### Stream Logs to Terminal

```bash
opencode --print-logs
```

Streams log output directly to stdout instead of writing to files. Useful when:
- Log files aren't being created (permissions issue)
- You need real-time output for debugging
- The process crashes before writing logs

### Run Without Plugins

```bash
opencode --pure
```

Runs OpenCode without loading any external plugins. Useful for troubleshooting plugin-related issues — if the problem disappears with `--pure`, a plugin is the cause. See [Disable Plugins](#disable-plugins) for more.

---

## Getting Help

If you're experiencing issues with OpenCode and the solutions above don't resolve them:

### Report Issues on GitHub

The best way to report bugs or request features:

[**github.com/anomalyco/opencode/issues**](https://github.com/anomalyco/opencode/issues)

Before creating a new issue, search existing issues to see if your problem has already been reported.

### Join the Discord

For real-time help and community discussion:

[**opencode.ai/discord**](https://opencode.ai/discord)
