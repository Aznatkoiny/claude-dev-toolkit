---
description: Guided hook creation workflow. Helps you create Claude Code hooks by walking through event selection, matcher configuration, and handler implementation.
disable-model-invocation: true
---

# /create-hook — Hook Creation Wizard

You are running the hook creation wizard. Guide the user through creating a Claude Code hook step-by-step.

## Step 1: What Should the Hook Do?

Ask the user what they want to automate. Use AskUserQuestion with these common patterns:

| Pattern | Event | Example |
|---------|-------|---------|
| Block dangerous commands | PreToolUse | Block `rm -rf /` |
| Validate file changes | PostToolUse | Check code style after edits |
| Auto-format code | PostToolUse | Run prettier after Write/Edit |
| Desktop notifications | Stop | Notify when task completes |
| Inject context at start | SessionStart | Load project-specific context |
| Re-inject after compaction | Notification | Restore important context |
| Protect specific files | PreToolUse | Block edits to production configs |
| Validate user prompts | UserPromptSubmit | Check for sensitive data in prompts |

If the user provided arguments (e.g., `/create-hook block dangerous commands`), infer the pattern from their description.

## Step 2: Choose Hook Type

Ask the user which type:
- **Command hook** (`type: "command"`): Runs a shell script. Best for: file operations, system checks, external tools.
- **Prompt hook** (`type: "prompt"`): Uses an LLM. Best for: semantic analysis, content review, context-dependent decisions.

## Step 3: Choose Location

Ask where the hook should live:
- **Plugin hook**: `hooks/hooks.json` in a plugin directory (portable, distributable)
- **Project hook**: `.claude/settings.json` (project-specific)
- **User hook**: `~/.claude/settings.json` (personal, all projects)

## Step 4: Build the Hook Configuration

Generate the hooks.json or settings.json entry:

```json
{
  "hooks": {
    "EVENT_NAME": [
      {
        "matcher": "TOOL_PATTERN",
        "hooks": [
          {
            "type": "command|prompt",
            "command": "path/to/handler.sh"
          }
        ]
      }
    ]
  }
}
```

For **command hooks**, also create the handler script:
- Parse JSON input from stdin using `jq`
- Output a JSON decision to stdout
- Make the script executable

For **prompt hooks**, configure the model and prompt text.

## Step 5: Implement the Handler

Write the handler script based on the user's requirements. Follow this template:

```bash
#!/bin/bash
# Hook handler for [EVENT_NAME]
# Matcher: [PATTERN]

input=$(cat)

# Extract relevant fields
tool_name=$(echo "$input" | jq -r '.tool_name // empty')
tool_input=$(echo "$input" | jq -r '.tool_input // empty')

# Your logic here
if [condition]; then
  echo '{"decision":"block","reason":"Explanation of why this was blocked"}'
else
  echo '{"decision":"approve"}'
fi
```

## Step 6: Test Instructions

Provide testing steps:
1. Save the configuration
2. Restart Claude Code (or start a new session)
3. Trigger the relevant event
4. Verify the hook fires correctly

Explain how to debug if the hook doesn't work:
- Check script is executable (`chmod +x`)
- Verify JSON output format
- Test the script manually: `echo '{"tool_name":"Bash","tool_input":{"command":"test"}}' | ./handler.sh`
- Check matcher regex matches the intended tools

## Rules

- Always use `${CLAUDE_PLUGIN_ROOT}` for paths in plugin hooks
- Hook scripts MUST output valid JSON to stdout
- Never print debug info to stdout (use stderr: `echo "debug" >&2`)
- Use `jq` for JSON parsing — it's reliable and available on most systems
- For cross-platform plugins, use the polyglot wrapper pattern
- Prompt hooks use API tokens — recommend `haiku` model for cost efficiency
