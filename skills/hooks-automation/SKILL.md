---
name: hooks-automation
description: >
  Comprehensive guide to Claude Code hooks — the system for running shell commands, LLM prompts, or
  subagents automatically at specific points in Claude Code's lifecycle. Use when creating, configuring,
  debugging, or understanding Claude Code hooks. Trigger phrases: "create a hook", "hook configuration",
  "hooks.json", "PreToolUse", "PostToolUse", "SessionStart", "hook events", "hook matchers",
  "automate with hooks", "prompt hooks", "agent hooks", "hook lifecycle", "UserPromptSubmit",
  "PermissionRequest", "PostToolUseFailure", "SubagentStart", "SubagentStop", "Stop hook",
  "TeammateIdle", "TaskCompleted", "PreCompact", "SessionEnd", "Notification hook".
  Also for blocking dangerous commands, auto-formatting code, desktop notifications, file protection,
  context re-injection after compaction, async hooks, MCP tool hooks, and hook debugging.
---

# Hooks Automation

## Overview

Hooks are user-defined shell commands, LLM prompts, or subagents that execute automatically at
specific points in Claude Code's lifecycle. They provide **deterministic control** over Claude Code's
behavior, ensuring certain actions always happen rather than relying on the LLM to choose to run them.
Use hooks to enforce project rules, automate repetitive tasks, and integrate Claude Code with your
existing tools.

There are three types of hook handlers. **Command hooks** (`type: "command"`) run a shell command
that receives JSON on stdin and communicates results through exit codes and stdout. **Prompt hooks**
(`type: "prompt"`) send a prompt to a Claude model for single-turn yes/no evaluation. **Agent hooks**
(`type: "agent"`) spawn a subagent that can use tools like Read, Grep, and Glob to verify conditions
before returning a decision.

Hooks are defined in JSON settings files or in skill/agent YAML frontmatter. The configuration has
three levels: choose a **hook event** (the lifecycle point), add a **matcher group** (a regex filter
for when it fires), and define one or more **hook handlers** (the command, prompt, or agent that runs).

## When to Use Hooks

- Enforce project rules deterministically (block dangerous commands, protect files)
- Automate post-edit workflows (formatting, linting, test runs)
- Send notifications when Claude needs input
- Inject context at session start or after compaction
- Validate tool inputs before execution
- Auto-approve or deny permission requests programmatically
- Run quality gates before task completion or agent stopping

## When NOT to Use Hooks

- For static context injection, use CLAUDE.md instead
- For one-time instructions, use the prompt directly
- For complex multi-step workflows that need judgment, consider skills or subagents
- For decisions that don't map to a specific lifecycle event

## Quick Start Example

The simplest hook blocks a dangerous command. Add this to `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/block-rm.sh"
          }
        ]
      }
    ]
  }
}
```

The script reads JSON input from stdin, checks the command, and returns a decision:

```bash
#!/bin/bash
# .claude/hooks/block-rm.sh
COMMAND=$(jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Destructive command blocked by hook"
    }
  }'
else
  exit 0  # allow the command
fi
```

How it resolves: the `PreToolUse` event fires, the matcher `"Bash"` matches the tool name,
the hook handler runs, and Claude Code reads the JSON decision to block the tool call.

## Quick Reference Router

| If you want to...                                    | Read this file                    |
| :--------------------------------------------------- | :-------------------------------- |
| Look up an event's input schema or output format     | `hook-events-reference.md`        |
| Understand hooks.json format, locations, precedence  | `hook-configuration.md`           |
| Use prompt-based or agent-based hooks                | `prompt-and-agent-hooks.md`       |
| See working recipes and practical examples           | `hook-recipes.md`                 |

## Hook Lifecycle

Hooks fire at specific points during a Claude Code session. The lifecycle flows as:

```
SessionStart
    |
    v
UserPromptSubmit  (user submits a prompt)
    |
    v
  [Agentic Loop]
    |
    +---> PreToolUse        (before tool call)
    |         |
    |     PermissionRequest  (if permission dialog shown)
    |         |
    |     PostToolUse        (after tool succeeds)
    |     PostToolUseFailure (after tool fails)
    |         |
    |     SubagentStart      (when subagent spawns)
    |     SubagentStop       (when subagent finishes)
    |         |
    |     Notification       (when notification fires)
    |         |
    +---> PreCompact         (before compaction)
    |
    v
  Stop                       (Claude finishes responding)
  TeammateIdle               (teammate about to go idle)
  TaskCompleted              (task marked as completed)
    |
    v
SessionEnd
```

When an event fires and a matcher matches, Claude Code passes JSON context about the event
to your hook handler. For command hooks, this arrives on stdin. Your handler can then inspect
the input, take action, and optionally return a decision.

## All 14 Hook Events

| Event                | When it fires                                     | Can block? |
| :------------------- | :------------------------------------------------ | :--------- |
| `SessionStart`       | When a session begins or resumes                  | No         |
| `UserPromptSubmit`   | When you submit a prompt, before processing       | Yes        |
| `PreToolUse`         | Before a tool call executes                       | Yes        |
| `PermissionRequest`  | When a permission dialog appears                  | Yes        |
| `PostToolUse`        | After a tool call succeeds                        | No         |
| `PostToolUseFailure` | After a tool call fails                           | No         |
| `Notification`       | When Claude Code sends a notification             | No         |
| `SubagentStart`      | When a subagent is spawned                        | No         |
| `SubagentStop`       | When a subagent finishes                          | Yes        |
| `Stop`               | When Claude finishes responding                   | Yes        |
| `TeammateIdle`       | When an agent team teammate is about to go idle   | Yes        |
| `TaskCompleted`      | When a task is being marked as completed          | Yes        |
| `PreCompact`         | Before context compaction                         | No         |
| `SessionEnd`         | When a session terminates                         | No         |

## Hook Types Summary

### Command Hooks (`type: "command"`)

Run a shell command. Receives JSON on stdin, returns decisions via exit codes and stdout.
Supports `async: true` for background execution.

**Key fields**: `command` (required), `async` (optional), `timeout` (default 600s)

### Prompt Hooks (`type: "prompt"`)

Send a prompt to a Claude model (Haiku by default) for single-turn evaluation.
The model returns `{ "ok": true }` or `{ "ok": false, "reason": "..." }`.

**Key fields**: `prompt` (required, use `$ARGUMENTS` for hook input), `model` (optional), `timeout` (default 30s)

### Agent Hooks (`type: "agent"`)

Spawn a subagent with tool access (Read, Grep, Glob) for multi-turn verification.
Same response format as prompt hooks. Up to 50 turns.

**Key fields**: `prompt` (required), `model` (optional), `timeout` (default 60s)

## Matcher Patterns Overview

The `matcher` field is a regex string that filters when hooks fire. Omit or use `""` / `"*"` to
match all occurrences. Different event types match on different fields:

- **Tool events** (`PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`): match on tool name (e.g., `Bash`, `Edit|Write`, `mcp__.*`)
- **`SessionStart`**: match on how session started (`startup`, `resume`, `clear`, `compact`)
- **`SessionEnd`**: match on exit reason (`clear`, `logout`, `prompt_input_exit`, `other`)
- **`Notification`**: match on type (`permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`)
- **`SubagentStart` / `SubagentStop`**: match on agent type (`Bash`, `Explore`, `Plan`, or custom names)
- **`PreCompact`**: match on trigger (`manual`, `auto`)
- **`UserPromptSubmit`, `Stop`, `TeammateIdle`, `TaskCompleted`**: no matcher support, always fire

MCP tools follow the pattern `mcp__<server>__<tool>` and can be matched with regex like
`mcp__memory__.*` or `mcp__.*__write.*`.

## Hook Input and Output Overview

All hooks receive JSON on stdin with common fields (`session_id`, `transcript_path`, `cwd`,
`permission_mode`, `hook_event_name`) plus event-specific fields. Hooks communicate results
through exit codes, stdout JSON, and stderr.

### Exit Code Semantics

- **Exit 0**: Success. Action proceeds. stdout parsed for JSON output.
- **Exit 2**: Blocking error. Action blocked. stderr fed back as error message.
- **Any other code**: Non-blocking error. stderr logged, execution continues.

### JSON Output

On exit 0, you can print a JSON object to stdout for structured control. Universal fields
include `continue` (stop Claude entirely if `false`), `stopReason`, `suppressOutput`, and
`systemMessage`. Event-specific fields vary; see `hook-events-reference.md` for details.

### Decision Control Patterns

| Events                                                                | Pattern              |
| :-------------------------------------------------------------------- | :------------------- |
| UserPromptSubmit, PostToolUse, PostToolUseFailure, Stop, SubagentStop | Top-level `decision: "block"` with `reason` |
| PreToolUse                                                            | `hookSpecificOutput` with `permissionDecision` (allow/deny/ask) |
| PermissionRequest                                                     | `hookSpecificOutput` with `decision.behavior` (allow/deny) |
| TeammateIdle, TaskCompleted                                           | Exit code 2 only (stderr as feedback) |

## Async Hooks

Set `"async": true` on command hooks to run in the background without blocking Claude.
Async hooks cannot block or return decisions. Results (`systemMessage` or `additionalContext`)
are delivered on the next conversation turn. Only `type: "command"` supports async.

## Security Considerations

- Hooks execute with your full user permissions
- Validate and sanitize inputs; never trust input data blindly
- Always quote shell variables: use `"$VAR"` not `$VAR`
- Block path traversal: check for `..` in file paths
- Use absolute paths for scripts, using `"$CLAUDE_PROJECT_DIR"` for the project root
- Avoid processing `.env`, `.git/`, keys, and other sensitive files

## Configuration Locations

| Location                              | Scope                         | Shareable?                         |
| :------------------------------------ | :---------------------------- | :--------------------------------- |
| `~/.claude/settings.json`             | All your projects             | No, local to your machine          |
| `.claude/settings.json`               | Single project                | Yes, can be committed to the repo  |
| `.claude/settings.local.json`         | Single project                | No, gitignored                     |
| Managed policy settings               | Organization-wide             | Yes, admin-controlled              |
| Plugin `hooks/hooks.json`             | When plugin is enabled        | Yes, bundled with the plugin       |
| Skill or agent frontmatter            | While the component is active | Yes, defined in the component file |

## Debugging Hooks

- Run `claude --debug` for full execution details (matched hooks, exit codes, output)
- Toggle verbose mode with `Ctrl+O` to see hook progress in the transcript
- Test hooks manually: `echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' | ./my-hook.sh`
- Use `/hooks` menu to view, add, and delete hooks interactively

## Reference File Index

1. **`references/hook-events-reference.md`** - Complete table of all 14 hook events with firing
   conditions, input/output schemas, matcher patterns, and decision control options for each event.

2. **`references/hook-configuration.md`** - hooks.json format, hook locations (user/project/plugin/
   skill-scoped), configuration hierarchy, `${CLAUDE_PLUGIN_ROOT}` for plugin hooks, matcher patterns,
   and cross-platform considerations.

3. **`references/prompt-and-agent-hooks.md`** - How prompt hooks work (`type: "prompt"` with `model`
   field), agent hooks (`type: "agent"`), differences from command hooks, when to use each, cost
   considerations, and complete examples.

4. **`references/hook-recipes.md`** - Working recipes: desktop notifications, auto-format with Prettier,
   protected files, context re-injection after compaction, rm -rf blocking, async test runners,
   MCP tool logging, session cleanup, and troubleshooting patterns.
