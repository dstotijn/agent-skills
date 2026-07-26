---
name: peer-consult
description: >
  Peer-consult: confer with a peer harness (another agent running in the same
  working directory) to sharpen your own thinking, then decide for yourself. Use
  when the user wants a second opinion or cross-model verification, wants
  another agent to red-team a plan, a design, code, or a piece of writing, wants
  to hash out a decision with a peer, or wants fresh eyes to get unstuck. The
  peer advises; you stay the driver and the only one who changes anything.
---

# Peer-consult

To **confer** is to reason with an equal, not to obey one and not to hand off
the work. A peer harness is another agent running in this same working
directory, with its own tools and its own skills already loaded, but with zero
knowledge of this conversation. You consult it to stress-test your thinking. You
remain the driver, you make the call, and you are the only one who changes
anything.

Fire this only when the user asks for it. Do not confer on your own initiative.

## Transport contract

You reach the peer through whatever peer-invocation mechanism this harness
provides. This skill names no peer, no tool, and no command on purpose: it runs
inside more than one harness, so the peer is always "the other agent," resolved
by the environment, not by this text.

What you need from that mechanism:

- Invoke the peer in the current working directory, so it can read files, grep,
  and run things itself.
- Continue the *same* peer session for follow-up rounds, so the back-and-forth
  keeps its context.
- Keep the peer read-only. It advises; it does not change anything.

If the mechanism cannot continue a session, one round is your ceiling. Say so
rather than faking a debate across cold restarts.

## Confer

### 1. Commit your position first

Before you send anything, state your own current view, even in a sentence:
what you think the answer is and why. You send this to the peer as part of the
brief. This gives you a real position to defend and keeps the peer's framing
from quietly becoming yours.

### 2. Write the brief

The peer arrives with its own skills loaded but remembers nothing of this
conversation. The harness transfers the peer's capabilities; it does not
transfer our plan, our constraints, or our dead ends. That gap is yours to
close.

A brief is ready to send only when a peer with no memory of this conversation
could act on it. It must carry:

1. The decision or question actually on the table.
2. The constraints, and what has already been tried or ruled out, and why.
3. Pointers to the relevant material by `file:line`. Point, do not paste: the
   peer can open the files itself.
4. Exactly what you want back: a critique, an alternative, a verification, a
   tie-break.
5. Your own current position, from step 1.

The failure this bar exists to stop is briefing the peer as if it shared the
chat ("as we discussed, the fix for that bug..."). It saw none of it.

### 3. Confer, round by round

Send the brief, then read the reply on its merits. Weigh the peer's
*reasoning*, never its verdict. It is an equal, not an authority:

- Push back when you disagree. That is the point of conferring.
- Never adopt a conclusion because the peer asserted it, and never drop a
  position you still believe is correct just to reach agreement.
- Watch for false consensus. Two models can rationalise toward a confident
  wrong answer as easily as one. If you find yourself agreeing fast, re-check
  against the position you committed in step 1.

Continue only while each round adds something new: a fresh argument, evidence,
a reframing. Stop the moment a round repeats what you already have. Consensus,
a stable impasse, and plain repetition are all *done*. Rounds cost real money,
so bias toward stopping. An impasse is a legitimate result, not a failure to
try harder; carry it to the report unresolved.

### 4. Report back

Give the user a distilled synthesis, not a transcript:

- Name the peer if you know which one it was ("Codex argued...", not "the peer
  argued...").
- Say what the peer contributed, where you converged, and where you diverged.
- Attribute the peer's ideas to the peer. Do not launder them into your own
  voice, and do not smooth over a disagreement to look decisive.
- State your call and why.
