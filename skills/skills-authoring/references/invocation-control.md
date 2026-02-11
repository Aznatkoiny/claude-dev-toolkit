# Invocation Control

By default, both you and Claude can invoke any skill. You can type `/skill-name` to invoke it directly, and Claude
can load it automatically when relevant to your conversation. Two frontmatter fields let you restrict this.

## User-Invocable vs Model-Invocable

### disable-model-invocation: true

Only the user can invoke the skill. Use this for workflows with side effects or where you want to control timing,
like `/commit`, `/deploy`, or `/send-slack-message`. You don't want Claude deciding to deploy because your code
looks ready.

```yaml
---
name: deploy
description: Deploy the application to production
disable-model-invocation: true
---

Deploy $ARGUMENTS to production:

1. Run the test suite
2. Build the application
3. Push to the deployment target
4. Verify the deployment succeeded
```

When `disable-model-invocation: true` is set:
- The skill description is NOT loaded into Claude's context
- Claude cannot see or invoke the skill
- The skill only runs when the user explicitly types `/deploy`

### user-invocable: false

Only Claude can invoke the skill. Use this for background knowledge that isn't actionable as a command. A
`legacy-system-context` skill explains how an old system works. Claude should know this when relevant, but
`/legacy-system-context` isn't a meaningful action for users to take.

```yaml
---
name: legacy-system-context
description: How the legacy billing system works
user-invocable: false
---

The legacy billing system uses...
```

When `user-invocable: false` is set:
- The skill is hidden from the `/` autocomplete menu
- Claude can still see the description and invoke it when relevant
- The full skill content loads when Claude invokes it

**Important note**: The `user-invocable` field only controls menu visibility, not Skill tool access. Use
`disable-model-invocation: true` to block programmatic invocation.

## Invocation and Context Loading Matrix

| Frontmatter                      | User can invoke | Claude can invoke | When loaded into context                                     |
| :------------------------------- | :-------------- | :---------------- | :----------------------------------------------------------- |
| (default)                        | Yes             | Yes               | Description always in context, full skill loads when invoked |
| `disable-model-invocation: true` | Yes             | No                | Description not in context, full skill loads when you invoke |
| `user-invocable: false`          | No              | Yes               | Description always in context, full skill loads when invoked |

In a regular session, skill descriptions are loaded into context so Claude knows what's available, but full skill
content only loads when invoked. Subagents with preloaded skills work differently: the full skill content is
injected at startup.

## Restricting Skill Access with Permissions

Three ways to control which skills Claude can invoke:

### Disable all skills

Deny the Skill tool in `/permissions`:

```
# Add to deny rules:
Skill
```

### Allow or deny specific skills

Using permission rules:

```
# Allow only specific skills
Skill(commit)
Skill(review-pr *)

# Deny specific skills
Skill(deploy *)
```

Permission syntax: `Skill(name)` for exact match, `Skill(name *)` for prefix match with any arguments.

### Hide individual skills

Add `disable-model-invocation: true` to their frontmatter. This removes the skill from Claude's context entirely.

## Restricting Tool Access Within a Skill

Use the `allowed-tools` field to limit which tools Claude can use when a skill is active. This creates a scoped
permission set for the skill:

```yaml
---
name: safe-reader
description: Read files without making changes
allowed-tools: Read, Grep, Glob
---
```

Skills that define `allowed-tools` grant Claude access to those tools without per-use approval when the skill is
active. Your permission settings still govern baseline approval behavior for all other tools. Built-in commands
like `/compact` and `/init` are not available through the Skill tool.

## Running Skills in a Subagent (context: fork)

Add `context: fork` to your frontmatter when you want a skill to run in isolation. The skill content becomes the
prompt that drives the subagent. It won't have access to your conversation history.

**Important**: `context: fork` only makes sense for skills with explicit instructions. If your skill contains
guidelines like "use these API conventions" without a task, the subagent receives the guidelines but no actionable
prompt, and returns without meaningful output.

### How context: fork works

1. A new isolated context is created
2. The subagent receives the skill content as its prompt
3. The `agent` field determines the execution environment (model, tools, and permissions)
4. Results are summarized and returned to your main conversation

### Skills and subagents comparison

| Approach                     | System prompt                             | Task                        | Also loads                   |
| :--------------------------- | :---------------------------------------- | :-------------------------- | :--------------------------- |
| Skill with `context: fork`   | From agent type (`Explore`, `Plan`, etc.) | SKILL.md content            | CLAUDE.md                    |
| Subagent with `skills` field | Subagent's markdown body                  | Claude's delegation message | Preloaded skills + CLAUDE.md |

With `context: fork`, you write the task in your skill and pick an agent type to execute it. For the inverse
(defining a custom subagent that uses skills as reference material), see the subagents documentation.

### Example: Research skill using Explore agent

```yaml
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:

1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Summarize findings with specific file references
```

The `agent` field specifies which subagent configuration to use. Options include built-in agents (`Explore`, `Plan`,
`general-purpose`) or any custom subagent from `.claude/agents/`. If omitted, uses `general-purpose`.

### Example: PR summary with dynamic context

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

This combines `context: fork` with dynamic context injection (`!`command``) to fetch live data and analyze it in
an isolated subagent.

## When to Use Each Mode

| Scenario                                      | Recommended configuration                              |
| :-------------------------------------------- | :----------------------------------------------------- |
| Reference/conventions (API style guide)        | Default (both can invoke)                              |
| Dangerous workflow (deploy, send message)       | `disable-model-invocation: true`                       |
| Background knowledge (legacy system docs)       | `user-invocable: false`                                |
| Isolated research or analysis                   | `context: fork` + `agent: Explore`                     |
| Read-only exploration                           | `allowed-tools: Read, Grep, Glob`                      |
| Heavy task that shouldn't pollute conversation  | `context: fork` + `agent: general-purpose`             |
| Dangerous workflow run in isolation             | `disable-model-invocation: true` + `context: fork`     |
