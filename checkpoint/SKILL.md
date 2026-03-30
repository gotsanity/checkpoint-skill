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

## Step 1: Detect Resume State

Scan `.checkpoints/` in the working directory for any checkpoint file with `"status": "resumed"`.

**If a resumed checkpoint exists**, use AskUserQuestion to prompt:

- **Update existing (Recommended)** — Merge new conversation details into the resumed checkpoint. Preserves continuity.
- **Archive** — Archive the resumed checkpoint to `.checkpoints/archive/` with status `archived`. Does not create a new checkpoint.
- **Defer** — Set the resumed checkpoint to `deferred` status. Does not create a new checkpoint.

**If no resumed checkpoint exists**, proceed to create a new checkpoint.

## Step 2: Gather Context

Scan the full conversation history and extract details into the JSON structure defined below. Be specific and concrete — vague summaries defeat the purpose. Not "fixing a bug" but "fixing a null reference in `processOrder()` at line 214 that occurs when orders come from the API endpoint but not from the UI, because the API serializer omits the `quantity` field when it's zero."

Every decision needs rationale and rejected alternatives. Every fact needs its source. Every constraint needs its impact. Without these, the next agent will re-evaluate from scratch.

The **scratchpad** has two sources: agent observations that don't fit other categories, and anything the user explicitly asked to "write down" during the conversation.

## Step 3: Name the Checkpoint

**If updating an existing checkpoint**: skip this step — keep the original name and ID.

**If an argument was provided** (`$ARGUMENTS`): use it as the checkpoint name. Slugify it for the ID.

**If no argument was provided**: use AskUserQuestion to ask the user for a checkpoint name. Suggest a short, descriptive default based on the conversation summary (e.g., "serializer-zero-bug", "auth-middleware-refactor"). The user can accept the suggestion or provide their own.

Slugify the name: lowercase, hyphens for spaces, strip non-alphanumeric characters except hyphens. Generate the ID as `{slug}-{YYYYMMDD-HHmmss}`. The filename is `{id}.json`.

## Step 4: Review and Capture User Notes

Present a summary of what will be saved: objective, decisions (count + brief list), facts (count + brief list), open questions, constraints, current hypothesis, referenced files, and any scratchpad notes already gathered.

After presenting the summary, use AskUserQuestion to ask: "Anything else you want recorded in this checkpoint?" with options:

- **Add a note** — User provides details they want carried over. Record their input verbatim as a scratchpad entry with `"source": "user"`. After recording, ask again until they choose "Looks good".
- **Looks good** — Proceed to write.

This gives the user a chance to ensure specific details they find important are captured exactly as they state them.

## Step 5: Write the Checkpoint

Create `.checkpoints/` if it doesn't exist.

**If updating an existing checkpoint**: append new entries to each section, update the objective and hypothesis if they evolved, add new referenced files, update `updated` timestamp, keep original `created` and `id`, add an entry to `resume_history`.

**If creating new**: write `{id}.json` with the structure below.

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

Report to the user:
- Checkpoint name, ID, and file path
- Summary of what was captured (counts: N decisions, N facts, N open questions, N files referenced)
- Remind them they can resume with `/checkpoint-resume {id}` or `/checkpoint-resume` to browse

## Stale Detection

When scanning `.checkpoints/` in Step 1, flag any checkpoint with status `active` whose `updated` timestamp is older than 14 days. Report these as potentially stale.
