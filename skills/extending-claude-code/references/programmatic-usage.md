# Programmatic Usage Reference

Complete guide to running Claude Code programmatically from the CLI using the Agent SDK.

The Agent SDK gives you the same tools, agent loop, and context management that power Claude Code. It's available as a CLI for scripts and CI/CD, or as Python and TypeScript packages for full programmatic control.

Note: The CLI was previously called "headless mode." The `-p` flag and all CLI options work the same way.

## Basic Usage

To run Claude Code programmatically from the CLI, pass `-p` with your prompt and any CLI options:

```bash
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"
```

Add the `-p` (or `--print`) flag to any `claude` command to run it non-interactively. All CLI options work with `-p`, including:

- `--continue` for continuing conversations
- `--allowedTools` for auto-approving tools
- `--output-format` for structured output

This example asks Claude a question about your codebase and prints the response:

```bash
claude -p "What does the auth module do?"
```

## Output Formats

Use `--output-format` to control how responses are returned:

- `text` (default): plain text output
- `json`: structured JSON with result, session ID, and metadata
- `stream-json`: newline-delimited JSON for real-time streaming

### JSON Output

This example returns a project summary as JSON with session metadata, with the text result in the `result` field:

```bash
claude -p "Summarize this project" --output-format json
```

### Structured Output with JSON Schema

To get output conforming to a specific schema, use `--output-format json` with `--json-schema` and a JSON Schema definition. The response includes metadata about the request (session ID, usage, etc.) with the structured output in the `structured_output` field.

This example extracts function names and returns them as an array of strings:

```bash
claude -p "Extract the main function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}'
```

### Parsing JSON Output with jq

Use a tool like jq to parse the response and extract specific fields:

```bash
# Extract the text result
claude -p "Summarize this project" --output-format json | jq -r '.result'

# Extract structured output
claude -p "Extract function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  | jq '.structured_output'
```

## Streaming Responses

Use `--output-format stream-json` with `--verbose` and `--include-partial-messages` to receive tokens as they're generated. Each line is a JSON object representing an event:

```bash
claude -p "Explain recursion" --output-format stream-json --verbose --include-partial-messages
```

The following example uses jq to filter for text deltas and display just the streaming text. The `-r` flag outputs raw strings (no quotes) and `-j` joins without newlines so tokens stream continuously:

```bash
claude -p "Write a poem" --output-format stream-json --verbose --include-partial-messages | \
  jq -rj 'select(.type == "stream_event" and .event.delta.type? == "text_delta") | .event.delta.text'
```

For programmatic streaming with callbacks and message objects, see the Agent SDK documentation on streaming output.

## Auto-Approve Tools

Use `--allowedTools` to let Claude use certain tools without prompting. This example runs a test suite and fixes failures, allowing Claude to execute Bash commands and read/edit files without asking for permission:

```bash
claude -p "Run the test suite and fix any failures" \
  --allowedTools "Bash,Read,Edit"
```

## Creating Commits

This example reviews staged changes and creates a commit with an appropriate message:

```bash
claude -p "Look at my staged changes and create an appropriate commit" \
  --allowedTools "Bash(git diff *),Bash(git log *),Bash(git status *),Bash(git commit *)"
```

The `--allowedTools` flag uses permission rule syntax. The trailing ` *` enables prefix matching, so `Bash(git diff *)` allows any command starting with `git diff`. The space before `*` is important: without it, `Bash(git diff*)` would also match `git diff-index`.

Note: User-invoked skills like `/commit` and built-in commands are only available in interactive mode. In `-p` mode, describe the task you want to accomplish instead.

## Customizing the System Prompt

Use `--append-system-prompt` to add instructions while keeping Claude Code's default behavior. This example pipes a PR diff to Claude and instructs it to review for security vulnerabilities:

```bash
gh pr diff "$1" | claude -p \
  --append-system-prompt "You are a security engineer. Review for vulnerabilities." \
  --output-format json
```

Use `--system-prompt` to fully replace the default prompt instead of appending.

## Continuing Conversations

Use `--continue` to continue the most recent conversation, or `--resume` with a session ID to continue a specific conversation.

### Basic continuation

```bash
# First request
claude -p "Review this codebase for performance issues"

# Continue the most recent conversation
claude -p "Now focus on the database queries" --continue
claude -p "Generate a summary of all issues found" --continue
```

### Resuming a specific session

If you're running multiple conversations, capture the session ID to resume a specific one:

```bash
session_id=$(claude -p "Start a review" --output-format json | jq -r '.session_id')
claude -p "Continue that review" --resume "$session_id"
```

## Multi-Turn Pipelines

Combine continuation with structured output for multi-step workflows:

```bash
# Step 1: Analyze the codebase
session_id=$(claude -p "Analyze this codebase for security issues" \
  --output-format json | jq -r '.session_id')

# Step 2: Focus on findings
claude -p "Which of the issues you found are critical?" \
  --resume "$session_id" --output-format json | jq -r '.result'

# Step 3: Generate fixes
claude -p "Generate fixes for the critical issues" \
  --resume "$session_id" \
  --allowedTools "Read,Edit"
```

## CI/CD Integration Patterns

### GitHub Actions

Use the Agent SDK in GitHub workflows by running `claude -p` with appropriate tool permissions:

```bash
claude -p "Run the test suite and fix any failures" \
  --allowedTools "Bash,Read,Edit" \
  --output-format json
```

### GitLab CI/CD

Use the Agent SDK in GitLab pipelines in the same way:

```bash
claude -p "Review the changes in this merge request" \
  --append-system-prompt "You are a code reviewer. Focus on correctness and security." \
  --output-format json
```

### Common CI/CD patterns

- **Automated code review:** Pipe PR diffs to Claude with `--append-system-prompt` for reviewer instructions
- **Test fixing:** Run `claude -p` with Bash/Read/Edit tools to diagnose and fix failing tests
- **Commit generation:** Use `--allowedTools` with git commands to create well-formatted commits
- **Documentation generation:** Use `--output-format json` with `--json-schema` for structured documentation output

## CLI Flags Summary

| Flag                         | Purpose                                          |
| ---------------------------- | ------------------------------------------------ |
| `-p`, `--print`              | Run non-interactively (required for programmatic use) |
| `--output-format`            | Output format: `text`, `json`, `stream-json`     |
| `--json-schema`              | JSON Schema for structured output                |
| `--allowedTools`             | Auto-approve specific tools                      |
| `--continue`                 | Continue most recent conversation                |
| `--resume`                   | Resume a specific session by ID                  |
| `--append-system-prompt`     | Add to the default system prompt                 |
| `--system-prompt`            | Replace the default system prompt entirely       |
| `--verbose`                  | Include detailed event info (useful with streaming) |
| `--include-partial-messages` | Include partial messages in streaming output     |
