---
name: subagents-and-teams
description: >
  Comprehensive guide to creating custom subagents and orchestrating agent teams in Claude Code.
  Use when creating custom subagents, configuring agent teams, understanding built-in subagents,
  or choosing between subagents and teams. Trigger phrases: "create a subagent", "custom agent",
  "agent teams", "built-in subagents", "Explore agent", "Plan agent", "agent configuration",
  "subagent tools", "permission modes", "agent frontmatter", "team coordination", "parallel agents",
  "delegate mode", "subagent hooks", "agent memory", "Task tool", "spawn teammates",
  "subagent vs team", "agent team display mode", "tmux agents", "agent team tasks".
---

# Subagents and Agent Teams

Claude Code supports two distinct approaches for parallelizing and delegating work: **subagents** and **agent teams**. Understanding when to use each is critical for effective workflows.

**Subagents** are specialized AI assistants that run within a single session. Each subagent operates in its own context window with a custom system prompt, specific tool access, and independent permissions. When Claude encounters a task matching a subagent's description, it delegates to that subagent, which works independently and returns results to the main conversation. Subagents cannot spawn other subagents.

**Agent teams** coordinate multiple independent Claude Code instances working together. One session acts as team lead, creating tasks, spawning teammates, and synthesizing results. Teammates work independently in their own context windows and can communicate directly with each other -- not just back to the lead. Agent teams are experimental and must be explicitly enabled.

Subagents help you:

- **Preserve context** by keeping exploration and implementation out of your main conversation
- **Enforce constraints** by limiting which tools a subagent can use
- **Reuse configurations** across projects with user-level subagents
- **Specialize behavior** with focused system prompts for specific domains
- **Control costs** by routing tasks to faster, cheaper models like Haiku

## When to Use Subagents vs Agent Teams

### Use Subagents When

- You need focused workers that report back results to a single session
- The task produces verbose output you want kept out of your main context
- You want to enforce specific tool restrictions or permission modes
- The work is self-contained and can return a summary
- You want to control costs by routing tasks to cheaper/faster models
- You need quick, parallel research on independent topics

### Use Agent Teams When

- Teammates need to share findings and challenge each other
- Work requires sustained discussion, coordination, and collaboration
- You want teammates to self-claim tasks from a shared task list
- The problem benefits from competing hypotheses investigated in parallel
- Changes span multiple layers (frontend, backend, tests) owned by different agents
- Tasks are too large for a single context window

### Use the Main Conversation When

- The task needs frequent back-and-forth or iterative refinement
- Multiple phases share significant context (planning, implementation, testing)
- You are making a quick, targeted change
- Latency matters -- subagents start fresh and may need time to gather context

## Quick Reference: Subagents vs Agent Teams

| Aspect | Subagents | Agent Teams |
|:---|:---|:---|
| **Context** | Own context window; results return to caller | Own context window; fully independent |
| **Communication** | Report results back to main agent only | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list with self-coordination |
| **Best for** | Focused tasks where only the result matters | Complex work requiring discussion and collaboration |
| **Token cost** | Lower: results summarized back to main context | Higher: each teammate is a separate Claude instance |
| **Nesting** | Cannot spawn other subagents | Cannot spawn nested teams |
| **Setup** | Available by default | Requires experimental flag |

## Built-in Subagents

Claude Code includes built-in subagents that it automatically uses when appropriate:

| Subagent | Model | Tools | Purpose |
|:---|:---|:---|:---|
| **Explore** | Haiku | Read-only (denied Write/Edit) | File discovery, code search, codebase exploration |
| **Plan** | Inherits | Read-only (denied Write/Edit) | Codebase research for planning |
| **General-purpose** | Inherits | All tools | Complex research, multi-step operations, code modifications |
| **Bash** | Inherits | Terminal context | Running terminal commands in a separate context |
| **statusline-setup** | Sonnet | Specialized | Configuring status line via `/statusline` |
| **Claude Code Guide** | Haiku | Specialized | Answering Claude Code feature questions |

The **Explore** subagent is invoked with a thoroughness level: **quick** (targeted lookups), **medium** (balanced), or **very thorough** (comprehensive analysis).

The **Plan** subagent prevents infinite nesting during plan mode (subagents cannot spawn other subagents) while still gathering necessary context.

The **General-purpose** subagent handles tasks requiring both exploration and modification, complex reasoning, or multiple dependent steps.

## Decision Guide

If you need... use this approach:

- **Quick codebase exploration** --> Built-in Explore subagent (Haiku, read-only)
- **Planning and research** --> Built-in Plan subagent (read-only, inherits model)
- **Complex multi-step task** --> Built-in General-purpose subagent (all tools)
- **Domain-specific analysis** --> Custom subagent with focused prompt and tools
- **Read-only code review** --> Custom subagent with Read/Grep/Glob tools only
- **Parallel independent research** --> Multiple subagents spawned concurrently
- **Parallel review with debate** --> Agent team with adversarial reviewers
- **Multi-file feature development** --> Agent team with file ownership per teammate
- **Competing hypothesis debugging** --> Agent team where teammates challenge each other
- **Cross-layer changes** --> Agent team with frontend/backend/test specialists

## Subagent Configuration Summary

### Supported Frontmatter Fields

| Field | Required | Description |
|:---|:---|:---|
| `name` | Yes | Unique identifier using lowercase letters and hyphens |
| `description` | Yes | When Claude should delegate to this subagent |
| `tools` | No | Tools the subagent can use (inherits all if omitted) |
| `disallowedTools` | No | Tools to deny, removed from inherited or specified list |
| `model` | No | `sonnet`, `opus`, `haiku`, or `inherit` (default: `inherit`) |
| `permissionMode` | No | `default`, `acceptEdits`, `dontAsk`, `delegate`, `bypassPermissions`, or `plan` |
| `maxTurns` | No | Maximum agentic turns before the subagent stops |
| `skills` | No | Skills to inject into subagent context at startup |
| `mcpServers` | No | MCP servers available to this subagent |
| `hooks` | No | Lifecycle hooks scoped to this subagent |
| `memory` | No | Persistent memory scope: `user`, `project`, or `local` |

### Subagent File Locations

| Location | Scope | Priority |
|:---|:---|:---|
| `--agents` CLI flag | Current session | 1 (highest) |
| `.claude/agents/` | Current project | 2 |
| `~/.claude/agents/` | All your projects | 3 |
| Plugin's `agents/` directory | Where plugin is enabled | 4 (lowest) |

When multiple subagents share the same name, the higher-priority location wins.

### Subagent Creation Workflow

1. **Define the subagent** as a Markdown file with YAML frontmatter (or use `/agents` command)
2. **Set the scope**: project, user, plugin, or CLI session
3. **Write a clear description** so Claude knows when to delegate
4. **Configure tools** -- restrict with `tools` (allowlist) or `disallowedTools` (denylist)
5. **Choose a model** -- `sonnet`, `opus`, `haiku`, or `inherit`
6. **Set permission mode** -- controls how permission prompts are handled
7. **Optionally add**: hooks, skills, memory, MCP servers, maxTurns

### Minimal Subagent Example

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

### CLI-Defined Subagent Example

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer. Use proactively after code changes.",
    "prompt": "You are a senior code reviewer. Focus on code quality, security, and best practices.",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  }
}'
```

## Working with Subagents

### Automatic Delegation

Claude automatically delegates based on the task description, the subagent's `description` field, and current context. To encourage proactive delegation, include phrases like "use proactively" in your subagent's description.

You can also request a specific subagent explicitly:

```
Use the test-runner subagent to fix failing tests
Have the code-reviewer subagent look at my recent changes
```

### Foreground vs Background Subagents

- **Foreground**: blocks the main conversation until complete; permission prompts pass through to you
- **Background**: runs concurrently; permissions are pre-approved before launch; auto-denies anything not pre-approved; MCP tools are not available

Press **Ctrl+B** to background a running task. Set `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` to disable.

### Common Subagent Patterns

**Isolate high-volume operations**: delegate test runs, documentation fetching, or log processing to keep verbose output out of your main context.

**Run parallel research**: spawn multiple subagents for independent investigations that Claude synthesizes afterward.

**Chain subagents**: use subagents in sequence where each completes its task and returns results for the next step.

### Resuming Subagents

Ask Claude to continue a previous subagent's work rather than starting fresh. Resumed subagents retain their full conversation history. Transcripts persist in `~/.claude/projects/{project}/{sessionId}/subagents/`.

## Agent Teams Overview

Agent teams are experimental. Enable with:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Architecture

Teams consist of:
- **Team lead**: main session that creates the team, spawns teammates, and coordinates
- **Teammates**: separate Claude Code instances working on assigned tasks
- **Task list**: shared work items with dependency tracking
- **Mailbox**: messaging system for inter-agent communication

### Display Modes

- **In-process**: all teammates in your main terminal (Shift+Up/Down to navigate, Ctrl+T for task list)
- **Split panes**: each teammate in its own pane (requires tmux or iTerm2)

Default is `"auto"` -- uses split panes inside tmux, in-process otherwise.

### Key Team Capabilities

- Natural language team creation and task description
- Task assignment by lead or self-claiming by teammates
- Plan approval workflows for risky tasks
- Delegate mode (Shift+Tab) to keep lead focused on coordination only
- Direct teammate messaging and broadcasting
- Quality gates via `TeammateIdle` and `TaskCompleted` hooks

### Team Use Cases

**Parallel code review**: spawn reviewers focused on security, performance, and test coverage independently:

```
Create an agent team to review PR #142. Spawn three reviewers:
- One focused on security implications
- One checking performance impact
- One validating test coverage
```

**Competing hypotheses**: make teammates adversarial to find the actual root cause:

```
Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
each other to try to disprove each other's theories, like a scientific debate.
```

### Team Best Practices

- Give teammates enough context in spawn prompts (conversation history does not carry over)
- Size tasks appropriately: 5-6 tasks per teammate keeps everyone productive
- Avoid file conflicts by assigning file ownership per teammate
- Monitor and steer progress rather than letting teams run unattended
- Start with research and review tasks before attempting parallel implementation

### Known Limitations

- No session resumption for in-process teammates (`/resume` and `/rewind` do not restore them)
- Task status can lag; teammates may fail to mark tasks as completed
- One team per session; clean up before starting a new one
- No nested teams; only the lead manages the team
- Permissions set at spawn; change individual teammate modes after spawning
- Split panes require tmux or iTerm2 (not supported in VS Code terminal, Windows Terminal, or Ghostty)

## Permission Modes Summary

| Mode | Behavior |
|:---|:---|
| `default` | Standard permission checking with prompts |
| `acceptEdits` | Auto-accept file edits |
| `dontAsk` | Auto-deny permission prompts (explicitly allowed tools still work) |
| `delegate` | Coordination-only mode for agent team leads |
| `bypassPermissions` | Skip all permission checks (use with extreme caution) |
| `plan` | Plan mode (read-only exploration) |

If the parent uses `bypassPermissions`, this takes precedence and cannot be overridden.

## Best Practices for Subagent Design

- **Design focused subagents**: each should excel at one specific task
- **Write detailed descriptions**: Claude uses the description to decide when to delegate
- **Limit tool access**: grant only necessary permissions for security and focus
- **Check into version control**: share project subagents (`.claude/agents/`) with your team
- **Use `user` memory scope** as the default for persistent learning across projects
- **Include memory instructions** in the subagent body to encourage proactive knowledge building

## Reference Files

For complete details, see the following reference documents:

- **[Subagent Configuration](references/subagent-configuration.md)**: Complete YAML frontmatter fields, tool restrictions, model selection, file locations, CLI flags, hooks, memory, skills, and example subagents
- **[Built-in Subagents](references/built-in-subagents.md)**: Explore, Plan, General-purpose, and other built-in subagents with their models, tools, and purposes
- **[Agent Teams](references/agent-teams.md)**: Enabling teams, display modes, task assignment, team coordination, use cases, best practices, troubleshooting, and limitations
- **[Permission Modes](references/permission-modes.md)**: All permission modes, what each allows, when to use each, security considerations, and common patterns
