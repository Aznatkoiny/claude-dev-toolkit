# Subagent Configuration Reference

Complete reference for creating and configuring custom subagents in Claude Code.

## Subagent File Format

Subagents are Markdown files with YAML frontmatter for configuration, followed by the system prompt body:

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

The frontmatter defines metadata and configuration. The body becomes the system prompt. Subagents receive only this system prompt (plus basic environment details like working directory), not the full Claude Code system prompt.

Subagents are loaded at session start. If you create a subagent by manually adding a file, restart your session or use `/agents` to load it immediately.

## Supported Frontmatter Fields

Only `name` and `description` are required. All other fields are optional.

| Field | Required | Description |
|:---|:---|:---|
| `name` | Yes | Unique identifier using lowercase letters and hyphens |
| `description` | Yes | When Claude should delegate to this subagent |
| `tools` | No | Tools the subagent can use. Inherits all tools if omitted |
| `disallowedTools` | No | Tools to deny, removed from inherited or specified list |
| `model` | No | Model to use: `sonnet`, `opus`, `haiku`, or `inherit`. Defaults to `inherit` |
| `permissionMode` | No | Permission mode: `default`, `acceptEdits`, `delegate`, `dontAsk`, `bypassPermissions`, or `plan` |
| `maxTurns` | No | Maximum number of agentic turns before the subagent stops |
| `skills` | No | Skills to load into the subagent's context at startup. Full skill content is injected, not just made available for invocation. Subagents don't inherit skills from the parent conversation |
| `mcpServers` | No | MCP servers available to this subagent. Each entry is either a server name referencing an already-configured server or an inline definition |
| `hooks` | No | Lifecycle hooks scoped to this subagent |
| `memory` | No | Persistent memory scope: `user`, `project`, or `local`. Enables cross-session learning |

## Where to Place Subagent Files

Store subagent files in different locations depending on scope. When multiple subagents share the same name, the higher-priority location wins.

| Location | Scope | Priority | How to create |
|:---|:---|:---|:---|
| `--agents` CLI flag | Current session | 1 (highest) | Pass JSON when launching Claude Code |
| `.claude/agents/` | Current project | 2 | Interactive or manual |
| `~/.claude/agents/` | All your projects | 3 | Interactive or manual |
| Plugin's `agents/` directory | Where plugin is enabled | 4 (lowest) | Installed with plugins |

**Project subagents** (`.claude/agents/`) are ideal for subagents specific to a codebase. Check them into version control so your team can use and improve them collaboratively.

**User subagents** (`~/.claude/agents/`) are personal subagents available in all your projects.

**Plugin subagents** come from plugins you have installed. They appear in `/agents` alongside your custom subagents.

## CLI-Defined Subagents

Pass subagents as JSON when launching Claude Code. They exist only for that session:

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

The `--agents` flag accepts JSON with the same frontmatter fields as file-based subagents: `description`, `prompt`, `tools`, `disallowedTools`, `model`, `permissionMode`, `mcpServers`, `hooks`, `maxTurns`, `skills`, and `memory`. Use `prompt` for the system prompt (equivalent to the markdown body in file-based subagents).

## Model Selection

The `model` field controls which AI model the subagent uses:

- **`sonnet`**: Balanced capability and speed
- **`opus`**: Most capable model
- **`haiku`**: Fast, low-latency, lower cost
- **`inherit`**: Use the same model as the main conversation (default if omitted)

## Tool Restrictions

By default, subagents inherit all tools from the main conversation, including MCP tools.

### Allowlist with `tools`

Specify exactly which tools the subagent can use:

```yaml
---
name: safe-researcher
description: Research agent with restricted capabilities
tools: Read, Grep, Glob, Bash
---
```

### Denylist with `disallowedTools`

Remove specific tools from the inherited set:

```yaml
---
name: safe-researcher
description: Research agent with restricted capabilities
tools: Read, Grep, Glob, Bash
disallowedTools: Write, Edit
---
```

### Restrict Spawnable Subagents with `Task(agent_type)`

When an agent runs as the main thread with `claude --agent`, use `Task(agent_type)` syntax to restrict which subagents it can spawn:

```yaml
---
name: coordinator
description: Coordinates work across specialized agents
tools: Task(worker, researcher), Read, Bash
---
```

This is an allowlist: only `worker` and `researcher` subagents can be spawned. To allow spawning any subagent without restrictions, use `Task` without parentheses:

```yaml
tools: Task, Read, Bash
```

If `Task` is omitted from the `tools` list entirely, the agent cannot spawn any subagents. This restriction only applies to agents running as the main thread with `claude --agent`. Subagents cannot spawn other subagents, so `Task(agent_type)` has no effect in subagent definitions.

### Disable Specific Subagents via Permissions

Prevent Claude from using specific subagents by adding them to the `deny` array in settings:

```json
{
  "permissions": {
    "deny": ["Task(Explore)", "Task(my-custom-agent)"]
  }
}
```

This works for both built-in and custom subagents. Also available via CLI:

```bash
claude --disallowedTools "Task(Explore)"
```

## Preloading Skills

Use the `skills` field to inject skill content into a subagent's context at startup:

```yaml
---
name: api-developer
description: Implement API endpoints following team conventions
skills:
  - api-conventions
  - error-handling-patterns
---

Implement API endpoints. Follow the conventions and patterns from the preloaded skills.
```

The full content of each skill is injected into the subagent's context, not just made available for invocation. Subagents don't inherit skills from the parent conversation; you must list them explicitly.

## Persistent Memory

The `memory` field gives the subagent a persistent directory that survives across conversations:

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
memory: user
---

You are a code reviewer. As you review code, update your agent memory with
patterns, conventions, and recurring issues you discover.
```

### Memory Scopes

| Scope | Location | Use when |
|:---|:---|:---|
| `user` | `~/.claude/agent-memory/<name-of-agent>/` | The subagent should remember learnings across all projects |
| `project` | `.claude/agent-memory/<name-of-agent>/` | The subagent's knowledge is project-specific and shareable via version control |
| `local` | `.claude/agent-memory-local/<name-of-agent>/` | The subagent's knowledge is project-specific but should not be checked into version control |

When memory is enabled:
- The subagent's system prompt includes instructions for reading and writing to the memory directory
- The subagent's system prompt includes the first 200 lines of `MEMORY.md` in the memory directory
- Read, Write, and Edit tools are automatically enabled so the subagent can manage its memory files

**Tips:**
- `user` is the recommended default scope
- Ask the subagent to consult its memory before starting work
- Ask the subagent to update its memory after completing a task
- Include memory instructions directly in the subagent's markdown body

## Hooks in Subagent Frontmatter

Define hooks that run only while a specific subagent is active:

```yaml
---
name: code-reviewer
description: Review code changes with automatic linting
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh $TOOL_INPUT"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
---
```

All hook events are supported. The most common events for subagents:

| Event | Matcher input | When it fires |
|:---|:---|:---|
| `PreToolUse` | Tool name | Before the subagent uses a tool |
| `PostToolUse` | Tool name | After the subagent uses a tool |
| `Stop` | (none) | When the subagent finishes (converted to `SubagentStop` at runtime) |

### Project-Level Hooks for Subagent Events

Configure hooks in `settings.json` that respond to subagent lifecycle events:

| Event | Matcher input | When it fires |
|:---|:---|:---|
| `SubagentStart` | Agent type name | When a subagent begins execution |
| `SubagentStop` | Agent type name | When a subagent completes |

Example:

```json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "db-agent",
        "hooks": [
          { "type": "command", "command": "./scripts/setup-db-connection.sh" }
        ]
      }
    ],
    "SubagentStop": [
      {
        "hooks": [
          { "type": "command", "command": "./scripts/cleanup-db-connection.sh" }
        ]
      }
    ]
  }
}
```

## The /agents Command

The `/agents` command provides an interactive interface for managing subagents:

- View all available subagents (built-in, user, project, and plugin)
- Create new subagents with guided setup or Claude generation
- Edit existing subagent configuration and tool access
- Delete custom subagents
- See which subagents are active when duplicates exist

## Automatic Delegation

Claude automatically delegates tasks based on:
- The task description in your request
- The `description` field in subagent configurations
- Current context

To encourage proactive delegation, include phrases like "use proactively" in the description field.

You can also request a specific subagent explicitly:

```
Use the test-runner subagent to fix failing tests
Have the code-reviewer subagent look at my recent changes
```

## Foreground vs Background Subagents

- **Foreground subagents** block the main conversation until complete. Permission prompts and clarifying questions are passed through to you.
- **Background subagents** run concurrently while you continue working. Before launching, Claude Code prompts for any tool permissions the subagent will need. Once running, the subagent auto-denies anything not pre-approved. MCP tools are not available in background subagents.

Press **Ctrl+B** to background a running task. Set `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` to disable background functionality.

## Resuming Subagents

Each subagent invocation creates a new instance with fresh context. To continue an existing subagent's work, ask Claude to resume it:

```
Use the code-reviewer subagent to review the authentication module
[Agent completes]

Continue that code review and now analyze the authorization logic
[Claude resumes the subagent with full context from previous conversation]
```

Subagent transcripts persist in `~/.claude/projects/{project}/{sessionId}/subagents/` as `agent-{agentId}.jsonl`.

## Auto-Compaction

Subagents support automatic compaction using the same logic as the main conversation. By default triggers at approximately 95% capacity. Set `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` to a lower percentage (e.g., `50`) to trigger earlier.

## Example Subagents

### Code Reviewer (Read-Only)

```markdown
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer ensuring high standards of code quality and security.

When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code is clear and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented
- Good test coverage
- Performance considerations addressed

Provide feedback organized by priority:
- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)

Include specific examples of how to fix issues.
```

### Debugger

```markdown
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior. Use proactively when encountering any issues.
tools: Read, Edit, Bash, Grep, Glob
---

You are an expert debugger specializing in root cause analysis.

When invoked:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix
5. Verify solution works

Debugging process:
- Analyze error messages and logs
- Check recent code changes
- Form and test hypotheses
- Add strategic debug logging
- Inspect variable states

For each issue, provide:
- Root cause explanation
- Evidence supporting the diagnosis
- Specific code fix
- Testing approach
- Prevention recommendations

Focus on fixing the underlying issue, not the symptoms.
```

### Data Scientist

```markdown
---
name: data-scientist
description: Data analysis expert for SQL queries, BigQuery operations, and data insights. Use proactively for data analysis tasks and queries.
tools: Bash, Read, Write
model: sonnet
---

You are a data scientist specializing in SQL and BigQuery analysis.

When invoked:
1. Understand the data analysis requirement
2. Write efficient SQL queries
3. Use BigQuery command line tools (bq) when appropriate
4. Analyze and summarize results
5. Present findings clearly

Key practices:
- Write optimized SQL queries with proper filters
- Use appropriate aggregations and joins
- Include comments explaining complex logic
- Format results for readability
- Provide data-driven recommendations

For each analysis:
- Explain the query approach
- Document any assumptions
- Highlight key findings
- Suggest next steps based on data

Always ensure queries are efficient and cost-effective.
```

### Database Query Validator (with Hook)

```markdown
---
name: db-reader
description: Execute read-only database queries. Use when analyzing data or generating reports.
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---

You are a database analyst with read-only access. Execute SELECT queries to answer questions about the data.

When asked to analyze data:
1. Identify which tables contain the relevant data
2. Write efficient SELECT queries with appropriate filters
3. Present results clearly with context

You cannot modify data. If asked to INSERT, UPDATE, DELETE, or modify schema, explain that you only have read access.
```

Validation script for the db-reader:

```bash
#!/bin/bash
# Blocks SQL write operations, allows SELECT queries

# Read JSON input from stdin
INPUT=$(cat)

# Extract the command field from tool_input using jq
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if [ -z "$COMMAND" ]; then
  exit 0
fi

# Block write operations (case-insensitive)
if echo "$COMMAND" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE|REPLACE|MERGE)\b' > /dev/null; then
  echo "Blocked: Write operations not allowed. Use SELECT queries only." >&2
  exit 2
fi

exit 0
```
