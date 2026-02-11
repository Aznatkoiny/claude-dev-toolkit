# Parallel Sessions

## Running Multiple Claude Sessions

Once you're effective with one Claude, multiply your output with parallel sessions, headless mode,
and fan-out patterns.

There are three main ways to run parallel sessions:

- **Claude Desktop**: Manage multiple local sessions visually. Each session gets its own isolated
  worktree.
- **Claude Code on the web**: Run on Anthropic's secure cloud infrastructure in isolated VMs.
- **Agent teams**: Automated coordination of multiple sessions with shared tasks, messaging, and
  a team lead.

## When Parallel Is Better Than Sequential

Parallel sessions work best when:

- Tasks are independent and don't modify the same files
- You need to explore multiple approaches to a problem
- You want to separate writing from reviewing
- You're running large migrations across many files
- You need isolated experiments that won't interfere with each other

Sequential is better when:

- Tasks have dependencies (one must complete before the next)
- Changes affect shared state or the same files
- You need to build on the output of a previous task
- The overall approach needs to be validated before expanding

## The Writer/Reviewer Pattern

A fresh context improves code review since Claude won't be biased toward code it just wrote.
Use two sessions with different roles:

| Session A (Writer) | Session B (Reviewer) |
|:-------------------|:---------------------|
| `Implement a rate limiter for our API endpoints` | |
| | `Review the rate limiter implementation in @src/middleware/rateLimiter.ts. Look for edge cases, race conditions, and consistency with our existing middleware patterns.` |
| `Here's the review feedback: [Session B output]. Address these issues.` | |

You can do something similar with tests: have one Claude write tests, then another write code to
pass them.

## Git Worktrees for Parallel Work

Give each session its own working directory using git worktrees:

```bash
git worktree add ../my-project-feature-a feature-a
git worktree add ../my-project-feature-b feature-b
```

Each worktree has its own checkout of the repository, so sessions can work on different branches
without interfering. This avoids merge conflicts and file contention between parallel sessions.

## Task Decomposition

Break large tasks into independent units that can be parallelized:

1. **Identify independent subtasks**: Look for work that doesn't share files or state
2. **Define clear boundaries**: Each subtask should have a clear input and output
3. **Merge results**: After parallel sessions complete, integrate changes

Example decomposition for adding a new feature:

- Session 1: Implement the backend API endpoint
- Session 2: Write the frontend component
- Session 3: Write tests for the API
- Session 4: Write tests for the frontend component

Then merge the branches and run integration tests.

## Fan-Out Across Files

For large migrations or analyses, distribute work across many parallel Claude invocations:

### Step 1: Generate a Task List

Have Claude list all files that need migrating (e.g., "list all 2,000 Python files that need
migrating").

### Step 2: Write a Script to Loop Through the List

```bash
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```

### Step 3: Test on a Few Files, Then Run at Scale

Refine your prompt based on what goes wrong with the first 2-3 files, then run on the full set.
The `--allowedTools` flag restricts what Claude can do, which matters when running unattended.

## Headless Mode

Use `claude -p "prompt"` in CI, pre-commit hooks, or scripts. Add `--output-format stream-json`
for streaming JSON output.

With `claude -p "your prompt"`, you can run Claude headlessly without an interactive session.
Headless mode is how you integrate Claude into CI pipelines, pre-commit hooks, or any automated
workflow. The output formats (plain text, JSON, streaming JSON) let you parse results
programmatically.

```bash
# One-off queries
claude -p "Explain what this project does"

# Structured output for scripts
claude -p "List all API endpoints" --output-format json

# Streaming for real-time processing
claude -p "Analyze this log file" --output-format stream-json
```

You can also integrate Claude into existing data/processing pipelines:

```bash
claude -p "<your prompt>" --output-format json | your_command
```

Use `--verbose` for debugging during development, and turn it off in production.

## Coordination Strategies

### Shared Git Repository

The simplest coordination method. Each session works on a separate branch. Merge when done.

### Agent Teams

For automated coordination, use agent teams. A team lead coordinates multiple sessions with
shared tasks, messaging, and automatic task assignment.

### Manual Coordination

For two or three sessions, manual coordination works well:

1. Start each session with a clear, scoped task
2. Let sessions run independently
3. Review output from each session
4. Feed review feedback back into sessions as needed
5. Merge results

## Safe Autonomous Mode

Use `claude --dangerously-skip-permissions` to bypass all permission checks and let Claude work
uninterrupted. This works well for contained workflows like fixing lint errors or generating
boilerplate code.

**Warning**: Letting Claude run arbitrary commands is risky and can result in data loss, system
corruption, or data exfiltration (e.g., via prompt injection attacks). To minimize these risks,
use `--dangerously-skip-permissions` in a container without internet access.

With sandboxing enabled (`/sandbox`), you get similar autonomy with better security. Sandbox
defines upfront boundaries rather than bypassing all checks.

## Permissions for Batch Operations

Use `--allowedTools` to scope permissions when running batch operations:

```bash
claude -p "Fix lint errors in $file" \
  --allowedTools "Edit,Bash(npm run lint)"
```

This restricts Claude to only the tools it needs, reducing risk when running unattended across
many files.
