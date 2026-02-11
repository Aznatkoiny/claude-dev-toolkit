# MCP Servers in Plugins Reference

Plugins can bundle MCP servers, automatically providing tools and integrations when the plugin
is enabled. Plugin MCP servers work identically to user-configured servers.

## How Plugin MCP Servers Work

- Plugins define MCP servers in `.mcp.json` at the plugin root or inline in `plugin.json`
- When a plugin is enabled, its MCP servers start automatically
- Plugin MCP tools appear alongside manually configured MCP tools
- Plugin servers are managed through plugin installation (not `/mcp` commands)

## Configuration Methods

### Method 1: .mcp.json at Plugin Root

Create a `.mcp.json` file at the root of your plugin directory:

```json
{
  "database-tools": {
    "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
    "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
    "env": {
      "DB_URL": "${DB_URL}"
    }
  }
}
```

### Method 2: Inline in plugin.json

Define MCP servers directly in the plugin manifest using the `mcpServers` field:

```json
{
  "name": "my-plugin",
  "mcpServers": {
    "plugin-api": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/api-server",
      "args": ["--port", "8080"]
    }
  }
}
```

## The ${CLAUDE_PLUGIN_ROOT} Variable

Use `${CLAUDE_PLUGIN_ROOT}` in MCP configurations to reference files relative to the plugin
root directory. This variable resolves to the absolute path of the plugin directory at runtime,
ensuring paths work regardless of where the plugin is installed.

**Example usage:**

```json
{
  "my-server": {
    "command": "${CLAUDE_PLUGIN_ROOT}/bin/my-mcp-server",
    "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config/settings.json"],
    "env": {
      "DATA_DIR": "${CLAUDE_PLUGIN_ROOT}/data"
    }
  }
}
```

## Plugin MCP Features

### Automatic Lifecycle

Servers start when the plugin is enabled. You must restart Claude Code to apply MCP server
changes (enabling or disabling plugins).

### Environment Variables

- Use `${CLAUDE_PLUGIN_ROOT}` for plugin-relative paths
- Access the same user environment variables as manually configured servers
- Reference user-specific values like `${DB_URL}` that plugin users set in their environment

### Transport Types

Plugin MCP servers support all transport types:

| Transport | Example |
|:----------|:--------|
| **stdio** | Local process bundled with the plugin |
| **HTTP** | Remote service URL |
| **SSE** | Legacy remote service URL |

Transport support may vary by server implementation.

## Complete Plugin Example

A plugin that bundles a custom database MCP server:

```
my-db-plugin/
├── .claude-plugin/
│   └── plugin.json
├── .mcp.json
├── servers/
│   └── db-server
├── config/
│   └── defaults.json
└── skills/
    └── db-queries/
        └── SKILL.md
```

**`.claude-plugin/plugin.json`:**

```json
{
  "name": "my-db-plugin",
  "description": "Database tools with MCP integration",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

**`.mcp.json`:**

```json
{
  "db-tools": {
    "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
    "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config/defaults.json"],
    "env": {
      "DB_CONNECTION_STRING": "${DB_CONNECTION_STRING}"
    }
  }
}
```

## Viewing Plugin MCP Servers

Within Claude Code, use the `/mcp` command to see all MCP servers including those from plugins:

```
> /mcp
```

Plugin servers appear in the list with indicators showing they come from plugins.

## Benefits of Plugin MCP Servers

| Benefit | Description |
|:--------|:------------|
| **Bundled distribution** | Tools and servers packaged together in a single installable unit |
| **Automatic setup** | No manual MCP configuration needed by the user |
| **Team consistency** | Everyone gets the same tools when the plugin is installed |
| **Version control** | Server configurations are versioned with the plugin |

## Important Notes

- Place `.mcp.json` at the plugin root, NOT inside the `.claude-plugin/` directory
- The `.claude-plugin/` directory should contain only `plugin.json`
- All component directories (skills, commands, hooks, etc.) go at the plugin root level
- Plugin MCP servers are managed through plugin installation, not through `claude mcp add`
  or `/mcp` commands
- If you make changes to MCP server configurations in a plugin, users must restart Claude Code
  to pick up the changes
