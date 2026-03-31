# Checkpoint Skill Suite for Claude Code

Save and resume conversation context across Claude Code sessions. When you're mid-task and need to stop, `/checkpoint` captures your decisions, facts, open questions, and working state to a structured JSON file. Pick up later with `/checkpoint-resume` — no re-discovery needed.

## Installation

Clone this repo somewhere convenient:

```bash
git clone https://github.com/gotsanity/checkpoint-skill.git
```

### Global (all projects)

Copy each skill directory into `~/.claude/skills/`:

```bash
# Linux/macOS
cp -r checkpoint checkpoint-archive checkpoint-defer checkpoint-list checkpoint-note checkpoint-resume ~/.claude/skills/

# Windows
xcopy checkpoint "%USERPROFILE%\.claude\skills\checkpoint" /E /I
xcopy checkpoint-archive "%USERPROFILE%\.claude\skills\checkpoint-archive" /E /I
xcopy checkpoint-defer "%USERPROFILE%\.claude\skills\checkpoint-defer" /E /I
xcopy checkpoint-list "%USERPROFILE%\.claude\skills\checkpoint-list" /E /I
xcopy checkpoint-note "%USERPROFILE%\.claude\skills\checkpoint-note" /E /I
xcopy checkpoint-resume "%USERPROFILE%\.claude\skills\checkpoint-resume" /E /I
```

### Per-project

Copy into your project's `.claude/skills/` directory:

```bash
# Linux/macOS
cp -r checkpoint checkpoint-archive checkpoint-defer checkpoint-list checkpoint-note checkpoint-resume /path/to/project/.claude/skills/

# Windows
xcopy checkpoint "C:\path\to\project\.claude\skills\checkpoint" /E /I
xcopy checkpoint-archive "C:\path\to\project\.claude\skills\checkpoint-archive" /E /I
xcopy checkpoint-defer "C:\path\to\project\.claude\skills\checkpoint-defer" /E /I
xcopy checkpoint-list "C:\path\to\project\.claude\skills\checkpoint-list" /E /I
xcopy checkpoint-note "C:\path\to\project\.claude\skills\checkpoint-note" /E /I
xcopy checkpoint-resume "C:\path\to\project\.claude\skills\checkpoint-resume" /E /I
```

## Usage

Skills are invoked as slash commands directly in the Claude Code prompt:

```
> /checkpoint serializer-bug
> /checkpoint-resume
> /checkpoint-list
```

Skills can also be triggered conversationally. Claude reads each skill's description and will invoke the matching skill automatically when your message aligns with it — no slash command needed:

```
> save a checkpoint called auth-refactor
> where did we leave off?
> what checkpoints do I have?
```

## Skills

| Command | Description |
|---------|-------------|
| `/checkpoint [name]` | Save current conversation context to disk |
| `/checkpoint-resume [id]` | Resume from a saved checkpoint |
| `/checkpoint-list` | List all checkpoints with status and age |
| `/checkpoint-note [note]` | Add a quick note to an active checkpoint |
| `/checkpoint-archive [id]` | Archive a completed checkpoint |
| `/checkpoint-defer [id]` | Pause a checkpoint without archiving |

## How It Works

Checkpoints are saved as JSON files in `.checkpoints/` in your working directory. Each checkpoint captures:

- **Objective** — what you were working on and why
- **Decisions** — what was decided, rationale, and rejected alternatives
- **Facts** — concrete details with sources
- **Open questions** — unresolved items
- **Constraints** — limitations and their impact
- **Current hypothesis** — where your thinking stands
- **Referenced files** — paths and summaries of relevant files
- **Scratchpad** — freeform notes from you or the agent

### Lifecycle

```
/checkpoint  -->  active
                    |
         /checkpoint-resume  -->  resumed
                    |                |
                    |         /checkpoint (update)
                    |                |
         /checkpoint-defer   /checkpoint-archive
                    |                |
                 deferred         archived
                    |          (.checkpoints/archive/)
         /checkpoint-resume
                    |
                 resumed
```

Checkpoints with status `active` and no updates for 14 days are flagged as stale.

## Session Start Hook

The first time you run `/checkpoint` or `/checkpoint-list`, the skill will ask if you'd like to see your checkpoints automatically at the start of each session. If you accept, it installs a `SessionStart` hook and adds a `Skill(checkpoint-list)` permission entry to your `settings.json` (project-level or global, matching how you installed the skills).

If your project has a `CLAUDE.md` with a heavy startup protocol (multi-file reads, task loading, etc.), the hook may compete for priority. For a guaranteed trigger, add `/checkpoint-list` directly to your `CLAUDE.md` session start section:

```markdown
## On Session Start
- Run /checkpoint-list to show saved checkpoints
```

## Example Workflow

```
# Working on a complex bug fix, need to stop for the day
> /checkpoint serializer-bug

# Next session (or different machine), pick up where you left off
> /checkpoint-resume serializer-bug

# Jot down a thought mid-session
> /checkpoint-note remember to check the API serializer edge case

# Done with the work
> /checkpoint-archive serializer-bug
```

## Storage

- Active checkpoints: `.checkpoints/` (add to `.gitignore`)
- Archived checkpoints: `.checkpoints/archive/`
- Format: JSON with ISO-8601 timestamps
- One file per checkpoint, named `{slug}-{YYYYMMDD-HHmmss}.json`

## License

MIT
