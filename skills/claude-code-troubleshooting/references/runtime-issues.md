# Runtime Issues

Detailed troubleshooting for issues that occur during Claude Code usage after successful installation.

## WSL2 Sandbox Setup

Sandboxing is supported on WSL2 but requires installing additional packages. If you see an error like
"Sandbox requires socat and bubblewrap" when running `/sandbox`, install the dependencies:

**Ubuntu/Debian:**

```bash
sudo apt-get install bubblewrap socat
```

**Fedora:**

```bash
sudo dnf install bubblewrap socat
```

WSL1 does not support sandboxing. If you see "Sandboxing requires WSL2", you need to upgrade to WSL2
or run Claude Code without sandboxing.

## Permissions and Authentication

### Repeated Permission Prompts

If you find yourself repeatedly approving the same commands, you can allow specific tools to run without
approval using the `/permissions` command. See the Permissions documentation for details on managing
permissions.

### Authentication Issues

If you're experiencing authentication problems:

1. Run `/logout` to sign out completely
2. Close Claude Code
3. Restart with `claude` and complete the authentication process again

If the browser doesn't open automatically during login, press `c` to copy the OAuth URL to your
clipboard, then paste it into your browser manually.

If problems persist, try removing stored authentication and forcing a clean login:

```bash
rm -rf ~/.config/claude-code/auth.json
claude
```

## Configuration File Locations

Claude Code stores configuration in several locations:

| File | Purpose |
|:-----|:--------|
| `~/.claude/settings.json` | User settings (permissions, hooks, model overrides) |
| `.claude/settings.json` | Project settings (checked into source control) |
| `.claude/settings.local.json` | Local project settings (not committed) |
| `~/.claude.json` | Global state (theme, OAuth, MCP servers) |
| `.mcp.json` | Project MCP servers (checked into source control) |
| `managed-settings.json` | Managed settings |
| `managed-mcp.json` | Managed MCP servers |

On Windows, `~` refers to your user home directory, such as `C:\Users\YourName`.

**Managed file locations:**

- macOS: `/Library/Application Support/ClaudeCode/`
- Linux/WSL: `/etc/claude-code/`
- Windows: `C:\Program Files\ClaudeCode\`

### Resetting Configuration

To reset Claude Code to default settings, you can remove the configuration files:

```bash
# Reset all user settings and state
rm ~/.claude.json
rm -rf ~/.claude/

# Reset project-specific settings
rm -rf .claude/
rm .mcp.json
```

**Warning**: This will remove all your settings, MCP server configurations, and session history.

## Performance and Stability

### High CPU or Memory Usage

Claude Code is designed to work with most development environments, but may consume significant resources
when processing large codebases. If you're experiencing performance issues:

1. Use `/compact` regularly to reduce context size
2. Close and restart Claude Code between major tasks
3. Consider adding large build directories to your `.gitignore` file

### Command Hangs or Freezes

If Claude Code seems unresponsive:

1. Press Ctrl+C to attempt to cancel the current operation
2. If unresponsive, you may need to close the terminal and restart

### Context Window Limits

If Claude Code's responses are degrading in quality during a long session:

1. Use `/compact` to summarize and compress the conversation context
2. Start a new session for unrelated tasks
3. Break large tasks into smaller, focused sessions
4. Avoid pasting extremely large files directly into the conversation

## Search and Discovery Issues

If Search tool, `@file` mentions, custom agents, and custom skills aren't working, install system
ripgrep:

```bash
# macOS (Homebrew)
brew install ripgrep

# Windows (winget)
winget install BurntSushi.ripgrep.MSVC

# Ubuntu/Debian
sudo apt install ripgrep

# Alpine Linux
apk add ripgrep

# Arch Linux
pacman -S ripgrep
```

Then set `USE_BUILTIN_RIPGREP=0` in your environment.

### Slow or Incomplete Search Results on WSL

Disk read performance penalties when working across file systems on WSL may result in
fewer-than-expected matches (but not a complete lack of search functionality) when using Claude Code
on WSL.

**Note**: `/doctor` will show Search as OK in this case.

**Solutions:**

1. **Submit more specific searches**: Reduce the number of files searched by specifying directories or
   file types: "Search for JWT validation logic in the auth-service package" or "Find use of md5 hash
   in JS files".

2. **Move project to Linux filesystem**: If possible, ensure your project is located on the Linux
   filesystem (`/home/`) rather than the Windows filesystem (`/mnt/c/`).

3. **Use native Windows instead**: Consider running Claude Code natively on Windows instead of through
   WSL, for better file system performance.

## IDE Integration Issues

### JetBrains IDE Not Detected on WSL2

If you're using Claude Code on WSL2 with JetBrains IDEs and getting "No available IDEs detected" errors,
this is likely due to WSL2's networking configuration or Windows Firewall blocking the connection.

#### WSL2 Networking Modes

WSL2 uses NAT networking by default, which can prevent IDE detection. You have two options:

**Option 1: Configure Windows Firewall (recommended)**

1. Find your WSL2 IP address:
   ```bash
   wsl hostname -I
   # Example output: 172.21.123.456
   ```

2. Open PowerShell as Administrator and create a firewall rule:
   ```powershell
   New-NetFirewallRule -DisplayName "Allow WSL2 Internal Traffic" -Direction Inbound -Protocol TCP -Action Allow -RemoteAddress 172.21.0.0/16 -LocalAddress 172.21.0.0/16
   ```
   (Adjust the IP range based on your WSL2 subnet from step 1)

3. Restart both your IDE and Claude Code

**Option 2: Switch to mirrored networking**

Add to `.wslconfig` in your Windows user directory:

```ini
[wsl2]
networkingMode=mirrored
```

Then restart WSL with `wsl --shutdown` from PowerShell.

**Note**: These networking issues only affect WSL2. WSL1 uses the host's network directly and doesn't
require these configurations.

### Reporting Windows IDE Integration Issues

If you're experiencing IDE integration problems on Windows, create an issue at
https://github.com/anthropics/claude-code/issues with the following information:

- Environment type: native Windows (Git Bash) or WSL1/WSL2
- WSL networking mode (if applicable): NAT or mirrored
- IDE name and version
- Claude Code extension/plugin version
- Shell type: Bash, Zsh, PowerShell, etc.

### Escape Key Not Working in JetBrains Terminals

If you're using Claude Code in JetBrains terminals (IntelliJ, PyCharm, etc.) and the `Esc` key doesn't
interrupt the agent as expected, this is likely due to a keybinding clash with JetBrains' default
shortcuts.

To fix this issue:

1. Go to Settings, then Tools, then Terminal
2. Either:
   - Uncheck "Move focus to the editor with Escape", or
   - Click "Configure terminal keybindings" and delete the "Switch focus to Editor" shortcut
3. Apply the changes

This allows the `Esc` key to properly interrupt Claude Code operations.

## Markdown Formatting Issues

### Missing Language Tags in Code Blocks

If you notice code blocks in generated markdown without language tags (bare triple backticks instead
of tagged blocks like ````javascript`):

**Solutions:**

1. **Ask Claude to add language tags**: Request "Add appropriate language tags to all code blocks in
   this markdown file."

2. **Use post-processing hooks**: Set up automatic formatting hooks to detect and add missing language
   tags. See the hooks guide for an example of a PostToolUse formatting hook.

3. **Manual verification**: After generating markdown files, review them for proper code block
   formatting and request corrections if needed.

### Inconsistent Spacing and Formatting

If generated markdown has excessive blank lines or inconsistent spacing:

**Solutions:**

1. **Request formatting corrections**: Ask Claude to "Fix spacing and formatting issues in this
   markdown file."

2. **Use formatting tools**: Set up hooks to run markdown formatters like `prettier` or custom
   formatting scripts on generated markdown files.

3. **Specify formatting preferences**: Include formatting requirements in your prompts or project
   memory files.

### Best Practices for Markdown Generation

To minimize formatting issues:

- **Be explicit in requests**: Ask for "properly formatted markdown with language-tagged code blocks"
- **Use project conventions**: Document your preferred markdown style in `CLAUDE.md`
- **Set up validation hooks**: Use post-processing hooks to automatically verify and fix common
  formatting issues

## Network and Proxy Issues

If Claude Code cannot connect to the Anthropic API:

1. **Check your internet connection**: Verify basic connectivity with `curl https://api.anthropic.com`
2. **Proxy configuration**: If behind a corporate proxy, set the standard proxy environment variables:
   ```bash
   export HTTP_PROXY=http://proxy.example.com:8080
   export HTTPS_PROXY=http://proxy.example.com:8080
   export NO_PROXY=localhost,127.0.0.1
   ```
3. **Firewall rules**: Ensure outbound HTTPS traffic to `api.anthropic.com` is allowed
4. **VPN interference**: Some VPNs may interfere with Claude Code's API connections. Try disconnecting
   your VPN temporarily to test

## API Key Issues

If you are using an API key (rather than OAuth) and encountering errors:

1. **Verify the key is valid**: Check that the key starts with the expected prefix and hasn't expired
2. **Check environment variable**: Ensure `ANTHROPIC_API_KEY` is set correctly:
   ```bash
   echo $ANTHROPIC_API_KEY
   ```
3. **Re-export the key**: If the key isn't persisting across sessions, add it to your shell profile:
   ```bash
   export ANTHROPIC_API_KEY="your-key-here"
   ```
4. **Check for whitespace**: Ensure there are no leading or trailing spaces in the key value

## ESM Module Errors

If you encounter errors related to ES modules (like "ERR_REQUIRE_ESM" or "Must use import to load
ES Module"):

1. **Update Node.js**: Ensure you are running a supported Node.js version
2. **Clear npm cache**: Run `npm cache clean --force` and reinstall
3. **Try the native installer**: The native installation avoids Node.js module resolution issues:
   ```bash
   curl -fsSL https://claude.ai/install.sh | bash
   ```

## Getting More Help

If you're experiencing issues not covered here:

1. Use the `/bug` command within Claude Code to report problems directly to Anthropic
2. Check the GitHub repository (https://github.com/anthropics/claude-code) for known issues
3. Run `/doctor` to diagnose issues -- it checks installation, settings, MCP servers, keybindings,
   context usage, and plugin loading
4. Ask Claude directly about its capabilities and features -- Claude has built-in access to its
   documentation
