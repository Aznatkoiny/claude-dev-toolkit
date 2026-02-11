# Remote MCP Servers Reference

## HTTP Transport (Recommended)

HTTP servers are the recommended option for connecting to remote MCP services. This is the
most widely supported transport for cloud-based services.

```bash
# Basic syntax
claude mcp add --transport http <name> <url>

# Example: Connect to Notion
claude mcp add --transport http notion https://mcp.notion.com/mcp

# Example with Bearer token
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"
```

### HTTP JSON Configuration

```json
{
  "mcpServers": {
    "my-api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```

## SSE Transport (Deprecated)

The SSE (Server-Sent Events) transport is deprecated. Use HTTP servers instead where available.

```bash
# Basic syntax
claude mcp add --transport sse <name> <url>

# Example: Connect to Asana
claude mcp add --transport sse asana https://mcp.asana.com/sse

# Example with authentication header
claude mcp add --transport sse private-api https://api.company.com/sse \
  --header "X-API-Key: your-key-here"
```

## Authentication

### OAuth 2.0

Many cloud-based MCP servers require OAuth authentication. Claude Code supports OAuth 2.0
for secure connections.

**Standard OAuth flow:**

1. Add the server that requires authentication:

```bash
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
```

2. Use the `/mcp` command within Claude Code:

```
> /mcp
```

Then follow the steps in your browser to login.

**Tips:**

- Authentication tokens are stored securely and refreshed automatically
- Use "Clear authentication" in the `/mcp` menu to revoke access
- If your browser does not open automatically, copy the provided URL
- OAuth authentication works with HTTP servers

### Pre-configured OAuth Credentials

Some MCP servers do not support automatic OAuth setup. If you see an error like "Incompatible
auth server: does not support dynamic client registration," the server requires pre-configured
credentials.

**Step 1:** Register an OAuth app with the server's developer portal. Note your client ID and
client secret. If the server requires a redirect URI, choose a port and register a redirect
URI in the format `http://localhost:PORT/callback`.

**Step 2:** Add the server with your credentials.

Using `claude mcp add`:

```bash
claude mcp add --transport http \
  --client-id your-client-id --client-secret --callback-port 8080 \
  my-server https://mcp.example.com/mcp
```

The `--client-secret` flag prompts for the secret with masked input.

Using `claude mcp add-json`:

```bash
claude mcp add-json my-server \
  '{"type":"http","url":"https://mcp.example.com/mcp","oauth":{"clientId":"your-client-id","callbackPort":8080}}' \
  --client-secret
```

Using environment variable (CI/non-interactive):

```bash
MCP_CLIENT_SECRET=your-secret claude mcp add --transport http \
  --client-id your-client-id --client-secret --callback-port 8080 \
  my-server https://mcp.example.com/mcp
```

**Step 3:** Authenticate in Claude Code by running `/mcp` and following the browser login flow.

**Tips:**

- The client secret is stored securely in your system keychain (macOS) or a credentials file,
  not in your config
- If the server uses a public OAuth client with no secret, use only `--client-id` without
  `--client-secret`
- These flags only apply to HTTP and SSE transports. They have no effect on stdio servers
- Use `claude mcp get <name>` to verify that OAuth credentials are configured for a server

### Header-Based Authentication

For servers that use API keys or bearer tokens:

```bash
# Bearer token
claude mcp add --transport http my-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"

# API key
claude mcp add --transport sse private-api https://api.company.com/sse \
  --header "X-API-Key: your-key-here"
```

In `.mcp.json`, use environment variable expansion for secrets:

```json
{
  "mcpServers": {
    "my-api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${MY_API_TOKEN}"
      }
    }
  }
}
```

## Security Considerations

- Use third-party MCP servers at your own risk. Anthropic has not verified the correctness or
  security of all listed servers.
- Be especially careful when using MCP servers that could fetch untrusted content, as these can
  expose you to prompt injection risk.
- For project-scoped servers (`.mcp.json`), Claude Code prompts for approval before use. Reset
  approval choices with `claude mcp reset-project-choices`.
- Store secrets in environment variables, not directly in `.mcp.json` files that are checked
  into version control.

## Managed MCP Configuration

Organizations can centrally control MCP servers using two approaches.

### Option 1: Exclusive Control with managed-mcp.json

Deploy a `managed-mcp.json` file to take exclusive control over all MCP servers. Users cannot
add, modify, or use any servers other than those defined in this file.

**System-wide paths (require admin privileges):**

| Platform | Path |
|:---------|:-----|
| macOS | `/Library/Application Support/ClaudeCode/managed-mcp.json` |
| Linux/WSL | `/etc/claude-code/managed-mcp.json` |
| Windows | `C:\Program Files\ClaudeCode\managed-mcp.json` |

These are system-wide paths (not user home directories like `~/Library/...`).

**Format** (same as standard `.mcp.json`):

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/"
    },
    "sentry": {
      "type": "http",
      "url": "https://mcp.sentry.dev/mcp"
    },
    "company-internal": {
      "type": "stdio",
      "command": "/usr/local/bin/company-mcp-server",
      "args": ["--config", "/etc/company/mcp-config.json"],
      "env": {
        "COMPANY_API_URL": "https://internal.company.com"
      }
    }
  }
}
```

### Option 2: Policy-Based Control with Allowlists/Denylists

Allow users to add their own servers while enforcing restrictions using `allowedMcpServers`
and `deniedMcpServers` in the managed settings file.

Each entry restricts servers in one of three ways:

| Restriction | Field | Description |
|:------------|:------|:------------|
| By server name | `serverName` | Matches the configured name |
| By command | `serverCommand` | Matches the exact command and arguments (stdio) |
| By URL pattern | `serverUrl` | Matches remote server URLs with wildcard support |

Each entry must have exactly one of `serverName`, `serverCommand`, or `serverUrl`.

**Example configuration:**

```json
{
  "allowedMcpServers": [
    { "serverName": "github" },
    { "serverName": "sentry" },
    { "serverCommand": ["npx", "-y", "@modelcontextprotocol/server-filesystem"] },
    { "serverCommand": ["python", "/usr/local/bin/approved-server.py"] },
    { "serverUrl": "https://mcp.company.com/*" },
    { "serverUrl": "https://*.internal.corp/*" }
  ],
  "deniedMcpServers": [
    { "serverName": "dangerous-server" },
    { "serverCommand": ["npx", "-y", "unapproved-package"] },
    { "serverUrl": "https://*.untrusted.com/*" }
  ]
}
```

### How Command-Based Restrictions Work

- Command arrays must match exactly -- both the command and all arguments in the correct order
- `["npx", "-y", "server"]` will NOT match `["npx", "server"]` or `["npx", "-y", "server", "--flag"]`
- When the allowlist contains any `serverCommand` entries, stdio servers must match one of those commands
- Stdio servers cannot pass by name alone when command restrictions are present

### How URL-Based Restrictions Work

URL patterns support wildcards using `*`:

- `https://mcp.company.com/*` - Allow all paths on a specific domain
- `https://*.example.com/*` - Allow any subdomain
- `http://localhost:*/*` - Allow any port on localhost

When the allowlist contains any `serverUrl` entries, remote servers must match one of those URL
patterns. Remote servers cannot pass by name alone when URL restrictions are present.

### Allowlist Behavior (allowedMcpServers)

| Value | Effect |
|:------|:-------|
| `undefined` (default) | No restrictions - users can configure any MCP server |
| Empty array `[]` | Complete lockdown - no MCP servers allowed |
| List of entries | Users can only configure servers matching by name, command, or URL |

### Denylist Behavior (deniedMcpServers)

| Value | Effect |
|:------|:-------|
| `undefined` (default) | No servers blocked |
| Empty array `[]` | No servers blocked |
| List of entries | Specified servers explicitly blocked across all scopes |

### Important Rules

- Option 1 and Option 2 can be combined. If `managed-mcp.json` exists, it has exclusive control
  and users cannot add servers. Allowlists/denylists still apply to the managed servers.
- Denylist takes absolute precedence. If a server matches a denylist entry, it is blocked even
  if it is on the allowlist.
- Name-based, command-based, and URL-based restrictions work together: a server passes if it
  matches either a name entry, a command entry, or a URL pattern (unless blocked by denylist).
- When using `managed-mcp.json`, users cannot add MCP servers through `claude mcp add` or
  configuration files.
