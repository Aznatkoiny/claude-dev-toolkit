# Permission Modes Reference

Complete reference for all permission modes available to subagents and agent teams in Claude Code.

## Overview

The `permissionMode` field controls how a subagent handles permission prompts. Subagents inherit the permission context from the main conversation but can override the mode. Permission modes determine what actions require user approval and what actions proceed automatically.

## Permission Modes Table

| Mode | Behavior | Best For |
|:---|:---|:---|
| `default` | Standard permission checking with prompts | General-purpose subagents where you want to review each action |
| `acceptEdits` | Auto-accept file edits (Write, Edit) | Trusted code modification agents where you want file changes without prompts |
| `dontAsk` | Auto-deny permission prompts (explicitly allowed tools still work) | Read-only research agents or constrained agents that should never prompt |
| `delegate` | Coordination-only mode for agent team leads. Restricts to team management tools | Agent team leads that should only orchestrate, not implement |
| `bypassPermissions` | Skip all permission checks | Fully trusted automation or CI/CD pipelines (use with extreme caution) |
| `plan` | Plan mode (read-only exploration) | Research and planning phases where no modifications should occur |

## Detailed Mode Descriptions

### `default`

Standard permission checking. The subagent prompts for approval before performing actions that require permissions, just like the main conversation.

Use when:
- The subagent performs a mix of read and write operations
- You want to review each potentially destructive action
- You are testing a new subagent and want full visibility

### `acceptEdits`

Automatically accepts file edit operations (Write and Edit tools) without prompting. Other permission-requiring operations still prompt normally.

Use when:
- The subagent is trusted to modify files correctly
- You want to reduce friction for code modification workflows
- The subagent has focused, well-tested behavior for file changes

### `dontAsk`

Automatically denies any operation that would normally require a permission prompt. Tools that are explicitly allowed in the subagent's `tools` field still work, but any permission check is auto-denied rather than prompting.

Use when:
- The subagent should never interrupt the user with prompts
- You want a fully autonomous agent within strict boundaries
- Combined with a restrictive `tools` list for maximum constraint

### `delegate`

Coordination-only mode designed for agent team leads. Restricts the agent to team management tools only: spawning teammates, sending messages, shutting down teammates, and managing tasks. Prevents the lead from implementing tasks directly.

Use when:
- You want the team lead to focus purely on orchestration
- The lead should break down work and assign it, not do it
- You want to ensure all implementation is done by teammates

Enable delegate mode by pressing **Shift+Tab** after starting a team.

### `bypassPermissions`

Skips all permission checks entirely. The subagent can execute any operation without approval.

**Use with extreme caution.** This mode is appropriate only for:
- Fully trusted automation pipelines
- CI/CD environments where no human oversight is available
- Controlled testing environments

If the parent uses `bypassPermissions`, this takes precedence and cannot be overridden by the subagent.

### `plan`

Read-only exploration mode. The subagent can read and search code but cannot make modifications. Used for research and planning phases.

Use when:
- Gathering context before presenting a plan
- The subagent should analyze but not change anything
- You want safe, non-destructive exploration

## Permission Inheritance

### Subagents

Subagents inherit the permission context from the main conversation but can override it with the `permissionMode` field. The one exception: if the parent uses `bypassPermissions`, this takes precedence and cannot be overridden.

### Agent Teams

Teammates start with the lead's permission settings. If the lead runs with `--dangerously-skip-permissions`, all teammates inherit that. After spawning, you can change individual teammate modes, but you cannot set per-teammate modes at spawn time.

## Setting Permission Mode in Subagent Frontmatter

```yaml
---
name: safe-researcher
description: Read-only research agent
tools: Read, Grep, Glob
permissionMode: dontAsk
---

You are a research agent. Explore the codebase and report findings.
```

## Setting Permission Mode via CLI

When using the `--agents` CLI flag:

```bash
claude --agents '{
  "researcher": {
    "description": "Read-only research agent",
    "prompt": "Explore the codebase and report findings.",
    "tools": ["Read", "Grep", "Glob"],
    "permissionMode": "dontAsk"
  }
}'
```

## Common Permission Patterns

### Read-Only Explorer

```yaml
---
name: explorer
description: Safe codebase exploration
tools: Read, Grep, Glob
permissionMode: plan
---
```

### Trusted Code Modifier

```yaml
---
name: fixer
description: Fix code issues autonomously
tools: Read, Edit, Write, Bash, Grep, Glob
permissionMode: acceptEdits
---
```

### Constrained Automation

```yaml
---
name: linter
description: Run linting and report issues
tools: Bash, Read
permissionMode: dontAsk
---
```

### Team Orchestrator

```yaml
---
name: coordinator
description: Coordinates work across specialized agents
tools: Task(worker, researcher), Read
permissionMode: delegate
---
```

## Security Considerations

- **Principle of least privilege**: start with the most restrictive mode that allows the subagent to do its work
- **Combine with tool restrictions**: permission modes work alongside `tools` and `disallowedTools` for defense in depth
- **Avoid `bypassPermissions` in shared environments**: anyone with access to the subagent file can trigger unrestricted operations
- **Review before committing**: if project subagents are checked into version control, review permission modes in code review
- **Use hooks for fine-grained control**: when permission modes are too broad, use `PreToolUse` hooks to validate specific operations
- **Test new subagents with `default` mode first**: verify behavior before relaxing permissions
