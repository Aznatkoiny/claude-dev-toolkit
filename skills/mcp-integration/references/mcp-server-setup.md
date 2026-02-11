# MCP Server Setup Reference

## Configuration Format

MCP servers are configured either via the `claude mcp` CLI or through `.mcp.json` files. The
underlying configuration structure is the same.

### .mcp.json File Format

The `.mcp.json` file at your project root stores project-scoped MCP server configurations. It
is designed to be checked into version control so all team members share the same MCP tools.

```json
{
  "mcpServers": {
    "server-name": {
      "command": "/path/to/server",
      "args": [],
      "env": {}
    }
  }
}
```

For HTTP servers:

```json
{
  "mcpServers": {
    "server-name": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```

### Environment Variable Expansion

`.mcp.json` files support environment variable expansion, allowing teams to share configurations
while keeping machine-specific paths and secrets flexible.

**Supported syntax:**

- `${VAR}` - Expands to the value of environment variable `VAR`
- `${VAR:-default}` - Expands to `VAR` if set, otherwise uses `default`

**Expansion locations:**

| Field | Description |
|:------|:------------|
| `command` | The server executable path |
| `args` | Command-line arguments |
| `env` | Environment variables passed to the server |
| `url` | For HTTP server types |
| `headers` | For HTTP server authentication |

**Example:**

```json
{
  "mcpServers": {
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```

If a required environment variable is not set and has no default value, Claude Code will fail
to parse the config.

## Adding Servers via CLI

### Add a stdio server

```bash
# Basic syntax
claude mcp add [options] <name> -- <command> [args...]

# Example: Airtable server
claude mcp add --transport stdio --env AIRTABLE_API_KEY=YOUR_KEY airtable \
  -- npx -y airtable-mcp-server

# Example: PostgreSQL
claude mcp add --transport stdio db -- npx -y @bytebase/dbhub \
  --dsn "postgresql://readonly:pass@prod.db.com:5432/analytics"
```

**Important: Option ordering.** All options (`--transport`, `--env`, `--scope`, `--header`)
must come before the server name. The `--` separates the server name from the command and
arguments passed to the MCP server.

Examples of correct ordering:

- `claude mcp add --transport stdio myserver -- npx server` runs `npx server`
- `claude mcp add --transport stdio --env KEY=value myserver -- python server.py --port 8080`
  runs `python server.py --port 8080` with `KEY=value` in environment

This prevents conflicts between Claude's flags and the server's flags.

### Add an HTTP server

```bash
# Basic syntax
claude mcp add --transport http <name> <url>

# Example: Notion
claude mcp add --transport http notion https://mcp.notion.com/mcp

# Example with Bearer token
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"
```

### Add an SSE server (deprecated)

```bash
# Basic syntax
claude mcp add --transport sse <name> <url>

# Example: Asana
claude mcp add --transport sse asana https://mcp.asana.com/sse

# Example with auth header
claude mcp add --transport sse private-api https://api.company.com/sse \
  --header "X-API-Key: your-key-here"
```

The SSE transport is deprecated. Use HTTP servers where available.

### Add from JSON configuration

```bash
# Basic syntax
claude mcp add-json <name> '<json>'

# HTTP server with headers
claude mcp add-json weather-api '{"type":"http","url":"https://api.weather.com/mcp","headers":{"Authorization":"Bearer token"}}'

# Stdio server
claude mcp add-json local-weather '{"type":"stdio","command":"/path/to/weather-cli","args":["--api-key","abc123"],"env":{"CACHE_DIR":"/tmp"}}'

# HTTP server with OAuth credentials
claude mcp add-json my-server '{"type":"http","url":"https://mcp.example.com/mcp","oauth":{"clientId":"your-client-id","callbackPort":8080}}' --client-secret
```

Make sure the JSON is properly escaped in your shell and conforms to the MCP server
configuration schema.

### Import from Claude Desktop

```bash
# Import servers (macOS and WSL only)
claude mcp add-from-claude-desktop

# With user scope
claude mcp add-from-claude-desktop --scope user
```

After running the command, an interactive dialog lets you select which servers to import.
If servers with the same names already exist, they get a numerical suffix (e.g., `server_1`).

## Managing Servers

```bash
# List all configured servers
claude mcp list

# Get details for a specific server
claude mcp get <name>

# Remove a server
claude mcp remove <name>

# Check server status within Claude Code
> /mcp
```

## Scope Hierarchy

MCP servers can be configured at three scope levels:

### Local Scope (default)

- Stored in `~/.claude.json` under your project's path
- Private to you, only accessible in the current project
- Ideal for personal servers, experimental configs, or sensitive credentials

```bash
# Add a local-scoped server (default)
claude mcp add --transport http stripe https://mcp.stripe.com

# Explicitly specify local scope
claude mcp add --transport http stripe --scope local https://mcp.stripe.com
```

**Note:** The term "local scope" for MCP servers differs from general local settings. MCP
local-scoped servers are stored in `~/.claude.json` (your home directory), while general
local settings use `.claude/settings.local.json` (in the project directory).

### Project Scope

- Stored in `.mcp.json` at the project root
- Designed to be checked into version control
- All team members share the same MCP tools

```bash
claude mcp add --transport http paypal --scope project https://mcp.paypal.com/mcp
```

For security, Claude Code prompts for approval before using project-scoped servers. Reset
approval choices with:

```bash
claude mcp reset-project-choices
```

### User Scope

- Stored in `~/.claude.json`
- Available across all projects on your machine
- Private to your user account

```bash
claude mcp add --transport http hubspot --scope user https://mcp.hubspot.com/anthropic
```

### Precedence

When servers with the same name exist at multiple scopes: **Local > Project > User**.
Local-scoped servers always take priority.

### Choosing the Right Scope

| Scope | Best For |
|:------|:---------|
| **Local** | Personal servers, experimental configs, sensitive credentials for one project |
| **Project** | Team-shared servers, project-specific tools, collaboration requirements |
| **User** | Personal utilities across multiple projects, frequently used services |

### Where MCP Servers Are Stored

| Scope | Location |
|:------|:---------|
| User and Local | `~/.claude.json` (in `mcpServers` field or under project paths) |
| Project | `.mcp.json` in your project root (checked into source control) |
| Managed | `managed-mcp.json` in system directories |

## Dynamic Tool Updates

Claude Code supports MCP `list_changed` notifications, allowing MCP servers to dynamically
update their available tools, prompts, and resources without requiring you to disconnect and
reconnect. When an MCP server sends a `list_changed` notification, Claude Code automatically
refreshes the available capabilities from that server.

## Startup Timeout

Configure MCP server startup timeout using the `MCP_TIMEOUT` environment variable:

```bash
MCP_TIMEOUT=10000 claude    # 10-second timeout
```

## Output Limits

- Warning threshold: 10,000 tokens per tool output
- Default maximum: 25,000 tokens
- Configure with `MAX_MCP_OUTPUT_TOKENS`:

```bash
export MAX_MCP_OUTPUT_TOKENS=50000
claude
```

Useful for MCP servers that query large datasets, generate detailed reports, or process
extensive log files.

## Windows Notes

On native Windows (not WSL), local MCP servers that use `npx` require the `cmd /c` wrapper:

```bash
claude mcp add --transport stdio my-server -- cmd /c npx -y @some/package
```

Without the `cmd /c` wrapper, you will encounter "Connection closed" errors because Windows
cannot directly execute `npx`.

## Practical Examples

### Monitor errors with Sentry

```bash
# Add the Sentry MCP server
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp

# Authenticate with your Sentry account
> /mcp

# Debug production issues
> "What are the most common errors in the last 24 hours?"
> "Show me the stack trace for error ID abc123"
> "Which deployment introduced these new errors?"
```

### Connect to GitHub for code reviews

```bash
# Add the GitHub MCP server
claude mcp add --transport http github https://api.githubcopilot.com/mcp/

# Authenticate if needed
> /mcp
# Select "Authenticate" for GitHub

# Work with GitHub
> "Review PR #456 and suggest improvements"
> "Create a new issue for the bug we just found"
> "Show me all open PRs assigned to me"
```

### Query your PostgreSQL database

```bash
# Add the database server with your connection string
claude mcp add --transport stdio db -- npx -y @bytebase/dbhub \
  --dsn "postgresql://readonly:pass@prod.db.com:5432/analytics"

# Query your database naturally
> "What's our total revenue this month?"
> "Show me the schema for the orders table"
> "Find customers who haven't made a purchase in 90 days"
```

## MCP Resources

MCP servers can expose resources that you reference using @ mentions:

```
> Can you analyze @github:issue://123 and suggest a fix?
> Please review the API documentation at @docs:file://api/authentication
> Compare @postgres:schema://users with @docs:file://database/user-model
```

- Resources are automatically fetched and included as attachments
- Resource paths are fuzzy-searchable in the @ mention autocomplete
- Claude Code automatically provides tools to list and read MCP resources
- Resources can contain any type of content (text, JSON, structured data, etc.)

## MCP Prompts as Commands

MCP servers can expose prompts that become available as slash commands:

```
> /mcp__github__list_prs
> /mcp__github__pr_review 456
> /mcp__jira__create_issue "Bug in login flow" high
```

- Type `/` to see all available commands including MCP prompts
- MCP prompts appear with the format `/mcp__servername__promptname`
- Arguments are parsed based on the prompt's defined parameters
- Prompt results are injected directly into the conversation
- Server and prompt names are normalized (spaces become underscores)

## MCP Tool Search

When many MCP servers are configured, tool definitions can consume significant context.
Claude Code automatically enables Tool Search when MCP tool descriptions exceed 10% of
the context window.

When triggered:

1. MCP tools are deferred rather than loaded into context upfront
2. Claude uses a search tool to discover relevant MCP tools when needed
3. Only the tools Claude actually needs are loaded into context
4. MCP tools continue to work exactly as before from your perspective

**Requires:** Sonnet 4 and later, or Opus 4 and later. Haiku models do not support tool search.

**Configuration:**

| Value | Behavior |
|:------|:---------|
| `auto` | Activates when MCP tools exceed 10% of context (default) |
| `auto:<N>` | Activates at custom threshold (e.g., `auto:5` for 5%) |
| `true` | Always enabled |
| `false` | Disabled, all MCP tools loaded upfront |

```bash
ENABLE_TOOL_SEARCH=auto:5 claude   # 5% threshold
ENABLE_TOOL_SEARCH=false claude    # Disable tool search
```

Or set in your `settings.json` `env` field. You can also disable via `disallowedTools`:

```json
{
  "permissions": {
    "deny": ["MCPSearch"]
  }
}
```

### For MCP Server Authors

With Tool Search enabled, server instructions help Claude understand when to search for
your tools. Add clear, descriptive server instructions that explain:

- What category of tasks your tools handle
- When Claude should search for your tools
- Key capabilities your server provides

## Using Claude Code as an MCP Server

Claude Code can itself serve as an MCP server:

```bash
claude mcp serve
```

Add to Claude Desktop `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "claude-code": {
      "type": "stdio",
      "command": "claude",
      "args": ["mcp", "serve"],
      "env": {}
    }
  }
}
```

If `claude` is not in your PATH, find the full path with `which claude` and use that.

The server provides access to Claude's tools like View, Edit, LS, etc. Note that this MCP
server only exposes Claude Code's tools to your MCP client -- your own client is responsible
for implementing user confirmation for individual tool calls.
