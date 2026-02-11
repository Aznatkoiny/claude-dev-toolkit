# Marketplace Distribution Reference

Plugin marketplaces are catalogs that help users discover and install plugins. This reference
covers how marketplaces work, how to create one, publishing plugins, and team distribution.

## How Marketplaces Work

A marketplace is a catalog of plugins that someone has created and shared. Using a marketplace
is a two-step process:

1. **Add the marketplace**: registers the catalog with Claude Code so you can browse what is
   available. No plugins are installed yet.
2. **Install individual plugins**: browse the catalog and install the plugins you want.

Think of it like adding an app store: adding the store gives you access to browse its collection,
but you still choose which apps to download individually.

## Official Anthropic Marketplace

The official Anthropic marketplace (`claude-plugins-official`) is automatically available when
you start Claude Code. Run `/plugin` and go to the **Discover** tab to browse available plugins.

Install a plugin from the official marketplace:

```shell
/plugin install plugin-name@claude-plugins-official
```

### Official Plugin Categories

**Code intelligence** (LSP plugins):

| Language | Plugin | Binary Required |
|:---------|:-------|:----------------|
| C/C++ | `clangd-lsp` | `clangd` |
| C# | `csharp-lsp` | `csharp-ls` |
| Go | `gopls-lsp` | `gopls` |
| Java | `jdtls-lsp` | `jdtls` |
| Kotlin | `kotlin-lsp` | `kotlin-language-server` |
| Lua | `lua-lsp` | `lua-language-server` |
| PHP | `php-lsp` | `intelephense` |
| Python | `pyright-lsp` | `pyright-langserver` |
| Rust | `rust-analyzer-lsp` | `rust-analyzer` |
| Swift | `swift-lsp` | `sourcekit-lsp` |
| TypeScript | `typescript-lsp` | `typescript-language-server` |

**External integrations** (pre-configured MCP servers):
- Source control: `github`, `gitlab`
- Project management: `atlassian` (Jira/Confluence), `asana`, `linear`, `notion`
- Design: `figma`
- Infrastructure: `vercel`, `firebase`, `supabase`
- Communication: `slack`
- Monitoring: `sentry`

**Development workflows**:
- `commit-commands`: Git commit workflows
- `pr-review-toolkit`: PR review agents
- `agent-sdk-dev`: Claude Agent SDK tools
- `plugin-dev`: Plugin creation toolkit

**Output styles**:
- `explanatory-output-style`: Educational insights
- `learning-output-style`: Interactive learning mode

## Adding Marketplaces

### From GitHub

Add a GitHub repository containing `.claude-plugin/marketplace.json`:

```shell
/plugin marketplace add owner/repo
```

Example:

```shell
/plugin marketplace add anthropics/claude-code
```

### From Other Git Hosts

Add any git repository by providing the full URL (GitLab, Bitbucket, self-hosted):

Using HTTPS:

```shell
/plugin marketplace add https://gitlab.com/company/plugins.git
```

Using SSH:

```shell
/plugin marketplace add git@gitlab.com:company/plugins.git
```

Add a specific branch or tag with `#`:

```shell
/plugin marketplace add https://gitlab.com/company/plugins.git#v1.0.0
```

### From Local Paths

Add a local directory containing `.claude-plugin/marketplace.json`:

```shell
/plugin marketplace add ./my-marketplace
```

Or add a direct path to a `marketplace.json` file:

```shell
/plugin marketplace add ./path/to/marketplace.json
```

### From Remote URLs

Add a remote `marketplace.json` file via URL:

```shell
/plugin marketplace add https://example.com/marketplace.json
```

**Note**: URL-based marketplaces have some limitations compared to Git-based marketplaces.
Plugins with relative paths may fail in URL-based marketplaces.

**Shortcut**: You can use `/plugin market` instead of `/plugin marketplace`, and `rm` instead
of `remove`.

## Installing Plugins

Install a plugin (defaults to user scope):

```shell
/plugin install plugin-name@marketplace-name
```

To choose a different installation scope, use the interactive UI: run `/plugin`, go to the
**Discover** tab, and press Enter on a plugin. Options:

- **User scope** (default): install for yourself across all projects
- **Project scope**: install for all collaborators on this repository (adds to `.claude/settings.json`)
- **Local scope**: install for yourself in this repository only

You may also see plugins with **managed** scope -- installed by administrators via managed
settings and cannot be modified.

Install with a specific scope from CLI:

```shell
claude plugin install formatter@your-org --scope project
claude plugin uninstall formatter@your-org --scope project
```

**Important**: Make sure you trust a plugin before installing it. Anthropic does not control
what MCP servers, files, or other software are included in plugins and cannot verify that they
work as intended.

## Managing Installed Plugins

Run `/plugin` and go to the **Installed** tab to view, enable, disable, or uninstall plugins.

Direct commands:

```shell
# Disable without uninstalling
/plugin disable plugin-name@marketplace-name

# Re-enable
/plugin enable plugin-name@marketplace-name

# Remove completely
/plugin uninstall plugin-name@marketplace-name
```

## Managing Marketplaces

### Interactive Interface

Run `/plugin` and go to the **Marketplaces** tab to:
- View all added marketplaces with sources and status
- Add new marketplaces
- Update marketplace listings to fetch latest plugins
- Remove marketplaces

### CLI Commands

```shell
# List all configured marketplaces
/plugin marketplace list

# Refresh plugin listings
/plugin marketplace update marketplace-name

# Remove a marketplace (also uninstalls its plugins)
/plugin marketplace remove marketplace-name
```

## Auto-Updates

Claude Code can automatically update marketplaces and their installed plugins at startup.

Toggle auto-update for individual marketplaces:
1. Run `/plugin` to open the plugin manager
2. Select **Marketplaces**
3. Choose a marketplace
4. Select **Enable auto-update** or **Disable auto-update**

**Defaults**:
- Official Anthropic marketplaces: auto-update enabled
- Third-party and local development marketplaces: auto-update disabled

**Environment variables**:

```shell
# Disable all automatic updates (Claude Code + plugins)
export DISABLE_AUTOUPDATER=true

# Keep plugin auto-updates while disabling Claude Code auto-updates
export DISABLE_AUTOUPDATER=true
export FORCE_AUTOUPDATE_PLUGINS=true
```

## Creating a Marketplace

To distribute your own plugins, create a repository with this structure:

```
my-marketplace/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    ├── plugin-one/
    │   ├── .claude-plugin/
    │   │   └── plugin.json
    │   └── commands/
    └── plugin-two/
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
```

The `marketplace.json` catalogs all available plugins:

```json
{
  "plugins": [
    {
      "name": "plugin-one",
      "description": "Description of plugin one",
      "path": "./plugins/plugin-one"
    },
    {
      "name": "plugin-two",
      "description": "Description of plugin two",
      "path": "./plugins/plugin-two"
    }
  ]
}
```

Each plugin listed must have its own `.claude-plugin/plugin.json` at the specified path.

## Team and Private Distribution

### Configure Team Marketplaces

Team admins can set up automatic marketplace installation for projects by adding marketplace
configuration to `.claude/settings.json`. When team members trust the repository folder,
Claude Code prompts them to install these marketplaces and plugins.

Use `extraKnownMarketplaces` and `enabledPlugins` in the project settings to configure
which marketplaces and plugins are available to the team.

### Private Repositories

For private or internal distribution:
- Use SSH URLs: `git@gitlab.com:company/plugins.git`
- Users must have repository access
- Works with any Git host that supports SSH or HTTPS authentication

### Distribution Checklist

1. Add documentation: include a `README.md` with installation and usage instructions
2. Version your plugin using semantic versioning in `plugin.json`
3. Create the `marketplace.json` with entries for all plugins
4. Push to a Git repository (GitHub, GitLab, Bitbucket, or self-hosted)
5. Share the marketplace add command with users
6. Have team members test before wider distribution

## Troubleshooting

### /plugin command not recognized

1. Check version: `claude --version` (requires 1.0.33 or later)
2. Update Claude Code:
   - Homebrew: `brew upgrade claude-code`
   - npm: `npm update -g @anthropic-ai/claude-code`
   - Native installer: re-run the install command
3. Restart Claude Code

### Common Issues

- **Marketplace not loading**: verify the URL is accessible and `.claude-plugin/marketplace.json` exists
- **Plugin installation failures**: check that plugin source URLs are accessible and repositories are public (or you have access)
- **Files not found after installation**: plugins are copied to a cache; paths referencing files outside the plugin directory will not work
- **Plugin skills not appearing**: clear the cache with `rm -rf ~/.claude/plugins/cache`, restart Claude Code, and reinstall
- **Code intelligence binary not found**: install the required language server binary and ensure it is in your `$PATH`
- **High memory usage from LSP**: disable the plugin with `/plugin disable <plugin-name>` and use built-in search tools instead
