# Prompt and Agent Hooks

How prompt-based hooks (`type: "prompt"`) and agent-based hooks (`type: "agent"`) work,
their differences from command hooks, when to use each, cost considerations, and examples.

## Overview

In addition to command hooks that run shell scripts, Claude Code supports two LLM-powered
hook types for decisions that require judgment rather than deterministic rules:

- **Prompt hooks** (`type: "prompt"`): Single-turn LLM evaluation. Fast and cheap.
- **Agent hooks** (`type: "agent"`): Multi-turn subagent with tool access. Thorough but slower.

Both return the same structured response: `{ "ok": true }` to allow or `{ "ok": false, "reason": "..." }`
to block.

## Supported Events

Prompt-based and agent-based hooks work with these events:

- `PreToolUse`
- `PostToolUse`
- `PostToolUseFailure`
- `PermissionRequest`
- `UserPromptSubmit`
- `Stop`
- `SubagentStop`
- `TaskCompleted`

`TeammateIdle` does **not** support prompt-based or agent-based hooks.

## Prompt Hooks (`type: "prompt"`)

### How They Work

1. Claude Code sends the hook input JSON and your prompt to a Claude model (Haiku by default)
2. The model responds with structured JSON containing a decision
3. Claude Code processes the decision automatically

### Configuration

| Field     | Required | Description                                                                              |
| :-------- | :------- | :--------------------------------------------------------------------------------------- |
| `type`    | yes      | Must be `"prompt"`                                                                       |
| `prompt`  | yes      | Prompt text. Use `$ARGUMENTS` as placeholder for hook input JSON. If `$ARGUMENTS` is absent, input JSON is appended |
| `model`   | no       | Model to use. Defaults to a fast model (Haiku)                                           |
| `timeout` | no       | Timeout in seconds. Default: 30                                                          |

### Response Schema

The LLM must respond with JSON:

```json
{
  "ok": true,
  "reason": "Explanation for the decision"
}
```

| Field    | Description                                                |
| :------- | :--------------------------------------------------------- |
| `ok`     | `true` allows the action, `false` prevents it              |
| `reason` | Required when `ok` is `false`. Explanation shown to Claude |

### Basic Example: Stop Hook

This hook asks the LLM whether all tasks are complete before allowing Claude to stop:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Check if all tasks are complete. If not, respond with {\"ok\": false, \"reason\": \"what remains to be done\"}."
          }
        ]
      }
    ]
  }
}
```

### Multi-Criteria Stop Hook

A more detailed prompt checking three conditions:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "You are evaluating whether Claude should stop working. Context: $ARGUMENTS\n\nAnalyze the conversation and determine if:\n1. All user-requested tasks are complete\n2. Any errors need to be addressed\n3. Follow-up work is needed\n\nRespond with JSON: {\"ok\": true} to allow stopping, or {\"ok\": false, \"reason\": \"your explanation\"} to continue working.",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

If the model returns `"ok": false`, Claude continues working and uses the `reason` as its
next instruction. `SubagentStop` hooks use the same format to evaluate whether a subagent
should stop.

## Agent Hooks (`type: "agent"`)

### How They Work

1. Claude Code spawns a subagent with your prompt and the hook's JSON input
2. The subagent can use tools: Read, Grep, Glob (up to 50 turns)
3. After investigation, the subagent returns a structured `{ "ok": true/false }` decision
4. Claude Code processes the decision the same way as a prompt hook

### Configuration

| Field     | Required | Description                                                                 |
| :-------- | :------- | :-------------------------------------------------------------------------- |
| `type`    | yes      | Must be `"agent"`                                                           |
| `prompt`  | yes      | Prompt describing what to verify. Use `$ARGUMENTS` for hook input JSON      |
| `model`   | no       | Model to use. Defaults to a fast model                                      |
| `timeout` | no       | Timeout in seconds. Default: 60                                             |

### Example: Verify Tests Pass

This hook verifies that all unit tests pass before allowing Claude to finish:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that all unit tests pass. Run the test suite and check the results. $ARGUMENTS",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

## Command vs Prompt vs Agent: When to Use Each

| Aspect              | Command Hook                | Prompt Hook                     | Agent Hook                          |
| :------------------ | :-------------------------- | :------------------------------ | :---------------------------------- |
| **Type**            | `"command"`                 | `"prompt"`                      | `"agent"`                           |
| **Execution**       | Runs a shell command        | Single LLM call                 | Multi-turn subagent with tools      |
| **Best for**        | Deterministic rules         | Judgment-based decisions        | Verification requiring file access  |
| **Speed**           | Fast (script execution)     | Fast (single API call)          | Slower (multiple turns)             |
| **Default timeout** | 600 seconds                 | 30 seconds                      | 60 seconds                          |
| **Tool access**     | Full system access          | None                            | Read, Grep, Glob                    |
| **Max turns**       | N/A                         | 1                               | 50                                  |
| **Async support**   | Yes                         | No                              | No                                  |
| **Input**           | JSON on stdin               | `$ARGUMENTS` placeholder        | `$ARGUMENTS` placeholder            |
| **Output**          | Exit codes + JSON stdout    | `{ "ok": bool, "reason": "." }` | `{ "ok": bool, "reason": "." }`    |

### Decision Guide

Use **command hooks** when:
- The decision is deterministic (pattern matching, file existence checks)
- You need to run external tools (formatters, linters, test suites)
- You need async/background execution
- You need full system access

Use **prompt hooks** when:
- The decision requires judgment or nuance
- The hook input data alone is sufficient to decide
- Speed matters (single LLM call is faster than spawning an agent)
- You want to evaluate conversation context or prompt content

Use **agent hooks** when:
- Verification requires inspecting actual files or test output
- The hook input data alone isn't enough; the agent needs to investigate
- You need multi-step verification (read files, search code, check state)
- Thoroughness matters more than speed

## Cost Considerations

- **Command hooks**: No LLM cost. Only compute/network costs of the script.
- **Prompt hooks**: One LLM API call per hook invocation. Uses Haiku by default (cheapest).
  Consider how frequently the event fires. A `Stop` hook fires once per response turn. A
  `PreToolUse` hook on `Bash` fires before every Bash command, which could be many times per session.
- **Agent hooks**: Multiple LLM API calls per invocation (up to 50 turns). More expensive than
  prompt hooks. Use judiciously on frequently-firing events.

To control costs:
- Use the cheapest sufficient hook type (command > prompt > agent)
- Use matchers to narrow when hooks fire
- Set appropriate timeouts to limit agent hook duration
- Avoid agent hooks on high-frequency events like `PreToolUse` unless necessary
- Use the `model` field to select faster/cheaper models when full capability isn't needed

## Common Patterns

### PreToolUse Prompt Hook: Validate Code Changes

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Review this code change for security issues. Tool: $ARGUMENTS. Check for SQL injection, XSS, command injection, and hardcoded credentials. Respond {\"ok\": true} if safe, {\"ok\": false, \"reason\": \"description of issue\"} if not."
          }
        ]
      }
    ]
  }
}
```

### Stop Agent Hook: Verify Implementation

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Before Claude stops, verify: 1) All files mentioned in the task exist, 2) No TODO comments remain in modified files, 3) Code compiles without errors. Context: $ARGUMENTS",
            "timeout": 90
          }
        ]
      }
    ]
  }
}
```

### UserPromptSubmit Prompt Hook: Content Filtering

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Check if this prompt asks Claude to do something outside the scope of the current project. Prompt: $ARGUMENTS. If it's on-topic, respond {\"ok\": true}. If off-topic, respond {\"ok\": false, \"reason\": \"Please keep requests focused on the current project.\"}."
          }
        ]
      }
    ]
  }
}
```
