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

## Step 1: Find the Checkpoint

**If an argument was provided** (`$ARGUMENTS`), match against checkpoint ID, name, filename slug, or search summaries for a keyword match. If multiple matches or no match, fall through to selection.

**If no argument was provided**, scan `.checkpoints/` for non-archived checkpoints (status: `active`, `resumed`, or `deferred`). If none exist, tell the user.

Present available checkpoints via AskUserQuestion showing: ID, name, summary, status, last updated. Flag any older than 14 days as potentially stale.

## Step 2: Load and Present

Read the selected checkpoint and present all sections to the conversation in a structured, readable format.

## Step 3: Mark as Resumed

Update the checkpoint file:
- Set `"status": "resumed"`
- Add to `resume_history`: `{"resumed_at": "ISO-8601", "resumed_by": "agent"}`
- Update `updated` timestamp

## Step 4: Orient

Suggest a starting point based on checkpoint content — open questions, current hypothesis, or clear next steps from the objective. Ask the user what they'd like to focus on.
