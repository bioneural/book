<h1 align="center">
  b o o k
  <br>
  <sub>a persistent task queue for autonomous agents</sub>
</h1>

A prioritized work queue that survives across sessions. Tasks enter, a dispatcher executes them via `claude -p`, and gate blocks produce reviews requiring human approval before work resumes. Intention persists even when an agent does not.

Ruby stdlib plus the sqlite3 gem. No other gems.

---

## How it works

book manages a queue of tasks across six statuses and three failure modes. A task starts as `pending`, gets picked up by a dispatcher, and terminates as `done`, `failed`, or `error`. When a policy gate fires during execution, the task becomes `blocked` and a review is created. A human approves or rejects that review. Approved tasks return to `pending`. Rejected tasks become `failed`.

Approvals are recorded for auditability, not enforcement. An `approved_by` field holds a plain string — whoever claims to be approving. An audit trail is the value, not cryptographic proof.

## Usage

### Queue management

```bash
# Initialize a database (created on demand, but explicit init is supported)
book init

# Add tasks with optional priority (higher = dispatched first)
book add "deploy staging environment" --priority 5
book add "update SSL certificates" --priority 10

# View the queue
book list          # all tasks, ordered by priority desc
book next          # highest-priority pending or retriable task (JSON)
book status        # counts by state

# Change priority
book reprioritize 1 --priority 20

# Remove a task
book drop 1
```

### Completing tasks manually

```bash
book done 1                              # mark completed
book fail 1 --reason "host unreachable"  # unrecoverable — terminal
book error 1 --reason "timeout"          # retriable — stays in queue
```

### Policy blocks and reviews

```bash
# Block a task (creates a review for human approval)
book block 1 --gate "requires-approval" --action "deploy to production"

# View pending reviews
book reviews

# Resolve a review
book approve 1 --as kerry --notes "approved for deploy window"
book reject 1 --as kerry --notes "not ready — staging tests failing"
```

### Dispatch

```bash
# Execute the next pending or retriable task via claude -p
book dispatch
```

A dispatch cycle:
1. Find highest-priority pending or error task
2. Mark it `running`
3. Send a structured prompt to `claude -p`
4. Parse the response for a status line (`STATUS: DONE`, `BLOCKED`, `ERROR`, or `FAILED`)
5. Update task status accordingly
6. If `BLOCKED`, create a review row for human approval

### Doctor

```bash
book doctor    # JSON report of prerequisites: ruby, sqlite3, claude CLI
```

## Task lifecycle

```
                ┌──────────┐
                │ pending  │◄──────────────────────┐
                └────┬─────┘                       │
                     │ dispatch                    │ approve
                     ▼                             │
                ┌──────────┐     block      ┌─────┴──────┐
                │ running  │───────────────►│  blocked   │
                └────┬─────┘                └─────┬──────┘
                     │                            │ reject
           ┌────────┬┴────────┐                   ▼
           ▼        ▼         ▼             ┌──────────┐
      ┌────────┐ ┌──────┐ ┌───────┐        │  failed  │
      │  done  │ │error │ │failed │        └──────────┘
      └────────┘ └──┬───┘ └───────┘
                    │
                    │ retry (retries < max_retries)
                    └──────────► pending
                    │
                    │ retries >= max_retries
                    └──────────► failed
```

Six statuses. Two are terminal (`done`, `failed`). One is retriable (`error`). One awaits human action (`blocked`). One is active (`running`).

## Dispatch

A dispatcher uses `claude -p` to execute tasks. Each invocation handles one task — a cron job or heartbeat handles repetition. A structured prompt instructs the model to end its response with a status line:

```
STATUS: DONE — deployed to staging
STATUS: BLOCKED — GATE: requires approval — ACTION: deploy to production
STATUS: ERROR — API timeout, retrying
STATUS: FAILED — credentials expired, cannot recover
```

Prior approvals for a task are included in a dispatch prompt, so the model knows what has already been approved.

## Human approval

When a task is blocked — whether by dispatcher output or by a manual `book block` command — a review row is created. `book reviews` lists pending reviews. A human resolves a review with one of five feedback actions:

| Action | Command | Effect |
|--------|---------|--------|
| **Authorize** | `book approve` | Unblock task, record approval |
| **Deny** | `book reject` | Fail task with rejection reason |
| **Redirect** | `book redirect` | Fail original task, create a replacement task |
| **Correct** | `book correct` | Unblock task, write a correction to crib |
| **Clarify** | `book clarify` | Unblock task, prepend clarification to task description |

```bash
book approve 1 --as kerry --notes "approved for deploy window"
book reject 1 --as kerry --notes "not ready"
book redirect 1 --as kerry --notes "wrong approach" --task "use blue-green deploy instead"
book correct 1 --as kerry --notes "use port 8080, not 3000"
book clarify 1 --as kerry --notes "scope is staging only, not production"
```

Correct and clarify actions write feedback to crib as operator-feedback entries, so the correction or clarification persists as memory for future sessions.

An approval records a reviewer's name and creates a row in the `approvals` table. Approvals persist and are surfaced to subsequent dispatch runs for the same task, so a model can act on previously granted permissions.

A rejection marks the task as `failed` with the reviewer's reason.

## Pause and resume

Dispatch can be paused to prevent task execution when human review is needed:

```bash
book pause     # creates .state/book/paused flag
book resume    # removes the flag
book paused    # returns JSON: {"paused": true/false}
```

When paused, `book dispatch` returns immediately without executing a task. Prophet's heartbeat auto-pauses dispatch when pending reviews exist and auto-resumes when all reviews are resolved.

## Claude semaphore

A file lock at `.state/book/claude.lock` prevents concurrent dispatch. If a dispatch is already running, subsequent calls skip with a log message instead of racing on `claude -p`.

## Schema

Three tables: `tasks`, `reviews`, `approvals`.

```sql
CREATE TABLE tasks (
  id           INTEGER PRIMARY KEY,
  description  TEXT NOT NULL,
  priority     INTEGER DEFAULT 0,
  status       TEXT DEFAULT 'pending',
  created_at   TEXT DEFAULT (datetime('now')),
  started_at   TEXT,
  completed_at TEXT,
  result       TEXT,
  retries      INTEGER DEFAULT 0,
  max_retries  INTEGER DEFAULT 3
);

CREATE TABLE reviews (
  id          INTEGER PRIMARY KEY,
  task_id     INTEGER REFERENCES tasks(id),
  gate_reason TEXT NOT NULL,
  action_desc TEXT NOT NULL,
  context     TEXT,
  status      TEXT DEFAULT 'pending',
  resolved_by TEXT,
  created_at  TEXT DEFAULT (datetime('now')),
  resolved_at TEXT,
  notes       TEXT
);

CREATE TABLE approvals (
  id           INTEGER PRIMARY KEY,
  task_id      INTEGER REFERENCES tasks(id),
  gate_reason  TEXT NOT NULL,
  description  TEXT NOT NULL,
  approved_by  TEXT NOT NULL,
  approved_at  TEXT DEFAULT (datetime('now')),
  notes        TEXT
);
```

State lives at `.state/book/book.db` by default. Override with the `BOOK_DB` environment variable.

## Smoke tests

```bash
./smoke-test           # full suite (dispatch tests require claude CLI)
./smoke-test --quick   # skip claude-dependent tests
```

## Prerequisites

- Ruby stdlib plus the sqlite3 gem
- claude CLI (for dispatch)

## Prophet integration

book is a standalone tool. When used with [prophet](https://github.com/bioneural/prophet), set `BOOK_HOME` to a book repo root or place book as a sibling directory. A prophet heartbeat can call `book dispatch` on each cycle, and a hooker inject policy can surface `book next` on every prompt.

---

## License

MIT
