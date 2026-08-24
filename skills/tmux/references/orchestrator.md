# Orchestrator patterns

Detailed workflows for the orchestrator window. See the "Orchestrator pattern"
section of `SKILL.md` for what the orchestrator is.

## Bootstrap sessions

Spin up new windows for the user when they want to start working on an issue.
Create the worktree (if applicable), open the window, set the name, start the
harness, and send an initial prompt that gives the session context about its
task. See `SKILL.md` for picking the harness, waiting for its TUI, and the
two-call `send-keys` pattern.

## Status sweep

When the user asks what's going on across their sessions (or you think a check-in
would be helpful), scan all non-orchestrator windows and summarize:

```bash
# Get IDs and names of all windows except the current (orchestrator) one
tmux list-windows -F '#{?window_active,,#{window_id} #{window_name}}'

# Peek at the last ~30 lines of a window's output
tmux capture-pane -t "$WID" -p -S -30 | tail -n 30
```

`capture-pane -S -30` starts the capture 30 lines into the scrollback and ends
at the bottom of the visible pane, so the raw output is typically 30 + pane
height lines. Pipe through `tail -n 30` if you actually want 30 lines.

Read each pane's recent output and give a concise summary, e.g.:

> - `asdf-123-auth`: running tests, 2 failing
> - `asdf-456-migration`: idle, composer waiting for input
> - `asdf-789-api`: mid-conversation, discussing schema design

Different harnesses render progress, tool calls, and idle state differently, so
read what the pane actually shows instead of pattern-matching one harness's
layout. A window sitting at a shell prompt means its harness exited: worth
flagging, since the user probably thinks a session is still there.

## Stale session detection

A session is stale when its composer is visible and empty (no in-flight tool
calls, no streaming output) *and* the pane has not changed between two captures
taken a few seconds apart. Concretely: capture-pane output is byte-identical
across two reads, and the bottom of the pane shows an idle composer rather than
a progress or "interrupt to stop" indicator. Each harness words those indicators
differently; the byte-identical check is the harness-agnostic half, so lean on
it and use the visible state to confirm. Flag it: "asdf-456 looks idle, still
need that one?" This helps the user notice sessions they may have forgotten
about.

## Context briefing

When the user is about to switch to another window, capture that pane's recent
output and summarize where the conversation left off. This saves them from
scrolling through history to remember what they were doing.

```bash
tmux capture-pane -t "$WID" -p -S -50 | tail -n 50
```

## Session inventory

Maintain a quick overview mapping window names to tickets, branch names,
worktree paths, and which harness is running in each:

```bash
tmux list-windows -F '#{window_index}: #{window_name} (#{pane_current_path})'
```

tmux cannot tell you reliably which harness a window is running (see "List
windows" in `SKILL.md`), so record that when you start the session. In a mixed
fleet it matters: it decides how you read the pane and how you talk about the
session to the user.

## Clean up

When the user is done with a session, kill the window. If they mention cleaning
up worktrees, compose with the worktree-cleanup skill for that.
