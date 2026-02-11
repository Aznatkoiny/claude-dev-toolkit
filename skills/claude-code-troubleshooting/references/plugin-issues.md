# Plugin Issues

Troubleshooting guide for Claude Code plugin components: plugins, skills, commands, hooks, MCP servers,
and LSP servers. This covers common structural errors, loading failures, and debugging techniques for
each component type.

## Plugin Not Loading

If your plugin does not appear in Claude Code after installation or when using `--plugin-dir`:

### Check Directory Structure

The most common cause of plugin loading failures is incorrect directory structure. Component directories
must be at the plugin root level, NOT inside `.claude-plugin/`.

**Correct structure:**

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json        # ONLY the manifest goes here
├── commands/               # At plugin root
├── skills/                 # At plugin root
├── agents/                 # At plugin root
├── hooks/                  # At plugin root (hooks.json)
├── .mcp.json               # At plugin root
└── .lsp.json               # At plugin root
```

**Wrong structure (common mistake):**

```
my-plugin/
├── .claude-plugin/
│   ├── plugin.json
│   ├── commands/           # WRONG: should not be inside .claude-plugin/
│   └── skills/             # WRONG: should not be inside .claude-plugin/
```

### Validate plugin.json

Ensure your `plugin.json` is valid JSON and contains the required fields:

```json
{
  "name": "my-plugin",
  "description": "A description of what the plugin does",
  "version": "1.0.0"
}
```

**Common plugin.json errors:**

- Trailing commas in JSON (not allowed)
- Missing required fields (`name`, `description`, `version`)
- Invalid version format (must be semantic versioning: `MAJOR.MINOR.PATCH`)
- Syntax errors (unclosed quotes, missing braces)

Validate your JSON:

```bash
# Quick JSON validation
python3 -c "import json; json.load(open('.claude-plugin/plugin.json'))"

# Or with jq
jq . .claude-plugin/plugin.json
```

### Check the Errors Tab

Run Claude Code and check the `/plugin` Errors tab for specific loading errors. This tab shows:

- JSON parsing errors in plugin.json
- Missing required fields
- Component loading failures
- Path resolution issues

### Restart Claude Code

After making any changes to plugin files, you must restart Claude Code to pick up updates. Plugin
files are read at startup and are not hot-reloaded.

## Skill Not Triggering

If your plugin's skill exists but Claude does not invoke it when expected:

### Verify Skill Structure

Each skill needs a `SKILL.md` file inside a named directory under `skills/`:

```
my-plugin/
└── skills/
    └── my-skill/
        ├── SKILL.md          # Required: skill definition with frontmatter
        └── references/       # Optional: reference files
            └── details.md
```

### Check SKILL.md Frontmatter

The `description` field in the SKILL.md frontmatter is critical -- Claude uses it to decide when to
invoke the skill. Ensure it contains:

- A clear description of when to use the skill
- Relevant trigger phrases that match how users describe the problem
- Keywords that Claude can match against user queries

```yaml
---
name: my-skill
description: >
  Use when the user asks about X, Y, or Z. Trigger phrases: "keyword1",
  "keyword2", "keyword3".
---
```

**Common reasons a skill doesn't trigger:**

- The `description` field is too vague or doesn't match user queries
- Missing or malformed YAML frontmatter (must be between `---` delimiters)
- The skill name conflicts with another skill (check for namespace collisions)
- SKILL.md file is empty or has only frontmatter with no body content

### Test with Explicit Invocation

For plugin skills, use the namespaced command to invoke directly:

```
/plugin-name:skill-name
```

If explicit invocation works but automatic triggering doesn't, the issue is in the description
field -- it needs better trigger phrases.

## Command Not Appearing

If a command defined in your plugin doesn't appear in the commands list:

### Verify Command File Location

Commands must be Markdown files in the `commands/` directory at the plugin root:

```
my-plugin/
└── commands/
    └── my-command.md       # Creates /plugin-name:my-command
```

### Check Command File Format

Command files should be plain Markdown. The filename (minus `.md`) becomes the command name.

### Check for Name Conflicts

If multiple plugins define commands with the same name, there may be conflicts. Use the
`/plugin-name:command-name` namespaced format to disambiguate.

## Hook Not Firing

If a hook defined in your plugin doesn't execute when the expected event occurs:

### Verify hooks.json Location and Format

Hooks must be defined in a `hooks.json` file in the `hooks/` directory at the plugin root:

```
my-plugin/
└── hooks/
    └── hooks.json
```

### Check hooks.json Structure

The hooks.json file must use the correct event names and structure:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hook": {
          "type": "command",
          "command": "${CLAUDE_PLUGIN_ROOT}/scripts/pre-write.sh"
        }
      }
    ],
    "PostToolUse": [
      {
        "matcher": ".*",
        "hook": {
          "type": "command",
          "command": "echo 'Tool used'"
        }
      }
    ]
  }
}
```

**Common hooks.json errors:**

- Using wrong event names (valid events: `PreToolUse`, `PostToolUse`, `Notification`, `Stop`,
  `SubagentStop`)
- Invalid JSON syntax
- Missing the outer `hooks` key
- Incorrect matcher regex patterns
- Script files not being executable (`chmod +x script.sh`)

### Use `${CLAUDE_PLUGIN_ROOT}` for Paths

Always use `${CLAUDE_PLUGIN_ROOT}` when referencing files within the plugin. This variable resolves
to the absolute path of the plugin directory at runtime:

```json
{
  "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format.sh"
}
```

**Do NOT use hardcoded absolute paths** -- they will break when the plugin is installed on different
machines:

```json
{
  "command": "/Users/myname/plugins/my-plugin/scripts/format.sh"
}
```

### Check Script Permissions

If the hook command references a script file, ensure it is executable:

```bash
chmod +x scripts/format.sh
```

### Test the Hook Command Independently

Run the hook command manually in your terminal to verify it works:

```bash
# Test the command outside of Claude Code
cd /path/to/my-plugin
./scripts/format.sh
```

## MCP Server Not Starting

If an MCP server defined in your plugin fails to connect:

### Verify .mcp.json Location

The `.mcp.json` file must be at the plugin root level:

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json
└── .mcp.json               # At plugin root
```

### Check .mcp.json Format

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/servers/my-server.js"],
      "env": {
        "MY_VAR": "value"
      }
    }
  }
}
```

**Common .mcp.json errors:**

- Server command not found (missing dependency or wrong path)
- Using hardcoded paths instead of `${CLAUDE_PLUGIN_ROOT}`
- Missing environment variables required by the server
- Port conflicts if the server listens on a specific port
- Invalid JSON syntax

### Check Server Dependencies

Ensure all dependencies required by the MCP server are installed:

```bash
# For Node.js servers
cd /path/to/my-plugin
npm install

# For Python servers
pip install -r requirements.txt
```

### Test the Server Independently

Try running the MCP server command directly:

```bash
node /path/to/my-plugin/servers/my-server.js
```

Check for startup errors, missing modules, or configuration issues.

### Check /doctor Output

Run `/doctor` inside Claude Code -- it specifically checks for MCP server configuration errors and
will report connection failures.

## LSP Server Issues

If an LSP server defined in your plugin is not working:

### Verify .lsp.json Location and Format

The `.lsp.json` file must be at the plugin root:

```json
{
  "lspServers": {
    "my-lsp": {
      "command": "my-language-server",
      "args": ["--stdio"],
      "languages": ["python"]
    }
  }
}
```

### Common LSP Issues

- Language server binary not installed or not in PATH
- Wrong communication mode (most LSP servers use `--stdio`)
- Language identifier mismatch (must match VS Code language identifiers)

## General Plugin Debugging Techniques

### Step 1: Isolate the Problem

Test one component at a time. If you have a plugin with skills, hooks, and MCP servers, temporarily
remove all but one component to determine which is causing issues.

### Step 2: Check Logs

Look for error output in the terminal where Claude Code is running. Many loading errors are printed
to stderr during startup.

### Step 3: Validate All JSON Files

Plugin configuration relies heavily on JSON files. Validate all of them:

```bash
# Validate all JSON files in the plugin
for f in $(find /path/to/my-plugin -name "*.json" -not -path "*/node_modules/*"); do
  echo "Checking $f..."
  python3 -c "import json; json.load(open('$f'))" 2>&1 || echo "  INVALID JSON: $f"
done
```

### Step 4: Use --plugin-dir for Local Testing

Always test plugins locally before publishing:

```bash
claude --plugin-dir ./my-plugin
```

You can load multiple plugins simultaneously:

```bash
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two
```

### Step 5: Compare Against Working Examples

If your plugin structure seems correct but still doesn't work, compare it against a known-working
plugin. Pay attention to:

- Exact directory names (case-sensitive on Linux/macOS)
- File locations relative to plugin root
- JSON field names and structure
- Version string format

## Common Structural Errors Summary

| Error | Cause | Fix |
|:------|:------|:----|
| Plugin not detected | Missing `.claude-plugin/plugin.json` | Create the manifest file with required fields |
| Components not loading | Directories inside `.claude-plugin/` | Move component directories to plugin root |
| Skill not triggering | Poor description in SKILL.md frontmatter | Add specific trigger phrases and keywords |
| Command not found | Wrong file extension or location | Ensure `.md` files are in `commands/` at root |
| Hook not firing | Wrong event name or invalid matcher | Check event names and regex patterns |
| Hook script fails | Hardcoded paths in commands | Use `${CLAUDE_PLUGIN_ROOT}` for all paths |
| Hook script fails | Script not executable | Run `chmod +x` on the script file |
| MCP server won't start | Missing dependencies | Install required packages, test server independently |
| JSON parse error | Trailing commas or syntax errors | Validate JSON with `jq` or `python3 -c` |
| Name conflicts | Duplicate names across plugins | Use namespaced commands: `/plugin-name:command-name` |
