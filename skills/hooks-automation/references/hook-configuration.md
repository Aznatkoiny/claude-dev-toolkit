# Hook Configuration

Complete reference for hooks.json format, hook locations, configuration hierarchy, matcher patterns,
environment variables, and cross-platform considerations.

## Configuration Structure

Hooks are defined in JSON settings files with three levels of nesting:

1. Choose a **hook event** to respond to (e.g., `PreToolUse`, `Stop`)
2. Add a **matcher group** to filter when it fires (e.g., "only for the Bash tool")
3. Define one or more **hook handlers** to run when matched

```json
{
  "hooks": {
    "<EventName>": [
      {
        "matcher": "<regex pattern>",
        "hooks": [
          {
            "type": "command",
            "command": "your-script.sh"
          }
        ]
      }
    ]
  }
}
```

## Hook Locations

Where you define a hook determines its scope:

| Location                              | Scope                         | Shareable?                         |
| :------------------------------------ | :---------------------------- | :--------------------------------- |
| `~/.claude/settings.json`             | All your projects             | No, local to your machine          |
| `.claude/settings.json`               | Single project                | Yes, can be committed to the repo  |
| `.claude/settings.local.json`         | Single project                | No, gitignored                     |
| Managed policy settings               | Organization-wide             | Yes, admin-controlled              |
| Plugin `hooks/hooks.json`             | When plugin is enabled        | Yes, bundled with the plugin       |
| Skill or agent frontmatter            | While the component is active | Yes, defined in the component file |

Enterprise administrators can use `allowManagedHooksOnly` to block user, project, and plugin hooks.

### User-Scoped Hooks

Stored in `~/.claude/settings.json`. Apply to all your projects on this machine:

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude needs attention\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

### Project-Scoped Hooks

Stored in `.claude/settings.json` (shareable) or `.claude/settings.local.json` (gitignored):

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

### Plugin Hooks

Define plugin hooks in `hooks/hooks.json` within the plugin directory. Plugin hooks include an
optional top-level `description` field. When a plugin is enabled, its hooks merge with user and
project hooks.

Use `${CLAUDE_PLUGIN_ROOT}` to reference scripts bundled with the plugin:

```json
{
  "description": "Automatic code formatting",
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

Plugin hooks appear as `[Plugin]` in the `/hooks` menu and are read-only.

### Skill and Agent Frontmatter Hooks

Hooks can be defined directly in skills and subagents using YAML frontmatter. These hooks are
scoped to the component's lifecycle and only run when that component is active.

All hook events are supported. For subagents, `Stop` hooks are automatically converted to
`SubagentStop` since that is the event that fires when a subagent completes.

```yaml
---
name: secure-operations
description: Perform operations with security checks
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---
```

The `once` field is available for skill-scoped hooks (not agents). If `true`, the hook runs
only once per session then is removed.

## Hook Handler Fields

### Common Fields (All Types)

| Field           | Required | Description                                                                   |
| :-------------- | :------- | :---------------------------------------------------------------------------- |
| `type`          | yes      | `"command"`, `"prompt"`, or `"agent"`                                         |
| `timeout`       | no       | Seconds before canceling. Defaults: 600 for command, 30 for prompt, 60 for agent |
| `statusMessage` | no       | Custom spinner message displayed while the hook runs                          |
| `once`          | no       | If `true`, runs only once per session (skills only, not agents)               |

### Command Hook Fields

| Field     | Required | Description                                                     |
| :-------- | :------- | :-------------------------------------------------------------- |
| `command` | yes      | Shell command to execute                                        |
| `async`   | no       | If `true`, runs in the background without blocking              |

### Prompt and Agent Hook Fields

| Field    | Required | Description                                                                 |
| :------- | :------- | :-------------------------------------------------------------------------- |
| `prompt` | yes      | Prompt text. Use `$ARGUMENTS` as placeholder for hook input JSON            |
| `model`  | no       | Model to use for evaluation. Defaults to a fast model                       |

## Matcher Patterns

The `matcher` field is a regex string that filters when hooks fire. Use `"*"`, `""`, or omit
`matcher` entirely to match all occurrences.

| Event                                                                  | What the matcher filters  | Example matcher values                                                         |
| :--------------------------------------------------------------------- | :------------------------ | :----------------------------------------------------------------------------- |
| `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest` | tool name                 | `Bash`, `Edit\|Write`, `mcp__.*`                                               |
| `SessionStart`                                                         | how the session started   | `startup`, `resume`, `clear`, `compact`                                        |
| `SessionEnd`                                                           | why the session ended     | `clear`, `logout`, `prompt_input_exit`, `bypass_permissions_disabled`, `other`  |
| `Notification`                                                         | notification type         | `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`        |
| `SubagentStart`, `SubagentStop`                                        | agent type                | `Bash`, `Explore`, `Plan`, or custom agent names                               |
| `PreCompact`                                                           | what triggered compaction | `manual`, `auto`                                                               |
| `UserPromptSubmit`, `Stop`, `TeammateIdle`, `TaskCompleted`            | no matcher support        | always fires on every occurrence                                               |

The matcher is a regex, so `Edit|Write` matches either tool and `Notebook.*` matches any tool
starting with Notebook.

### Matching MCP Tools

MCP tools follow the naming pattern `mcp__<server>__<tool>`:

- `mcp__memory__create_entities` - Memory server's create entities tool
- `mcp__filesystem__read_file` - Filesystem server's read file tool
- `mcp__github__search_repositories` - GitHub server's search tool

Use regex patterns to target specific MCP tools or groups:

- `mcp__memory__.*` - matches all tools from the memory server
- `mcp__.*__write.*` - matches any tool containing "write" from any server

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__memory__.*",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Memory operation initiated' >> ~/mcp-operations.log"
          }
        ]
      },
      {
        "matcher": "mcp__.*__write.*",
        "hooks": [
          {
            "type": "command",
            "command": "/home/user/scripts/validate-mcp-write.py"
          }
        ]
      }
    ]
  }
}
```

## Environment Variables

| Variable               | Available in     | Description                                            |
| :--------------------- | :--------------- | :----------------------------------------------------- |
| `$CLAUDE_PROJECT_DIR`  | All hooks        | The project root. Wrap in quotes for paths with spaces |
| `${CLAUDE_PLUGIN_ROOT}`| Plugin hooks     | The plugin's root directory                            |
| `$CLAUDE_ENV_FILE`     | SessionStart     | File path to persist env vars for the session          |
| `$CLAUDE_CODE_REMOTE`  | All hooks        | Set to `"true"` in remote web environments             |

### Reference Scripts by Path

Use `$CLAUDE_PROJECT_DIR` for project scripts:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/check-style.sh"
          }
        ]
      }
    ]
  }
}
```

Use `${CLAUDE_PLUGIN_ROOT}` for plugin scripts:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

## Cross-Platform Wrapper Pattern

When creating hooks that must work on both Unix and Windows, use a wrapper pattern. Create
both a `.sh` script and a `.cmd` script, then reference the appropriate one or use a
dispatcher that detects the platform.

Example dispatcher script:

```bash
#!/bin/bash
# run-hook.sh - Unix version
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')
# ... your hook logic
```

```cmd
@echo off
REM run-hook.cmd - Windows version
REM Read JSON from stdin and process
```

In your hooks.json, reference the platform-appropriate script or use shell detection:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/run-hook.sh"
          }
        ]
      }
    ]
  }
}
```

## The /hooks Menu

Type `/hooks` in Claude Code to open the interactive hooks manager. Each hook is labeled:

- `[User]`: from `~/.claude/settings.json`
- `[Project]`: from `.claude/settings.json`
- `[Local]`: from `.claude/settings.local.json`
- `[Plugin]`: from a plugin's `hooks/hooks.json` (read-only)

## Disable or Remove Hooks

- Delete the hook entry from the settings file, or use `/hooks` menu to delete
- Set `"disableAllHooks": true` to temporarily disable all hooks without removing them
- Use the toggle in the `/hooks` menu for the same effect
- There is no way to disable an individual hook while keeping it in the configuration

## Hook Snapshot Behavior

Claude Code captures a snapshot of hooks at startup and uses it throughout the session. This
prevents malicious or accidental hook modifications from taking effect mid-session without review.

If hooks are modified externally while a session is running, Claude Code warns you and requires
review in the `/hooks` menu before changes apply. Hooks added through the `/hooks` menu take
effect immediately.

## Execution Details

- All matching hooks run in parallel
- Identical handlers are deduplicated automatically
- Handlers run in the current directory with Claude Code's environment
- Hook timeout is 10 minutes by default, configurable per hook with `timeout` (in seconds)
