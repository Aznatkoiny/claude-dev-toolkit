---
name: extending-claude-code
description: >
  Comprehensive guide for extending Claude Code with custom functionality. Use when deciding how to extend Claude Code, comparing extension mechanisms, creating output styles, or running Claude Code programmatically. Trigger phrases: "extend Claude Code", "which extension", "CLAUDE.md vs skill", "subagent vs team", "output style", "custom output", "programmatic Claude", "Agent SDK", "claude -p", "headless mode", "structured output", "CI/CD integration", "MCP vs skill", "hook vs skill", "plugin packaging".
---

# Extending Claude Code

## Overview

Claude Code combines a model that reasons about your code with built-in tools for file operations, search, execution, and web access. The built-in tools cover most coding tasks. On top of this core, Claude Code provides an extension layer: features you add to customize what Claude knows, connect it to external services, and automate workflows.

Extensions plug into different parts of the agentic loop. CLAUDE.md adds persistent context Claude sees every session. Skills add reusable knowledge and invocable workflows. MCP connects Claude to external services and tools. Subagents run their own loops in isolated context, returning summaries. Agent teams coordinate multiple independent sessions with shared tasks and peer-to-peer messaging. Hooks run outside the loop entirely as deterministic scripts. Plugins and marketplaces package and distribute these features.

Skills are the most flexible extension. A skill is a markdown file containing knowledge, workflows, or instructions. You can invoke skills with a slash command like `/deploy`, or Claude can load them automatically when relevant. Skills can run in your current conversation or in an isolated context via subagents. New to Claude Code? Start with CLAUDE.md for project conventions, then add other extensions as you need them.

## When to Use This Skill

Use this skill when you need to:

- Decide which extension mechanism fits your use case
- Compare features like CLAUDE.md vs Skills, Subagents vs Agent Teams, or MCP vs Skills
- Understand how extensions layer and combine
- Create custom output styles for non-engineering use cases
- Run Claude Code programmatically via CLI, scripts, or CI/CD pipelines
- Get structured output from Claude Code
- Understand context costs of different extensions

## Quick Reference: Routing Table

| You want to...                                | Go to                          |
| --------------------------------------------- | ------------------------------ |
| Compare all extension types side-by-side      | extension-comparison.md        |
| Decide between two similar features           | extension-comparison.md        |
| Run Claude Code from scripts or CI/CD         | programmatic-usage.md          |
| Get structured JSON output                    | programmatic-usage.md          |
| Create or customize output styles             | output-styles-reference.md     |
| Understand output style vs CLAUDE.md          | output-styles-reference.md     |

## Extension Comparison Table

| Feature          | What it does                                          | When to use it                                           | Example                                                                          |
| ---------------- | ----------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **CLAUDE.md**    | Persistent context loaded every conversation          | Project conventions, "always do X" rules                 | "Use pnpm, not npm. Run tests before committing."                                |
| **Skill**        | Instructions, knowledge, and workflows Claude can use | Reusable content, reference docs, repeatable tasks       | `/review` runs your code review checklist; API docs skill with endpoint patterns |
| **Subagent**     | Isolated execution context returning summarized results | Context isolation, parallel tasks, specialized workers  | Research task that reads many files but returns only key findings                 |
| **Agent teams**  | Coordinate multiple independent Claude Code sessions  | Parallel research, feature dev, debugging with hypotheses | Spawn reviewers to check security, performance, and tests simultaneously        |
| **MCP**          | Connect to external services                          | External data or actions                                 | Query your database, post to Slack, control a browser                            |
| **Hook**         | Deterministic script that runs on events              | Predictable automation, no LLM involved                  | Run ESLint after every file edit                                                 |
| **Plugin**       | Bundle skills, hooks, subagents, MCP into one unit    | Reuse across repos, distribute to others                 | Plugin with deploy skill + linting hook + database MCP                           |
| **Output Style** | Modify how Claude responds (tone, format, structure)  | Non-engineering uses, teaching mode, custom formatting   | Explanatory style that adds educational insights while coding                    |

**Plugins** are the packaging layer. A plugin bundles skills, hooks, subagents, and MCP servers into a single installable unit. Plugin skills are namespaced (like `/my-plugin:review`) so multiple plugins can coexist. Use plugins to reuse the same setup across repositories or distribute to others via a marketplace.

## Decision Guide

If you need... use:

- **"Always do X" rules and project conventions** --> CLAUDE.md
- **Reference material Claude needs sometimes** --> Skill
- **A repeatable workflow triggered by a slash command** --> Skill with `/<name>`
- **Context isolation or parallel workers** --> Subagent
- **Multiple agents that coordinate and discuss** --> Agent team
- **Connection to an external service** --> MCP server
- **Deterministic automation on events (no LLM)** --> Hook
- **Bundling extensions for distribution** --> Plugin
- **Changing how Claude formats responses** --> Output style
- **Running Claude from scripts or CI/CD** --> `claude -p` (Agent SDK CLI)
- **Structured JSON output from Claude** --> `claude -p --output-format json`

## Compare Similar Features

Some features can seem similar. Here's how to tell them apart.

### Skill vs Subagent

Skills and subagents solve different problems:

- **Skills** are reusable content you can load into any context
- **Subagents** are isolated workers that run separately from your main conversation

| Aspect          | Skill                                          | Subagent                                                         |
| --------------- | ---------------------------------------------- | ---------------------------------------------------------------- |
| **What it is**  | Reusable instructions, knowledge, or workflows | Isolated worker with its own context                             |
| **Key benefit** | Share content across contexts                  | Context isolation. Work happens separately, only summary returns |
| **Best for**    | Reference material, invocable workflows        | Tasks that read many files, parallel work, specialized workers   |

Skills can be reference or action. Reference skills provide knowledge Claude uses throughout your session (like your API style guide). Action skills tell Claude to do something specific (like `/deploy` that runs your deployment workflow).

Use a subagent when you need context isolation or when your context window is getting full. The subagent might read dozens of files or run extensive searches, but your main conversation only receives a summary.

They can combine: a subagent can preload specific skills (`skills:` field). A skill can run in isolated context using `context: fork`.

### CLAUDE.md vs Skill

Both store instructions, but they load differently and serve different purposes.

| Aspect                    | CLAUDE.md                    | Skill                                   |
| ------------------------- | ---------------------------- | --------------------------------------- |
| **Loads**                 | Every session, automatically | On demand                               |
| **Can include files**     | Yes, with `@path` imports    | Yes, with `@path` imports               |
| **Can trigger workflows** | No                           | Yes, with `/<name>`                     |
| **Best for**              | "Always do X" rules          | Reference material, invocable workflows |

Put it in CLAUDE.md if Claude should always know it: coding conventions, build commands, project structure, "never do X" rules.

Put it in a skill if it's reference material Claude needs sometimes (API docs, style guides) or a workflow you trigger with `/<name>` (deploy, review, release).

Rule of thumb: Keep CLAUDE.md under ~500 lines. If it's growing, move reference content to skills.

### Subagent vs Agent Team

Both parallelize work, but they're architecturally different:

- **Subagents** run inside your session and report results back to your main context
- **Agent teams** are independent Claude Code sessions that communicate with each other

| Aspect            | Subagent                                         | Agent team                                          |
| ----------------- | ------------------------------------------------ | --------------------------------------------------- |
| **Context**       | Own context window; results return to the caller | Own context window; fully independent               |
| **Communication** | Reports results back to the main agent only      | Teammates message each other directly               |
| **Coordination**  | Main agent manages all work                      | Shared task list with self-coordination             |
| **Best for**      | Focused tasks where only the result matters      | Complex work requiring discussion and collaboration |
| **Token cost**    | Lower: results summarized back to main context   | Higher: each teammate is a separate Claude instance |

Use a subagent when you need a quick, focused worker: research a question, verify a claim, review a file. The subagent does the work and returns a summary.

Use an agent team when teammates need to share findings, challenge each other, and coordinate independently. Best for research with competing hypotheses, parallel code review, and feature development where each teammate owns a separate piece.

Transition point: If you're running parallel subagents but hitting context limits, or if your subagents need to communicate with each other, agent teams are the natural next step. Agent teams are experimental and disabled by default.

### MCP vs Skill

MCP connects Claude to external services. Skills extend what Claude knows, including how to use those services effectively.

| Aspect         | MCP                                                  | Skill                                                   |
| -------------- | ---------------------------------------------------- | ------------------------------------------------------- |
| **What it is** | Protocol for connecting to external services         | Knowledge, workflows, and reference material            |
| **Provides**   | Tools and data access                                | Knowledge, workflows, reference material                |
| **Examples**   | Slack integration, database queries, browser control | Code review checklist, deploy workflow, API style guide |

MCP gives Claude the ability to interact with external systems. Without MCP, Claude can't query your database or post to Slack. Skills give Claude knowledge about how to use those tools effectively, plus workflows you can trigger with `/<name>`.

Example: An MCP server connects Claude to your database. A skill teaches Claude your data model, common query patterns, and which tables to use for different tasks.

## Context Cost by Feature

Every feature you add consumes some of Claude's context. Too much can fill up your context window, but it can also add noise that makes Claude less effective. Understanding these trade-offs helps you build an effective setup.

| Feature         | When it loads             | What loads                                    | Context cost                                 |
| --------------- | ------------------------- | --------------------------------------------- | -------------------------------------------- |
| **CLAUDE.md**   | Session start             | Full content                                  | Every request                                |
| **Skills**      | Session start + when used | Descriptions at start, full content when used | Low (descriptions every request)*            |
| **MCP servers** | Session start             | All tool definitions and schemas              | Every request                                |
| **Subagents**   | When spawned              | Fresh context with specified skills           | Isolated from main session                   |
| **Hooks**       | On trigger                | Nothing (runs externally)                     | Zero, unless hook returns additional context |

*By default, skill descriptions load at session start so Claude can decide when to use them. Set `disable-model-invocation: true` in a skill's frontmatter to hide it from Claude entirely until you invoke it manually. This reduces context cost to zero for skills you only trigger yourself.

### How Each Feature Loads

**CLAUDE.md** loads at session start. Full content of all CLAUDE.md files (managed, user, and project levels). Claude reads CLAUDE.md files from your working directory up to the root, and discovers nested ones in subdirectories as it accesses those files. Keep CLAUDE.md under ~500 lines.

**Skills** load descriptions at session start; full content loads when used. For user-only skills (`disable-model-invocation: true`), nothing loads until you invoke them. Claude matches your task against skill descriptions to decide which are relevant. In subagents, skills are fully preloaded at launch rather than on-demand.

**MCP servers** load at session start. All tool definitions and JSON schemas from connected servers load. Tool search (enabled by default) loads MCP tools up to 10% of context and defers the rest until needed. Run `/mcp` to see token costs per server.

**Subagents** load on demand with fresh, isolated context containing: the system prompt, full content of skills listed in the agent's `skills:` field, CLAUDE.md and git status (inherited from parent), and whatever context the lead agent passes in the prompt.

**Hooks** fire at specific lifecycle events like tool execution, session boundaries, prompt submission, permission requests, and compaction. Nothing loads by default. Hooks run as external scripts with zero context cost unless the hook returns output.

## How Features Layer

Features can be defined at multiple levels: user-wide, per-project, via plugins, or through managed policies. You can also nest CLAUDE.md files in subdirectories or place skills in specific packages of a monorepo. When the same feature exists at multiple levels:

- **CLAUDE.md files** are additive: all levels contribute content simultaneously. Files from your working directory and above load at launch; subdirectories load as you work in them. When instructions conflict, Claude uses judgment to reconcile them, with more specific instructions typically taking precedence.
- **Skills and subagents** override by name: when the same name exists at multiple levels, one definition wins based on priority (managed > user > project for skills; managed > CLI flag > project > user > plugin for subagents). Plugin skills are namespaced to avoid conflicts.
- **MCP servers** override by name: local > project > user.
- **Hooks** merge: all registered hooks fire for their matching events regardless of source.

## Common Combination Patterns

Each extension solves a different problem. Real setups combine them based on your workflow.

| Pattern                | How it works                                                                     | Example                                                                                            |
| ---------------------- | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Skill + MCP**        | MCP provides the connection; a skill teaches Claude how to use it well           | MCP connects to your database, a skill documents your schema and query patterns                    |
| **Skill + Subagent**   | A skill spawns subagents for parallel work                                       | `/review` skill kicks off security, performance, and style subagents in isolated context           |
| **CLAUDE.md + Skills** | CLAUDE.md holds always-on rules; skills hold reference material loaded on demand | CLAUDE.md says "follow our API conventions," a skill contains the full API style guide             |
| **Hook + MCP**         | A hook triggers external actions through MCP                                     | Post-edit hook sends a Slack notification when Claude modifies critical files                      |

## Output Styles Quick Reference

Output styles modify Claude Code's system prompt to change how it responds. Three built-in styles are available:

- **Default**: Standard system prompt for software engineering tasks
- **Explanatory**: Adds educational "Insights" between coding tasks, explaining implementation choices
- **Learning**: Collaborative learn-by-doing mode with `TODO(human)` markers for you to implement

Custom output styles are Markdown files with frontmatter saved at `~/.claude/output-styles/` (user) or `.claude/output-styles/` (project). Key frontmatter fields: `name`, `description`, `keep-coding-instructions` (default: false).

See output-styles-reference.md for full details on creating custom styles.

## Programmatic Usage Quick Reference

Run Claude Code non-interactively with the `-p` flag:

```bash
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"
```

Key flags:
- `--output-format`: `text` (default), `json`, or `stream-json`
- `--json-schema`: JSON Schema for structured output
- `--allowedTools`: Auto-approve specific tools
- `--continue` / `--resume`: Continue conversations
- `--append-system-prompt`: Add to system prompt
- `--system-prompt`: Replace system prompt entirely

See programmatic-usage.md for complete examples and CI/CD integration patterns.

## Reference Files

| File                        | Contents                                                          |
| --------------------------- | ----------------------------------------------------------------- |
| extension-comparison.md     | Full comparison of all extension types, feature matching, X vs Y  |
| programmatic-usage.md       | CLI usage, structured output, streaming, CI/CD integration        |
| output-styles-reference.md  | Built-in styles, custom styles, frontmatter, comparisons          |
