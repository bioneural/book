---
description: "Book domain taste — accumulated preferences for the task queue system"
events: [PreToolUse, UserPromptSubmit]
always: true
last_reviewed: "2026-03-14"
review_interval: 30
---

## Book domain taste

Accumulated preferences for work within the book (task queue) repository.

### Architecture
- Single Ruby script (bin/book) with subcommands. No framework.
- SQLite for all persistence. WAL mode, busy timeout, results_as_hash.
- Structured logging via spill when available, stderr fallback.
- Task dispatch via `claude -p` with structured prompts.

### Interface
- CLI output: IDs on stdout, diagnostics on stderr.
- JSON for machine-readable output (next, paused, doctor).
- Tab-separated for human-readable tables (list, status, reviews).

### Behavior
- Fail-loud on data operations. Fail-open on optional integrations.
- File-lock semaphore for dispatch exclusion.
- Auto-pause when pending reviews exist. Auto-resume when cleared.
