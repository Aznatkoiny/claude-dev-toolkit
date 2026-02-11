# claude-dev-toolkit

A comprehensive Claude Code plugin for building Claude Code extensions. Contains everything you need to create skills, plugins, hooks, subagents, agent teams, MCP servers, and output styles.

## What's Included

### Skills (8)

| Skill | Description |
|-------|-------------|
| `skills-authoring` | Create SKILL.md files with frontmatter, triggers, references, and fork mode |
| `plugin-development` | Build plugins with plugin.json, directory layout, marketplace distribution |
| `hooks-automation` | Configure hooks across 14 lifecycle events with matchers and handlers |
| `subagents-and-teams` | Create custom subagents and coordinate agent teams |
| `mcp-integration` | Set up MCP servers with stdio/SSE transports and authentication |
| `claude-code-best-practices` | Context management, CLAUDE.md patterns, and parallel sessions |
| `extending-claude-code` | Extension comparison, output styles, and programmatic usage |
| `claude-code-troubleshooting` | Diagnose installation, runtime, and plugin issues |

### Agents (3)

| Agent | Description |
|-------|-------------|
| `plugin-builder` | Scaffolds and validates complete plugins end-to-end |
| `hook-builder` | Creates and debugs hook configurations |
| `agent-builder` | Creates subagent definitions and team configurations |

### Commands (3)

| Command | Description |
|---------|-------------|
| `/create-plugin` | Interactive wizard to scaffold a new plugin |
| `/create-hook` | Guided hook creation workflow |
| `/claude-code-guide` | General Q&A about Claude Code extensibility |

### Hooks

- **PostToolUse** prompt hook: Validates plugin component files for convention compliance after Write/Edit operations

### Output Style

- **Developer Documentation**: Optimized for extension development — code-first, decision tables, copy-paste ready examples

## Installation

### Via plugin directory (development)

```bash
claude --plugin-dir /path/to/claude-dev-toolkit/
```

### Via marketplace

```bash
/plugin marketplace add Aznatkoiny/claude-dev-toolkit
/plugin install claude-dev-toolkit@Aznatkoiny
```

## Usage

### Skills trigger automatically

Ask Claude questions like:
- "How do I create a skill?"
- "Help me set up a hook"
- "What MCP servers are available?"
- "How do I build a plugin?"

### Use commands directly

```
/claude-dev-toolkit:create-plugin my-awesome-plugin
/claude-dev-toolkit:create-hook
/claude-dev-toolkit:claude-code-guide how do hooks work?
```

### Agents activate on matching scenarios

The builder agents activate when Claude detects you need scaffolding help — for example, when you say "build me a plugin that..." or "create a hook for...".

## License

MIT
