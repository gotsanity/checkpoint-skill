---
name: checkpoint
description: >
  Save conversation context to disk so work can be resumed later or by another agent.
  Triggers when the user wants to save progress, create a checkpoint, preserve context,
  or hand off work. For adding a single note, use /checkpoint-note instead.
allowed-tools: Bash,Write,Read,Glob,Grep,AskUserQuestion
argument-hint: "[checkpoint-name]"
---

# Checkpoint

Save the current conversation's context to a structured JSON file on disk. A fresh agent with no conversation history can resume from this checkpoint without re-asking or re-discovering anything.

## Output Ordering

Every step that involves tool calls (reading, writing, scanning files) must complete ALL tool work before producing any user-facing text. User-visible output — summaries, confirmations, questions — must be the last thing in a response, never followed by tool calls in the same turn. This prevents important information from getting buried by tool-use output that the user doesn't need to read.

## Index

All checkpoint skills use `.checkpoints/index.json` as the single source of truth for which checkpoints exist and their current state. This avoids scanning the directory on every invocation — the index is a single file read.

```json
{
  "version": 1,
  "last_active": "checkpoint-id or null",
  "checkpoints": [
    {
      "id": "string",
      "name": "string",
      "status": "active | resumed | deferred",
      "summary": "one-line summary",
      "created": "ISO-8601",
      "updated": "ISO-8601",
      "file": "filename.json"
    }
  ]
}
```

- `last_active` points to whichever checkpoint was most recently created, resumed, or updated.
- Archived checkpoints are pruned from the index (they still exist in `.checkpoints/archive/`).
- If `index.json` does not exist, rebuild it by scanning `.checkpoints/*.json` for non-archived checkpoint files, reading their `id`, `name`, `status`, `metadata.summary`, `created`, `updated`, and filename. Write the rebuilt index, then proceed normally.

## Step 1: Load Index and Detect Resume State

Read `.checkpoints/index.json`. If it doesn't exist, rebuild it (see Index section above).

Check the index for any entry with `"status": "resumed"`.

**If a resumed checkpoint exists**, use AskUserQuestion to prompt:

- **Update existing (Recommended)** — Merge new conversation details into the resumed checkpoint. Preserves continuity.
- **Archive** — Archive the resumed checkpoint to `.checkpoints/archive/` with status `archived`, prune it from the index. Does not create a new checkpoint.
- **Defer** — Set the resumed checkpoint to `deferred` status in both the file and the index. Does not create a new checkpoint.

**If no resumed checkpoint exists**, proceed to create a new checkpoint.

Also flag any index entry with status `active` whose `updated` timestamp is older than 14 days as potentially stale. Report these briefly when presenting the AskUserQuestion or before proceeding.

## Step 2: Gather Context (silent — no output to user)

Scan the full conversation history and extract details into the JSON structure defined below. Do not output anything to the user during this step — just build the internal data structure.

Be specific and concrete — vague summaries defeat the purpose. Not "fixing a bug" but "fixing a null reference in `processOrder()` at line 214 that occurs when orders come from the API endpoint but not from the UI, because the API serializer omits the `quantity` field when it's zero."

Every decision needs rationale and rejected alternatives. Every fact needs its source. Every constraint needs its impact. Without these, the next agent will re-evaluate from scratch.

The **scratchpad** has two sources: agent observations that don't fit other categories, and anything the user explicitly asked to "write down" during the conversation.

## Step 3: Name the Checkpoint

**If updating an existing checkpoint**: skip this step — keep the original name and ID.

**If an argument was provided** (`$ARGUMENTS`): use it as the checkpoint name. Slugify it for the ID. Do not output anything yet — proceed to Step 4.

**If no argument was provided**: use AskUserQuestion to ask the user for a checkpoint name. Suggest a short, descriptive default based on the conversation summary (e.g., "serializer-zero-bug", "auth-middleware-refactor"). The user can accept the suggestion or provide their own. This question must be the only output in this response — do not combine it with summaries or tool calls.

Slugify the name: lowercase, hyphens for spaces, strip non-alphanumeric characters except hyphens. Generate the ID as `{slug}-{YYYYMMDD-HHmmss}`. The filename is `{id}.json`.

## Step 4: Review and Capture User Notes

Present a summary of what will be saved: objective, decisions (count + brief list), facts (count + brief list), open questions, constraints, current hypothesis, referenced files, and any scratchpad notes already gathered.

This summary and the following question must be the only output in this response — do not perform any file writes or tool calls in the same turn. The summary is the whole point of this step and must remain visible to the user.

Use AskUserQuestion to ask: "Anything else you want recorded in this checkpoint?" with options:

- **Add a note** — User provides details they want carried over. Record their input verbatim as a scratchpad entry with `"source": "user"`. After recording, ask again until they choose "Looks good".
- **Looks good** — Proceed to Step 5.

This gives the user a chance to ensure specific details they find important are captured exactly as they state them.

## Step 5: Write the Checkpoint and Update Index

Create `.checkpoints/` if it doesn't exist.

**If updating an existing checkpoint**: append new entries to each section, update the objective and hypothesis if they evolved, add new referenced files, update `updated` timestamp, keep original `created` and `id`, add an entry to `resume_history`.

**If creating new**: write `{id}.json` with the structure below.

After writing the checkpoint file, update `index.json`:
- **New checkpoint**: add an entry to `checkpoints` array and set `last_active` to this checkpoint's ID.
- **Updated checkpoint**: update the matching entry's `summary`, `status`, and `updated` fields. Set `last_active` to this checkpoint's ID.

```json
{
  "id": "{slug}-{YYYYMMDD-HHmmss}",
  "name": "User-provided or accepted checkpoint name",
  "created": "ISO-8601 timestamp",
  "updated": "ISO-8601 timestamp",
  "status": "active",
  "metadata": {
    "working_directory": "absolute path",
    "git_branch": "current branch name or null",
    "git_commit": "current HEAD short hash or null",
    "summary": "One-line plain-language summary of what this conversation was about"
  },
  "resume_history": [],
  "objective": "Detailed description of what we were working on and why",
  "decisions": [
    {
      "decision": "What was decided",
      "rationale": "Why",
      "alternatives_rejected": ["What was rejected and why"]
    }
  ],
  "facts": [
    {
      "fact": "Concrete detail",
      "source": "Where or how this was learned"
    }
  ],
  "open_questions": [
    "Unresolved question"
  ],
  "constraints": [
    {
      "constraint": "What the limitation is",
      "impact": "How it affects the work"
    }
  ],
  "current_hypothesis": "Where our thinking stands, or null if not applicable",
  "referenced_files": [
    {
      "path": "relative/path/to/file",
      "summary": "What this file is and why it matters"
    }
  ],
  "scratchpad": [
    {
      "note": "The note content",
      "source": "agent or user",
      "timestamp": "ISO-8601 timestamp"
    }
  ]
}
```

## Step 6: Confirm

After the file write and index update in Step 5 are complete, report to the user. This confirmation must be the final output — no tool calls after it.

- Checkpoint name, ID, and file path
- Summary of what was captured (counts: N decisions, N facts, N open questions, N files referenced)
- Remind them they can resume with `/checkpoint-resume {id}` or `/checkpoint-resume` to browse
