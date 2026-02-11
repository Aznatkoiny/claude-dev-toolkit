---
description: Interactive wizard to scaffold a new Claude Code plugin. Creates the complete directory structure, plugin.json, and starter components based on your requirements.
disable-model-invocation: true
---

# /create-plugin — Plugin Scaffolding Wizard

You are running the plugin creation wizard. Guide the user through creating a new Claude Code plugin step-by-step.

## Step 1: Gather Information

Ask the user these questions (use AskUserQuestion tool for structured input):

1. **Plugin name**: What should the plugin be called? (lowercase, hyphens allowed)
2. **Description**: What does the plugin do? (1-2 sentences)
3. **Author name**: Who is the author?
4. **Components**: Which components does the plugin need?
   - Skills (domain knowledge, workflows)
   - Commands (slash commands)
   - Hooks (event-driven automation)
   - Agents (autonomous subagents)
   - MCP servers (external tool integration)
   - Output styles (response formatting)

If the user provided arguments with the command (e.g., `/create-plugin my-plugin`), use the first argument as the plugin name and skip that question.

## Step 2: Choose Location

Ask where to create the plugin:
- Current directory: `./$PLUGIN_NAME/`
- Home directory: `~/claude-plugins/$PLUGIN_NAME/`
- Custom path

## Step 3: Scaffold

Create the directory structure based on their choices:

```bash
mkdir -p $PLUGIN_PATH/.claude-plugin
mkdir -p $PLUGIN_PATH/skills    # if skills selected
mkdir -p $PLUGIN_PATH/commands  # if commands selected
mkdir -p $PLUGIN_PATH/hooks     # if hooks selected
mkdir -p $PLUGIN_PATH/agents    # if agents selected
mkdir -p $PLUGIN_PATH/output-styles  # if output styles selected
```

Write `plugin.json`:
```json
{
  "name": "$PLUGIN_NAME",
  "version": "1.0.0",
  "description": "$DESCRIPTION",
  "author": {"name": "$AUTHOR"}
}
```

## Step 4: Create Starter Files

For each selected component, create a minimal starter file:

**Skill**: Create `skills/example/SKILL.md` with template frontmatter and body
**Command**: Create `commands/example.md` with template frontmatter and instructions
**Hook**: Create `hooks/hooks.json` with a commented example configuration
**Agent**: Create `agents/example.md` with template frontmatter and system prompt
**Output Style**: Create `output-styles/example.md` with template frontmatter

## Step 5: Write README

Create a README.md with:
- Plugin name and description
- What's included (list components)
- Installation instructions
- Usage examples

## Step 6: Test Instructions

Tell the user how to test:
```bash
claude --plugin-dir $PLUGIN_PATH/
```

Or for marketplace-based testing, explain the development marketplace setup.

## Rules

- Always create `.claude-plugin/plugin.json` first — it's the minimum viable plugin
- Use `${CLAUDE_PLUGIN_ROOT}` in any hook scripts or MCP configs
- Make hook scripts executable
- Keep starter files minimal but functional
- Follow naming conventions: lowercase, hyphens for directories
