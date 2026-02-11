# Supporting Files

Skills can include multiple files in their directory beyond the required `SKILL.md`. This keeps the main file
focused on essentials while letting Claude access detailed reference material only when needed. Large reference
docs, API specifications, or example collections don't need to load into context every time the skill runs.

## Directory Structure Patterns

Each skill is a directory with `SKILL.md` as the entrypoint. Other files are optional.

### Basic skill (no supporting files)

```
my-skill/
└── SKILL.md           # Main instructions (required)
```

### Skill with reference material

```
my-skill/
├── SKILL.md           # Main instructions (required)
├── reference.md       # Detailed API docs - loaded when needed
└── examples.md        # Usage examples - loaded when needed
```

### Skill with templates and scripts

```
my-skill/
├── SKILL.md           # Main instructions (required)
├── template.md        # Template for Claude to fill in
├── examples/
│   └── sample.md      # Example output showing expected format
└── scripts/
    └── validate.sh    # Script Claude can execute
```

### Skill with detailed reference hierarchy

```
my-skill/
├── SKILL.md              # Overview and navigation
├── references/
│   ├── api-spec.md       # Complete API specification
│   ├── error-codes.md    # Error code reference
│   └── schemas.md        # Data schemas
├── templates/
│   ├── endpoint.md       # Template for new endpoints
│   └── test.md           # Template for tests
└── scripts/
    └── helper.py         # Utility script
```

## Referencing Supporting Files

Reference supporting files from `SKILL.md` so Claude knows what each file contains and when to load it:

```markdown
## Additional resources

- For complete API details, see [reference.md](reference.md)
- For usage examples, see [examples.md](examples.md)
```

Best practice: Keep `SKILL.md` under 500 lines. Move detailed reference material to separate files.

## Dynamic Context Injection

The `!`command`` syntax runs shell commands before the skill content is sent to Claude. The command output replaces
the placeholder, so Claude receives actual data, not the command itself.

### How it works

1. Each `!`command`` executes immediately (before Claude sees anything)
2. The output replaces the placeholder in the skill content
3. Claude receives the fully-rendered prompt with actual data

This is preprocessing, not something Claude executes. Claude only sees the final result.

### Example: Pull request summary

This skill fetches live PR data with the GitHub CLI:

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request...
```

When this skill runs, each `!`command`` is replaced with the command's output before Claude sees the prompt.

### Tips for dynamic context

- Use `context: fork` with dynamic context so the injected data scopes to a subagent
- Combine with `agent: Explore` for read-only analysis of injected data
- Commands run in the current working directory
- Failed commands inject their error output, so handle accordingly

## String Substitutions

Skills support string substitution for dynamic values:

| Variable               | Description                                                                                                                                  |
| :--------------------- | :------------------------------------------------------------------------------------------------------------------------------------------- |
| `$ARGUMENTS`           | All arguments passed when invoking the skill. If not present in content, arguments are appended as `ARGUMENTS: <value>`.                     |
| `$ARGUMENTS[N]`        | Access a specific argument by 0-based index, such as `$ARGUMENTS[0]` for the first argument.                                                 |
| `$N`                   | Shorthand for `$ARGUMENTS[N]`, such as `$0` for the first argument or `$1` for the second.                                                   |
| `${CLAUDE_SESSION_ID}` | The current session ID. Useful for logging, creating session-specific files, or correlating skill output with sessions.                      |

## Extended Example: Codebase Visualizer

This example shows a full skill that bundles and runs a Python script to generate interactive HTML output.

### Directory structure

```
codebase-visualizer/
├── SKILL.md
└── scripts/
    └── visualize.py
```

### SKILL.md

````yaml
---
name: codebase-visualizer
description: Generate an interactive collapsible tree visualization of your codebase. Use when exploring a new repo, understanding project structure, or identifying large files.
allowed-tools: Bash(python *)
---

# Codebase Visualizer

Generate an interactive HTML tree view that shows your project's file structure with collapsible directories.

## Usage

Run the visualization script from your project root:

```bash
python ~/.claude/skills/codebase-visualizer/scripts/visualize.py .
```

This creates `codebase-map.html` in the current directory and opens it in your default browser.

## What the visualization shows

- **Collapsible directories**: Click folders to expand/collapse
- **File sizes**: Displayed next to each file
- **Colors**: Different colors for different file types
- **Directory totals**: Shows aggregate size of each folder
````

### scripts/visualize.py

The script scans a directory tree and generates a self-contained HTML file with:

- A summary sidebar showing file count, directory count, total size, and number of file types
- A bar chart breaking down the codebase by file type (top 8 by size)
- A collapsible tree where you can expand and collapse directories, with color-coded file type indicators

The script requires Python but uses only built-in libraries, so there are no packages to install. See the source
documentation for the full script implementation. The key pattern is:

1. Scan the directory tree, collecting stats
2. Generate self-contained HTML with embedded CSS and JavaScript
3. Use safe DOM construction methods (createElement, textContent) for rendering the tree
4. Open the result in the default browser

This pattern works for any visual output: dependency graphs, test coverage reports, API documentation, or database
schema visualizations. The bundled script does the heavy lifting while Claude handles orchestration.

### Enabling extended thinking

To enable extended thinking in a skill, include the word "ultrathink" anywhere in your skill content.
