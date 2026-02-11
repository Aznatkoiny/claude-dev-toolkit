# Installation Issues

Detailed troubleshooting for Claude Code installation problems across all platforms.

## Windows Installation Issues: Errors in WSL

You might encounter the following issues when installing Claude Code in WSL.

### OS/Platform Detection Issues

If you receive an error during installation, WSL may be using Windows `npm`. Try:

- Run `npm config set os linux` before installation
- Install with `npm install -g @anthropic-ai/claude-code --force --no-os-check` (Do NOT use `sudo`)

### Node Not Found Errors

If you see `exec: node: not found` when running `claude`, your WSL environment may be using a Windows
installation of Node.js. You can confirm this with `which npm` and `which node`, which should point to
Linux paths starting with `/usr/` rather than `/mnt/c/`.

To fix this, install Node via your Linux distribution's package manager or via nvm
(https://github.com/nvm-sh/nvm).

### nvm Version Conflicts

If you have nvm installed in both WSL and Windows, you may experience version conflicts when switching
Node versions in WSL. This happens because WSL imports the Windows PATH by default, causing Windows
nvm/npm to take priority over the WSL installation.

**How to identify this issue:**

- Running `which npm` and `which node` -- if they point to Windows paths (starting with `/mnt/c/`),
  Windows versions are being used
- Experiencing broken functionality after switching Node versions with nvm in WSL

**Primary solution: Ensure nvm is properly loaded in your shell**

The most common cause is that nvm isn't loaded in non-interactive shells. Add the following to your
shell configuration file (`~/.bashrc`, `~/.zshrc`, etc.):

```bash
# Load nvm if it exists
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

Or run directly in your current session:

```bash
source ~/.nvm/nvm.sh
```

**Alternative: Adjust PATH order**

If nvm is properly loaded but Windows paths still take priority, you can explicitly prepend your Linux
paths to PATH in your shell configuration:

```bash
export PATH="$HOME/.nvm/versions/node/$(node -v)/bin:$PATH"
```

**Important**: Avoid disabling Windows PATH importing (`appendWindowsPath = false`) as this breaks the
ability to call Windows executables from WSL. Similarly, avoid uninstalling Node.js from Windows if you
use it for Windows development.

## Windows: "Claude Code on Windows requires git-bash"

Claude Code on native Windows requires Git for Windows (https://git-scm.com/downloads/win) which
includes Git Bash. If Git is installed but not detected:

1. Set the path explicitly in PowerShell before running Claude:
   ```powershell
   $env:CLAUDE_CODE_GIT_BASH_PATH="C:\Program Files\Git\bin\bash.exe"
   ```

2. Or add it to your system environment variables permanently through System Properties, then
   Environment Variables.

If Git is installed in a non-standard location, adjust the path accordingly.

## Windows: "installMethod is native, but claude command not found"

If you see this error after installation, the `claude` command isn't in your PATH. Fix it:

1. **Open Environment Variables**: Press `Win + R`, type `sysdm.cpl`, and press Enter. Click
   **Advanced**, then **Environment Variables**.

2. **Edit User PATH**: Under "User variables", select **Path** and click **Edit**. Click **New**
   and add:
   ```
   %USERPROFILE%\.local\bin
   ```

3. **Restart your terminal**: Close and reopen PowerShell or CMD for changes to take effect.

Verify installation:

```bash
claude doctor
```

## Linux and Mac: Permission or Command Not Found Errors

When installing Claude Code with npm, `PATH` problems may prevent access to `claude`. You may also
encounter permission errors if your npm global prefix is not user writable (for example, `/usr` or
`/usr/local`).

### Recommended Solution: Native Claude Code Installation

Claude Code has a native installation that doesn't depend on npm or Node.js. Use the following command
to run the native installer.

**macOS, Linux, WSL:**

```bash
# Install stable version (default)
curl -fsSL https://claude.ai/install.sh | bash

# Install latest version
curl -fsSL https://claude.ai/install.sh | bash -s latest

# Install specific version number
curl -fsSL https://claude.ai/install.sh | bash -s 1.0.58
```

**Windows PowerShell:**

```powershell
# Install stable version (default)
irm https://claude.ai/install.ps1 | iex

# Install latest version
& ([scriptblock]::Create((irm https://claude.ai/install.ps1))) latest

# Install specific version number
& ([scriptblock]::Create((irm https://claude.ai/install.ps1))) 1.0.58
```

This installs the appropriate build of Claude Code for your operating system and architecture and adds
a symlink to the installation at `~/.local/bin/claude` (or `%USERPROFILE%\.local\bin\claude.exe` on
Windows).

**Tip**: Make sure that you have the installation directory in your system PATH.

### Verifying PATH Configuration

After installation, verify that the Claude Code binary is accessible:

```bash
# Check if claude is in PATH
which claude

# Check PATH includes the installation directory
echo $PATH | tr ':' '\n' | grep -i local

# On macOS/Linux, ensure ~/.local/bin is in PATH
# Add to ~/.bashrc, ~/.zshrc, or ~/.profile:
export PATH="$HOME/.local/bin:$PATH"
```

### npm-Specific PATH Fixes

If using the npm installation method, check your npm global prefix:

```bash
# Check npm global prefix
npm config get prefix

# If prefix is /usr or /usr/local, change to user directory
npm config set prefix ~/.npm-global

# Add to PATH
export PATH="$HOME/.npm-global/bin:$PATH"
```

## Platform-Specific Summary

| Platform | Common Issue | Quick Fix |
|:---------|:-------------|:----------|
| WSL | Windows npm used instead of Linux npm | `npm config set os linux` then reinstall |
| WSL | Node not found | Install Node via Linux package manager or nvm |
| WSL | nvm conflicts with Windows | Ensure nvm loads in shell config, check `which node` |
| Windows | Git Bash not detected | Set `CLAUDE_CODE_GIT_BASH_PATH` environment variable |
| Windows | `claude` command not found after install | Add `%USERPROFILE%\.local\bin` to PATH |
| macOS | Permission denied on npm install | Use native installer: `curl -fsSL https://claude.ai/install.sh \| bash` |
| Linux | Permission denied on npm install | Use native installer or fix npm prefix |
| All | npm install fails entirely | Use the native installer (recommended) |
