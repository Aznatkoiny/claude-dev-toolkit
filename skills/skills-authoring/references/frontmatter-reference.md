# Frontmatter Reference

Skills are configured through YAML frontmatter at the top of `SKILL.md`, placed between `---` markers.

## Complete Fields Table

All fields are optional. Only `description` is recommended so Claude knows when to use the skill.

| Field                      | Required    | Default     | Description                                                                                                                                           |
| :------------------------- | :---------- | :---------- | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`                     | No          | directory name | Display name for the skill. If omitted, uses the directory name. Lowercase letters, numbers, and hyphens only (max 64 characters).                    |
| `description`              | Recommended | first paragraph | What the skill does and when to use it. Claude uses this to decide when to apply the skill. If omitted, uses the first paragraph of markdown content. |
| `argument-hint`            | No          | none        | Hint shown during autocomplete to indicate expected arguments. Example: `[issue-number]` or `[filename] [format]`.                                    |
| `disable-model-invocation` | No          | `false`     | Set to `true` to prevent Claude from automatically loading this skill. Use for workflows you want to trigger manually with `/name`.                   |
| `user-invocable`           | No          | `true`      | Set to `false` to hide from the `/` menu. Use for background knowledge users shouldn't invoke directly.                                               |
| `allowed-tools`            | No          | none        | Tools Claude can use without asking permission when this skill is active.                                                                             |
| `model`                    | No          | none        | Model to use when this skill is active.                                                                                                               |
| `context`                  | No          | none        | Set to `fork` to run in a forked subagent context.                                                                                                    |
| `agent`                    | No          | `general-purpose` | Which subagent type to use when `context: fork` is set. Options: `Explore`, `Plan`, `general-purpose`, or any custom agent from `.claude/agents/`.   |
| `hooks`                    | No          | none        | Hooks scoped to this skill's lifecycle. See hooks documentation for configuration format.                                                             |

## Field Details and Examples

### name

The `name` field becomes the `/slash-command` for invoking the skill. Must contain only lowercase letters, numbers, and hyphens. Maximum 64 characters.

```yaml
---
name: fix-issue
---
```

If omitted, the directory name is used. A skill at `.claude/skills/fix-issue/SKILL.md` automatically gets the name `fix-issue`.

### description

The most important field. Claude reads all skill descriptions to decide which skills are relevant to the current conversation. Write it to include keywords users would naturally use.

```yaml
---
name: explain-code
description: Explains code with visual diagrams and analogies. Use when explaining how code works, teaching about a codebase, or when the user asks "how does this work?"
---
```

If omitted, Claude uses the first paragraph of the markdown body as the description. Explicit descriptions are strongly recommended.

### argument-hint

Displayed during autocomplete to show users what arguments the skill expects. Purely cosmetic -- does not affect argument parsing.

```yaml
---
name: fix-issue
argument-hint: "[issue-number]"
---
```

```yaml
---
name: migrate-component
argument-hint: "[component] [from-framework] [to-framework]"
---
```

### disable-model-invocation

Prevents Claude from automatically loading this skill. The skill only runs when the user explicitly types `/skill-name`. Use for workflows with side effects.

```yaml
---
name: deploy
description: Deploy the application to production
disable-model-invocation: true
---
```

When set to `true`, the skill description is NOT loaded into Claude's context. Claude cannot see or invoke it.

### user-invocable

Controls whether the skill appears in the `/` autocomplete menu. Set to `false` for background knowledge that Claude should use automatically but users shouldn't invoke directly.

```yaml
---
name: legacy-system-context
description: How the legacy billing system works
user-invocable: false
---
```

Note: The `user-invocable` field only controls menu visibility, not Skill tool access. Use `disable-model-invocation: true` to block programmatic invocation.

### allowed-tools

Limits which tools Claude can use when the skill is active. Tools listed here are auto-approved (no per-use permission prompt).

```yaml
---
name: safe-reader
description: Read files without making changes
allowed-tools: Read, Grep, Glob
---
```

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---
```

### model

Override the model used when the skill is active.

```yaml
---
name: quick-fix
description: Fast simple fixes
model: claude-sonnet-4-5-20250929
---
```

### context

Set to `fork` to run the skill in an isolated subagent. The skill content becomes the prompt that drives the subagent. It will not have access to the conversation history.

```yaml
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---
```

**Important**: `context: fork` only makes sense for skills with explicit instructions. If your skill contains guidelines like "use these API conventions" without a task, the subagent receives the guidelines but no actionable prompt, and returns without meaningful output.

### agent

Specifies which subagent configuration to use when `context: fork` is set. Options include built-in agents (`Explore`, `Plan`, `general-purpose`) or any custom subagent from `.claude/agents/`. If omitted, uses `general-purpose`.

```yaml
---
name: deep-research
context: fork
agent: Explore
---
```

### hooks

Hooks scoped to the skill's lifecycle. Follows the same format as the hooks configuration.

```yaml
---
name: my-skill
hooks:
  skill-start:
    - command: echo "Skill started"
  skill-end:
    - command: echo "Skill ended"
---
```

## String Substitutions

Skills support string substitution for dynamic values in the skill content. These are replaced at invocation time.

| Variable               | Description                                                                                                                                  |
| :--------------------- | :------------------------------------------------------------------------------------------------------------------------------------------- |
| `$ARGUMENTS`           | All arguments passed when invoking the skill. If `$ARGUMENTS` is not present in the content, arguments are appended as `ARGUMENTS: <value>`. |
| `$ARGUMENTS[N]`        | Access a specific argument by 0-based index, such as `$ARGUMENTS[0]` for the first argument.                                                 |
| `$N`                   | Shorthand for `$ARGUMENTS[N]`, such as `$0` for the first argument or `$1` for the second.                                                   |
| `${CLAUDE_SESSION_ID}` | The current session ID. Useful for logging, creating session-specific files, or correlating skill output with sessions.                      |

### Examples

**Using $ARGUMENTS for all arguments:**

```yaml
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---

Fix GitHub issue $ARGUMENTS following our coding standards.

1. Read the issue description
2. Understand the requirements
3. Implement the fix
4. Write tests
5. Create a commit
```

When you run `/fix-issue 123`, Claude receives "Fix GitHub issue 123 following our coding standards..."

If you invoke a skill with arguments but the skill doesn't include `$ARGUMENTS`, Claude Code appends `ARGUMENTS: <your input>` to the end of the skill content so Claude still sees what you typed.

**Using positional arguments ($ARGUMENTS[N] or $N):**

```yaml
---
name: migrate-component
description: Migrate a component from one framework to another
---

Migrate the $ARGUMENTS[0] component from $ARGUMENTS[1] to $ARGUMENTS[2].
Preserve all existing behavior and tests.
```

Running `/migrate-component SearchBar React Vue` replaces `$ARGUMENTS[0]` with `SearchBar`, `$ARGUMENTS[1]` with `React`, and `$ARGUMENTS[2]` with `Vue`.

The same skill using the `$N` shorthand:

```yaml
---
name: migrate-component
description: Migrate a component from one framework to another
---

Migrate the $0 component from $1 to $2.
Preserve all existing behavior and tests.
```

**Using ${CLAUDE_SESSION_ID}:**

```yaml
---
name: session-logger
description: Log activity for this session
---

Log the following to logs/${CLAUDE_SESSION_ID}.log:

$ARGUMENTS
```

**Using argument-hint to show expected format:**

```yaml
---
name: deploy
description: Deploy the application to an environment
argument-hint: "[staging|production]"
disable-model-invocation: true
---

Deploy $ARGUMENTS to the target environment.
```
