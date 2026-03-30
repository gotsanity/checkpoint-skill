---
name: checkpoint-archive
description: >
  Archive a checkpoint — move it to .checkpoints/archive/ and mark it as completed.
  Use when the user is done with a checkpoint and wants to clean up.
allowed-tools: Bash,Read,Write,Glob,AskUserQuestion
argument-hint: "[checkpoint-id]"
---

# Checkpoint Archive

Move a checkpoint to `.checkpoints/archive/` and set its status to `archived`.

## Flow

1. **If an argument was provided**, find the matching checkpoint. If not, scan `.checkpoints/` for non-archived checkpoints and prompt the user to select one via AskUserQuestion.

2. **Update the checkpoint** — set `"status": "archived"` and update the `updated` timestamp.

3. **Move the file** to `.checkpoints/archive/`, creating the directory if needed.

4. **Confirm** — tell the user which checkpoint was archived and where it now lives.
