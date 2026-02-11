---
name: plugin-development
description: >
  Comprehensive guide for creating, modifying, testing, releasing, and maintaining Claude Code plugins.
  Use when creating a plugin, defining plugin structure, writing plugin.json manifests, publishing to
  marketplaces, installing plugins, setting up plugin directories, distributing plugins to teams,
  migrating standalone configurations to plugin format, or working with plugin components like skills,
  agents, hooks, MCP servers, and LSP servers. Trigger phrases: "create a plugin", "plugin structure",
  "plugin.json", "marketplace", "install plugin", "plugin directory", "distribute plugin",
  "plugin manifest", "plugin marketplace", "migrate to plugin", "plugin components", "LSP plugin".
---

# Plugin Development Guide

Plugins let you extend Claude Code with custom functionality that can be shared across projects and
teams. A plugin is a self-contained directory containing a manifest file (`plugin.json`) and one or
more components: skills, commands, agents, hooks, MCP server configs, and LSP server configs.

Plugins use namespaced skill names (like `/my-plugin:hello`) to prevent conflicts when multiple
plugins define skills with the same name. They support semantic versioning, marketplace distribution,
and scoped installation (user, project, local, or managed).

This guide covers the full plugin lifecycle: planning, creating, adding components, testing locally,
distributing through marketplaces, and migrating existing standalone configurations into plugin format.

## When to Use This Skill

- Creating a new Claude Code plugin from scratch
- Adding components (skills, commands, agents, hooks, MCP, LSP) to an existing plugin
- Writing or editing a `plugin.json` manifest
- Setting up plugin directory structure
- Testing plugins locally with `--plugin-dir`
- Publishing plugins to a marketplace
- Installing plugins from marketplaces
- Migrating standalone `.claude/` configurations into plugin format
- Setting up team or organization marketplaces
- Debugging plugin loading or configuration issues

## When NOT to Use This Skill

- Writing standalone skills in `.claude/skills/` (use the skills-authoring skill instead)
- Configuring hooks directly in `settings.json` without a plugin (use hooks-automation skill)
- Setting up MCP servers without a plugin wrapper (use mcp-integration skill)
- General Claude Code configuration or troubleshooting

## Quick Reference

| Topic | Reference File |
|:------|:---------------|
| plugin.json schema, required/optional fields, marketplace.json format | `references/plugin-manifest.md` |
| Directory structure, component types, path conventions, `${CLAUDE_PLUGIN_ROOT}` | `references/plugin-components.md` |
| Marketplace creation, publishing, installation commands, team distribution | `references/marketplace-distribution.md` |
| Converting standalone configs to plugins, comparison tables, step-by-step migration | `references/migration-guide.md` |

## Plugin Development Workflow

### Phase 1: Plan

Decide whether you need a plugin or standalone configuration:

| Approach | Skill Names | Best For |
|:---------|:------------|:---------|
| **Standalone** (`.claude/` directory) | `/hello` | Personal workflows, project-specific customizations, quick experiments |
| **Plugins** (directories with `.claude-plugin/plugin.json`) | `/plugin-name:hello` | Sharing with teammates, distributing to community, versioned releases, reusable across projects |

**Use standalone configuration when:**

- You are customizing Claude Code for a single project
- The configuration is personal and does not need to be shared
- You are experimenting with skills or hooks before packaging them
- You want short skill names like `/hello` or `/review`

**Use plugins when:**

- You want to share functionality with your team or community
- You need the same skills/agents across multiple projects
- You want version control and easy updates for your extensions
- You are distributing through a marketplace
- You are okay with namespaced skills like `/my-plugin:hello`

**Tip:** Start with standalone configuration in `.claude/` for quick iteration, then convert to a
plugin when you are ready to share.

### Phase 2: Create the Plugin

1. Create a directory for your plugin:

```bash
mkdir my-plugin
```

2. Create the manifest directory and file:

```bash
mkdir my-plugin/.claude-plugin
```

Create `my-plugin/.claude-plugin/plugin.json`:

```json
{
  "name": "my-plugin",
  "description": "A description of what your plugin does",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

See `references/plugin-manifest.md` for all available fields.

### Phase 3: Add Components

Add any combination of components to your plugin root directory:

- **`commands/`** - Slash commands as Markdown files
- **`skills/`** - Agent Skills with `SKILL.md` files (model-invoked)
- **`agents/`** - Custom agent definitions
- **`hooks/`** - Event handlers in `hooks.json`
- **`.mcp.json`** - MCP server configurations
- **`.lsp.json`** - LSP server configurations

See `references/plugin-components.md` for detailed component documentation.

### Phase 4: Test Locally

Load your plugin with the `--plugin-dir` flag:

```bash
claude --plugin-dir ./my-plugin
```

Test each component:

- Try your commands with `/plugin-name:command-name`
- Check that agents appear in `/agents`
- Verify hooks trigger correctly
- Confirm MCP servers connect

You can load multiple plugins at once:

```bash
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two
```

Restart Claude Code after making changes to pick up updates.

**Debugging tips:**

1. Check the structure: ensure directories are at the plugin root, NOT inside `.claude-plugin/`
2. Test components individually
3. Check the `/plugin` Errors tab for loading issues

### Phase 5: Release and Distribute

1. Add documentation: include a `README.md` with installation and usage instructions
2. Version your plugin using semantic versioning in `plugin.json`
3. Create or use a marketplace for distribution
4. Test with others before wider distribution

See `references/marketplace-distribution.md` for marketplace setup and publishing.

## Critical Rules

### Directory Structure Rules

- **`.claude-plugin/` contains ONLY `plugin.json`**. Never put `commands/`, `agents/`, `skills/`,
  or `hooks/` inside `.claude-plugin/`. All component directories go at the plugin root level.

- Correct structure:
  ```
  my-plugin/
  ├── .claude-plugin/
  │   └── plugin.json        # ONLY the manifest goes here
  ├── commands/               # At plugin root
  ├── skills/                 # At plugin root
  ├── agents/                 # At plugin root
  ├── hooks/                  # At plugin root
  ├── .mcp.json               # At plugin root
  └── .lsp.json               # At plugin root
  ```

### Path Conventions

- Use `${CLAUDE_PLUGIN_ROOT}` in hook commands and MCP configurations to reference files
  relative to the plugin root. This variable resolves to the absolute path of the plugin
  directory at runtime, ensuring paths work regardless of where the plugin is installed.

- All component paths are relative to the plugin root directory.

### Naming and Versioning

- The `name` field in `plugin.json` becomes the namespace prefix for all skills in the plugin.
  A plugin named `my-tools` with a skill called `review` creates the command `/my-tools:review`.

- Use semantic versioning (`MAJOR.MINOR.PATCH`) for the `version` field.

### Installation Scopes

- **User scope**: installed for yourself across all projects (default)
- **Project scope**: installed for all collaborators on a repository (stored in `.claude/settings.json`)
- **Local scope**: installed for yourself in a specific repository only
- **Managed scope**: installed by administrators via managed settings (read-only)

## Reference File Index

| File | Contents |
|:-----|:---------|
| `references/plugin-manifest.md` | Complete `plugin.json` schema with all required and optional fields, `marketplace.json` format, version conventions |
| `references/plugin-components.md` | Plugin directory structure, how each component type works (skills, commands, hooks, MCP, LSP, agents), path conventions with `${CLAUDE_PLUGIN_ROOT}` |
| `references/marketplace-distribution.md` | How marketplaces work, creating a marketplace, publishing plugins, installation commands, private/team distribution, auto-updates |
| `references/migration-guide.md` | Converting standalone configs to plugin format, comparison tables, step-by-step migration instructions |
