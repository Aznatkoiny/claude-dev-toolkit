# Hook Events Reference

Complete reference for all 14 Claude Code hook events, including firing conditions, input schemas,
matcher patterns, and decision control options.

## Common Input Fields

All hook events receive these fields via stdin as JSON:

| Field             | Description                                                                  |
| :---------------- | :--------------------------------------------------------------------------- |
| `session_id`      | Current session identifier                                                   |
| `transcript_path` | Path to conversation JSON                                                    |
| `cwd`             | Current working directory when the hook is invoked                           |
| `permission_mode` | Current permission mode: `"default"`, `"plan"`, `"acceptEdits"`, `"dontAsk"`, or `"bypassPermissions"` |
| `hook_event_name` | Name of the event that fired                                                 |

Example common input:

```json
{
  "session_id": "abc123",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/my-project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

## Universal JSON Output Fields

These fields work across all events when your hook exits 0 and prints JSON to stdout:

| Field            | Default | Description                                                                |
| :--------------- | :------ | :------------------------------------------------------------------------- |
| `continue`       | `true`  | If `false`, Claude stops processing entirely. Takes precedence over event-specific decisions |
| `stopReason`     | none    | Message shown to the user when `continue` is `false`. Not shown to Claude  |
| `suppressOutput` | `false` | If `true`, hides stdout from verbose mode output                           |
| `systemMessage`  | none    | Warning message shown to the user                                          |

To stop Claude entirely regardless of event type:

```json
{ "continue": false, "stopReason": "Build failed, fix errors before continuing" }
```

## Exit Code Behavior Per Event

| Hook event           | Can block? | What happens on exit 2                                             |
| :------------------- | :--------- | :----------------------------------------------------------------- |
| `PreToolUse`         | Yes        | Blocks the tool call                                               |
| `PermissionRequest`  | Yes        | Denies the permission                                              |
| `UserPromptSubmit`   | Yes        | Blocks prompt processing and erases the prompt                     |
| `Stop`               | Yes        | Prevents Claude from stopping, continues the conversation          |
| `SubagentStop`       | Yes        | Prevents the subagent from stopping                                |
| `TeammateIdle`       | Yes        | Prevents the teammate from going idle                              |
| `TaskCompleted`      | Yes        | Prevents the task from being marked as completed                   |
| `PostToolUse`        | No         | Shows stderr to Claude (tool already ran)                          |
| `PostToolUseFailure` | No         | Shows stderr to Claude (tool already failed)                       |
| `Notification`       | No         | Shows stderr to user only                                          |
| `SubagentStart`      | No         | Shows stderr to user only                                          |
| `SessionStart`       | No         | Shows stderr to user only                                          |
| `SessionEnd`         | No         | Shows stderr to user only                                          |
| `PreCompact`         | No         | Shows stderr to user only                                          |

## Decision Control Patterns Summary

| Events                                                                | Decision pattern     | Key fields                                                        |
| :-------------------------------------------------------------------- | :------------------- | :---------------------------------------------------------------- |
| UserPromptSubmit, PostToolUse, PostToolUseFailure, Stop, SubagentStop | Top-level `decision` | `decision: "block"`, `reason`                                     |
| TeammateIdle, TaskCompleted                                           | Exit code only       | Exit code 2 blocks the action, stderr is fed back as feedback     |
| PreToolUse                                                            | `hookSpecificOutput` | `permissionDecision` (allow/deny/ask), `permissionDecisionReason` |
| PermissionRequest                                                     | `hookSpecificOutput` | `decision.behavior` (allow/deny)                                  |

---

## SessionStart

Runs when Claude Code starts a new session or resumes an existing one.

### Matcher Values

| Matcher   | When it fires                          |
| :-------- | :------------------------------------- |
| `startup` | New session                            |
| `resume`  | `--resume`, `--continue`, or `/resume` |
| `clear`   | `/clear`                               |
| `compact` | Auto or manual compaction              |

### Input Schema

Additional fields beyond common input:

| Field        | Description                                                                          |
| :----------- | :----------------------------------------------------------------------------------- |
| `source`     | How the session started: `"startup"`, `"resume"`, `"clear"`, or `"compact"`          |
| `model`      | The model identifier (e.g., `"claude-sonnet-4-5-20250929"`)                          |
| `agent_type` | Present if started with `claude --agent <name>`. Contains the agent name             |

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.claude/projects/.../00893aaf.jsonl",
  "cwd": "/Users/...",
  "permission_mode": "default",
  "hook_event_name": "SessionStart",
  "source": "startup",
  "model": "claude-sonnet-4-5-20250929"
}
```

### Decision Control

Stdout text is added as context for Claude. You can also return structured JSON:

| Field               | Description                                                               |
| :------------------ | :------------------------------------------------------------------------ |
| `additionalContext` | String added to Claude's context. Multiple hooks' values are concatenated |

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "My additional context here"
  }
}
```

### Persist Environment Variables

SessionStart hooks have access to `CLAUDE_ENV_FILE`. Write `export` statements to persist
environment variables for all subsequent Bash commands:

```bash
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=production' >> "$CLAUDE_ENV_FILE"
  echo 'export DEBUG_LOG=true' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

To capture all env changes from setup commands:

```bash
#!/bin/bash
ENV_BEFORE=$(export -p | sort)
source ~/.nvm/nvm.sh
nvm use 20
if [ -n "$CLAUDE_ENV_FILE" ]; then
  ENV_AFTER=$(export -p | sort)
  comm -13 <(echo "$ENV_BEFORE") <(echo "$ENV_AFTER") >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

> Note: `CLAUDE_ENV_FILE` is only available for SessionStart hooks. Other hook types do not have access.

---

## UserPromptSubmit

Runs when the user submits a prompt, before Claude processes it.

### Matcher

No matcher support. Always fires on every prompt submission.

### Input Schema

| Field    | Description                        |
| :------- | :--------------------------------- |
| `prompt` | The text the user submitted        |

```json
{
  "session_id": "abc123",
  "hook_event_name": "UserPromptSubmit",
  "prompt": "Write a function to calculate the factorial of a number"
}
```

### Decision Control

| Field               | Description                                                                                        |
| :------------------ | :------------------------------------------------------------------------------------------------- |
| `decision`          | `"block"` prevents the prompt from being processed and erases it from context. Omit to allow       |
| `reason`            | Shown to the user when `decision` is `"block"`. Not added to context                               |
| `additionalContext` | String added to Claude's context                                                                   |

Plain text stdout is also added as context (non-JSON text on exit 0).

```json
{
  "decision": "block",
  "reason": "Explanation for decision",
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "My additional context here"
  }
}
```

---

## PreToolUse

Runs after Claude creates tool parameters and before processing the tool call.

### Matcher

Matches on tool name: `Bash`, `Edit`, `Write`, `Read`, `Glob`, `Grep`, `Task`, `WebFetch`,
`WebSearch`, and any MCP tool names (pattern: `mcp__<server>__<tool>`).

### Input Schema

Additional fields beyond common input:

| Field         | Description                        |
| :------------ | :--------------------------------- |
| `tool_name`   | Name of the tool being called      |
| `tool_input`  | Arguments passed to the tool       |
| `tool_use_id` | Unique identifier for this call    |

#### Tool Input Schemas by Tool

**Bash**:

| Field               | Type    | Description                                   |
| :------------------ | :------ | :-------------------------------------------- |
| `command`           | string  | The shell command to execute                   |
| `description`       | string  | Optional description of what the command does  |
| `timeout`           | number  | Optional timeout in milliseconds               |
| `run_in_background` | boolean | Whether to run in background                   |

**Write**:

| Field       | Type   | Description                        |
| :---------- | :----- | :--------------------------------- |
| `file_path` | string | Absolute path to the file to write |
| `content`   | string | Content to write to the file       |

**Edit**:

| Field         | Type    | Description                        |
| :------------ | :------ | :--------------------------------- |
| `file_path`   | string  | Absolute path to the file to edit  |
| `old_string`  | string  | Text to find and replace           |
| `new_string`  | string  | Replacement text                   |
| `replace_all` | boolean | Whether to replace all occurrences |

**Read**:

| Field       | Type   | Description                                |
| :---------- | :----- | :----------------------------------------- |
| `file_path` | string | Absolute path to the file to read          |
| `offset`    | number | Optional line number to start reading from |
| `limit`     | number | Optional number of lines to read           |

**Glob**:

| Field     | Type   | Description                                    |
| :-------- | :----- | :--------------------------------------------- |
| `pattern` | string | Glob pattern to match files against            |
| `path`    | string | Optional directory to search in                |

**Grep**:

| Field         | Type    | Description                                              |
| :------------ | :------ | :------------------------------------------------------- |
| `pattern`     | string  | Regular expression pattern to search for                 |
| `path`        | string  | Optional file or directory to search in                  |
| `glob`        | string  | Optional glob pattern to filter files                    |
| `output_mode` | string  | `"content"`, `"files_with_matches"`, or `"count"`        |
| `-i`          | boolean | Case insensitive search                                  |
| `multiline`   | boolean | Enable multiline matching                                |

**WebFetch**:

| Field    | Type   | Description                          |
| :------- | :----- | :----------------------------------- |
| `url`    | string | URL to fetch content from            |
| `prompt` | string | Prompt to run on the fetched content |

**WebSearch**:

| Field             | Type   | Description                                       |
| :---------------- | :----- | :------------------------------------------------ |
| `query`           | string | Search query                                      |
| `allowed_domains` | array  | Optional: only include results from these domains |
| `blocked_domains` | array  | Optional: exclude results from these domains      |

**Task**:

| Field           | Type   | Description                                  |
| :-------------- | :----- | :------------------------------------------- |
| `prompt`        | string | The task for the agent to perform            |
| `description`   | string | Short description of the task                |
| `subagent_type` | string | Type of specialized agent to use             |
| `model`         | string | Optional model alias to override the default |

### Decision Control

Uses `hookSpecificOutput` with richer control than other events:

| Field                      | Description                                                                           |
| :------------------------- | :------------------------------------------------------------------------------------ |
| `permissionDecision`       | `"allow"` bypasses permission, `"deny"` prevents tool call, `"ask"` prompts the user  |
| `permissionDecisionReason` | For `"allow"`/`"ask"`: shown to user. For `"deny"`: shown to Claude                   |
| `updatedInput`             | Modifies the tool's input parameters before execution                                 |
| `additionalContext`        | String added to Claude's context before tool executes                                 |

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Database writes are not allowed",
    "updatedInput": {
      "field_to_modify": "new value"
    },
    "additionalContext": "Current environment: production."
  }
}
```

> Note: Top-level `decision` and `reason` are deprecated for PreToolUse. Use `hookSpecificOutput.permissionDecision` and `hookSpecificOutput.permissionDecisionReason` instead.

---

## PermissionRequest

Runs when the user is shown a permission dialog.

### Matcher

Matches on tool name, same values as PreToolUse.

### Input Schema

Same as PreToolUse (`tool_name`, `tool_input`) but without `tool_use_id`. Includes an optional
`permission_suggestions` array with "always allow" options.

```json
{
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf node_modules"
  },
  "permission_suggestions": [
    { "type": "toolAlwaysAllow", "tool": "Bash" }
  ]
}
```

### Decision Control

| Field                | Description                                                                |
| :------------------- | :------------------------------------------------------------------------- |
| `behavior`           | `"allow"` grants the permission, `"deny"` denies it                        |
| `updatedInput`       | For `"allow"` only: modifies tool input before execution                   |
| `updatedPermissions` | For `"allow"` only: applies permission rule updates (like "always allow")  |
| `message`            | For `"deny"` only: tells Claude why the permission was denied              |
| `interrupt`          | For `"deny"` only: if `true`, stops Claude                                 |

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedInput": {
        "command": "npm run lint"
      }
    }
  }
}
```

> Note: `PermissionRequest` hooks do not fire in non-interactive mode (`-p`). Use `PreToolUse` hooks for automated permission decisions.

---

## PostToolUse

Runs immediately after a tool completes successfully.

### Matcher

Matches on tool name, same values as PreToolUse.

### Input Schema

Includes both `tool_input` (arguments sent to the tool) and `tool_response` (the result):

```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/path/to/file.txt",
    "content": "file content"
  },
  "tool_response": {
    "filePath": "/path/to/file.txt",
    "success": true
  },
  "tool_use_id": "toolu_01ABC123..."
}
```

### Decision Control

| Field                  | Description                                                    |
| :--------------------- | :------------------------------------------------------------- |
| `decision`             | `"block"` prompts Claude with the `reason`. Omit to allow     |
| `reason`               | Explanation shown to Claude when `decision` is `"block"`       |
| `additionalContext`    | Additional context for Claude to consider                      |
| `updatedMCPToolOutput` | For MCP tools only: replaces the tool's output                 |

```json
{
  "decision": "block",
  "reason": "Explanation for decision",
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Additional information for Claude"
  }
}
```

---

## PostToolUseFailure

Runs when a tool execution fails.

### Matcher

Matches on tool name, same values as PreToolUse.

### Input Schema

Same as PostToolUse but with error info instead of tool_response:

| Field          | Description                                                     |
| :------------- | :-------------------------------------------------------------- |
| `error`        | String describing what went wrong                               |
| `is_interrupt` | Optional boolean: whether failure was caused by user interruption |

```json
{
  "hook_event_name": "PostToolUseFailure",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  },
  "tool_use_id": "toolu_01ABC123...",
  "error": "Command exited with non-zero status code 1",
  "is_interrupt": false
}
```

### Decision Control

| Field               | Description                                                   |
| :------------------ | :------------------------------------------------------------ |
| `additionalContext` | Additional context for Claude to consider alongside the error |

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUseFailure",
    "additionalContext": "Additional information about the failure"
  }
}
```

---

## Notification

Runs when Claude Code sends notifications.

### Matcher Values

| Matcher               | Description                    |
| :-------------------- | :----------------------------- |
| `permission_prompt`   | Permission dialog shown        |
| `idle_prompt`         | Claude is idle, waiting        |
| `auth_success`        | Authentication succeeded       |
| `elicitation_dialog`  | Elicitation dialog shown       |

### Input Schema

| Field               | Description                    |
| :------------------ | :----------------------------- |
| `message`           | Notification text              |
| `title`             | Optional notification title    |
| `notification_type` | Which type fired               |

```json
{
  "hook_event_name": "Notification",
  "message": "Claude Code needs your permission to use Bash",
  "title": "Permission needed",
  "notification_type": "permission_prompt"
}
```

### Decision Control

Cannot block or modify notifications. Can return `additionalContext` to add context.

---

## SubagentStart

Runs when a Claude Code subagent is spawned via the Task tool.

### Matcher

Matches on agent type: `Bash`, `Explore`, `Plan`, or custom agent names.

### Input Schema

| Field        | Description                                 |
| :----------- | :------------------------------------------ |
| `agent_id`   | Unique identifier for the subagent          |
| `agent_type` | Agent name (built-in or custom)             |

```json
{
  "hook_event_name": "SubagentStart",
  "agent_id": "agent-abc123",
  "agent_type": "Explore"
}
```

### Decision Control

Cannot block subagent creation. Can inject context:

| Field               | Description                            |
| :------------------ | :------------------------------------- |
| `additionalContext` | String added to the subagent's context |

---

## SubagentStop

Runs when a Claude Code subagent has finished responding.

### Matcher

Matches on agent type, same values as SubagentStart.

### Input Schema

| Field                    | Description                                           |
| :----------------------- | :---------------------------------------------------- |
| `stop_hook_active`       | Whether Claude is continuing due to a previous stop hook |
| `agent_id`               | Unique identifier for the subagent                    |
| `agent_type`             | Agent name (used for matcher filtering)               |
| `agent_transcript_path`  | Path to the subagent's own transcript                 |

```json
{
  "hook_event_name": "SubagentStop",
  "stop_hook_active": false,
  "agent_id": "def456",
  "agent_type": "Explore",
  "agent_transcript_path": "~/.claude/projects/.../subagents/agent-def456.jsonl"
}
```

### Decision Control

Same as Stop hooks: `decision: "block"` with `reason` to prevent the subagent from stopping.

---

## Stop

Runs when the main Claude Code agent has finished responding. Does not run on user interrupts.

### Matcher

No matcher support. Always fires.

### Input Schema

| Field              | Description                                                  |
| :----------------- | :----------------------------------------------------------- |
| `stop_hook_active` | `true` when Claude is already continuing due to a stop hook. Check this to prevent infinite loops |

```json
{
  "hook_event_name": "Stop",
  "stop_hook_active": true
}
```

### Decision Control

| Field      | Description                                                                |
| :--------- | :------------------------------------------------------------------------- |
| `decision` | `"block"` prevents Claude from stopping. Omit to allow                     |
| `reason`   | Required when `decision` is `"block"`. Tells Claude why it should continue |

```json
{
  "decision": "block",
  "reason": "Must be provided when Claude is blocked from stopping"
}
```

---

## TeammateIdle

Runs when an agent team teammate is about to go idle after finishing its turn.

### Matcher

No matcher support. Always fires.

### Input Schema

| Field           | Description                                   |
| :-------------- | :-------------------------------------------- |
| `teammate_name` | Name of the teammate about to go idle         |
| `team_name`     | Name of the team                              |

```json
{
  "hook_event_name": "TeammateIdle",
  "teammate_name": "researcher",
  "team_name": "my-project"
}
```

### Decision Control

Exit code only. Exit 2 with stderr feedback prevents the teammate from going idle.

```bash
#!/bin/bash
if [ ! -f "./dist/output.js" ]; then
  echo "Build artifact missing. Run the build before stopping." >&2
  exit 2
fi
exit 0
```

---

## TaskCompleted

Runs when a task is being marked as completed (via TaskUpdate or when a teammate finishes with
in-progress tasks).

### Matcher

No matcher support. Always fires.

### Input Schema

| Field              | Description                                             |
| :----------------- | :------------------------------------------------------ |
| `task_id`          | Identifier of the task being completed                  |
| `task_subject`     | Title of the task                                       |
| `task_description` | Detailed description (may be absent)                    |
| `teammate_name`    | Name of the teammate completing (may be absent)         |
| `team_name`        | Name of the team (may be absent)                        |

```json
{
  "hook_event_name": "TaskCompleted",
  "task_id": "task-001",
  "task_subject": "Implement user authentication",
  "task_description": "Add login and signup endpoints",
  "teammate_name": "implementer",
  "team_name": "my-project"
}
```

### Decision Control

Exit code only. Exit 2 with stderr feedback prevents the task from being marked as completed.

```bash
#!/bin/bash
INPUT=$(cat)
TASK_SUBJECT=$(echo "$INPUT" | jq -r '.task_subject')
if ! npm test 2>&1; then
  echo "Tests not passing. Fix failing tests before completing: $TASK_SUBJECT" >&2
  exit 2
fi
exit 0
```

---

## PreCompact

Runs before Claude Code runs a compact operation.

### Matcher Values

| Matcher  | When it fires                                |
| :------- | :------------------------------------------- |
| `manual` | `/compact`                                   |
| `auto`   | Auto-compact when context window is full     |

### Input Schema

| Field                 | Description                                                         |
| :-------------------- | :------------------------------------------------------------------ |
| `trigger`             | `"manual"` or `"auto"`                                              |
| `custom_instructions` | For `manual`: what user passed to `/compact`. For `auto`: empty     |

```json
{
  "hook_event_name": "PreCompact",
  "trigger": "manual",
  "custom_instructions": ""
}
```

### Decision Control

No decision control. Cannot block compaction.

---

## SessionEnd

Runs when a Claude Code session ends.

### Matcher Values

| Reason                        | Description                                |
| :---------------------------- | :----------------------------------------- |
| `clear`                       | Session cleared with `/clear`              |
| `logout`                      | User logged out                            |
| `prompt_input_exit`           | User exited while prompt input was visible |
| `bypass_permissions_disabled` | Bypass permissions mode was disabled       |
| `other`                       | Other exit reasons                         |

### Input Schema

| Field    | Description                                     |
| :------- | :---------------------------------------------- |
| `reason` | Why the session ended (see matcher values above) |

```json
{
  "hook_event_name": "SessionEnd",
  "reason": "other"
}
```

### Decision Control

No decision control. Cannot block session termination. Useful for cleanup tasks and logging.
