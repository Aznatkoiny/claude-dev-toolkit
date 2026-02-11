---
name: claude-code-troubleshooting
description: >
  Comprehensive troubleshooting guide for diagnosing and fixing Claude Code issues across installation,
  runtime, and plugin systems. Use when troubleshooting Claude Code issues, fixing installation problems,
  resolving runtime errors, debugging plugin issues, or diagnosing configuration problems. Trigger phrases:
  "troubleshoot", "not working", "error", "installation issue", "permission denied", "plugin not loading",
  "skill not triggering", "hook not firing", "MCP server error", "WSL issue", "npm error", "command not found",
  "Claude Code problem", "debug", "sandbox error", "socat", "bubblewrap", "authentication issue", "API key",
  "context window", "search not working", "IDE not detected", "JetBrains", "ripgrep", "doctor", "reset config",
  "high CPU", "memory usage", "freeze", "hang", "PATH issue", "nvm conflict", "node not found", "Git Bash".
---

# Claude Code Troubleshooting Guide

Claude Code is a powerful terminal-based AI development tool, but installation and runtime environments
vary widely across platforms and configurations. This guide provides systematic approaches to diagnosing
and resolving the most common issues users encounter with Claude Code.

The troubleshooting content is organized into three categories: installation issues (getting Claude Code
running for the first time), runtime issues (problems that occur during normal usage), and plugin issues
(problems with plugins, skills, commands, hooks, and MCP servers). Each category has a dedicated reference
file with detailed solutions.

When debugging an issue, start with the symptom-based diagnosis table below to quickly identify which
reference file contains the relevant solution. For general health checks, run `/doctor` inside Claude Code
to get an automated diagnostic report.

## When to Use This Skill

- Claude Code fails to install or the `claude` command is not found
- You encounter errors during startup or authentication
- Sandbox, networking, or permission errors appear at runtime
- Plugins, skills, commands, hooks, or MCP servers are not loading or behaving unexpectedly
- Claude Code is slow, unresponsive, or consuming excessive resources
- Search, file discovery, or IDE integration is not working correctly
- You need to reset configuration or clear cached state

## When NOT to Use This Skill

- Creating new plugins, skills, or hooks (use plugin-development, skills-authoring, or hooks-automation)
- Configuring MCP servers for the first time (use mcp-integration)
- Learning best practices for prompting or project setup (use claude-code-best-practices)
- Extending Claude Code with new capabilities (use extending-claude-code)

## Symptom-Based Diagnosis

Use this table to quickly route from a symptom to the right solution.

| Symptom | Category | Read This |
|:--------|:---------|:----------|
| `npm` errors during install, OS/platform detection failure | Installation | `references/installation-issues.md` |
| `exec: node: not found` or wrong Node.js version | Installation | `references/installation-issues.md` |
| nvm version conflicts between WSL and Windows | Installation | `references/installation-issues.md` |
| `permission denied` or `EACCES` errors during install | Installation | `references/installation-issues.md` |
| `command not found: claude` after installation | Installation | `references/installation-issues.md` |
| "Claude Code on Windows requires git-bash" | Installation | `references/installation-issues.md` |
| "installMethod is native, but claude command not found" | Installation | `references/installation-issues.md` |
| "Sandbox requires socat and bubblewrap" | Runtime | `references/runtime-issues.md` |
| "Sandboxing requires WSL2" | Runtime | `references/runtime-issues.md` |
| Authentication failures, OAuth not opening | Runtime | `references/runtime-issues.md` |
| Repeated permission prompts | Runtime | `references/runtime-issues.md` |
| High CPU or memory usage | Runtime | `references/runtime-issues.md` |
| Claude Code hangs or freezes | Runtime | `references/runtime-issues.md` |
| Search, `@file`, or custom agents/skills not working | Runtime | `references/runtime-issues.md` |
| Slow or incomplete search results on WSL | Runtime | `references/runtime-issues.md` |
| JetBrains IDE not detected on WSL2 | Runtime | `references/runtime-issues.md` |
| Escape key not working in JetBrains terminals | Runtime | `references/runtime-issues.md` |
| Missing language tags in generated markdown | Runtime | `references/runtime-issues.md` |
| Plugin not appearing after installation | Plugin | `references/plugin-issues.md` |
| Skill not triggering or command not found | Plugin | `references/plugin-issues.md` |
| Hook not firing on expected events | Plugin | `references/plugin-issues.md` |
| MCP server not starting or connection refused | Plugin | `references/plugin-issues.md` |
| "Plugin loading error" in `/plugin` Errors tab | Plugin | `references/plugin-issues.md` |

## Quick Fixes Checklist

Try these common fixes first before diving into detailed troubleshooting.

### Installation Quick Fixes

1. **Run `/doctor`** to get an automated health check of your installation
2. **Try the native installer** if npm-based installation is failing:
   ```bash
   # macOS, Linux, WSL
   curl -fsSL https://claude.ai/install.sh | bash

   # Windows PowerShell
   irm https://claude.ai/install.ps1 | iex
   ```
3. **Check your PATH** -- ensure `~/.local/bin` (or `%USERPROFILE%\.local\bin` on Windows) is in PATH:
   ```bash
   # Check if claude is reachable
   which claude

   # Verify PATH includes installation directory
   echo $PATH | tr ':' '\n' | grep local
   ```
4. **Verify Node.js version** -- Claude Code requires a compatible Node.js version:
   ```bash
   node --version
   ```
5. **On WSL**, confirm `which node` and `which npm` point to Linux paths (`/usr/` not `/mnt/c/`)

### Runtime Quick Fixes

1. **Restart Claude Code** -- close the terminal and open a fresh session
2. **Use `/compact`** to reduce context size if responses are degrading
3. **Press Ctrl+C** if Claude Code appears frozen
4. **Re-authenticate** by running `/logout`, closing Claude Code, then restarting with `claude`
5. **Install ripgrep** if search features are broken:
   ```bash
   # macOS
   brew install ripgrep

   # Ubuntu/Debian
   sudo apt install ripgrep

   # Windows
   winget install BurntSushi.ripgrep.MSVC
   ```
6. **Install sandbox dependencies** on WSL2 if you see sandbox errors:
   ```bash
   sudo apt-get install bubblewrap socat
   ```

### Plugin Quick Fixes

1. **Restart Claude Code** after making any plugin changes
2. **Check `/plugin` Errors tab** for loading issues
3. **Verify directory structure** -- component directories go at plugin root, NOT inside `.claude-plugin/`
4. **Check `plugin.json`** for valid JSON and required fields (`name`, `description`, `version`)
5. **Test with `--plugin-dir`** to load local plugins: `claude --plugin-dir ./my-plugin`

## General Debugging Approach

Follow this systematic process when troubleshooting any Claude Code issue.

### Step 1: Identify the Category

Determine whether the issue is an installation, runtime, or plugin problem:

- **Installation**: Claude Code won't install, the `claude` command doesn't exist, or it crashes on first launch
- **Runtime**: Claude Code launches but encounters errors during use (auth, sandbox, performance, search, IDE)
- **Plugin**: A specific plugin component (skill, command, hook, MCP server) isn't working as expected

### Step 2: Run Built-in Diagnostics

```bash
claude doctor
```

The `/doctor` command checks:
- Installation type, version, and search functionality
- Auto-update status and available versions
- Invalid settings files (malformed JSON, incorrect types)
- MCP server configuration errors
- Keybinding configuration problems
- Context usage warnings (large CLAUDE.md files, high MCP token usage, unreachable permission rules)
- Plugin and agent loading errors

### Step 3: Check Configuration Files

Claude Code stores configuration in several locations:

| File | Purpose |
|:-----|:--------|
| `~/.claude/settings.json` | User settings (permissions, hooks, model overrides) |
| `.claude/settings.json` | Project settings (checked into source control) |
| `.claude/settings.local.json` | Local project settings (not committed) |
| `~/.claude.json` | Global state (theme, OAuth, MCP servers) |
| `.mcp.json` | Project MCP servers (checked into source control) |

Managed file locations by platform:
- **macOS**: `/Library/Application Support/ClaudeCode/`
- **Linux/WSL**: `/etc/claude-code/`
- **Windows**: `C:\Program Files\ClaudeCode\`

### Step 4: Reset if Needed

If configuration is corrupted, you can reset to defaults:

```bash
# Reset all user settings and state
rm ~/.claude.json
rm -rf ~/.claude/

# Reset project-specific settings
rm -rf .claude/
rm .mcp.json
```

**Warning**: This removes all settings, MCP server configurations, and session history.

### Step 5: Collect Diagnostic Information

Before seeking additional help, gather the following information:

```bash
# Check Claude Code version
claude --version

# Run health check
claude doctor

# Check Node.js version (if using npm installation)
node --version
npm --version

# Check platform info
uname -a
```

For plugin issues, also check:

```bash
# Validate plugin JSON
jq . /path/to/plugin/.claude-plugin/plugin.json

# Check file permissions
ls -la /path/to/plugin/
```

### Step 6: Get More Help

If the issue persists after troubleshooting:

1. Use `/bug` within Claude Code to report problems directly to Anthropic
2. Check the [GitHub repository](https://github.com/anthropics/claude-code) for known issues
3. Ask Claude directly about its capabilities -- Claude has built-in access to its documentation
4. When filing an issue, include: your platform, Claude Code version, Node.js version, error messages,
   and the output of `claude doctor`

## Platform-Specific Notes

Different platforms have different common issues. Here is a quick guide to platform-specific concerns.

### Windows (Native)

- Requires Git for Windows (Git Bash)
- PATH must include `%USERPROFILE%\.local\bin`
- Use PowerShell for the native installer
- Set `CLAUDE_CODE_GIT_BASH_PATH` if Git Bash is not auto-detected

### Windows (WSL)

- Verify `which node` and `which npm` point to Linux paths, not `/mnt/c/`
- Load nvm properly in shell config for non-interactive shells
- Install `bubblewrap` and `socat` for sandbox support (WSL2 only)
- Keep projects on the Linux filesystem (`/home/`) for best search performance
- WSL1 does not support sandboxing

### macOS

- Use `brew install ripgrep` if search is not working
- Native installer places binary at `~/.local/bin/claude`
- Add `~/.local/bin` to PATH in `~/.zshrc` if needed

### Linux

- Native installer is recommended over npm for permission issues
- Install ripgrep from your package manager if search is broken
- Managed settings are stored in `/etc/claude-code/`

## Reference File Index

| File | Contents |
|:-----|:---------|
| `references/installation-issues.md` | WSL npm detection, Node.js version conflicts, nvm conflicts, native installer, PATH configuration, Windows Git Bash requirement, Windows PATH issues, Linux/Mac permission errors |
| `references/runtime-issues.md` | WSL2 sandbox setup, authentication issues, permission prompts, performance/stability, search and discovery issues, IDE integration, markdown formatting |
| `references/plugin-issues.md` | Plugin not loading, skill not triggering, command not appearing, hook not firing, MCP server not starting, directory structure errors, debugging techniques for each component type |
