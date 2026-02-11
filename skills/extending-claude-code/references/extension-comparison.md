# Extension Comparison Reference

Full comparison of all Claude Code extension mechanisms, including feature matching, context costs, layering rules, and head-to-head comparisons.

## Feature Overview Table

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

## Context Cost by Feature

| Feature         | When it loads             | What loads                                    | Context cost                                 |
| --------------- | ------------------------- | --------------------------------------------- | -------------------------------------------- |
| **CLAUDE.md**   | Session start             | Full content                                  | Every request                                |
| **Skills**      | Session start + when used | Descriptions at start, full content when used | Low (descriptions every request)*            |
| **MCP servers** | Session start             | All tool definitions and schemas              | Every request                                |
| **Subagents**   | When spawned              | Fresh context with specified skills           | Isolated from main session                   |
| **Hooks**       | On trigger                | Nothing (runs externally)                     | Zero, unless hook returns additional context |

*Set `disable-model-invocation: true` in a skill's frontmatter to hide it from Claude entirely until manually invoked. This reduces context cost to zero for skills you only trigger yourself.

## How Features Layer

Features can be defined at multiple levels: user-wide, per-project, via plugins, or through managed policies.

- **CLAUDE.md files** are additive: all levels contribute content simultaneously. Files from your working directory and above load at launch; subdirectories load as you work in them. When instructions conflict, Claude uses judgment to reconcile them, with more specific instructions typically taking precedence.
- **Skills and subagents** override by name: when the same name exists at multiple levels, one definition wins based on priority (managed > user > project for skills; managed > CLI flag > project > user > plugin for subagents). Plugin skills are namespaced to avoid conflicts.
- **MCP servers** override by name: local > project > user.
- **Hooks** merge: all registered hooks fire for their matching events regardless of source.

## How Features Load

### CLAUDE.md

- **When:** Session start
- **What loads:** Full content of all CLAUDE.md files (managed, user, and project levels).
- **Inheritance:** Claude reads CLAUDE.md files from your working directory up to the root, and discovers nested ones in subdirectories as it accesses those files.
- **Tip:** Keep CLAUDE.md under ~500 lines. Move reference material to skills, which load on-demand.

### Skills

- **When:** Depends on configuration. By default, descriptions load at session start and full content loads when used. For user-only skills (`disable-model-invocation: true`), nothing loads until you invoke them.
- **What loads:** For model-invocable skills, Claude sees names and descriptions in every request. When you invoke a skill with `/<name>` or Claude loads it automatically, the full content loads into your conversation.
- **How Claude chooses skills:** Claude matches your task against skill descriptions to decide which are relevant. If descriptions are vague or overlap, Claude may load the wrong skill or miss one.
- **Context cost:** Low until used. User-only skills have zero cost until invoked.
- **In subagents:** Skills passed to a subagent are fully preloaded into its context at launch. Subagents don't inherit skills from the main session; you must specify them explicitly.
- **Tip:** Use `disable-model-invocation: true` for skills with side effects. This saves context and ensures only you trigger them.

### MCP Servers

- **When:** Session start.
- **What loads:** All tool definitions and JSON schemas from connected servers.
- **Context cost:** Tool search (enabled by default) loads MCP tools up to 10% of context and defers the rest until needed.
- **Reliability note:** MCP connections can fail silently mid-session. If a server disconnects, its tools disappear without warning.
- **Tip:** Run `/mcp` to see token costs per server. Disconnect servers you're not actively using.

### Subagents

- **When:** On demand, when you or Claude spawns one for a task.
- **What loads:** Fresh, isolated context containing:
  - The system prompt (shared with parent for cache efficiency)
  - Full content of skills listed in the agent's `skills:` field
  - CLAUDE.md and git status (inherited from parent)
  - Whatever context the lead agent passes in the prompt
- **Context cost:** Isolated from main session. Subagents don't inherit your conversation history or invoked skills.
- **Tip:** Use subagents for work that doesn't need your full conversation context. Their isolation prevents bloating your main session.

### Hooks

- **When:** On trigger. Hooks fire at specific lifecycle events like tool execution, session boundaries, prompt submission, permission requests, and compaction.
- **What loads:** Nothing by default. Hooks run as external scripts.
- **Context cost:** Zero, unless the hook returns output that gets added as messages to your conversation.
- **Tip:** Hooks are ideal for side effects (linting, logging) that don't need to affect Claude's context.

## Feature Matching Guide

**If you need... use:**

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

## Head-to-Head Comparisons

### Skill vs Subagent

Skills and subagents solve different problems:

- **Skills** are reusable content you can load into any context
- **Subagents** are isolated workers that run separately from your main conversation

| Aspect          | Skill                                          | Subagent                                                         |
| --------------- | ---------------------------------------------- | ---------------------------------------------------------------- |
| **What it is**  | Reusable instructions, knowledge, or workflows | Isolated worker with its own context                             |
| **Key benefit** | Share content across contexts                  | Context isolation. Work happens separately, only summary returns |
| **Best for**    | Reference material, invocable workflows        | Tasks that read many files, parallel work, specialized workers   |

**Skills can be reference or action.** Reference skills provide knowledge Claude uses throughout your session (like your API style guide). Action skills tell Claude to do something specific (like `/deploy` that runs your deployment workflow).

**Use a subagent** when you need context isolation or when your context window is getting full. The subagent might read dozens of files or run extensive searches, but your main conversation only receives a summary. Since subagent work doesn't consume your main context, this is also useful when you don't need the intermediate work to remain visible. Custom subagents can have their own instructions and can preload skills.

**They can combine.** A subagent can preload specific skills (`skills:` field). A skill can run in isolated context using `context: fork`. See Skills documentation for details.

### CLAUDE.md vs Skill

Both store instructions, but they load differently and serve different purposes.

| Aspect                    | CLAUDE.md                    | Skill                                   |
| ------------------------- | ---------------------------- | --------------------------------------- |
| **Loads**                 | Every session, automatically | On demand                               |
| **Can include files**     | Yes, with `@path` imports    | Yes, with `@path` imports               |
| **Can trigger workflows** | No                           | Yes, with `/<name>`                     |
| **Best for**              | "Always do X" rules          | Reference material, invocable workflows |

**Put it in CLAUDE.md** if Claude should always know it: coding conventions, build commands, project structure, "never do X" rules.

**Put it in a skill** if it's reference material Claude needs sometimes (API docs, style guides) or a workflow you trigger with `/<name>` (deploy, review, release).

**Rule of thumb:** Keep CLAUDE.md under ~500 lines. If it's growing, move reference content to skills.

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

**Use a subagent** when you need a quick, focused worker: research a question, verify a claim, review a file. The subagent does the work and returns a summary. Your main conversation stays clean.

**Use an agent team** when teammates need to share findings, challenge each other, and coordinate independently. Agent teams are best for research with competing hypotheses, parallel code review, and new feature development where each teammate owns a separate piece.

**Transition point:** If you're running parallel subagents but hitting context limits, or if your subagents need to communicate with each other, agent teams are the natural next step.

Note: Agent teams are experimental and disabled by default.

### MCP vs Skill

MCP connects Claude to external services. Skills extend what Claude knows, including how to use those services effectively.

| Aspect         | MCP                                                  | Skill                                                   |
| -------------- | ---------------------------------------------------- | ------------------------------------------------------- |
| **What it is** | Protocol for connecting to external services         | Knowledge, workflows, and reference material            |
| **Provides**   | Tools and data access                                | Knowledge, workflows, reference material                |
| **Examples**   | Slack integration, database queries, browser control | Code review checklist, deploy workflow, API style guide |

These solve different problems and work well together:

**MCP** gives Claude the ability to interact with external systems. Without MCP, Claude can't query your database or post to Slack.

**Skills** give Claude knowledge about how to use those tools effectively, plus workflows you can trigger with `/<name>`. A skill might include your team's database schema and query patterns, or a `/post-to-slack` workflow with your team's message formatting rules.

**Example:** An MCP server connects Claude to your database. A skill teaches Claude your data model, common query patterns, and which tables to use for different tasks.

## Common Combination Patterns

| Pattern                | How it works                                                                     | Example                                                                                            |
| ---------------------- | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Skill + MCP**        | MCP provides the connection; a skill teaches Claude how to use it well           | MCP connects to your database, a skill documents your schema and query patterns                    |
| **Skill + Subagent**   | A skill spawns subagents for parallel work                                       | `/review` skill kicks off security, performance, and style subagents in isolated context           |
| **CLAUDE.md + Skills** | CLAUDE.md holds always-on rules; skills hold reference material loaded on demand | CLAUDE.md says "follow our API conventions," a skill contains the full API style guide             |
| **Hook + MCP**         | A hook triggers external actions through MCP                                     | Post-edit hook sends a Slack notification when Claude modifies critical files                      |
