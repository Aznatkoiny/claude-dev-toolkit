# Prompt Engineering for Claude Code

## Being Specific vs Vague

Claude can infer intent, but it can't read your mind. Reference specific files, mention
constraints, and point to example patterns. The more precise your instructions, the fewer
corrections you'll need.

| Strategy | Before | After |
|:---------|:-------|:------|
| **Scope the task.** Specify which file, what scenario, and testing preferences. | "add tests for foo.py" | "write a test for foo.py covering the edge case where the user is logged out. avoid mocks." |
| **Point to sources.** Direct Claude to the source that can answer a question. | "why does ExecutionFactory have such a weird api?" | "look through ExecutionFactory's git history and summarize how its api came to be" |
| **Reference existing patterns.** Point Claude to patterns in your codebase. | "add a calendar widget" | "look at how existing widgets are implemented on the home page to understand the patterns. HotDogWidget.php is a good example. follow the pattern to implement a new calendar widget that lets the user select a month and paginate forwards/backwards to pick a year. build from scratch without libraries other than the ones already used in the codebase." |
| **Describe the symptom.** Provide the symptom, the likely location, and what "fixed" looks like. | "fix the login bug" | "users report that login fails after session timeout. check the auth flow in src/auth/, especially token refresh. write a failing test that reproduces the issue, then fix it" |

Vague prompts can be useful when you're exploring and can afford to course-correct. A prompt like
"what would you improve in this file?" can surface things you wouldn't have thought to ask about.

## Providing Rich Content

You can provide rich data to Claude in several ways:

- **Reference files with `@`** instead of describing where code lives. Claude reads the file
  before responding.
- **Paste images directly**. Copy/paste or drag and drop images into the prompt.
- **Give URLs** for documentation and API references. Use `/permissions` to allowlist
  frequently-used domains.
- **Pipe in data** by running `cat error.log | claude` to send file contents directly.
- **Let Claude fetch what it needs**. Tell Claude to pull context itself using Bash commands, MCP
  tools, or by reading files.

## The Explore-Plan-Code Workflow

Letting Claude jump straight to coding can produce code that solves the wrong problem. Use Plan
Mode to separate exploration from execution.

The recommended workflow has four phases:

### Phase 1: Explore

Enter Plan Mode. Claude reads files and answers questions without making changes.

```
read /src/auth and understand how we handle sessions and login.
also look at how we manage environment variables for secrets.
```

### Phase 2: Plan

Ask Claude to create a detailed implementation plan.

```
I want to add Google OAuth. What files need to change?
What's the session flow? Create a plan.
```

Press `Ctrl+G` to open the plan in your text editor for direct editing before Claude proceeds.

### Phase 3: Implement

Switch back to Normal Mode and let Claude code, verifying against its plan.

```
implement the OAuth flow from your plan. write tests for the
callback handler, run the test suite and fix any failures.
```

### Phase 4: Commit

Ask Claude to commit with a descriptive message and create a PR.

```
commit with a descriptive message and open a PR
```

### When to Skip Planning

Plan Mode is useful, but also adds overhead. For tasks where the scope is clear and the fix is
small (like fixing a typo, adding a log line, or renaming a variable) ask Claude to do it directly.

Planning is most useful when:

- You're uncertain about the approach
- The change modifies multiple files
- You're unfamiliar with the code being modified

If you could describe the diff in one sentence, skip the plan.

## Asking Codebase Questions

When onboarding to a new codebase, use Claude Code for learning and exploration. You can ask
Claude the same sorts of questions you would ask another engineer:

- How does logging work?
- How do I make a new API endpoint?
- What does `async move { ... }` do on line 134 of `foo.rs`?
- What edge cases does `CustomerOnboardingFlowImpl` handle?
- Why does this code call `foo()` instead of `bar()` on line 333?

Using Claude Code this way is an effective onboarding workflow, improving ramp-up time and
reducing load on other engineers. No special prompting required -- ask questions directly.

## The Interview Technique

For larger features, have Claude interview you first. Start with a minimal prompt and ask Claude
to interview you using the `AskUserQuestion` tool:

```
I want to build [brief description]. Interview me in detail using the AskUserQuestion tool.

Ask about technical implementation, UI/UX, edge cases, concerns, and tradeoffs. Don't ask obvious
questions, dig into the hard parts I might not have considered.

Keep interviewing until we've covered everything, then write a complete spec to SPEC.md.
```

Claude asks about things you might not have considered yet, including technical implementation,
UI/UX, edge cases, and tradeoffs.

Once the spec is complete, start a fresh session to execute it. The new session has clean context
focused entirely on implementation, and you have a written spec to reference.

## Debugging Patterns

### Verification-First Debugging

Always provide verification criteria so Claude can check its own work:

| Strategy | Before | After |
|:---------|:-------|:------|
| **Provide verification criteria** | "implement a function that validates email addresses" | "write a validateEmail function. example test cases: user@example.com is true, invalid is false, user@.com is false. run the tests after implementing" |
| **Verify UI changes visually** | "make the dashboard look better" | "[paste screenshot] implement this design. take a screenshot of the result and compare it to the original. list differences and fix them" |
| **Address root causes, not symptoms** | "the build is failing" | "the build fails with this error: [paste error]. fix it and verify the build succeeds. address the root cause, don't suppress the error" |

### Using Screenshots and Images

For UI work, paste screenshots directly into Claude Code. You can:

- Paste a design mockup and ask Claude to implement it
- Ask Claude to take a screenshot of its implementation and compare to the original
- Use the Claude in Chrome extension to open new tabs, test the UI, and iterate until it works

### Providing Environment Details

When reporting issues, include:

- The exact error message (paste it directly)
- Relevant file paths
- What you expected vs what happened
- Steps to reproduce

The more context you provide upfront, the fewer round trips needed to diagnose the problem.

## Communication Tips

### Course-Correct Early and Often

The best results come from tight feedback loops:

- **`Esc`**: Stop Claude mid-action with the Esc key. Context is preserved, so you can redirect.
- **`Esc + Esc` or `/rewind`**: Open the rewind menu and restore previous conversation and code
  state, or summarize from a selected message.
- **"Undo that"**: Have Claude revert its changes.
- **`/clear`**: Reset context between unrelated tasks. Long sessions with irrelevant context can
  reduce performance.

If you've corrected Claude more than twice on the same issue in one session, the context is
cluttered with failed approaches. Run `/clear` and start fresh with a more specific prompt that
incorporates what you learned. A clean session with a better prompt almost always outperforms a
long session with accumulated corrections.

### CLI Tools for External Services

Tell Claude Code to use CLI tools like `gh`, `aws`, `gcloud`, and `sentry-cli` when interacting
with external services. CLI tools are the most context-efficient way to interact with external
services.

If you use GitHub, install the `gh` CLI. Claude knows how to use it for creating issues, opening
pull requests, and reading comments. Without `gh`, Claude can still use the GitHub API, but
unauthenticated requests often hit rate limits.

Claude is also effective at learning CLI tools it doesn't already know. Try prompts like:

```
Use 'foo-cli-tool --help' to learn about foo tool, then use it to solve A, B, C.
```
