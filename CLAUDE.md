# Checkpoint Skill Suite

## Project Structure

```
checkpoint-skill/
├── checkpoint/SKILL.md        — Save conversation context to disk
├── checkpoint-resume/SKILL.md — Resume from a saved checkpoint
├── checkpoint-list/SKILL.md   — List all checkpoints with status and age
├── checkpoint-note/SKILL.md   — Add a note to a checkpoint's scratchpad
├── checkpoint-archive/SKILL.md— Archive a completed checkpoint
├── checkpoint-defer/SKILL.md  — Pause a checkpoint without archiving
├── checkpoint-config/SKILL.md — Configure hooks, permissions, and setup
└── README.md
```

Each skill is a standalone `SKILL.md` with YAML frontmatter (name, description, allowed-tools, argument-hint) and markdown instructions.

## Storage

- Active checkpoints: `.checkpoints/`
- Archived checkpoints: `.checkpoints/archive/`
- Checkpoint index: `.checkpoints/index.json`
- One file per checkpoint: `{slug}-{YYYYMMDD-HHmmss}.json`

## Index Schema

The index is the source of truth for which checkpoints exist and their current state. Skills read the index instead of scanning the directory. Archived checkpoints are pruned from the index.

```json
{
  "version": 1,
  "last_active": "checkpoint-id or null",
  "checkpoints": [
    {
      "id": "auth-refactor-20260329-140000",
      "name": "auth-refactor",
      "status": "active | resumed | deferred",
      "summary": "one-line summary",
      "created": "ISO-8601",
      "updated": "ISO-8601",
      "file": "auth-refactor-20260329-140000.json"
    }
  ]
}
```

If `index.json` is missing, rebuild it by scanning `.checkpoints/*.json`.

## Checkpoint Schema

```json
{
  "id": "{slug}-{YYYYMMDD-HHmmss}",
  "name": "string",
  "created": "ISO-8601",
  "updated": "ISO-8601",
  "status": "active | resumed | deferred | archived",
  "metadata": {
    "working_directory": "absolute path",
    "git_branch": "string | null",
    "git_commit": "short hash | null",
    "summary": "one-line summary"
  },
  "resume_history": [{ "resumed_at": "ISO-8601", "resumed_by": "agent" }],
  "objective": "string",
  "decisions": [{ "decision": "", "rationale": "", "alternatives_rejected": [] }],
  "facts": [{ "fact": "", "source": "" }],
  "open_questions": ["string"],
  "constraints": [{ "constraint": "", "impact": "" }],
  "current_hypothesis": "string | null",
  "referenced_files": [{ "path": "", "summary": "" }],
  "scratchpad": [{ "note": "", "source": "agent | user", "timestamp": "ISO-8601" }]
}
```

## Session File

`.checkpoints/session.json` tracks the checkpoint currently in use. Written by `checkpoint` (on create/update) and `checkpoint-resume` (on resume). Cleared by `checkpoint-archive` and `checkpoint-defer`.

```json
{
  "checkpoint_id": "auth-refactor-20260329-140000",
  "name": "auth-refactor",
  "summary": "one-line summary",
  "started_at": "ISO-8601"
}
```

When cleared, write `null` values: `{"checkpoint_id": null, "name": null, "summary": null, "started_at": null}`.

## Lifecycle

`active` → `resumed` → `archived` or `deferred` → `resumed` (re-resumable)

Checkpoints with status `active` and no update for 14 days are flagged stale.

## Guardrails

- **Append-only updates**: When updating an existing checkpoint, never remove, replace, condense, or summarize prior entries. All array fields are appended to. Text fields (`objective`, `current_hypothesis`) preserve the original in full with the update appended below a timestamp header. Only a user may explicitly request condensing or removing information.
