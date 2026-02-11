# Migration Guide: Standalone to Plugin

This guide covers converting existing standalone configurations in `.claude/` into plugin format
for easier sharing and distribution.

## When to Migrate

Migrate from standalone to plugin when:

- You want to share your skills, commands, or hooks with teammates
- You need the same configuration across multiple projects
- You want version control and easy updates
- You plan to distribute through a marketplace

Stay with standalone if:

- The configuration is personal to one project
- You are still experimenting and iterating
- You want short skill names without namespacing

## Comparison Table

| Aspect | Standalone (`.claude/`) | Plugin |
|:-------|:------------------------|:-------|
| Availability | Only in one project | Shared via marketplaces, reusable across projects |
| Command files | `.claude/commands/` | `plugin-name/commands/` |
| Skill files | `.claude/skills/` | `plugin-name/skills/` |
| Agent files | `.claude/agents/` | `plugin-name/agents/` |
| Hook configuration | `hooks` in `.claude/settings.json` | `hooks/hooks.json` in plugin directory |
| MCP configuration | `.claude/.mcp.json` or project `.mcp.json` | `.mcp.json` at plugin root |
| Skill names | `/hello`, `/review` | `/plugin-name:hello`, `/plugin-name:review` |
| Sharing | Must manually copy files | Install with `/plugin install` |
| Versioning | No built-in versioning | Semantic versioning in `plugin.json` |
| Updates | Manual file copying | Marketplace auto-updates |

## Step-by-Step Migration

### Step 1: Create the Plugin Structure

Create a new plugin directory with the manifest:

```bash
mkdir -p my-plugin/.claude-plugin
```

Create the manifest file at `my-plugin/.claude-plugin/plugin.json`:

```json
{
  "name": "my-plugin",
  "description": "Migrated from standalone configuration",
  "version": "1.0.0"
}
```

### Step 2: Copy Commands

If you have commands in `.claude/commands/`, copy them to the plugin:

```bash
cp -r .claude/commands my-plugin/
```

**What changes**: Command `/hello` becomes `/my-plugin:hello` due to namespacing.

### Step 3: Copy Skills

If you have skills in `.claude/skills/`, copy them to the plugin:

```bash
cp -r .claude/skills my-plugin/
```

Skills are model-invoked and namespaced under the plugin name automatically.

### Step 4: Copy Agents

If you have agents in `.claude/agents/`, copy them to the plugin:

```bash
cp -r .claude/agents my-plugin/
```

### Step 5: Migrate Hooks

Hooks require a format change. In standalone configuration, hooks live inside
`.claude/settings.json` or `settings.local.json`. In a plugin, they go in a separate file.

Create the hooks directory:

```bash
mkdir my-plugin/hooks
```

Create `my-plugin/hooks/hooks.json` by copying the `hooks` object from your settings file.
The hook format is identical -- you just move it to a dedicated file.

**Before** (in `.claude/settings.json`):

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
  },
  "other_settings": "..."
}
```

**After** (in `my-plugin/hooks/hooks.json`):

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

The hook command receives hook input as JSON on stdin. Use `jq` to extract fields like file paths.

**Path references**: If your hooks reference local scripts, update paths to use
`${CLAUDE_PLUGIN_ROOT}`:

```json
{
  "type": "command",
  "command": "${CLAUDE_PLUGIN_ROOT}/scripts/lint.sh"
}
```

### Step 6: Migrate MCP Configuration

If you have MCP servers configured in `.claude/.mcp.json` or a project-level `.mcp.json`,
copy it to the plugin root:

```bash
cp .mcp.json my-plugin/.mcp.json
```

Update any local paths to use `${CLAUDE_PLUGIN_ROOT}`:

**Before**:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["./servers/my-server.js"]
    }
  }
}
```

**After**:

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

### Step 7: Test the Migrated Plugin

Load your plugin to verify everything works:

```bash
claude --plugin-dir ./my-plugin
```

Test each component:

- Run your commands (remember they are now namespaced: `/my-plugin:command-name`)
- Check agents appear in `/agents`
- Verify hooks trigger correctly on the expected events
- Confirm MCP servers connect

### Step 8: Clean Up

After verifying the plugin works correctly, you can remove the original files from `.claude/`
to avoid duplicates. The plugin version takes precedence when loaded.

## Final Plugin Structure

After migration, your plugin directory should look like:

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── review.md
│   └── deploy.md
├── skills/
│   └── code-review/
│       └── SKILL.md
├── agents/
│   └── reviewer.md
├── hooks/
│   └── hooks.json
├── .mcp.json
└── README.md
```

## Common Migration Pitfalls

1. **Forgetting namespacing**: All commands change from `/name` to `/plugin-name:name`. Update
   any documentation or scripts that reference command names.

2. **Putting components inside `.claude-plugin/`**: Only `plugin.json` goes inside
   `.claude-plugin/`. All other directories must be at the plugin root.

3. **Hardcoded paths in hooks/MCP**: Replace local paths with `${CLAUDE_PLUGIN_ROOT}` so
   they work after installation (plugins are copied to a cache directory).

4. **Not testing after migration**: Always test with `claude --plugin-dir ./my-plugin` before
   distributing.

5. **Leaving duplicates**: After successful migration, remove the original standalone files
   from `.claude/` to prevent confusion or conflicts.
