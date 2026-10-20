---
name: hindsight
description: Selective, explicit cross-session memory through the configured Hindsight CLI. Use when a non-trivial task may depend on a past preference, decision, confirmed workaround, or earlier failure; when the user asks what was previously decided; or when the user explicitly asks to remember something. Retain only durable, non-sensitive, future-useful knowledge. Never automatically store conversations, source files, logs, secrets, or routine progress.
---

# Hindsight Memory

Use the configured `hindsight` CLI for deliberate, agent-initiated memory calls. Do not install hooks, capture sessions automatically, or send every interaction to Hindsight.

Briefly tell the user before calling Hindsight and explain why. Do not include sensitive details in that notice.

Pass `-o json` to read commands so their output is clean and parseable.

## Resolve the bank first

Every command takes a bank ID as its first positional argument. There is no implicit
default bank, and `default` is not a real one: a fresh install has no banks at all.
List them and use a real ID (shown below as `<bank>`):

```bash
hindsight -o json bank list
```

Both failure modes here are silent, so check rather than assume:

- Recalling from a bank that does not exist returns `{"results": []}` and exit 0.
  That is indistinguishable from a bank with nothing relevant in it.
- Retaining into a bank that does not exist creates it, so a typo silently produces
  a new empty bank instead of an error.

A repository may name its bank in its local Git config; use that bank for knowledge
that belongs to the repository:

```bash
git config --get hindsight.bank
```

Use the `global` bank for knowledge that would help in any repository. Before
retaining, ask whether the memory would help in another repository and pick the bank
accordingly. If the project or user fixes a bank ID and its configuration, use that
one and do not invent another.

## Recall selectively

Recall when a non-trivial task is likely to benefit from a past:

- user preference or working convention;
- project or architecture decision and its rationale;
- confirmed workaround, bug cause, or fix;
- failure or lesson that could prevent repeated work;
- fact the user explicitly asks you to remember or retrieve.

Use a focused query and a budget of `--max-tokens 1024` to `2048` instead of the
4096 default:

```bash
hindsight -o json memory recall <bank> "user preferences relevant to this task" --max-tokens 1024
hindsight -o json memory recall <bank> "previous decisions and confirmed problems in this area" --max-tokens 2048
```

Filter on a known identity with `--tags`. The default `--tags-match any` also returns
untagged memories; `any_strict` excludes them:

```bash
hindsight -o json memory recall <bank> "prior work and blockers" --tags issue:ABC-1234 --tags-match any_strict --max-tokens 1024
```

Recall cannot filter on `--context`.

Use reflection only when synthesis across several memories would materially help. It
is an agent loop of up to 10 LLM calls, so it is far slower and costlier than recall:

```bash
hindsight -o json memory reflect <bank> "What past decisions or lessons should guide this task?"
```

Bank directives steer only `reflect`, not retain.

Do not recall for trivial questions, or when the current repository, documentation, or live system is the authoritative source. Treat recalled information as fallible and potentially stale; verify it before relying on it.

## Fix the record

When you verify that a recalled claim is wrong, correct it: re-retain the corrected
memory under the same document ID, or delete the document if it no longer holds:

```bash
hindsight document delete <bank> <document_id>
```

## Fetch exact source text when needed

Recall and reflection return extracted facts and can omit exact values. When a result points to a stored document and an exact literal is necessary, fetch that document:

```bash
hindsight -o json document get <bank> <document_id>
```

Only `document get` populates `original_text`; `document list` returns it empty, so
use it to find a document ID, not to read one:

```bash
hindsight -o json document list <bank>
```

Read `original_text`, but expose or use only the part needed for the task. It is
populated only when the bank stores document text, so treat an empty value as
"not available" rather than "no such text".

## Retain deliberately

Retain only when the user explicitly asks, or when you learn something durable, confirmed, non-sensitive, and likely to save future work:

- a stable user preference that no instruction file records;
- a confirmed technical decision and its rationale;
- a proven procedure or workaround;
- a verified bug cause and fix;
- a non-obvious constraint that will matter again.

If the user keeps preferences and conventions in instruction files such as `AGENTS.md` or `CLAUDE.md`, propose a change to the relevant file instead of retaining the preference.

Write a self-contained memory containing the useful fact, why it matters, and the confirmed outcome. Keep preferences and decisions concise; a root cause or procedure may be fuller, with the steps and evidence, because extraction distills it. Include exact values only when they are necessary and safe to persist.

Shape every retain call the same way:

- Always pass `--doc-id` with a stable, descriptive ID such as `<scope>--<slug>`. The
  default ID is `cli_put_<YYYYmmdd_HHMMSS>`, and reusing a document ID replaces the
  old document, so two retains in the same second (likely with `--async`) silently
  overwrite each other. Re-retaining under the same ID is how you correct a memory.
- Tag the memory with identities you know, such as `--document-tags issue:ABC-1234`,
  not with content types. The CLI sends one item per call, so this flag tags exactly
  that memory even though its help text calls it deprecated.
- Give `--context` a short description of the source and situation, not a bare label.
  It feeds fact extraction.
- Pass `--timestamp` (ISO 8601) when the decision or event happened in the past; it
  defaults to now.

```bash
hindsight memory retain <bank> "The user prefers explicit Hindsight recall and retention calls; do not enable automatic transcript or Git ingestion." \
  --doc-id user--explicit-memory-calls --context "User preference stated while setting up agent memory"
hindsight memory retain <bank> "The build requires Node 20 because Node 18 fails with the confirmed ESM loader incompatibility." \
  --doc-id abc-1234--node20-esm-loader --document-tags issue:ABC-1234 \
  --context "Root cause confirmed while fixing a failing CI build in <repo>"
hindsight memory retain <bank> "Run the integration suite with NODE_ENV=test; this selects the isolated test configuration." \
  --doc-id <repo>--integration-test-env --context "Procedure verified while running the <repo> integration suite" \
  --timestamp 2026-01-15
```

Retain is the slowest operation by a wide margin: extraction runs an LLM over the
content, and a single call can take minutes. Queue it with `--async` when you do not
need the result in-turn, and do not treat a long wait as a hang.

Never retain:

- complete conversations, transcripts, or session dumps;
- raw source files, patches, logs, command output, or tool results;
- routine progress or facts that are easy to recover from an authoritative source;
- speculation, unverified hypotheses, or transient state;
- passwords, API keys, tokens, credentials, private keys, or `.env` contents;
- personal or customer data unless the user explicitly approves that specific retention.

Hindsight redacts secrets at retain time only when the bank has a `memory_defense` policy, so never rely on it. If sensitivity or future value is ambiguous, ask before retaining. Never send a broad payload merely because Hindsight can extract facts from it.

## If Hindsight is unavailable

Do not start Hindsight, Docker, or another service solely for an optional memory call. Continue without memory and briefly report that Hindsight was unavailable. If memory is essential because the user explicitly requested it, explain the blocker.

Do not trust the CLI's error text on its own. It reports a client-side timeout as
"Cannot connect to Hindsight API / the server is not running", and it reports an
upstream provider error as "API endpoint not found (404)". Confirm with `hindsight
health` before telling the user Hindsight is down, and read the server logs for the
real cause when a call fails against a healthy server.
