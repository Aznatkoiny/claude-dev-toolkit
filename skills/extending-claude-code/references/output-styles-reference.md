# Output Styles Reference

Complete guide to Claude Code output styles: built-in styles, custom style creation, frontmatter fields, file locations, and comparisons with related features.

Output styles allow you to use Claude Code as any type of agent while keeping its core capabilities, such as running local scripts, reading/writing files, and tracking TODOs.

## Built-in Output Styles

### Default

The existing system prompt, designed to help you complete software engineering tasks efficiently.

### Explanatory

Provides educational "Insights" in between helping you complete software engineering tasks. Helps you understand implementation choices and codebase patterns.

### Learning

Collaborative, learn-by-doing mode where Claude will not only share "Insights" while coding, but also ask you to contribute small, strategic pieces of code yourself. Claude Code will add `TODO(human)` markers in your code for you to implement.

## How Output Styles Work

Output styles directly modify Claude Code's system prompt.

- All output styles exclude instructions for efficient output (such as responding concisely).
- Custom output styles exclude instructions for coding (such as verifying code with tests), unless `keep-coding-instructions` is true.
- All output styles have their own custom instructions added to the end of the system prompt.
- All output styles trigger reminders for Claude to adhere to the output style instructions during the conversation.

## Changing Your Output Style

You can either:

- Run `/output-style` to access a menu and select your output style (this can also be accessed from the `/config` menu)
- Run `/output-style [style]`, such as `/output-style explanatory`, to directly switch to a style

These changes apply to the local project level and are saved in `.claude/settings.local.json`. You can also directly edit the `outputStyle` field in a settings file at a different level.

## Creating a Custom Output Style

Custom output styles are Markdown files with frontmatter and the text that will be added to the system prompt:

```markdown
---
name: My Custom Style
description:
  A brief description of what this style does, to be displayed to the user
---

# Custom Style Instructions

You are an interactive CLI tool that helps users with software engineering
tasks. [Your custom instructions here...]

## Specific Behaviors

[Define how the assistant should behave in this style...]
```

### File Locations

You can save output style files at:

- **User level:** `~/.claude/output-styles/`
- **Project level:** `.claude/output-styles/`

### Frontmatter Fields

| Frontmatter                | Purpose                                                                     | Default                 |
| :------------------------- | :-------------------------------------------------------------------------- | :---------------------- |
| `name`                     | Name of the output style, if not the file name                              | Inherits from file name |
| `description`              | Description of the output style. Used only in the UI of `/output-style`     | None                    |
| `keep-coding-instructions` | Whether to keep the parts of Claude Code's system prompt related to coding. | false                   |

## Comparisons to Related Features

### Output Styles vs CLAUDE.md vs --append-system-prompt

Output styles completely "turn off" the parts of Claude Code's default system prompt specific to software engineering. Neither CLAUDE.md nor `--append-system-prompt` edit Claude Code's default system prompt.

- **Output styles:** Replace parts of the default system prompt. Use for fundamentally changing Claude's behavior.
- **CLAUDE.md:** Adds contents as a user message *following* Claude Code's default system prompt. Use for project conventions and rules.
- **--append-system-prompt:** Appends content to the system prompt. Use for additional instructions in programmatic/CI contexts.

### Output Styles vs Agents (Subagents)

Output styles directly affect the main agent loop and only affect the system prompt. Agents are invoked to handle specific tasks and can include additional settings like the model to use, the tools they have available, and some context about when to use the agent.

### Output Styles vs Skills

Output styles modify how Claude responds (formatting, tone, structure) and are always active once selected. Skills are task-specific prompts that you invoke with `/skill-name` or that Claude loads automatically when relevant. Use output styles for consistent formatting preferences; use skills for reusable workflows and tasks.

## Examples of Custom Styles

### Technical Writer Style

```markdown
---
name: Technical Writer
description: Focuses on clear documentation and technical writing
keep-coding-instructions: true
---

# Technical Writer Mode

You are a technical writer assistant. When helping with code:

- Add clear docstrings to all functions
- Write README sections for new features
- Explain complex logic with inline comments
- Suggest documentation improvements alongside code changes

## Formatting Rules

- Use active voice
- Keep sentences under 25 words
- Define acronyms on first use
```

### Data Analyst Style

```markdown
---
name: Data Analyst
description: Optimized for data analysis and exploration tasks
---

# Data Analyst Mode

You are a data analyst assistant. Focus on:

- Exploring datasets and summarizing findings
- Creating visualizations and charts
- Writing SQL queries and data transformations
- Statistical analysis and hypothesis testing

## Output Format

- Always show your reasoning
- Include data summaries in tables
- Suggest next steps for analysis
```

### Minimal Style

```markdown
---
name: Minimal
description: Extremely concise responses
keep-coding-instructions: true
---

# Minimal Mode

Respond with the absolute minimum text needed. No explanations unless asked.
Show only code changes, commands, or direct answers.
```
