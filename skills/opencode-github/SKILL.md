# OpenCode GitHub Integration Skill

> **📚 Official Docs:** For the latest information, always refer to the official documentation:
> [https://opencode.ai/docs/github/](https://opencode.ai/docs/github/)

## Overview

This skill covers OpenCode's GitHub integration, enabling you to use OpenCode directly within GitHub issues and pull requests. OpenCode integrates with your GitHub workflow — mention `/opencode` or `/oc` in your comment, and OpenCode will execute tasks within your GitHub Actions runner.

## Features

- **Triage issues**: Ask OpenCode to look into an issue and explain it to you.
- **Fix and implement**: Ask OpenCode to fix an issue or implement a feature. It will work in a new branch and submit a PR with all the changes.
- **Secure**: OpenCode runs inside your GitHub's runners.

## Installation

### Automated Installation

Run the following command in a project that is in a GitHub repo:

```bash
opencode github install
```

This will walk you through:
1. Installing the GitHub app
2. Creating the workflow file
3. Setting up required secrets

### Manual Setup

#### Step 1: Install the GitHub App

Head over to [**github.com/apps/opencode-agent**](https://github.com/apps/opencode-agent). Make sure it's installed on the target repository.

#### Step 2: Add the Workflow

Add the following workflow file to `.github/workflows/opencode.yml` in your repo. Make sure to set the appropriate `model` and required API keys in `env`.

```yaml
name: opencode
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
jobs:
  opencode:
    if: |
      contains(github.event.comment.body, '/oc') ||
      contains(github.event.comment.body, '/opencode')
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          fetch-depth: 1
          persist-credentials: false

      - name: Run OpenCode
        uses: anomalyco/opencode/github@latest
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          model: anthropic/claude-sonnet-4-20250514
          # share: true
          # github_token: xxxx
```

#### Step 3: Store API Keys in Secrets

In your organization or project **settings**, expand **Secrets and variables** on the left and select **Actions**. Add the required API keys (e.g., `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc., depending on your chosen provider).

## Configuration Options

| Option | Required | Description |
|--------|----------|-------------|
| `model` | **Yes** | The model to use with OpenCode. Takes the format of `provider/model`. Example: `anthropic/claude-sonnet-4-20250514`. |
| `agent` | No | The agent to use. Must be a primary agent. Falls back to `default_agent` from config or `"build"` if not found. |
| `share` | No | Whether to share the OpenCode session. Defaults to **true** for public repositories. |
| `prompt` | No | Optional custom prompt to override the default behavior. Use this to customize how OpenCode processes requests. Required for `issues`, `schedule`, and `workflow_dispatch` events. |
| `token` | No | Optional GitHub access token for performing operations such as creating comments, committing changes, and opening pull requests. By default, OpenCode uses the installation access token from the OpenCode GitHub App, so commits, comments, and pull requests appear as coming from the app. Alternatively, you can use a [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) (PAT). |
| `use_github_token` | No | Use the GitHub Action runner's built-in `GITHUB_TOKEN` instead of the app token. Requires granting the necessary permissions in your workflow. |

### Using `use_github_token`

If you prefer to use the GitHub Action runner's [built-in `GITHUB_TOKEN`](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token) without installing the OpenCode GitHub App, set `use_github_token: true` and grant the required permissions in your workflow:

```yaml
permissions:
  id-token: write
  contents: write
  pull-requests: write
  issues: write
```

## Supported Events

| Event Type | Triggered By | Details |
|------------|--------------|---------|
| `issue_comment` | Comment on an issue or PR | Mention `/opencode` or `/oc` in your comment. OpenCode reads context and can create branches, open PRs, or reply. |
| `pull_request_review_comment` | Comment on specific code lines in a PR | Mention `/opencode` or `/oc` while reviewing code. OpenCode receives file path, line numbers, and diff context. |
| `issues` | Issue opened or edited | Automatically trigger OpenCode when issues are created or modified. **Requires `prompt` input.** |
| `pull_request` | PR opened or updated | Automatically trigger OpenCode when PRs are opened, synchronized, or reopened. Useful for automated reviews. |
| `schedule` | Cron-based schedule | Run OpenCode on a schedule. **Requires `prompt` input.** Output goes to logs and PRs (no issue to comment on). |
| `workflow_dispatch` | Manual trigger from GitHub UI | Trigger OpenCode on demand via Actions tab. **Requires `prompt` input.** Output goes to logs and PRs. |

## Complete Workflow Examples

### Basic Issue Comment Workflow

This is the most common setup — responds to `/opencode` or `/oc` mentions in issue and PR comments.

```yaml
name: opencode
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
jobs:
  opencode:
    if: |
      contains(github.event.comment.body, '/oc') ||
      contains(github.event.comment.body, '/opencode')
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          fetch-depth: 1
          persist-credentials: false

      - name: Run OpenCode
        uses: anomalyco/opencode/github@latest
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          model: anthropic/claude-sonnet-4-20250514
```

### Scheduled Task (Cron, Monday 9am)

Run OpenCode on a schedule to perform automated tasks. The `prompt` input is **required** since there's no comment to extract instructions from. Scheduled workflows run without a user context to permission-check, so the workflow must grant `contents: write` and `pull-requests: write` if you expect OpenCode to create branches or PRs.

```yaml
name: Scheduled OpenCode Task
on:
  schedule:
    - cron: "0 9 * * 1" # Every Monday at 9am UTC
jobs:
  opencode:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write
      pull-requests: write
      issues: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          persist-credentials: false

      - name: Run OpenCode
        uses: anomalyco/opencode/github@latest
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          model: anthropic/claude-sonnet-4-20250514
          prompt: |
            Review the codebase for any TODO comments and create a summary.
            If you find issues worth addressing, open an issue to track them.
```

### PR Auto-Review (on open/sync)

Automatically review PRs when they are opened or updated. For `pull_request` events, if no `prompt` is provided, OpenCode defaults to reviewing the pull request.

```yaml
name: opencode-review
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: read
      issues: read
    steps:
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - uses: anomalyco/opencode/github@latest
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          model: anthropic/claude-sonnet-4-20250514
          use_github_token: true
          prompt: |
            Review this pull request:
            - Check for code quality issues
            - Look for potential bugs
            - Suggest improvements
```

### Issue Triage (with Account Age Check to Reduce Spam)

Automatically triage new issues. This example filters to accounts older than 30 days to reduce spam. For `issues` events, the `prompt` input is **required** since there's no comment to extract instructions from.

```yaml
name: Issue Triage
on:
  issues:
    types: [opened]
jobs:
  triage:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write
      pull-requests: write
      issues: write
    steps:
      - name: Check account age
        id: check
        uses: actions/github-script@v7
        with:
          script: |
            const user = await github.rest.users.getByUsername({
              username: context.payload.issue.user.login
            });
            const created = new Date(user.data.created_at);
            const days = (Date.now() - created) / (1000 * 60 * 60 * 24);
            return days >= 30;
          result-encoding: string

      - uses: actions/checkout@v6
        if: steps.check.outputs.result == 'true'
        with:
          persist-credentials: false

      - uses: anomalyco/opencode/github@latest
        if: steps.check.outputs.result == 'true'
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          model: anthropic/claude-sonnet-4-20250514
          prompt: |
            Review this issue. If there's a clear fix or relevant docs:
            - Provide documentation links
            - Add error handling guidance for code examples
            Otherwise, do not comment.
```

### Manual Trigger (workflow_dispatch)

Trigger OpenCode on demand from the GitHub Actions UI. Useful for running tasks manually without writing a comment or waiting for a scheduled run.

```yaml
name: Manual OpenCode Task
on:
  workflow_dispatch:
    inputs:
      prompt:
        description: 'Task for OpenCode to perform'
        required: true
        type: string
      model:
        description: 'Model to use'
        required: false
        default: 'anthropic/claude-sonnet-4-20250514'
        type: string
jobs:
  opencode:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write
      pull-requests: write
      issues: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          persist-credentials: false

      - name: Run OpenCode
        uses: anomalyco/opencode/github@latest
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          model: ${{ inputs.model || 'anthropic/claude-sonnet-4-20250514' }}
          prompt: ${{ inputs.prompt }}
```

## Custom Prompts

Override the default prompt to customize OpenCode's behavior for your workflow. This is useful for enforcing specific review criteria, coding standards, or focus areas relevant to your project.

```yaml
- uses: anomalyco/opencode/github@latest
  with:
    model: anthropic/claude-sonnet-4-20250514
    prompt: |
      Review this pull request:
      - Check for code quality issues
      - Look for potential bugs
      - Suggest improvements
```

You can tailor prompts for different event types:

- **PR reviews**: Focus on code quality, security, performance, or style.
- **Issue triage**: Classify issues, request missing info, or auto-close duplicates.
- **Scheduled tasks**: Run audits, generate reports, or clean up stale branches.
- **Issue fixes**: Provide step-by-step solutions with code examples.

## Usage Examples

### Explain an Issue

Add this comment in a GitHub issue:

```
/opencode explain this issue
```

OpenCode will read the entire thread, including all comments, and reply with a clear explanation.

### Fix an Issue

In a GitHub issue, say:

```
/opencode fix this
```

OpenCode will create a new branch, implement the changes, and open a PR with the changes.

### Review PRs and Make Changes

Leave the following comment on a GitHub PR:

```
Delete the attachment from S3 when the note is removed /oc
```

OpenCode will implement the requested change and commit it to the same PR.

### Review Specific Code Lines

Leave a comment directly on code lines in the PR's "Files" tab. OpenCode automatically detects the file, line numbers, and diff context to provide precise responses.

```
[Comment on specific lines in Files tab]
/oc add error handling here
```

When commenting on specific lines, OpenCode receives:
- The exact file being reviewed
- The specific lines of code
- The surrounding diff context
- Line number information

This allows for more targeted requests without needing to specify file paths or line numbers manually.

## Permissions

The following permissions are required for OpenCode to function correctly:

| Permission | Purpose |
|------------|---------|
| `id-token: write` | Required for OpenCode's authentication and verification. |
| `contents: write` | Required for creating branches, committing changes, and pushing code. |
| `pull-requests: write` | Required for creating and updating pull requests, and adding PR comments. |
| `issues: write` | Required for commenting on and managing issues. |

For read-only operations (e.g., PR reviews that only add comments), you can use more restrictive permissions:

```yaml
permissions:
  id-token: write
  contents: read
  pull-requests: read
  issues: read
```

## Self-Hosted GitLab Reference

OpenCode also supports GitLab integration, which is handled separately from the GitHub integration. For GitLab-specific setup and configuration, refer to the [OpenCode GitLab documentation](https://opencode.ai/docs/gitlab/).

## Troubleshooting

### Common Issues

1. **Workflow not triggering**: Ensure the GitHub app is installed on the repository and the workflow file is in `.github/workflows/`.

2. **Permission denied errors**: Verify that the `permissions` block in your workflow includes the required scopes.

3. **Model not found**: Ensure the `model` parameter uses the correct `provider/model` format (e.g., `anthropic/claude-sonnet-4-20250514`).

4. **API key not found**: Confirm that the required secrets (e.g., `ANTHROPIC_API_KEY`) are stored in your repository or organization settings under **Settings > Secrets and variables > Actions**.

5. **No response to /opencode**: Check the `if` condition in the workflow — it must match the comment body containing `/opencode` or `/oc`.

## References

- [OpenCode GitHub Documentation](https://opencode.ai/docs/github/)
- [OpenCode GitHub App](https://github.com/apps/opencode-agent)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub GITHUB_TOKEN Documentation](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token)
