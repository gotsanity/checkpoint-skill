---
name: checkpoint-list
description: >
  List all saved checkpoints with their status, summary, and age.
  Use when the user wants to see what checkpoints exist, check status, or browse saved context.
allowed-tools: Bash,Read,Glob
---

# Checkpoint List

Display all checkpoints in `.checkpoints/` (including archived ones in `.checkpoints/archive/`).

## Flow

1. **Scan** `.checkpoints/` and `.checkpoints/archive/` for all `.json` files.

2. **For each checkpoint**, read and extract: ID, summary, status, created date, last updated date.

3. **Flag stale checkpoints** — any with status `active` whose `updated` timestamp is older than 14 days.

4. **Present as a table**, grouped by status:

   **Active / Resumed**
   | ID | Summary | Updated | Age |

   **Deferred**
   | ID | Summary | Updated | Age |

   **Stale** (active > 14 days without update)
   | ID | Summary | Updated | Age |

   **Archived**
   | ID | Summary | Archived | Age |

5. If there are stale checkpoints, suggest the user archive or defer them.
