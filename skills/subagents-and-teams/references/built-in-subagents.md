# Built-in Subagents Reference

Claude Code includes built-in subagents that Claude automatically uses when appropriate. Each inherits the parent conversation's permissions with additional tool restrictions.

## Built-in Subagents Table

| Subagent | Model | Tools | Purpose | When Claude Uses It |
|:---|:---|:---|:---|:---|
| **Explore** | Haiku (fast, low-latency) | Read-only tools (denied Write and Edit) | File discovery, code search, codebase exploration | When it needs to search or understand a codebase without making changes |
| **Plan** | Inherits from main conversation | Read-only tools (denied Write and Edit) | Codebase research for planning | During plan mode when Claude needs to understand the codebase |
| **General-purpose** | Inherits from main conversation | All tools | Complex research, multi-step operations, code modifications | When the task requires both exploration and modification, complex reasoning, or multiple dependent steps |
| **Bash** | Inherits from main conversation | (terminal context) | Running terminal commands in a separate context | When terminal commands need isolated context |
| **statusline-setup** | Sonnet | (specialized) | Configuring status line | When you run `/statusline` to configure your status line |
| **Claude Code Guide** | Haiku | (specialized) | Answering questions about Claude Code features | When you ask questions about Claude Code features |

## Explore Subagent

A fast, read-only agent optimized for searching and analyzing codebases.

- **Model**: Haiku (fast, low-latency)
- **Tools**: Read-only tools (denied access to Write and Edit tools)
- **Purpose**: File discovery, code search, codebase exploration

Claude delegates to Explore when it needs to search or understand a codebase without making changes. This keeps exploration results out of your main conversation context.

When invoking Explore, Claude specifies a thoroughness level:
- **quick**: targeted lookups
- **medium**: balanced exploration
- **very thorough**: comprehensive analysis

## Plan Subagent

A research agent used during plan mode to gather context before presenting a plan.

- **Model**: Inherits from main conversation
- **Tools**: Read-only tools (denied access to Write and Edit tools)
- **Purpose**: Codebase research for planning

When you are in plan mode and Claude needs to understand your codebase, it delegates research to the Plan subagent. This prevents infinite nesting (subagents cannot spawn other subagents) while still gathering necessary context.

## General-purpose Subagent

A capable agent for complex, multi-step tasks that require both exploration and action.

- **Model**: Inherits from main conversation
- **Tools**: All tools
- **Purpose**: Complex research, multi-step operations, code modifications

Claude delegates to general-purpose when the task requires:
- Both exploration and modification
- Complex reasoning to interpret results
- Multiple dependent steps

## Other Helper Agents

Claude Code includes additional helper agents for specific tasks. These are typically invoked automatically:

- **Bash**: Inherits model. Used for running terminal commands in a separate context.
- **statusline-setup**: Uses Sonnet. Used when you run `/statusline` to configure your status line.
- **Claude Code Guide**: Uses Haiku. Used when you ask questions about Claude Code features.

## Disabling Built-in Subagents

You can prevent Claude from using specific built-in subagents by adding them to the `deny` array in your settings:

```json
{
  "permissions": {
    "deny": ["Task(Explore)", "Task(Plan)"]
  }
}
```

Or via CLI:

```bash
claude --disallowedTools "Task(Explore)"
```

## Custom Subagents Beyond Built-ins

Beyond the built-in subagents, you can create your own with custom prompts, tool restrictions, permission modes, hooks, and skills. Custom subagents are defined as Markdown files with YAML frontmatter or passed via the `--agents` CLI flag. See the subagent configuration reference for full details.
