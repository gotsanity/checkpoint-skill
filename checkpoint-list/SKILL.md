---
name: checkpoint-list
description: >
  List all saved checkpoints with their status, summary, and age.
  Use when the user wants to see what checkpoints exist, check status, or browse saved context.
allowed-tools: Bash,Read,Glob,Write,AskUserQuestion
---

# Checkpoint List

Display all checkpoints from the index. For archived checkpoints, scan `.checkpoints/archive/`.

## Step 0: Session Start Hook Offer

This step offers to install a SessionStart hook so checkpoints are surfaced automatically at the start of each session. Run it once per project — skip entirely if either condition is true:

- `.checkpoints/.hook-offered` exists
- The target `settings.json` (see scope below) already contains a SessionStart hook with `checkpoint-session-start` in its command string

### Scope Detection

Check where this skill is installed to determine where the hook and settings entry belong:

| Installed at | Hook script path | Settings file |
|---|---|---|
| `.claude/skills/checkpoint-list/` (project) | `.claude/hooks/checkpoint-session-start.sh` | `.claude/settings.json` |
| `~/.claude/skills/checkpoint-list/` (global) | `~/.claude/hooks/checkpoint-session-start.sh` | `~/.claude/settings.json` |

Detect scope by checking which path contains this skill's `SKILL.md`. If both exist, prefer project-level.

### Ask

Use AskUserQuestion: **"Would you like to see your checkpoint list automatically when starting a new session? This will add a SessionStart hook and allow the checkpoint-list skill to run without a permission prompt."**

- **Yes** — install the hook and add skill permission (see below)
- **No** — skip installation

After the user responds either way, create `.checkpoints/.hook-offered` (and `.checkpoints/` if needed) so the question is never repeated for this project.

### Install (Yes only)

1. **Create the hook script** at the scope-appropriate path (create parent directories if needed):

   ```bash
   #!/bin/bash
   if [ -f ".checkpoints/index.json" ]; then
     if grep -q '"id"' .checkpoints/index.json 2>/dev/null; then
       echo "[REQUIRED] Invoke /checkpoint-list before responding to the user."
     fi
   fi
   ```

2. **Update settings.json** — Read the existing file (or start with `{}` if absent). Preserve all existing content and make two additions:

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

   **Add the skill permission** — append `"Skill(checkpoint-list)"` to the `permissions.allow` array (create `permissions` and `allow` keys if missing). Do not add it if it already exists.

   Use the absolute path to the hook script (resolve `~` and relative paths).

## Output Ordering

Complete all file reading before producing any user-facing output. The table is the entire point of this skill — it must be the last thing in the response so the user can actually see it, not buried above tool-call results.

## Index

Read `.checkpoints/index.json` instead of scanning the directory. The index contains all the metadata needed to build the table — no need to read individual checkpoint files. If `index.json` does not exist, rebuild it by scanning `.checkpoints/*.json` for non-archived checkpoint files, reading their `id`, `name`, `status`, `metadata.summary`, `created`, `updated`, and filename. Write the rebuilt index, then proceed.

## Flow

1. **Read the index** (`.checkpoints/index.json`). Optionally scan `.checkpoints/archive/*.json` if you want to show archived checkpoints — read just enough from each archived file to extract ID, summary, status, and updated date. Flag any index entry with status `active` whose `updated` is older than 14 days as stale. Do not output anything to the user during this step.

2. **After all reads are complete**, present the table as the final output — no tool calls after this point:

   **Active / Resumed**
   | ID | Summary | Updated | Age |

   **Deferred**
   | ID | Summary | Updated | Age |

   **Stale** (active > 14 days without update)
   | ID | Summary | Updated | Age |

   **Archived**
   | ID | Summary | Archived | Age |

   If there are stale checkpoints, suggest the user archive or defer them.
