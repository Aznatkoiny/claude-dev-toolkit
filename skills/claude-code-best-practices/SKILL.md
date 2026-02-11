---
name: claude-code-best-practices
description: >
  Comprehensive guide for optimizing Claude Code workflows and effectiveness. Use when writing or
  improving CLAUDE.md files, managing context windows, crafting effective prompts, running parallel
  sessions, debugging workflow issues, or improving Claude Code productivity. Covers the
  explore-plan-code workflow, verification strategies, context management, prompt engineering,
  session management, and scaling with parallel sessions. Trigger phrases: "best practices",
  "CLAUDE.md", "context management", "context window", "prompt engineering", "parallel sessions",
  "Claude Code tips", "effective workflow", "explore-plan-code", "verification strategies",
  "session management", "fan out", "headless mode", "subagents", "course correct", "rewind".
---

# Best Practices for Claude Code

Claude Code is an agentic coding environment. Unlike a chatbot that answers questions and waits,
Claude Code can read your files, run commands, make changes, and autonomously work through problems
while you watch, redirect, or step away entirely.

This changes how you work. Instead of writing code yourself and asking Claude to review it, you
describe what you want and Claude figures out how to build it. Claude explores, plans, and
implements. But this autonomy comes with a learning curve -- Claude works within certain constraints
you need to understand.

This skill covers patterns that have proven effective across Anthropic's internal teams and for
engineers using Claude Code across various codebases, languages, and environments. The single most
important thing to understand: **Claude's context window fills up fast, and performance degrades
as it fills.** Everything else follows from this constraint.

## When to Use This Skill

- Optimizing your Claude Code workflow for better results
- Writing or improving CLAUDE.md files
- Managing context windows during long sessions
- Crafting more effective prompts for Claude Code
- Running parallel Claude Code sessions
- Debugging why Claude Code is producing poor results
- Setting up verification strategies for Claude's work
- Learning the explore-plan-code workflow
- Scaling Claude Code usage with headless mode or fan-out patterns

## When NOT to Use This Skill

- Troubleshooting Claude Code installation or configuration errors (use claude-code-troubleshooting)
- Creating plugins (use plugin-development)
- Writing skills or SKILL.md files (use skills-authoring)
- Setting up hooks (use hooks-automation)
- Configuring MCP servers (use mcp-integration)
- Creating subagents or agent teams (use subagents-and-teams)

## Quick Reference

| Topic | Reference File |
|:------|:---------------|
| Context window management, CLAUDE.md best practices, verification strategies, scoping tasks | `references/context-management.md` |
| Effective prompts, being specific vs vague, explore-plan-code workflow, debugging patterns, rich content | `references/prompt-engineering.md` |
| Multiple sessions, git worktrees, task decomposition, fan-out patterns, headless mode | `references/parallel-sessions.md` |

## The Fundamental Constraint: Context Window

Claude's context window holds your entire conversation, including every message, every file Claude
reads, and every command output. This can fill up fast -- a single debugging session or codebase
exploration might generate and consume tens of thousands of tokens.

LLM performance degrades as context fills. When the context window is getting full, Claude may
start "forgetting" earlier instructions or making more mistakes. The context window is the most
important resource to manage.

## Core Principles

### 1. Give Claude a Way to Verify Its Work

This is the single highest-leverage thing you can do. Claude performs dramatically better when it
can verify its own work -- run tests, compare screenshots, and validate outputs.

Without clear success criteria, Claude might produce something that looks right but doesn't work.
You become the only feedback loop, and every mistake requires your attention.

| Strategy | Before | After |
|:---------|:-------|:------|
| **Provide verification criteria** | "implement a function that validates email addresses" | "write a validateEmail function. example test cases: user@example.com is true, invalid is false, user@.com is false. run the tests after implementing" |
| **Verify UI changes visually** | "make the dashboard look better" | "[paste screenshot] implement this design. take a screenshot of the result and compare it to the original. list differences and fix them" |
| **Address root causes, not symptoms** | "the build is failing" | "the build fails with this error: [paste error]. fix it and verify the build succeeds. address the root cause, don't suppress the error" |

Your verification can also be a test suite, a linter, or a Bash command that checks output.
Invest in making your verification rock-solid.

### 2. Explore First, Then Plan, Then Code

Letting Claude jump straight to coding can produce code that solves the wrong problem. Separate
research and planning from implementation to avoid solving the wrong problem.

The recommended workflow has four phases:

1. **Explore**: Enter Plan Mode. Claude reads files and answers questions without making changes.
2. **Plan**: Ask Claude to create a detailed implementation plan. Press `Ctrl+G` to open the plan
   in your text editor for direct editing.
3. **Implement**: Switch back to Normal Mode and let Claude code, verifying against its plan.
4. **Commit**: Ask Claude to commit with a descriptive message and create a PR.

Plan Mode is useful but adds overhead. For tasks where the scope is clear and the fix is small
(like fixing a typo, adding a log line, or renaming a variable), ask Claude to do it directly.
Planning is most useful when you're uncertain about the approach, when the change modifies multiple
files, or when you're unfamiliar with the code being modified.

See `references/prompt-engineering.md` for the full explore-plan-code workflow with examples.

### 3. Provide Specific Context

The more precise your instructions, the fewer corrections you'll need. Reference specific files,
mention constraints, and point to example patterns.

| Strategy | Before | After |
|:---------|:-------|:------|
| **Scope the task** | "add tests for foo.py" | "write a test for foo.py covering the edge case where the user is logged out. avoid mocks." |
| **Point to sources** | "why does ExecutionFactory have such a weird api?" | "look through ExecutionFactory's git history and summarize how its api came to be" |
| **Reference existing patterns** | "add a calendar widget" | "look at how existing widgets are implemented on the home page to understand the patterns. HotDogWidget.php is a good example. follow the pattern to implement a new calendar widget." |
| **Describe the symptom** | "fix the login bug" | "users report that login fails after session timeout. check the auth flow in src/auth/, especially token refresh. write a failing test that reproduces the issue, then fix it" |

See `references/prompt-engineering.md` for more on crafting effective prompts.

### 4. Manage Context Aggressively

Run `/clear` between unrelated tasks to reset context. Claude Code automatically compacts
conversation history when you approach context limits, preserving important code and decisions
while freeing space.

Key context management techniques:

- Use `/clear` frequently between tasks to reset the context window entirely
- Run `/compact <instructions>` for targeted compaction (e.g., `/compact Focus on the API changes`)
- Use `Esc + Esc` or `/rewind` to summarize from a checkpoint
- Customize compaction behavior in CLAUDE.md
- Use subagents to delegate research without cluttering your main context

See `references/context-management.md` for detailed strategies.

## CLAUDE.md Best Practices

CLAUDE.md is a special file Claude reads at the start of every conversation. Include Bash commands,
code style, and workflow rules. This gives Claude persistent context it can't infer from code alone.

Run `/init` to generate a starter CLAUDE.md based on your project structure, then refine over time.

**What to include:**

- Bash commands Claude can't guess
- Code style rules that differ from defaults
- Testing instructions and preferred test runners
- Repository etiquette (branch naming, PR conventions)
- Architectural decisions specific to your project
- Developer environment quirks (required env vars)
- Common gotchas or non-obvious behaviors

**What to exclude:**

- Anything Claude can figure out by reading code
- Standard language conventions Claude already knows
- Detailed API documentation (link to docs instead)
- Information that changes frequently
- Long explanations or tutorials
- File-by-file descriptions of the codebase
- Self-evident practices like "write clean code"

Keep it concise. For each line, ask: "Would removing this cause Claude to make mistakes?" If not,
cut it. Bloated CLAUDE.md files cause Claude to ignore your actual instructions.

See `references/context-management.md` for more CLAUDE.md patterns.

## Session Management

### Course-Correct Early and Often

The best results come from tight feedback loops:

- **`Esc`**: Stop Claude mid-action. Context is preserved, so you can redirect.
- **`Esc + Esc` or `/rewind`**: Open the rewind menu and restore previous conversation/code state.
- **"Undo that"**: Have Claude revert its changes.
- **`/clear`**: Reset context between unrelated tasks.

If you've corrected Claude more than twice on the same issue in one session, the context is
cluttered with failed approaches. Run `/clear` and start fresh with a more specific prompt.

### Resume Conversations

Run `claude --continue` to pick up where you left off, or `--resume` to choose from recent sessions.
Use `/rename` to give sessions descriptive names so you can find them later.

### Use Subagents for Investigation

Delegate research with "use subagents to investigate X". They explore in a separate context,
keeping your main conversation clean for implementation:

```
Use subagents to investigate how our authentication system handles token
refresh, and whether we have any existing OAuth utilities I should reuse.
```

## Common Failure Patterns

Recognizing these patterns early saves time:

- **The kitchen sink session**: You start with one task, then ask something unrelated, then go
  back. Context is full of irrelevant information. **Fix**: `/clear` between unrelated tasks.

- **Correcting over and over**: Claude does something wrong, you correct it, still wrong, correct
  again. Context is polluted with failed approaches. **Fix**: After two corrections, `/clear` and
  write a better initial prompt.

- **The over-specified CLAUDE.md**: Too long, so Claude ignores half of it. Important rules get
  lost in the noise. **Fix**: Ruthlessly prune. If Claude already does something correctly without
  the instruction, delete it or convert it to a hook.

- **The trust-then-verify gap**: Claude produces plausible-looking code that doesn't handle edge
  cases. **Fix**: Always provide verification (tests, scripts, screenshots).

- **The infinite exploration**: You ask Claude to "investigate" without scoping it. Claude reads
  hundreds of files, filling the context. **Fix**: Scope narrowly or use subagents.

## Scaling with Parallel Sessions

Run multiple Claude sessions to speed up development, run isolated experiments, or start complex
workflows. Use git worktrees to give each session its own working directory:

```bash
git worktree add ../my-project-feature-a feature-a
git worktree add ../my-project-feature-b feature-b
```

For large migrations, fan out across files with headless mode:

```bash
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```

See `references/parallel-sessions.md` for detailed patterns.

## Reference File Index

| File | Contents |
|:-----|:---------|
| `references/context-management.md` | Context window as primary constraint, CLAUDE.md best practices, scoping tasks, verification strategies with Before/After tables, subagent delegation |
| `references/prompt-engineering.md` | Effective prompts, being specific vs vague, explore-plan-code workflow in detail, rich content (images, URLs, pipes), debugging patterns, interview technique |
| `references/parallel-sessions.md` | Multiple sessions, git worktrees, writer/reviewer pattern, fan-out across files, headless mode, task decomposition, safe autonomous mode |
