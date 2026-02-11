# Plugin Manifest Reference

The plugin manifest is the `plugin.json` file located at `.claude-plugin/plugin.json` within your
plugin directory. It defines your plugin's identity, metadata, and version information.

## Manifest Location

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json    <-- The manifest file
├── commands/
├── skills/
└── ...
```

The `.claude-plugin/` directory must contain ONLY `plugin.json`. Do not place any other files
or directories inside `.claude-plugin/`.

## Required Fields

| Field | Type | Description |
|:------|:-----|:------------|
| `name` | string | Unique identifier for the plugin. Becomes the namespace prefix for all skills (e.g., a plugin named `my-tools` creates commands like `/my-tools:hello`). Must be URL-safe. |
| `description` | string | Human-readable description shown in the plugin manager when browsing or installing plugins. |
| `version` | string | Version string following semantic versioning (`MAJOR.MINOR.PATCH`). |

## Optional Fields

| Field | Type | Description |
|:------|:-----|:------------|
| `author` | object | Attribution information. Contains `name` (string). |
| `homepage` | string | URL to the plugin's homepage or documentation. |
| `repository` | string | URL to the plugin's source code repository. |
| `license` | string | SPDX license identifier (e.g., `"MIT"`, `"Apache-2.0"`). |

## Complete Example

```json
{
  "name": "code-quality",
  "description": "Code review and quality tools for Python and TypeScript projects",
  "version": "2.1.0",
  "author": {
    "name": "Dev Team"
  },
  "homepage": "https://github.com/example/code-quality-plugin",
  "repository": "https://github.com/example/code-quality-plugin",
  "license": "MIT"
}
```

## Minimal Example

```json
{
  "name": "my-plugin",
  "description": "A greeting plugin to learn the basics",
  "version": "1.0.0"
}
```

## Version Conventions

Use semantic versioning for the `version` field:

- **MAJOR** (e.g., `2.0.0`): Breaking changes that require users to update their workflows
- **MINOR** (e.g., `1.1.0`): New features added in a backward-compatible manner
- **PATCH** (e.g., `1.0.1`): Backward-compatible bug fixes

Start with `1.0.0` for your initial release. Increment appropriately as you make changes.

## Name as Namespace

The `name` field serves dual purpose:

1. **Identifier**: Used to reference the plugin in installation commands and marketplace listings
2. **Namespace**: All skills defined in the plugin are prefixed with this name

For example, with `"name": "my-tools"`:
- A command file at `commands/review.md` becomes `/my-tools:review`
- A skill at `skills/lint/SKILL.md` is namespaced under `my-tools`

Choose a name that is:
- Descriptive of the plugin's purpose
- URL-safe (no spaces or special characters)
- Unlikely to conflict with other plugins

## Marketplace Manifest (marketplace.json)

When distributing plugins through a marketplace, the marketplace itself needs a manifest file
at `.claude-plugin/marketplace.json`. This file catalogs all plugins available in the marketplace.

### marketplace.json Structure

The `marketplace.json` file lists all plugins available in the marketplace. Each entry points
to a plugin directory within the repository.

```json
{
  "plugins": [
    {
      "name": "commit-commands",
      "description": "Git commit workflows including commit, push, and PR creation",
      "path": "./plugins/commit-commands"
    },
    {
      "name": "code-review",
      "description": "Code review tools and agents",
      "path": "./plugins/code-review"
    }
  ]
}
```

### marketplace.json Fields

| Field | Type | Description |
|:------|:-----|:------------|
| `plugins` | array | List of plugin entries |
| `plugins[].name` | string | Plugin name (must match the `name` in the plugin's `plugin.json`) |
| `plugins[].description` | string | Short description shown when browsing the marketplace |
| `plugins[].path` | string | Relative path from the marketplace root to the plugin directory |

### Marketplace Location

The `marketplace.json` must be located at `.claude-plugin/marketplace.json` in the repository root:

```
my-marketplace-repo/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   ├── plugin-one/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   └── commands/
│   └── plugin-two/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
```

Each plugin listed in `marketplace.json` must have its own `.claude-plugin/plugin.json` at the
specified path.
