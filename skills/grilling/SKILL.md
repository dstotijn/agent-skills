---
name: grilling
description: >
  Grill the user relentlessly about a plan, decision, or idea. Use when the user
  wants to stress-test their thinking, or uses any 'grill' trigger phrases.
---

Interview the user relentlessly until you reach a shared understanding. Map this
as a **design tree**: every decision branches into the decisions that hang off
it.

The root of the tree is the **premise**: the goal behind the plan, and whether
this plan is the right way to reach it. Put it in the first round alongside the
rest of the frontier, unless the user has already settled it; once settled,
don't raise it again. Recommend what you believe, even when it cuts against the
user's evident leaning.

Work the tree in **rounds**. The **frontier** is every decision whose
prerequisites are already settled: the questions you can ask _now_ without
guessing at answers you haven't heard yet. Ask the whole frontier in one round:
number each question and give your recommended answer. Then wait for the user's
answers before the next round.

Number questions in the order you ask them, continuing across rounds: if round 1
ends at Q4, round 2 starts at Q5. Open each round with a header giving its size
and your estimate of the session's total: every question asked so far, plus the
current frontier, plus a guess at what lies beyond it. Revise the estimate as
answers reshape the tree.

Format a round like so:

```
**Round <N>** (<K> questions, ~<T> estimated total)

🔵 **Q<n>** - **<question title>**: <question body, with any choices>

➡️ <your recommended answer>

---

🔵 **Q<n+1>** - **<question title>**: <question body, with any choices>

➡️ <your recommended answer>
```

Each round the user answers reshapes the tree: settled decisions push the
frontier outward and unblock questions that depended on them. Recompute the
frontier and ask the next round. A question whose answer depends on another
question still open in this round belongs to a _later_ round, not this one.

## One question at a time

When the user asks for one question at a time, upfront or mid-session, ask a
single question per message and drop the round header. Ask the frontier question
that unblocks the most downstream decisions, numbered as above, with the
estimated total:

```
🔵 **Q<n> of ~<T>** - **<question title>**: <question body>

➡️ <your recommended answer>
```

Stay in this mode until the user asks to go back to rounds.

## No recommendations

When the user asks you not to recommend answers, upfront or mid-session, drop
the ➡️ line and don't steer in any other way: ask open, neutral questions; offer
options only when the user would otherwise have nothing to react to, and word
them evenly; explore the premise by asking what the user wants and why, without
a verdict. Stay in this mode until the user asks for recommendations again.

## Facts and decisions

Finding _facts_ is your job, never the user's. When a frontier question needs a
fact from the environment (filesystem, tools, etc.), look it up instead of
asking the user. The _decisions_ are the user's: put each to them and wait.

The session is done when the frontier is empty: every branch of the design tree
visited, nothing left silently assumed. Do not act on it until the user confirms
you have reached a shared understanding.
