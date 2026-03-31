---
name: checkpoint-resume
description: >
  Resume work from a previously saved checkpoint. Loads full conversation context so you
  can pick up where you left off. Triggers when the user wants to continue prior work,
  restore context, or asks "where were we".
allowed-tools: Bash,Read,Glob,Grep,AskUserQuestion,Write
argument-hint: "[checkpoint-id-or-name]"
---

# Checkpoint Resume

Load a saved checkpoint and restore working context.

## Output Ordering

Complete all tool work (file reads, writes, scans) before producing any user-facing text. User-visible output — the restored context, suggestions, questions — must be the last thing in a response, never followed by tool calls in the same turn. This prevents important information from getting buried.

## Index

Read `.checkpoints/index.json` instead of scanning the directory. The index contains all checkpoint metadata needed for selection. If `index.json` does not exist, rebuild it by scanning `.checkpoints/*.json` for non-archived checkpoint files, reading their `id`, `name`, `status`, `metadata.summary`, `created`, `updated`, and filename. Write the rebuilt index, then proceed.

## Step 1: Find the Checkpoint

**If an argument was provided** (`$ARGUMENTS`), match against the index entries by ID, name, or summary keyword. If multiple matches or no match, fall through to selection.

**If no argument was provided**, check the index. If it has a `last_active` value, default to that checkpoint. Present available checkpoints via AskUserQuestion showing: name, summary, status, last updated. Flag any with `updated` older than 14 days as potentially stale.

If the index has no checkpoints, tell the user there are none to resume.

## Step 2: Load and Mark as Resumed (silent — no output to user)

Read the selected checkpoint file (path from index: `.checkpoints/{file}`). Then immediately update it:
- Set `"status": "resumed"`
- Add to `resume_history`: `{"resumed_at": "ISO-8601", "resumed_by": "agent"}`
- Update `updated` timestamp

Then update `index.json`:
- Set the matching entry's `status` to `"resumed"` and update its `updated` field.
- Set `last_active` to this checkpoint's ID.

Also write `.checkpoints/session.json` to track the current session:
```json
{
  "checkpoint_id": "<checkpoint-id>",
  "name": "<checkpoint-name>",
  "summary": "<one-line summary>",
  "started_at": "<ISO-8601 timestamp>"
}
```

Complete the checkpoint file update, index update, and session file write before producing any output.

## Step 3: Present and Orient

Now that all file operations are done, present the checkpoint content and orientation together in a single response. No further tool calls after this output.

- Present all checkpoint sections in a structured, readable format
- Suggest a starting point based on checkpoint content — open questions, current hypothesis, or clear next steps from the objective
- Ask the user what they'd like to focus on
