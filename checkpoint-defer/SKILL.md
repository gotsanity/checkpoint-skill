---
name: checkpoint-defer
description: >
  Defer a checkpoint — mark it as deferred so it stays available but is not actively being worked on.
  Use when the user wants to pause work on a checkpoint without archiving it.
allowed-tools: Bash,Read,Write,Glob,AskUserQuestion
argument-hint: "[checkpoint-id]"
---

# Checkpoint Defer

Mark a checkpoint as `deferred`. It stays in `.checkpoints/` and remains available for future resume, but signals that work is paused.

## Output Ordering

Complete all file operations before producing any user-facing output. Confirmations must be the last thing in the response — never followed by tool calls.

## Index

Read `.checkpoints/index.json` instead of scanning the directory. If `index.json` does not exist, rebuild it by scanning `.checkpoints/*.json` for non-archived checkpoint files, reading their `id`, `name`, `status`, `metadata.summary`, `created`, `updated`, and filename. Write the rebuilt index, then proceed.

## Flow

1. **If an argument was provided**, find the matching entry in the index by ID or name. If not, present entries with status `active` or `resumed` from the index and prompt the user to select one via AskUserQuestion. If prompting, the question must be the only output in that response — no tool calls in the same turn.

2. **Update the checkpoint** — set `"status": "deferred"` and update the `updated` timestamp in the checkpoint file. Then update `index.json`: set the matching entry's `status` to `"deferred"` and update its `updated` field. Also clear `.checkpoints/session.json` if its `checkpoint_id` matches the deferred checkpoint — write `{"checkpoint_id": null, "name": null, "summary": null, "started_at": null}`. Do not output anything during this step.

3. **Confirm** — after the file update and index update are complete, tell the user which checkpoint was deferred. Remind them it can be resumed later with `/checkpoint-resume`. This must be the final output.
