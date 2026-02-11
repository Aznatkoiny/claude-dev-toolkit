# Plugin Components Reference

Plugins can include multiple component types. All component directories and files live at the
plugin root level, never inside `.claude-plugin/`.

## Directory Structure

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest (ONLY file in .claude-plugin/)
├── commands/                 # Slash commands (Markdown files)
│   ├── review.md
│   └── deploy.md
├── skills/                   # Agent Skills (model-invoked)
│   └── code-review/
│       └── SKILL.md
├── agents/                   # Custom agent definitions
│   └── reviewer.md
├── hooks/                    # Event handlers
│   └── hooks.json
├── .mcp.json                 # MCP server configurations
├── .lsp.json                 # LSP server configurations
└── README.md                 # Documentation (optional)
```

**Critical rule**: The `.claude-plugin/` directory contains ONLY `plugin.json`. Do not put
`commands/`, `agents/`, `skills/`, or `hooks/` inside `.claude-plugin/`.

## Component Types

### Commands (`commands/`)

Commands are slash commands defined as Markdown files. Each `.md` file in the `commands/`
directory becomes a user-invokable command, namespaced by the plugin name.

**File naming**: The filename (without `.md`) becomes the command name. A file at
`commands/hello.md` in a plugin named `my-plugin` creates the command `/my-plugin:hello`.

**Example command file** (`commands/hello.md`):

```markdown
---
description: Greet the user with a personalized message
---

# Hello Command

Greet the user named "$ARGUMENTS" warmly and ask how you can help them today.
Make the greeting personal and encouraging.
```

**Key features**:
- `$ARGUMENTS` placeholder captures any text the user provides after the command name
- Frontmatter `description` is shown in help listings
- `disable-model-invocation: true` in frontmatter prevents automatic invocation (user must
  explicitly call the command)

### Skills (`skills/`)

Skills are model-invoked capabilities. Unlike commands, Claude automatically uses Skills based
on task context without the user explicitly calling them.

**Structure**: Each skill is a folder containing a `SKILL.md` file:

```
skills/
└── code-review/
    └── SKILL.md
```

**Example SKILL.md**:

```yaml
---
name: code-review
description: >
  Reviews code for best practices and potential issues. Use when reviewing code,
  checking PRs, or analyzing code quality.
---

When reviewing code, check for:
1. Code organization and structure
2. Error handling
3. Security concerns
4. Test coverage
```

**Key features**:
- Frontmatter requires `name` and `description` fields
- The `description` tells Claude when to automatically invoke the skill
- Skills are loaded when Claude Code starts; restart to pick up changes

### Agents (`agents/`)

Agents are custom agent definitions that appear in the `/agents` listing. Each agent is
defined as a Markdown file in the `agents/` directory.

**File naming**: The filename (without `.md`) becomes the agent name, namespaced by the
plugin name.

### Hooks (`hooks/`)

Hooks are event handlers that run shell commands in response to Claude Code events. Plugin
hooks are defined in `hooks/hooks.json`.

**Example hooks.json**:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npm run lint:fix"
          }
        ]
      }
    ]
  }
}
```

The format is the same as the `hooks` object in `.claude/settings.json` or `settings.local.json`.
The hook command receives hook input as JSON on stdin.

### MCP Servers (`.mcp.json`)

MCP (Model Context Protocol) server configurations let your plugin connect Claude to external
services. Place an `.mcp.json` file at the plugin root.

**Path references**: Use `${CLAUDE_PLUGIN_ROOT}` to reference files relative to the plugin
directory:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/servers/my-server.js"]
    }
  }
}
```

The `${CLAUDE_PLUGIN_ROOT}` variable resolves to the absolute path of the plugin directory at
runtime, ensuring paths work regardless of where the plugin is installed.

### LSP Servers (`.lsp.json`)

LSP (Language Server Protocol) configurations give Claude real-time code intelligence for
specific programming languages. Place an `.lsp.json` file at the plugin root.

**Example .lsp.json**:

```json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

Users installing your LSP plugin must have the language server binary installed on their machine.

**What LSP provides to Claude**:
- **Automatic diagnostics**: After every file edit, the language server reports errors, warnings,
  type errors, missing imports, and syntax issues automatically
- **Code navigation**: Jump to definitions, find references, get type info on hover, list symbols,
  find implementations, and trace call hierarchies

**Note**: For common languages (TypeScript, Python, Rust, Go, etc.), pre-built LSP plugins are
available from the official Anthropic marketplace. Create custom LSP plugins only for languages
not already covered.

### Output Styles

Output style plugins customize how Claude responds. These are typically included as part of a
plugin's skill or command definitions.

Examples from the official marketplace:
- **explanatory-output-style**: Educational insights about implementation choices
- **learning-output-style**: Interactive learning mode for skill building

## Path Conventions

### `${CLAUDE_PLUGIN_ROOT}`

Use this variable in hook commands and MCP configurations to build paths relative to your
plugin root:

```json
{
  "type": "command",
  "command": "${CLAUDE_PLUGIN_ROOT}/scripts/lint.sh"
}
```

This ensures your plugin works correctly regardless of where it is installed on the user's
system (plugins are copied to a cache directory during installation).

### Relative Paths

All component directories are relative to the plugin root:
- `commands/` resolves to `<plugin-root>/commands/`
- `skills/` resolves to `<plugin-root>/skills/`
- `agents/` resolves to `<plugin-root>/agents/`
- `hooks/` resolves to `<plugin-root>/hooks/`

### Important Path Caveat

Plugins are copied to a cache directory during installation. Paths referencing files outside
the plugin directory will not work after installation. Keep all referenced files within the
plugin directory and use `${CLAUDE_PLUGIN_ROOT}` for absolute path resolution.

## Organizing Complex Plugins

For plugins with many components, organize by functionality:

```
my-complex-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── review.md
│   ├── deploy.md
│   └── test.md
├── skills/
│   ├── code-review/
│   │   └── SKILL.md
│   └── testing/
│       └── SKILL.md
├── agents/
│   ├── reviewer.md
│   └── deployer.md
├── hooks/
│   └── hooks.json
├── servers/
│   └── custom-mcp-server.js
├── .mcp.json
└── README.md
```
