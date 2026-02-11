# Agent Teams Reference

Complete reference for orchestrating teams of Claude Code sessions working together.

## Overview

Agent teams let you coordinate multiple Claude Code instances working together. One session acts as the team lead, coordinating work, assigning tasks, and synthesizing results. Teammates work independently, each in its own context window, and communicate directly with each other.

Unlike subagents (which run within a single session and can only report back to the main agent), you can interact with individual teammates directly without going through the lead.

## Experimental Status

Agent teams are experimental and disabled by default. Enable them by setting `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` to `1`:

### Via settings.json

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Via Environment Variable

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

## Architecture

An agent team consists of:

| Component | Role |
|:---|:---|
| **Team lead** | The main Claude Code session that creates the team, spawns teammates, and coordinates work |
| **Teammates** | Separate Claude Code instances that each work on assigned tasks |
| **Task list** | Shared list of work items that teammates claim and complete |
| **Mailbox** | Messaging system for communication between agents |

Teams and tasks are stored locally:
- **Team config**: `~/.claude/teams/{team-name}/config.json`
- **Task list**: `~/.claude/tasks/{team-name}/`

The team config contains a `members` array with each teammate's name, agent ID, and agent type. Teammates can read this file to discover other team members.

## Starting a Team

Tell Claude to create a team and describe the task and structure in natural language:

```
I'm designing a CLI tool that helps developers track TODO comments across
their codebase. Create an agent team to explore this from different angles: one
teammate on UX, one on technical architecture, one playing devil's advocate.
```

Claude creates the team, spawns teammates, and coordinates work based on your prompt.

### How Teams Get Started

- **You request a team**: give Claude a task and explicitly ask for an agent team
- **Claude proposes a team**: if Claude determines your task would benefit from parallel work, it may suggest creating a team. You confirm before it proceeds.

Claude will not create a team without your approval.

## Display Modes

Agent teams support two display modes:

### In-Process Mode

All teammates run inside your main terminal.
- Use **Shift+Up/Down** to select a teammate
- Type to message teammates directly
- Press **Enter** to view a teammate's session
- Press **Escape** to interrupt their current turn
- Press **Ctrl+T** to toggle the task list
- Works in any terminal, no extra setup required

### Split-Pane Mode

Each teammate gets its own pane.
- See everyone's output at once
- Click into a pane to interact directly
- Requires tmux or iTerm2

### Configuration

The default is `"auto"`, which uses split panes if you are already inside a tmux session, and in-process otherwise. The `"tmux"` setting enables split-pane mode and auto-detects whether to use tmux or iTerm2.

In `settings.json`:

```json
{
  "teammateMode": "in-process"
}
```

Or per session:

```bash
claude --teammate-mode in-process
```

### Split-Pane Requirements

- **tmux**: install through your system's package manager
- **iTerm2**: install the `it2` CLI, then enable the Python API in iTerm2 Settings > General > Magic > Enable Python API

Note: `tmux` has known limitations on certain operating systems and traditionally works best on macOS. Using `tmux -CC` in iTerm2 is the suggested entrypoint into tmux.

## Specifying Teammates and Models

Claude decides the number of teammates based on your task, or you can specify exactly:

```
Create a team with 4 teammates to refactor these modules in parallel.
Use Sonnet for each teammate.
```

## Task Assignment and Claiming

The shared task list coordinates work across the team. Tasks have three states: **pending**, **in progress**, and **completed**. Tasks can depend on other tasks -- a pending task with unresolved dependencies cannot be claimed until those dependencies are completed.

- **Lead assigns**: tell the lead which task to give to which teammate
- **Self-claim**: after finishing a task, a teammate picks up the next unassigned, unblocked task

Task claiming uses file locking to prevent race conditions when multiple teammates try to claim the same task simultaneously.

## Plan Approval Workflow

For complex or risky tasks, require teammates to plan before implementing:

```
Spawn an architect teammate to refactor the authentication module.
Require plan approval before they make any changes.
```

The workflow:
1. Teammate works in read-only plan mode
2. Teammate sends plan approval request to lead when planning is complete
3. Lead reviews and approves or rejects with feedback
4. If rejected, teammate revises and resubmits
5. Once approved, teammate exits plan mode and begins implementation

The lead makes approval decisions autonomously. Influence its judgment with criteria: "only approve plans that include test coverage" or "reject plans that modify the database schema."

## Delegate Mode

Without delegate mode, the lead sometimes starts implementing tasks itself instead of waiting for teammates. Delegate mode restricts the lead to coordination-only tools: spawning, messaging, shutting down teammates, and managing tasks.

Useful when you want the lead to focus entirely on orchestration.

To enable: start a team first, then press **Shift+Tab** to cycle into delegate mode.

## Teammate Communication

### Direct Messaging

- **message**: send a message to one specific teammate
- **broadcast**: send to all teammates simultaneously (use sparingly, costs scale with team size)

### Automatic Notifications

- **Automatic message delivery**: when teammates send messages, they are delivered automatically to recipients
- **Idle notifications**: when a teammate finishes and stops, they automatically notify the lead
- **Shared task list**: all agents can see task status and claim available work

## Permissions in Teams

Teammates start with the lead's permission settings. If the lead runs with `--dangerously-skip-permissions`, all teammates do too. After spawning, you can change individual teammate modes, but you cannot set per-teammate modes at spawn time.

## Context and Communication

Each teammate has its own context window. When spawned, a teammate loads the same project context as a regular session: CLAUDE.md, MCP servers, and skills. It also receives the spawn prompt from the lead. The lead's conversation history does not carry over.

## Shutting Down Teammates

```
Ask the researcher teammate to shut down
```

The lead sends a shutdown request. The teammate can approve (exiting gracefully) or reject with an explanation.

## Cleaning Up the Team

```
Clean up the team
```

This removes shared team resources. The lead checks for active teammates and fails if any are still running -- shut them down first.

Always use the lead to clean up. Teammates should not run cleanup because their team context may not resolve correctly, potentially leaving resources in an inconsistent state.

## Quality Gates with Hooks

Use hooks to enforce rules when teammates finish work or tasks complete:

- **`TeammateIdle`**: runs when a teammate is about to go idle. Exit with code 2 to send feedback and keep the teammate working.
- **`TaskCompleted`**: runs when a task is being marked complete. Exit with code 2 to prevent completion and send feedback.

## Token Usage

Agent teams use significantly more tokens than a single session. Each teammate has its own context window, and token usage scales with the number of active teammates.

- For research, review, and new feature work: the extra tokens are usually worthwhile
- For routine tasks: a single session is more cost-effective

## Use Case Examples

### Parallel Code Review

Split review criteria into independent domains for thorough simultaneous attention:

```
Create an agent team to review PR #142. Spawn three reviewers:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings.
```

### Competing Hypotheses Investigation

When root cause is unclear, make teammates adversarial:

```
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
each other to try to disprove each other's theories, like a scientific
debate. Update the findings doc with whatever consensus emerges.
```

## Best Practices

### Give Teammates Enough Context

Include task-specific details in the spawn prompt since the lead's conversation history does not carry over:

```
Spawn a security reviewer teammate with the prompt: "Review the authentication module
at src/auth/ for security vulnerabilities. Focus on token handling, session
management, and input validation. The app uses JWT tokens stored in
httpOnly cookies. Report any issues with severity ratings."
```

### Size Tasks Appropriately

- **Too small**: coordination overhead exceeds the benefit
- **Too large**: teammates work too long without check-ins
- **Just right**: self-contained units that produce a clear deliverable

Having 5-6 tasks per teammate keeps everyone productive and lets the lead reassign work if someone gets stuck.

### Avoid File Conflicts

Two teammates editing the same file leads to overwrites. Break work so each teammate owns a different set of files.

### Monitor and Steer

Check in on progress, redirect approaches that are not working, and synthesize findings as they come in. Letting a team run unattended too long increases risk of wasted effort.

### Start with Research and Review

If you are new to agent teams, start with tasks that have clear boundaries and don't require writing code: reviewing a PR, researching a library, or investigating a bug.

## Known Limitations

- **No session resumption with in-process teammates**: `/resume` and `/rewind` do not restore in-process teammates. After resuming, tell the lead to spawn new teammates.
- **Task status can lag**: teammates sometimes fail to mark tasks as completed. Check whether work is done and update manually or nudge the teammate.
- **Shutdown can be slow**: teammates finish their current request or tool call before shutting down.
- **One team per session**: clean up the current team before starting a new one.
- **No nested teams**: teammates cannot spawn their own teams. Only the lead can manage the team.
- **Lead is fixed**: the session that creates the team is the lead for its lifetime. You cannot promote a teammate to lead.
- **Permissions set at spawn**: all teammates start with the lead's permission mode. You can change individual teammate modes after spawning.
- **Split panes require tmux or iTerm2**: the default in-process mode works in any terminal. Split-pane mode is not supported in VS Code's integrated terminal, Windows Terminal, or Ghostty.

`CLAUDE.md` works normally: teammates read CLAUDE.md files from their working directory.

## Troubleshooting

### Teammates Not Appearing

- In in-process mode, press **Shift+Down** to cycle through active teammates
- Ensure the task was complex enough to warrant a team
- For split panes, verify tmux is installed: `which tmux`
- For iTerm2, verify the `it2` CLI is installed and the Python API is enabled

### Too Many Permission Prompts

Pre-approve common operations in your permission settings before spawning teammates.

### Teammates Stopping on Errors

Check output using Shift+Up/Down or by clicking the pane, then give additional instructions or spawn a replacement.

### Lead Shuts Down Before Work Is Done

Tell the lead to keep going or wait for teammates to finish before proceeding.

### Orphaned tmux Sessions

```bash
tmux ls
tmux kill-session -t <session-name>
```
