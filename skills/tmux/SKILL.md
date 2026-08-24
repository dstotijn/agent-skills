---
name: tmux
description: >
  Manage tmux windows to run and coordinate multiple CLI coding agent sessions.
  Use when the user mentions tmux, wants to open, list, kill, or rename a
  window, send a command or prompt to another window, spin up a worktree with an
  agent running in it (e.g. "tmux worktree for ASDF-123"), or run an
  orchestrator workflow coordinating parallel agent sessions.
---

# tmux Session Management

Manage tmux windows to run multiple agent sessions from an orchestrator window.
This skill assumes it is always running inside tmux (`$TMUX` is set).

**"Harness"** here means a CLI coding agent that takes over the terminal with
its own TUI. This skill names no harness on purpose: the user runs more than
one, a single orchestrator often drives a mix of them, and the tmux mechanics
are the same for all. Where behavior genuinely differs between harnesses, the
skill says how to find out rather than baking in one harness's quirk.

## Which harness to launch

1. If the user named one, use that.
2. Otherwise default to the harness this session is running in. You know which
   one that is without probing for it.
3. If the request is ambiguous and the choice matters, ask.

**Do not identify a harness from environment variables.** Harness marker
variables are inherited by child processes, so a session started from inside
another harness's window carries the parent's markers. They tell you what
started the process tree, not what is running in it.

Check the command exists before sending it:

```bash
command -v <harness-command>
```

A typo or a harness that is not installed just leaves a shell error in a window
you are not looking at, and the prompt you send next goes to that shell.

## Starting a harness in a new window

### Creating a window and launching the harness

Use a two-step approach (create the window, then send the command) so the shell
stays alive after the harness exits. Always pass `-d` to `new-window` so the
orchestrator window stays focused: the user is talking to you here and should
not be yanked into the new window mid-conversation. If they later want to switch
to it, they'll ask, and you use `tmux select-window` then. Capture the new
window's ID with `-P -F` and use it as the target for follow-up commands instead
of the name:

```bash
WID=$(tmux new-window -d -P -F '#{window_id}' -n "<name>" -c "<directory>")
tmux set-option -w -t "$WID" automatic-rename on
tmux send-keys -t "$WID" '<harness-command>' Enter
```

**Why re-enable `automatic-rename`:** passing `-n` sets `automatic-rename off`
for that window, which pins the name for the window's whole lifetime. The `-n`
name is only meant as a provisional label until the pane reports a real title,
so hand control straight back. Skip the `set-option` line only if the user asked
for a name that should stick.

**Why target by ID, not name:** tmux window names are not unique. If two windows
share a name, `tmux send-keys -t "<name>"` resolves to the lowest-indexed match
silently, which makes it easy to send a prompt to the wrong session. Window IDs
(`@N`-form) are stable for the window's lifetime and immune to renames or
duplicates. Use IDs whenever you need to act on a specific window you just
created or looked up. Name targeting is fine only for ad-hoc commands where the
user has specified the name explicitly and uniqueness is obvious.

### Sending an initial prompt

Launch the harness bare and send the prompt through its composer once it is up,
rather than passing the prompt as a CLI argument. Argument syntax and flags vary
per harness; the composer works the same everywhere.

The harness needs a moment to start, and anything sent before its TUI is ready
lands in the shell instead. Confirm it is up before sending:

```bash
tmux capture-pane -t "$WID" -p -S -20 | tail -n 20
```

Look for the harness's own UI (its banner, its composer prompt) rather than a
shell prompt. If it isn't up yet, wait a couple of seconds and capture again.
Then send the prompt with the two-call pattern described under
[Send text to another window](#send-text-to-another-window).

### Window name from branch name

Set the window name (the `-n` flag, shown in the status bar via
`#{window_name}`) to the branch name. Whether that name survives depends on the
harness: with `automatic-rename on` and an `automatic-rename-format` of
`#{pane_title}`, tmux replaces the name with whatever the pane reports via
OSC 2, and harnesses differ in what they report. Some report their current task,
which supersedes the branch name within a second of startup. Some report the
working directory, which looks roughly like the name you set. Some report
nothing at all, and the `-n` name is the only thing the user ever sees. Pick a
good one regardless, then look at the window list once to see which of the three
you are dealing with.

If the branch name is longer than ~30 characters and starts with a ticket ID
(e.g., `asdf-123-`), shorten the descriptive part while always preserving the ID:

| Branch name                              | Window name                |
|------------------------------------------|----------------------------|
| `asdf-123-auth-refactor`                 | `asdf-123-auth-refactor`   |
| `asdf-456-implement-real-time-collab-v2` | `asdf-456-realtime-collab` |
| `feature/quick-fix`                      | `feature/quick-fix`        |

When shortening: drop filler words, abbreviate where obvious, keep it recognizable.

## Worktree + tmux flow

When the user asks something like "tmux worktree for asdf-123" or "open a window
for this branch", the expected sequence is:

1. **Create the worktree** using the project's worktree skill or script. Many
   projects have a worktree creation script (e.g., `mise run worktree:new <branch>`).
   Check the project's AGENTS.md, mise tasks, or scripts directory to find it.
   The tmux skill does not own worktree creation: it composes with whatever
   worktree tooling the project provides.

2. **Extract the worktree path** from the script output (look for the `Location:`
   line or similar).

3. **Create a tmux window** in that directory with the branch name as its name.

4. **Start the harness** by sending its command to the window.

## Common tmux operations

### List windows

```bash
tmux list-windows -F '#{window_id} #{window_index}: #{window_name} (#{pane_current_path})'
```

Add `#{pane_current_command}` to see whether a window is sitting at a shell or
running something. Treat the value as a yes/no signal, not as a harness
identity: it reports the foreground process name, so an idle shell shows up as
`fish`/`zsh`, but a harness may report its runtime (`node`), its own binary
name, or something unrecognizable like a bare version string. Track which
harness you started in which window at creation time; you cannot reliably
re-derive it later.

### Kill a window

```bash
tmux kill-window -t "$WID"   # or "<name-or-index>" for ad-hoc use
```

### Rename a window

```bash
tmux rename-window -t "$WID" "<new-name>"
```

This pins the name: `rename-window` sets `automatic-rename off` for the window,
so the new name survives every later title change from the pane. That is the
right behavior when the user asks for a specific name. To hand naming back to
the pane afterwards:

```bash
tmux set-option -w -t "$WID" automatic-rename on
```

If a window's name looks frozen while its pane title moves on, this is why.
Check with `tmux show-options -w -t "$WID" automatic-rename`.

### Send text to another window

**Always send the text and the submit keypress as two separate calls**, with the
text in literal mode (`-l`):

```bash
tmux send-keys -t "$WID" -l '<prompt text>'
tmux send-keys -t "$WID" Enter
```

An Enter packaged into the same `send-keys` call as the text only inserts a
newline into a harness composer, it does not submit. The symptom is that the
message appears in the target window's composer and just sits there unsent. A
separate Enter keypress submits it. The two-call form is also correct for a
plain shell command, so use it unconditionally instead of tracking which target
needs which.

After sending, confirm the submit actually took:

```bash
tmux capture-pane -t "$WID" -p -S -20 | tail -n 20
```

If the text is still sitting in the composer, that harness may bind submit to
something other than Enter. Look at its own footer hints in the pane before
guessing, and never spam Enter to force it: where Enter does submit, the extra
keypresses send empty or duplicate messages.

**Quoting:** literal mode stops tmux from interpreting keysyms inside the
string. For long or multi-line prompts that contain quotes, write them to a temp
file and pipe:

```bash
tmux send-keys -t "$WID" -l "$(cat /tmp/prompt.txt)"
tmux send-keys -t "$WID" Enter
```

### Switch to a window

```bash
tmux select-window -t "$WID"   # or "<name-or-index>" for ad-hoc use
```

## Orchestrator pattern

The user works on several tasks simultaneously, each in its own tmux window
with an agent session, and those sessions are not necessarily all the same
harness. The orchestrator is the window where the user talks to you: think
of it as a dashboard and switchboard that helps them stay on top of everything.

See **`references/orchestrator.md`** for the detailed workflows: bootstrapping
sessions, status sweeps across windows, stale session detection, context
briefings before a window switch, session inventory, and cleanup.
