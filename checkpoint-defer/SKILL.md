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

## Flow

1. **If an argument was provided**, find the matching checkpoint. If not, scan `.checkpoints/` for checkpoints with status `active` or `resumed` and prompt the user to select one via AskUserQuestion.

2. **Update the checkpoint** — set `"status": "deferred"` and update the `updated` timestamp.

3. **Confirm** — tell the user which checkpoint was deferred. Remind them it can be resumed later with `/checkpoint-resume`.
