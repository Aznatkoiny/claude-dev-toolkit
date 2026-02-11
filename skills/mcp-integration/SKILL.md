---
name: mcp-integration
description: >
  Comprehensive guide for setting up and managing MCP (Model Context Protocol) servers in Claude Code.
  Use when configuring MCP servers, connecting external services via MCP, setting up .mcp.json files,
  choosing between stdio/HTTP/SSE transports, authenticating with remote MCP servers, bundling MCP
  servers in plugins, managing MCP scopes (local/project/user), using MCP resources and prompts,
  configuring managed MCP for organizations, or troubleshooting MCP connections. Trigger phrases:
  "MCP server", "Model Context Protocol", ".mcp.json", "connect external service", "MCP in plugin",
  "stdio transport", "SSE transport", "HTTP transport", "remote MCP", "MCP authentication",
  "MCP tools", "claude mcp add", "MCP OAuth", "MCP scope", "managed MCP", "MCP tool search",
  "MCP resources", "MCP prompts".
---

# MCP Integration Guide

The Model Context Protocol (MCP) is an open standard for connecting AI tools to external data sources,
APIs, and services. Claude Code supports MCP natively, allowing you to connect to hundreds of external
tools through a unified protocol. MCP servers give Claude Code access to your tools, databases, and
APIs -- from GitHub and Sentry to PostgreSQL and custom internal services.

MCP servers come in two main flavors: **local stdio servers** that run as processes on your machine
(ideal for direct system access and custom scripts), and **remote servers** that connect over HTTP
or SSE transport (ideal for cloud-based services). Claude Code manages the lifecycle of these servers,
handles authentication, and makes their tools available seamlessly during your conversations.

With MCP configured, you can ask Claude Code to implement features from issue trackers, analyze
monitoring data, query databases, integrate designs, and automate multi-step workflows -- all by
connecting the right MCP servers for your stack.

## When to Use This Skill

- Adding an MCP server to Claude Code (any transport type)
- Configuring `.mcp.json` for project-shared MCP servers
- Choosing the right transport: stdio vs HTTP vs SSE
- Authenticating with remote MCP servers via OAuth or API keys
- Setting up environment variables for MCP servers
- Understanding MCP scopes (local, project, user)
- Bundling MCP servers inside a plugin
- Managing MCP servers (list, get, remove)
- Using MCP resources via @ mentions
- Using MCP prompts as slash commands
- Configuring managed MCP for enterprise/team deployments
- Configuring MCP tool search for large numbers of tools
- Importing MCP servers from Claude Desktop
- Using Claude Code itself as an MCP server

## When NOT to Use This Skill

- Creating a full plugin (use plugin-development skill)
- Writing custom skills or agents (use skills-authoring or subagents-and-teams)
- General Claude Code configuration unrelated to MCP (use claude-code-best-practices)
- Troubleshooting non-MCP issues (use claude-code-troubleshooting)

## Quick Reference

| Topic | Reference File |
|:------|:---------------|
| .mcp.json format, stdio setup, env vars, scope hierarchy, CLI commands, examples | `references/mcp-server-setup.md` |
| HTTP/SSE transports, remote server config, OAuth authentication, headers, security | `references/remote-servers.md` |
| Bundling MCP in plugins, plugin.json mcp field, `${CLAUDE_PLUGIN_ROOT}`, examples | `references/mcp-in-plugins.md` |

## MCP Setup Workflow

### Step 1: Choose Your Transport

| Transport | Use Case | Command Flag |
|:----------|:---------|:-------------|
| **HTTP** (recommended) | Cloud-based services, remote APIs | `--transport http` |
| **SSE** (deprecated) | Legacy remote servers | `--transport sse` |
| **stdio** | Local processes, custom scripts, direct system access | `--transport stdio` |

### Step 2: Add the Server

```bash
# Remote HTTP server
claude mcp add --transport http <name> <url>

# Remote HTTP with auth header
claude mcp add --transport http <name> <url> --header "Authorization: Bearer <token>"

# Local stdio server
claude mcp add --transport stdio <name> -- <command> [args...]

# Local stdio with environment variables
claude mcp add --transport stdio --env KEY=value <name> -- <command> [args...]
```

**Important:** All options (`--transport`, `--env`, `--scope`, `--header`) must come before the
server name. The `--` (double dash) separates the server name from the command and arguments
passed to the MCP server.

### Step 3: Choose the Scope

```bash
# Local scope (default) - private to you, current project only
claude mcp add --transport http stripe https://mcp.stripe.com

# Project scope - shared via .mcp.json, checked into version control
claude mcp add --transport http stripe --scope project https://mcp.stripe.com

# User scope - available to you across all projects
claude mcp add --transport http stripe --scope user https://mcp.stripe.com
```

### Step 4: Authenticate (if needed)

For remote servers requiring OAuth:

```
> /mcp
# Select the server, then follow the browser login flow
```

### Step 5: Verify

```bash
# List all configured servers
claude mcp list

# Get details for a specific server
claude mcp get <name>

# Check status within Claude Code
> /mcp
```

## Common MCP Server Examples

### GitHub

```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp/
# Then authenticate:
> /mcp
```

### Sentry

```bash
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
> /mcp
```

### Notion

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

### PostgreSQL

```bash
claude mcp add --transport stdio db -- npx -y @bytebase/dbhub \
  --dsn "postgresql://readonly:pass@prod.db.com:5432/analytics"
```

### Airtable

```bash
claude mcp add --transport stdio --env AIRTABLE_API_KEY=YOUR_KEY airtable \
  -- npx -y airtable-mcp-server
```

## MCP Server Categories

MCP servers connect Claude Code to many types of external services:

| Category | Examples |
|:---------|:---------|
| **Code & Version Control** | GitHub, GitLab, Bitbucket |
| **Issue Tracking** | Jira, Linear, Asana |
| **Monitoring & Observability** | Sentry, Datadog, PagerDuty |
| **Databases** | PostgreSQL, MySQL, MongoDB, Supabase |
| **Communication** | Slack, Discord, Gmail |
| **Design & Documentation** | Figma, Notion, Confluence |
| **Cloud & Infrastructure** | AWS, GCP, Cloudflare |
| **CRM & Sales** | HubSpot, Salesforce |
| **Custom & Internal** | Build your own using the MCP SDK |

**Note:** Use third-party MCP servers at your own risk. Anthropic has not verified the correctness
or security of all listed servers. Be especially careful with MCP servers that could fetch
untrusted content, as these can expose you to prompt injection risk.

Find hundreds more MCP servers at https://github.com/modelcontextprotocol/servers or build your
own using the MCP SDK at https://modelcontextprotocol.io/quickstart/server.

## Managing Servers

```bash
# List all configured servers
claude mcp list

# Get details for a specific server
claude mcp get <name>

# Remove a server
claude mcp remove <name>

# Add from JSON configuration
claude mcp add-json <name> '<json>'

# Import from Claude Desktop (macOS and WSL only)
claude mcp add-from-claude-desktop
```

## MCP Scopes

| Scope | Storage | Visibility | Use Case |
|:------|:--------|:-----------|:---------|
| **Local** (default) | `~/.claude.json` under project path | Private, current project | Personal servers, sensitive credentials |
| **Project** | `.mcp.json` in project root | Shared via version control | Team-shared servers, project-specific tools |
| **User** | `~/.claude.json` | Private, all projects | Personal utilities across projects |

**Precedence:** Local > Project > User. When servers with the same name exist at multiple scopes,
local-scoped servers take priority.

## Environment Variable Expansion in .mcp.json

`.mcp.json` files support environment variable expansion:

- `${VAR}` - Expands to the value of environment variable VAR
- `${VAR:-default}` - Expands to VAR if set, otherwise uses default

Variables can be expanded in: `command`, `args`, `env`, `url`, and `headers` fields.

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

## MCP Resources

MCP servers can expose resources that you reference using @ mentions:

```
> Can you analyze @github:issue://123 and suggest a fix?
> Compare @postgres:schema://users with @docs:file://database/user-model
```

Resources are automatically fetched and included as attachments. Type `@` to see all available
resources from connected servers.

## MCP Prompts as Commands

MCP servers can expose prompts that become available as slash commands:

```
> /mcp__github__list_prs
> /mcp__github__pr_review 456
> /mcp__jira__create_issue "Bug in login flow" high
```

Type `/` to discover available prompts from connected servers.

## MCP Tool Search

When you have many MCP servers, tool definitions can consume significant context. Claude Code
automatically enables Tool Search when MCP tool descriptions exceed 10% of the context window.
When active, tools are loaded on-demand rather than all at once.

Configure with the `ENABLE_TOOL_SEARCH` environment variable:

| Value | Behavior |
|:------|:---------|
| `auto` | Activates when MCP tools exceed 10% of context (default) |
| `auto:<N>` | Activates at custom threshold percentage |
| `true` | Always enabled |
| `false` | Disabled, all MCP tools loaded upfront |

```bash
ENABLE_TOOL_SEARCH=auto:5 claude   # 5% threshold
ENABLE_TOOL_SEARCH=false claude    # Disable tool search
```

## MCP Output Limits

- Warning threshold: 10,000 tokens per tool output
- Default maximum: 25,000 tokens
- Configure with `MAX_MCP_OUTPUT_TOKENS`:

```bash
export MAX_MCP_OUTPUT_TOKENS=50000
claude
```

## Using Claude Code as an MCP Server

Claude Code can itself serve as an MCP server for other applications:

```bash
claude mcp serve
```

Add to Claude Desktop's `claude_desktop_config.json`:

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

If `claude` is not in your PATH, use the full path from `which claude`.

## Managed MCP for Organizations

Organizations can centrally control MCP servers through two options:

1. **`managed-mcp.json`**: Deploy a fixed set of MCP servers that users cannot modify.
   Located at system-wide paths requiring admin privileges:
   - macOS: `/Library/Application Support/ClaudeCode/managed-mcp.json`
   - Linux/WSL: `/etc/claude-code/managed-mcp.json`
   - Windows: `C:\Program Files\ClaudeCode\managed-mcp.json`

2. **Allowlists/Denylists**: Allow users to add their own servers within policy constraints
   using `allowedMcpServers` and `deniedMcpServers` in managed settings.

See `references/remote-servers.md` for details on managed configurations.

## Reference File Index

| File | Contents |
|:-----|:---------|
| `references/mcp-server-setup.md` | .mcp.json configuration format, stdio server setup, environment variables, scope hierarchy, CLI commands, JSON configuration, importing from Claude Desktop, practical examples |
| `references/remote-servers.md` | HTTP and SSE transports, remote server configuration, OAuth 2.0 authentication, pre-configured OAuth credentials, headers, managed MCP configuration, allowlists/denylists, security considerations |
| `references/mcp-in-plugins.md` | Bundling MCP servers in plugins, .mcp.json at plugin root, plugin.json inline MCP, `${CLAUDE_PLUGIN_ROOT}`, automatic lifecycle, transport types, viewing plugin MCP servers |
