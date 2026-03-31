---
name: checkpoint-archive
description: >
  Archive a checkpoint — move it to .checkpoints/archive/ and mark it as completed.
  Use when the user is done with a checkpoint and wants to clean up.
allowed-tools: Bash,Read,Write,Glob,AskUserQuestion
argument-hint: "[checkpoint-id]"
---

# Checkpoint Archive

Move a checkpoint to `.checkpoints/archive/` and set its status to `archived`. Prune it from the index.

## Output Ordering

Complete all file operations before producing any user-facing output. Confirmations must be the last thing in the response — never followed by tool calls.

## Index

Read `.checkpoints/index.json` instead of scanning the directory. If `index.json` does not exist, rebuild it by scanning `.checkpoints/*.json` for non-archived checkpoint files, reading their `id`, `name`, `status`, `metadata.summary`, `created`, `updated`, and filename. Write the rebuilt index, then proceed.

## Flow

1. **If an argument was provided**, find the matching entry in the index by ID or name. If not, present non-archived entries from the index and prompt the user to select one via AskUserQuestion. If prompting, the question must be the only output in that response — no tool calls in the same turn.

2. **Update and move** — Read the checkpoint file, set `"status": "archived"`, update the `updated` timestamp, and move the file to `.checkpoints/archive/` (creating the directory if needed). Then update `index.json`: remove the entry from the `checkpoints` array. If `last_active` pointed to this checkpoint, set it to `null`. Do not output anything during this step.

3. **Confirm** — after all file operations are complete, tell the user which checkpoint was archived and where it now lives. This must be the final output.
