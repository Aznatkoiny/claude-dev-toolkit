# Context Management

## The Context Window: Your Primary Constraint

Most best practices for Claude Code are based on one constraint: Claude's context window fills up
fast, and performance degrades as it fills.

Claude's context window holds your entire conversation, including every message, every file Claude
reads, and every command output. A single debugging session or codebase exploration might generate
and consume tens of thousands of tokens.

This matters because LLM performance degrades as context fills. When the context window is getting
full, Claude may start "forgetting" earlier instructions or making more mistakes. The context
window is the most important resource to manage.

## Managing Context Effectively

### Clear Between Tasks

Use `/clear` frequently between unrelated tasks to reset the context window entirely. Long sessions
with irrelevant context can reduce performance.

If you've corrected Claude more than twice on the same issue in one session, the context is
cluttered with failed approaches. Run `/clear` and start fresh with a more specific prompt that
incorporates what you learned. A clean session with a better prompt almost always outperforms a
long session with accumulated corrections.

### Compaction

Claude Code automatically compacts conversation history when you approach context limits. This
preserves important code and decisions while freeing space.

For more control:

- Run `/compact <instructions>` to compact with specific focus, like `/compact Focus on the API changes`
- Use `Esc + Esc` or `/rewind`, select a message checkpoint, and choose **Summarize from here** to
  condense messages from that point forward while keeping earlier context intact
- Customize compaction behavior in CLAUDE.md with instructions like "When compacting, always
  preserve the full list of modified files and any test commands" to ensure critical context
  survives summarization

### Use Subagents to Protect Context

Since context is your fundamental constraint, subagents are one of the most powerful tools
available. When Claude researches a codebase it reads lots of files, all of which consume your
context. Subagents run in separate context windows and report back summaries:

```
Use subagents to investigate how our authentication system handles token
refresh, and whether we have any existing OAuth utilities I should reuse.
```

The subagent explores the codebase, reads relevant files, and reports back with findings -- all
without cluttering your main conversation.

You can also use subagents for verification after Claude implements something:

```
use a subagent to review this code for edge cases
```

### Rewind with Checkpoints

Every action Claude makes creates a checkpoint. Double-tap `Escape` or run `/rewind` to open the
rewind menu. You can restore conversation only, restore code only, restore both, or summarize from
a selected message.

Instead of carefully planning every move, you can tell Claude to try something risky. If it doesn't
work, rewind and try a different approach. Checkpoints persist across sessions, so you can close
your terminal and still rewind later.

**Note**: Checkpoints only track changes made by Claude, not external processes. This isn't a
replacement for git.

### Resume Conversations

Claude Code saves conversations locally. When a task spans multiple sessions:

```bash
claude --continue    # Resume the most recent conversation
claude --resume      # Select from recent conversations
```

Use `/rename` to give sessions descriptive names ("oauth-migration", "debugging-memory-leak") so
you can find them later. Treat sessions like branches -- different workstreams can have separate,
persistent contexts.

## CLAUDE.md Best Practices

CLAUDE.md is a special file that Claude reads at the start of every conversation. Include Bash
commands, code style, and workflow rules. This gives Claude persistent context it can't infer
from code alone.

Run `/init` to generate a starter CLAUDE.md file based on your current project structure, then
refine over time.

### Format and Style

There's no required format for CLAUDE.md files, but keep it short and human-readable:

```markdown
# Code style
- Use ES modules (import/export) syntax, not CommonJS (require)
- Destructure imports when possible (eg. import { foo } from 'bar')

# Workflow
- Be sure to typecheck when you're done making a series of code changes
- Prefer running single tests, and not the whole test suite, for performance
```

CLAUDE.md is loaded every session, so only include things that apply broadly. For domain knowledge
or workflows that are only relevant sometimes, use skills instead. Claude loads them on demand
without bloating every conversation.

### What to Include vs Exclude

| Include | Exclude |
|:--------|:--------|
| Bash commands Claude can't guess | Anything Claude can figure out by reading code |
| Code style rules that differ from defaults | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners | Detailed API documentation (link to docs instead) |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Developer environment quirks (required env vars) | File-by-file descriptions of the codebase |
| Common gotchas or non-obvious behaviors | Self-evident practices like "write clean code" |

### Keep It Concise

For each line, ask: "Would removing this cause Claude to make mistakes?" If not, cut it. Bloated
CLAUDE.md files cause Claude to ignore your actual instructions.

If Claude keeps doing something you don't want despite having a rule against it, the file is
probably too long and the rule is getting lost. If Claude asks you questions that are answered in
CLAUDE.md, the phrasing might be ambiguous. Treat CLAUDE.md like code: review it when things go
wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts.

You can tune instructions by adding emphasis (e.g., "IMPORTANT" or "YOU MUST") to improve
adherence.

### Importing Files

CLAUDE.md files can import additional files using `@path/to/import` syntax:

```markdown
See @README.md for project overview and @package.json for available npm commands.

# Additional Instructions
- Git workflow: @docs/git-instructions.md
- Personal overrides: @~/.claude/my-project-instructions.md
```

### Placement Locations

You can place CLAUDE.md files in several locations:

- **Home folder (`~/.claude/CLAUDE.md`)**: Applies to all Claude sessions
- **Project root (`./CLAUDE.md`)**: Check into git to share with your team, or name it
  `CLAUDE.local.md` and `.gitignore` it
- **Parent directories**: Useful for monorepos where both `root/CLAUDE.md` and
  `root/foo/CLAUDE.md` are pulled in automatically
- **Child directories**: Claude pulls in child CLAUDE.md files on demand when working with files
  in those directories

## Scoping Tasks and Providing File Paths

The more precise your instructions, the fewer corrections you'll need:

| Strategy | Before | After |
|:---------|:-------|:------|
| **Scope the task** | "add tests for foo.py" | "write a test for foo.py covering the edge case where the user is logged out. avoid mocks." |
| **Point to sources** | "why does ExecutionFactory have such a weird api?" | "look through ExecutionFactory's git history and summarize how its api came to be" |
| **Reference existing patterns** | "add a calendar widget" | "look at how existing widgets are implemented on the home page to understand the patterns. HotDogWidget.php is a good example. follow the pattern to implement a new calendar widget that lets the user select a month and paginate forwards/backwards to pick a year. build from scratch without libraries other than the ones already used in the codebase." |
| **Describe the symptom** | "fix the login bug" | "users report that login fails after session timeout. check the auth flow in src/auth/, especially token refresh. write a failing test that reproduces the issue, then fix it" |

## Verification Strategies

Claude performs dramatically better when it can verify its own work. Include tests, screenshots,
or expected outputs so Claude can check itself. This is the single highest-leverage thing you can
do.

Without clear success criteria, Claude might produce something that looks right but actually
doesn't work. You become the only feedback loop, and every mistake requires your attention.

| Strategy | Before | After |
|:---------|:-------|:------|
| **Provide verification criteria** | "implement a function that validates email addresses" | "write a validateEmail function. example test cases: user@example.com is true, invalid is false, user@.com is false. run the tests after implementing" |
| **Verify UI changes visually** | "make the dashboard look better" | "[paste screenshot] implement this design. take a screenshot of the result and compare it to the original. list differences and fix them" |
| **Address root causes, not symptoms** | "the build is failing" | "the build fails with this error: [paste error]. fix it and verify the build succeeds. address the root cause, don't suppress the error" |

Your verification can also be a test suite, a linter, or a Bash command that checks output. Invest
in making your verification rock-solid.

## Common Context-Related Failure Patterns

- **The kitchen sink session**: You start with one task, then ask something unrelated, then go
  back. Context is full of irrelevant information.
  **Fix**: `/clear` between unrelated tasks.

- **Correcting over and over**: Claude does something wrong, you correct it, still wrong, correct
  again. Context is polluted with failed approaches.
  **Fix**: After two failed corrections, `/clear` and write a better initial prompt.

- **The over-specified CLAUDE.md**: Too long, Claude ignores half of it because important rules
  get lost in the noise.
  **Fix**: Ruthlessly prune. If Claude already does something correctly without the instruction,
  delete it or convert it to a hook.

- **The infinite exploration**: You ask Claude to "investigate" something without scoping it.
  Claude reads hundreds of files, filling the context.
  **Fix**: Scope investigations narrowly or use subagents so the exploration doesn't consume your
  main context.
