---
name: checkpoint-note
description: >
  Add a quick note to a checkpoint's scratchpad. Triggers when the user wants to jot down,
  record, or append a note to an existing checkpoint. For saving full conversation context,
  use /checkpoint instead.
allowed-tools: Read,Edit,Glob,Bash
argument-hint: "[checkpoint-name] [note text]"
---

# Checkpoint Note

Quick, quiet note append. Only visible output: the final confirmation line (and a selection prompt if needed).

## Find the Checkpoint

Scan `.checkpoints/` for non-archived checkpoints (status: `active`, `resumed`, or `deferred`).

- **None exist**: tell the user. Stop.
- **Exactly one**: use it automatically.
- **Multiple + `$ARGUMENTS` matches a name/ID**: use the match.
- **Multiple + no match**: ask the user to pick. Show name, summary, status. Recommend any `resumed` checkpoint as default.

## Capture the Note

Check in order:

1. **Inline in arguments** — `$ARGUMENTS` has text beyond the matched checkpoint name → that text is the note.
2. **Arguments are the note** — one checkpoint (auto-selected) and `$ARGUMENTS` doesn't match its name → entire argument string is the note.
3. **Ask** — no note text provided → ask in plain text: "What would you like to add to this checkpoint?" Do not use AskUserQuestion.

Record verbatim. Do not rephrase.

## Write

Append to the checkpoint's `scratchpad` array:

```json
{
  "note": "exact user text",
  "source": "user",
  "timestamp": "ISO-8601"
}
```

Get timestamp via `date -u +"%Y-%m-%dT%H:%M:%SZ"`. Update the checkpoint's `updated` field to match. Use Edit, not Write.

## Confirm

One line: `Note added to "<checkpoint name>".` — nothing else.
