---
name: checkpoint-config
description: >
  Configure the checkpoint skill suite — install or update the SessionStart hook, set skill
  permissions, and manage setup state. Invoked automatically on first use of /checkpoint or
  /checkpoint-list. Also callable directly when the user wants to reconfigure checkpoint
  settings, change hook scope, reinstall the session-start hook, or reset checkpoint configuration.
allowed-tools: Bash,Read,Write,Glob,AskUserQuestion
---

# Checkpoint Config

Set up and manage configuration for the checkpoint skill suite. This skill handles the SessionStart hook that surfaces checkpoints at the beginning of each session, along with the skill permissions needed for it to work.

Other checkpoint skills invoke this automatically on first run. Users can also call it directly to reconfigure — for example after changing from per-project to global installation, or to reinstall a hook that was removed.

## Output Ordering

Complete all file operations before producing any user-facing output. Confirmations and questions must be the last thing in a response — never followed by tool calls in the same turn.

## When Invoked Automatically (guard check from another skill)

Other checkpoint skills call this when `.checkpoints/.hook-offered` does not exist. In this mode, run the full flow below — scope detection, ask, install — then return control to the calling skill.

## When Invoked Directly by the User

If `$ARGUMENTS` contains `--reset` or the user mentions reconfiguring/reinstalling, delete `.checkpoints/.hook-offered` first so the flow runs fresh regardless of prior state. Then proceed with the full flow below.

If invoked directly without `--reset` and `.checkpoints/.hook-offered` already exists, tell the user the hook is already configured. Offer to reconfigure by running again with `--reset`.

## Step 1: Scope Detection

Determine where the checkpoint skills are installed by checking which paths exist:

- **Project-level**: `.claude/skills/checkpoint/SKILL.md` relative to the working directory
- **Global**: `~/.claude/skills/checkpoint/SKILL.md`

If both exist, prefer project-level.

This determines where the hook script and settings entry go:

| Scope | Hook script path | Settings file |
|---|---|---|
| Project | `.claude/hooks/checkpoint-session-start.sh` | `.claude/settings.json` |
| Global | `~/.claude/hooks/checkpoint-session-start.sh` | `~/.claude/settings.json` |

## Step 2: Ask the User

Use AskUserQuestion: **"Would you like to see your checkpoint list automatically when starting a new session? This will add a SessionStart hook and allow the checkpoint-list skill to run without a permission prompt."**

- **Yes** — proceed to Step 3
- **No** — skip to Step 4

This question must be the only output in this response — no tool calls in the same turn.

## Step 3: Install (Yes only)

### Hook Script

Create the hook script at the scope-appropriate path (create parent directories if needed):

```bash
#!/bin/bash
if [ -f ".checkpoints/index.json" ]; then
  if grep -q '"id"' .checkpoints/index.json 2>/dev/null; then
    echo "[REQUIRED] Invoke /checkpoint-list before responding to the user."
  fi
fi
```

### Settings Update

Read the existing settings.json at the scope-appropriate path (or start with `{}` if absent). Preserve all existing content and make two additions:

**Add the SessionStart hook** — append to the `hooks.SessionStart` array (create `hooks` and `SessionStart` keys if missing). Each entry requires a `matcher` (empty string to match all) and a `hooks` array:

```json
{
  "matcher": "",
  "hooks": [
    {
      "type": "command",
      "command": "bash \"<absolute-path-to-hook-script>\""
    }
  ]
}
```

Use the absolute path to the hook script (resolve `~` and relative paths).

**Add the skill permission** — append `"Skill(checkpoint-list)"` to the `permissions.allow` array (create `permissions` and `allow` keys if missing). Do not add it if it already exists.

## Step 4: Write Marker

Regardless of whether the user chose yes or no, create `.checkpoints/.hook-offered` (and `.checkpoints/` if needed). This prevents the question from being asked again for this project.

## Step 5: Confirm

After all file operations are complete, report what was done:

- If installed: confirm the hook script location, which settings.json was updated, and that the skill permission was added.
- If skipped: confirm the choice was recorded and the question won't be asked again. Mention they can reconfigure later with `/checkpoint-config --reset`.

This must be the final output — no tool calls after it.
