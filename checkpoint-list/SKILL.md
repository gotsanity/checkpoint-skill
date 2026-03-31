---
name: checkpoint-list
description: >
  List all saved checkpoints with their status, summary, and age.
  Use when the user wants to see what checkpoints exist, check status, or browse saved context.
allowed-tools: Bash,Read,Glob
---

# Checkpoint List

Display all checkpoints from the index. For archived checkpoints, scan `.checkpoints/archive/`.

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
