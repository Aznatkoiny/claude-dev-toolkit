---
name: skills-authoring
description: >
  Guide for creating, configuring, and distributing Claude Code skills. Use when the user asks about writing skills,
  creating custom slash commands, building SKILL.md files, skill frontmatter, skill configuration, skill arguments,
  skill invocation control, skill supporting files, skill templates, skill directory structure, dynamic context
  injection in skills, subagent skills, context fork, sharing skills via plugins, skill troubleshooting,
  disable-model-invocation, user-invocable settings, allowed-tools in skills, or extending Claude with custom
  capabilities. Also triggers for: "create a skill", "write a skill", "new slash command", "custom command",
  "how do skills work", "skill not triggering", "skill triggers too often", "$ARGUMENTS", "pass arguments to skill".
---

# Skills Authoring Guide

Skills extend what Claude can do in Claude Code. You create a `SKILL.md` file with instructions, and Claude adds it
to its toolkit. Claude uses skills when relevant, or users can invoke one directly with `/skill-name`.

Custom slash commands have been merged into skills. A file at `.claude/commands/review.md` and a skill at
`.claude/skills/review/SKILL.md` both create `/review` and work the same way. Existing `.claude/commands/` files
keep working. Skills add optional features: a directory for supporting files, frontmatter to control whether the
user or Claude invokes them, and the ability for Claude to load them automatically when relevant.

Claude Code skills follow the Agent Skills open standard (agentskills.io), which works across multiple AI tools.
Claude Code extends the standard with additional features like invocation control, subagent execution, and dynamic
context injection.

## When to Use This Skill

- Creating a new skill from scratch
- Configuring skill frontmatter (name, description, invocation control, allowed-tools)
- Adding supporting files, templates, or scripts to a skill
- Passing arguments to skills ($ARGUMENTS, $N, ${CLAUDE_SESSION_ID})
- Injecting dynamic context with shell commands (!`command`)
- Running skills in subagents (context: fork)
- Distributing skills via plugins, projects, or managed settings
- Troubleshooting skills that don't trigger or trigger too often

## When NOT to Use This Skill

- For built-in commands like `/help` and `/compact` -- see interactive mode docs
- For creating subagent definitions (`.claude/agents/`) -- see the subagents skill
- For creating plugins that package skills -- see the plugin-development skill
- For hook configuration -- see the hooks-automation skill

## Quick Reference

| Need to...                              | Read This                                                  |
| :-------------------------------------- | :--------------------------------------------------------- |
| Understand all frontmatter fields       | [frontmatter-reference.md](references/frontmatter-reference.md) |
| Add templates, scripts, or references   | [supporting-files.md](references/supporting-files.md)      |
| Control who can invoke a skill          | [invocation-control.md](references/invocation-control.md)  |
| Pass arguments to a skill               | [frontmatter-reference.md](references/frontmatter-reference.md) (string substitutions section) |
| Run a skill in a subagent               | [invocation-control.md](references/invocation-control.md) (subagent section) |
| Inject dynamic context                  | [supporting-files.md](references/supporting-files.md) (dynamic context section) |

## Core Workflow: Creating a Skill

### Step 1: Create the skill directory

Create a directory for the skill. Where you store it determines who can use it:

| Location   | Path                                                     | Applies to                     |
| :--------- | :------------------------------------------------------- | :----------------------------- |
| Enterprise | See managed settings                                     | All users in your organization |
| Personal   | `~/.claude/skills/<skill-name>/SKILL.md`                 | All your projects              |
| Project    | `.claude/skills/<skill-name>/SKILL.md`                   | This project only              |
| Plugin     | `<plugin>/skills/<skill-name>/SKILL.md`                  | Where plugin is enabled        |

When skills share the same name across levels, higher-priority locations win: enterprise > personal > project.
Plugin skills use a `plugin-name:skill-name` namespace, so they cannot conflict with other levels. If you have files
in `.claude/commands/`, those work the same way, but if a skill and a command share the same name, the skill takes
precedence.

**Automatic discovery from nested directories**: When you work with files in subdirectories, Claude Code automatically
discovers skills from nested `.claude/skills/` directories. For example, if you're editing a file in
`packages/frontend/`, Claude Code also looks for skills in `packages/frontend/.claude/skills/`. This supports
monorepo setups where packages have their own skills.

**Skills from additional directories**: Skills defined in `.claude/skills/` within directories added via `--add-dir`
are loaded automatically and picked up by live change detection, so you can edit them during a session without
restarting.

```bash
mkdir -p ~/.claude/skills/my-skill
```

### Step 2: Write SKILL.md

Every skill needs a `SKILL.md` file with two parts: YAML frontmatter (between `---` markers) that tells Claude
when to use the skill, and markdown content with instructions Claude follows when the skill is invoked.

```yaml
---
name: explain-code
description: Explains code with visual diagrams and analogies. Use when explaining how code works, teaching about a codebase, or when the user asks "how does this work?"
---

When explaining code, always include:

1. **Start with an analogy**: Compare the code to something from everyday life
2. **Draw a diagram**: Use ASCII art to show the flow, structure, or relationships
3. **Walk through the code**: Explain step-by-step what happens
4. **Highlight a gotcha**: What's a common mistake or misconception?

Keep explanations conversational. For complex concepts, use multiple analogies.
```

The `name` field becomes the `/slash-command`, and the `description` helps Claude decide when to load it
automatically. All frontmatter fields are optional. Only `description` is recommended so Claude knows when to use
the skill. See [frontmatter-reference.md](references/frontmatter-reference.md) for all available fields.

### Step 3: Choose the skill type

Think about how you want the skill to be invoked. This guides what content to include:

**Reference content** adds knowledge Claude applies to your current work. Conventions, patterns, style guides,
domain knowledge. This content runs inline so Claude can use it alongside your conversation context.

```yaml
---
name: api-conventions
description: API design patterns for this codebase
---

When writing API endpoints:
- Use RESTful naming conventions
- Return consistent error formats
- Include request validation
```

**Task content** gives Claude step-by-step instructions for a specific action, like deployments, commits, or code
generation. These are often actions you want to invoke directly with `/skill-name` rather than letting Claude decide
when to run them. Add `disable-model-invocation: true` to prevent Claude from triggering it automatically.

```yaml
---
name: deploy
description: Deploy the application to production
context: fork
disable-model-invocation: true
---

Deploy the application:
1. Run the test suite
2. Build the application
3. Push to the deployment target
```

### Step 4: Add supporting files (optional)

Skills can include multiple files in their directory. This keeps `SKILL.md` focused on the essentials while
letting Claude access detailed reference material only when needed. See
[supporting-files.md](references/supporting-files.md) for patterns and examples.

```
my-skill/
├── SKILL.md           # Main instructions (required)
├── template.md        # Template for Claude to fill in
├── examples/
│   └── sample.md      # Example output showing expected format
└── scripts/
    └── validate.sh    # Script Claude can execute
```

### Step 5: Configure invocation control (optional)

By default, both the user and Claude can invoke any skill. Use frontmatter fields to restrict this:

- `disable-model-invocation: true` -- Only the user can invoke (for workflows with side effects)
- `user-invocable: false` -- Only Claude can invoke (for background knowledge)

See [invocation-control.md](references/invocation-control.md) for details and examples.

### Step 6: Test the skill

You can test a skill two ways:

**Let Claude invoke it automatically** by asking something that matches the description:

```
How does this code work?
```

**Or invoke it directly** with the skill name:

```
/explain-code src/auth/login.ts
```

## Key Concepts Summary

- **SKILL.md** is the required entrypoint; frontmatter configures behavior, markdown body provides instructions
- **Description** is the most important field -- Claude uses it to decide when to load the skill
- **String substitutions** (`$ARGUMENTS`, `$N`, `${CLAUDE_SESSION_ID}`) make skills dynamic
- **Dynamic context** (`!`command``) runs shell commands at invocation time, injecting output into the prompt
- **`context: fork`** runs skills in isolated subagents that don't share conversation history
- **`disable-model-invocation: true`** prevents Claude from auto-triggering dangerous workflows
- **`user-invocable: false`** hides skills from the `/` menu for background knowledge only
- **`allowed-tools`** restricts which tools Claude can use when the skill is active
- **Supporting files** keep SKILL.md focused; reference them so Claude knows when to load them
- **Skill descriptions budget**: scales at 2% of context window (fallback 16,000 chars); override with `SLASH_COMMAND_TOOL_CHAR_BUDGET`

## Sharing Skills

Skills can be distributed at different scopes:

- **Project skills**: Commit `.claude/skills/` to version control
- **Plugins**: Create a `skills/` directory in your plugin
- **Managed**: Deploy organization-wide through managed settings

## Troubleshooting

**Skill not triggering**: Check that the description includes keywords users would naturally say. Verify the skill
appears when asking "What skills are available?" Try invoking directly with `/skill-name`.

**Skill triggers too often**: Make the description more specific. Add `disable-model-invocation: true` if you only
want manual invocation.

**Claude doesn't see all skills**: Skill descriptions are loaded into context so Claude knows what's available.
If you have many skills, they may exceed the character budget. Run `/context` to check for a warning about excluded
skills. Override the limit with the `SLASH_COMMAND_TOOL_CHAR_BUDGET` environment variable.

## Reference File Index

| File | Description |
| :--- | :---------- |
| [frontmatter-reference.md](references/frontmatter-reference.md) | Complete frontmatter fields table, all YAML options, required vs optional fields, string substitutions, examples |
| [supporting-files.md](references/supporting-files.md) | Directory structure patterns, reference files, templates, dynamic context injection, the codebase visualizer example |
| [invocation-control.md](references/invocation-control.md) | User-invocable vs model-invocable, disable-model-invocation, context: fork, subagent execution, permission rules |
