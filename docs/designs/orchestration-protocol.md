# Orchestration Protocol: Musician-Conductor Interaction Design

**Date:** 2026-02-07
**Status:** Active
**Depends on:** [Implementation Hook v2](implementation-hook-v2-multi-session.md)

---

## Problem

The conductor skill (Tier 0) and musician skill (Tier 1) communicate through a shared SQLite database (`comms.db`) via the comms-link MCP server. Both skills need a single, canonical reference for: database schema, state machine, message formats, heartbeat protocol, session lifecycle, and handoff procedures. Without this, each skill's documentation drifts independently and assumptions conflict.

This document is the **source of truth** for the interaction protocol between conductor and musician.

## Architecture Overview

```
Tier 0: Conductor (main session)
  ├─ Owns task-00 row in orchestration_tasks
  ├─ Creates phases, generates task instructions
  ├─ Monitors musicians via database polling
  ├─ Handles reviews, errors, phase transitions
  └─ Communicates with user for launches and escalations

Tier 1: Musician (EXTERNAL Claude sessions)
  ├─ Owns task-NN row in orchestration_tasks
  ├─ Reads task instruction file
  ├─ Does direct integration work + delegates to Tier 2 subagents
  ├─ Manages verification checkpoints and review cycles
  └─ Handles context-aware pausing and session handoff

Tier 2: Subagents (launched by musician)
  ├─ Receive scoped instruction sections from musician
  ├─ Do focused implementation (code, tests, docs)
  └─ Return tested code for musician integration
```

**Communication channel:** All inter-tier communication flows through two database tables (`orchestration_tasks` and `orchestration_messages`) accessed exclusively via comms-link MCP. No direct file-based or environment-variable communication between sessions.

---

## Key Decisions

These decisions were finalized on 2026-02-07 and are binding for both skills.

### 1. Message Format: String Body + message_type Column

Messages use human-readable string bodies with structured conventions (section headers, key-value pairs). A `message_type` enum column on `orchestration_messages` enables clean filtering without body parsing.

**Rationale:** Claude produces and parses structured markdown more reliably than JSON. The `message_type` column gives watchers cheap type discrimination without parsing the body.

### 2. New Database Columns: message_type Only

No new columns on `orchestration_tasks`. One new column (`message_type`) on `orchestration_messages`.

**Rejected columns:**
- `review_loop_count` — derivable from `SELECT COUNT(*) WHERE message_type = 'review_request'`
- `last_heartbeat_session_id` — redundant with `session_id` on tasks table
- `session_role` — derivable from `task_id = 'task-00'` convention

### 3. Guard Clause: 3 Claimable States

Only three states allow a new session to claim a task: `watching`, `fix_proposed`, `exit_requested`.

**Rationale:** Other states (`needs_review`, `review_approved`, `review_failed`, `error`) represent active protocol steps. If a session dies in those states, staleness detection moves the task to `fix_proposed`, which is then claimable. This prevents race conditions.

### 4. State Machine: No New States

The existing 11 states are sufficient. No `paused_for_context_check` or `resuming` states.

**Rationale:** Context warnings use `error` + `last_error = 'context_exhaustion_warning'` discriminator. Recovery is handled by the claim itself — no intermediate state needed.

### 5. Hook Preset Selection: task-00 Convention

The stop hook determines role by checking `task_id`: `task-00` = conductor preset, anything else = musician preset. No `session_role` column needed.

---

## Database Schema

All database operations MUST use comms-link MCP (query for SELECT, execute for writes). Direct `sqlite3` CLI access creates WAL isolation issues — comms-link cannot see changes made by sqlite3 and vice versa.

### Table: orchestration_tasks

```sql
CREATE TABLE orchestration_tasks (
    task_id TEXT PRIMARY KEY,
    state TEXT NOT NULL CHECK (state IN (
        'watching', 'reviewing', 'exit_requested', 'complete',
        'working', 'needs_review', 'review_approved', 'review_failed',
        'error', 'fix_proposed', 'exited'
    )),
    instruction_path TEXT,
    session_id TEXT,
    worked_by TEXT,
    started_at TEXT,
    completed_at TEXT,
    report_path TEXT,
    retry_count INTEGER DEFAULT 0,
    last_heartbeat TEXT,
    last_error TEXT
);
```

**Column semantics:**

| Column | Type | Purpose |
|--------|------|---------|
| `task_id` | TEXT PK | `task-00` (conductor), `task-01`..`task-NN` (musicians), `fallback-{session_id}` (guard block exits) |
| `state` | TEXT | Current state in the state machine (see State Machine section) |
| `instruction_path` | TEXT | Path to task instruction file (e.g., `docs/plans/implementation/task-03.md`) |
| `session_id` | TEXT | Actual Claude Code session ID (set by SessionStart hook via `$CLAUDE_SESSION_ID`) |
| `worked_by` | TEXT | Worker identifier with succession: `musician-task-03`, `musician-task-03-S2`, etc. |
| `started_at` | TEXT | ISO datetime when task was first claimed |
| `completed_at` | TEXT | ISO datetime when task reached terminal state |
| `report_path` | TEXT | Path to completion/error report file |
| `retry_count` | INTEGER | Error retry count (0-5). At 5, musician self-exits. |
| `last_heartbeat` | TEXT | ISO datetime of last heartbeat update. Updated on every state transition and by watcher refresh. |
| `last_error` | TEXT | Description of most recent error. Used to discriminate error subtypes (e.g., `context_exhaustion_warning`, `conductor_timeout`). |

### Table: orchestration_messages

```sql
CREATE TABLE orchestration_messages (
    id INTEGER PRIMARY KEY,
    task_id TEXT,
    from_session TEXT,
    message TEXT,
    message_type TEXT CHECK (message_type IN (
        'review_request', 'error', 'context_warning', 'completion',
        'emergency', 'handoff', 'approval', 'fix_proposal',
        'rejection', 'instruction', 'claim_blocked', 'resumption'
    )),
    timestamp TEXT DEFAULT CURRENT_TIMESTAMP
);
```

**Column semantics:**

| Column | Type | Purpose |
|--------|------|---------|
| `id` | INTEGER PK | Auto-incrementing message ID |
| `task_id` | TEXT | Which task this message concerns |
| `from_session` | TEXT | `$CLAUDE_SESSION_ID` of sender, or `task-00` for conductor |
| `message` | TEXT | Human-readable message body with structured sections |
| `message_type` | TEXT | Enum for cheap filtering without body parsing |
| `timestamp` | TEXT | ISO datetime, defaults to `CURRENT_TIMESTAMP` |

### Initialization SQL

```sql
-- Clean start for new implementation run
DROP TABLE IF EXISTS orchestration_tasks;
DROP TABLE IF EXISTS orchestration_messages;

-- Create tables (use DDL above)

-- Insert conductor row
INSERT INTO orchestration_tasks (task_id, state, last_heartbeat)
VALUES ('task-00', 'watching', datetime('now'));
```

---

## State Machine

### All States (11)

**Conductor-only states (task-00):**

| State | Set By | Meaning |
|-------|--------|---------|
| `watching` | Conductor | Monitoring execution tasks |
| `reviewing` | Conductor | Actively reviewing a submission |
| `exit_requested` | Conductor | Needs to exit (context, user consultation) |
| `complete` | Conductor | All tasks done |

**Musician states (task-01+):**

| State | Set By | Meaning |
|-------|--------|---------|
| `watching` | Conductor | Task created, not yet claimed |
| `working` | Musician | Actively executing task steps |
| `needs_review` | Musician | Checkpoint reached, awaiting conductor review |
| `review_approved` | Conductor | Review passed, musician should continue |
| `review_failed` | Conductor | Review rejected, musician should revise |
| `error` | Musician | Hit an error or context warning, awaiting conductor |
| `fix_proposed` | Conductor | Fix/handoff instructions sent, ready for claim |
| `complete` | Musician | Task finished (final DB write) |
| `exited` | Musician or Conductor | Terminated without completion (final DB write) |
| `exit_requested` | Conductor | Conductor wants musician to wrap up |

### Valid Transitions

```
Musician-initiated:
  watching → working              (atomic claim at bootstrap)
  working → needs_review          (checkpoint reached, tests pass)
  working → error                 (failure, context warning, conductor timeout)
  review_approved → working       (musician resumes after approval)
  review_failed → working         (musician applies feedback, resumes)
  fix_proposed → working          (musician applies fix, resumes)
  working → complete              (after final approval + cleanup — LAST write)
  working → exited                (clean handoff for context exit — LAST write)
  error → exited                  (unrecoverable — e.g., double timeout, 5th retry)

Conductor-initiated:
  watching → watching             (task-00: continues monitoring)
  watching → reviewing            (task-00: begins review)
  reviewing → watching            (task-00: review complete, back to monitoring)
  needs_review → review_approved  (task-NN: review passes)
  needs_review → review_failed    (task-NN: review rejected)
  error → fix_proposed            (task-NN: fix/guidance sent)
  watching → exit_requested       (either: conductor wrapping up)
  * → complete                    (task-00: all tasks done)
  * → exited                      (staleness detection: mark abandoned task)
```

### Guard Clause (Atomic Claim)

A new musician session claims a task with:

```sql
UPDATE orchestration_tasks
SET state = 'working',
    session_id = '$CLAUDE_SESSION_ID',
    worked_by = 'musician-task-{NN}',
    started_at = datetime('now'),
    last_heartbeat = datetime('now'),
    retry_count = 0
WHERE task_id = 'task-{NN}'
  AND state IN ('watching', 'fix_proposed', 'exit_requested');
```

**Only 3 states are claimable.** Verify `rows_affected = 1`. If 0, the guard blocked — create a fallback row (see Fallback Row Pattern).

### Fallback Row Pattern

When the guard blocks a claim, the session must exit cleanly. The stop hook needs an `exited` state for this session:

```sql
-- Always succeeds (no guard — unique PK)
INSERT INTO orchestration_tasks (task_id, state, session_id, last_heartbeat)
VALUES ('fallback-$CLAUDE_SESSION_ID', 'exited', '$CLAUDE_SESSION_ID', datetime('now'));

-- Notify conductor
INSERT INTO orchestration_messages (task_id, from_session, message, message_type)
VALUES ('task-{NN}', '$CLAUDE_SESSION_ID',
    'CLAIM BLOCKED: Guard prevented claim on task-{NN}. Created fallback row to exit cleanly.',
    'claim_blocked');
```

### Terminal States

`complete` and `exited` are terminal. The stop hook allows session exit when the task is in either state. No further state transitions should occur after reaching a terminal state.

### Heartbeat Rule

**Every state transition MUST update `last_heartbeat`:**

```sql
UPDATE orchestration_tasks
SET state = '{new_state}', last_heartbeat = datetime('now')
WHERE task_id = '{task_id}';
```

This is non-negotiable. Omitting `last_heartbeat` from a state update is a bug.

---

## Heartbeat Protocol

**Refresh rate:** 8 minutes (480 seconds) — watchers refresh heartbeat if older than this.
**Staleness threshold:** 9 minutes (540 seconds) — conductor considers session dead if heartbeat exceeds this.
**Buffer:** 1-minute gap prevents false alarms while catching genuine crashes.

### Heartbeat Update Points

1. **State transitions** — automatic, every state change includes `last_heartbeat = datetime('now')`
2. **Step boundaries** — musician updates when starting/completing steps
3. **Watcher refresh** — both background and pause watchers check heartbeat age each poll cycle; refresh if >480 seconds

### Staleness Detection (Conductor / Monitoring Subagent)

```sql
SELECT task_id, state,
       (julianday('now') - julianday(last_heartbeat)) * 86400 as seconds_stale
FROM orchestration_tasks
WHERE state IN ('working', 'review_approved', 'review_failed', 'fix_proposed')
  AND (julianday('now') - julianday(last_heartbeat)) * 86400 > 540;
```

When a task is stale:
1. Session is likely dead
2. Conductor sets state to `fix_proposed` (or `exited` if unrecoverable)
3. Conductor informs user to launch replacement session

### Conductor Heartbeat (task-00)

Musicians check the conductor's heartbeat before declaring timeout:

```sql
SELECT last_heartbeat,
       (julianday('now') - julianday(last_heartbeat)) * 86400 as orch_stale
FROM orchestration_tasks WHERE task_id = 'task-00';
```

- If `orch_stale < 540`: conductor alive but busy — keep waiting
- If `orch_stale >= 540`: conductor may be down — escalate

The conductor MUST refresh its own heartbeat periodically. The monitoring subagent also refreshes `task-00` heartbeat during its poll cycle.

---

## Message Formats

All messages use human-readable structured text. The `message_type` column enables filtering without body parsing.

### Review Request (musician → conductor)

**message_type:** `review_request`

```
REVIEW REQUEST (Smoothness: X/9):
  Checkpoint: N of M
  Context Usage: XX%
  Self-Correction: YES/NO (details if YES)
  Deviations: N (severity — description)
  Agents Remaining: N (~X% each, ~Y% total)
  Proposal: path/to/proposal.md
  Summary: what was accomplished
  Files Modified: N
  Tests: status (M total, N new)
```

### Context Warning (musician → conductor)

**message_type:** `context_warning`
**Task state:** `error` with `last_error = 'context_exhaustion_warning'`

```
CONTEXT WARNING: XX% usage
  Self-Correction: YES/NO
  Agents Remaining: N (~X% each, ~Y% total)
  Agents That Fit in 65% Budget: N
  Deviations: N (details)
  Proposal: what musician suggests doing
  Awaiting conductor instructions
```

### Error Report (musician → conductor)

**message_type:** `error`
**Task state:** `error`

```
ERROR (Retry N/5):
  Context Usage: XX%
  Self-Correction: YES/NO
  Error: description
  Report: docs/implementation/reports/task-{NN}-error-retry-{N}.md
  Awaiting conductor fix proposal
```

### Completion Report (musician → conductor)

**message_type:** `completion`
**Task state:** `needs_review` (completion requires final approval before `complete`)

```
TASK COMPLETE (Smoothness: X/9):
  Context Usage: XX%
  Self-Correction: YES/NO
  Deviations: N
  Report: docs/implementation/reports/task-{NN}-completion.md
  Summary: All deliverables created, tests passing
  Files Modified: N
  Tests: All passing (M tests, N new)
```

### Clean Exit / Handoff (musician → conductor)

**message_type:** `handoff`
**Task state:** `exited`

```
EXITED: Context exhaustion, clean handoff prepared.
  HANDOFF: temp/task-{NN}-HANDOFF
  Context Usage: XX%
  Last Completed Step: N
  Remaining Steps: list
```

### Emergency Broadcast (conductor → all musicians)

**message_type:** `emergency`

One INSERT per affected task_id. Each musician's watcher monitors only its own task_id.

```
EMERGENCY: description of cross-cutting issue
  Action Required: what musician should do
  Urgency: immediate / next-checkpoint
```

### Approval (conductor → musician)

**message_type:** `approval`

```
REVIEW APPROVED: feedback
  (Optional) Set self-correction flag to false — this was minor.
  Proceed with remaining steps.
```

### Rejection (conductor → musician)

**message_type:** `rejection`

```
REVIEW FAILED (Smoothness: X/9):
  Issue: what's wrong
  Required: specific changes needed
  Retry: instructions for re-submission
```

### Fix Proposal (conductor → musician)

**message_type:** `fix_proposal`

```
FIX PROPOSAL (Retry N/5):
  Root cause: analysis
  Fix: specific instructions
  Retry: what to do after applying fix
```

### Task Instruction (conductor → musician)

**message_type:** `instruction`

```
TASK INSTRUCTION: task-{NN}
  Instruction file: docs/plans/implementation/task-{NN}.md
  Phase: N
  Dependencies: none / list
```

### Claim Blocked (musician → conductor)

**message_type:** `claim_blocked`

```
CLAIM BLOCKED: Guard prevented claim on task-{NN}.
  Created fallback row to exit cleanly. Conductor intervention needed.
```

### Resumption Status (new musician → conductor)

**message_type:** `resumption`

```
RESUMPTION: musician-task-{NN}-S{N} taking over
  Previous session: {session_id from HANDOFF}
  HANDOFF: present / missing / stale
  Context Usage: XX% (fresh session)
  Deviations found: N (severity breakdown)
  Status/comms mismatches: N or none
  Pending conductor messages: N
  Self-correction in previous session: YES/NO
  Assessment: clean handoff / needs verification / needs conductor guidance
  Resuming from: step N, description
```

---

## Session Lifecycle

### Musician Bootstrap (9 steps)

1. **Parse task identity** — Extract task number from launch prompt → `task-{NN}`
2. **Session identity** — `$CLAUDE_SESSION_ID` is injected by SessionStart hook into system prompt
3. **Atomic claim** — Guard clause UPDATE (3 claimable states only)
4. **Guard block fallback** — If claim fails: insert fallback row, notify conductor, exit
5. **Check for HANDOFF** — If `temp/task-{NN}-HANDOFF` exists, this is a resumption (see Resumption below)
6. **Read task instructions** — Query `orchestration_messages` for instruction message
7. **Initialize temp files** — Create `temp/task-{NN}-status`, `temp/task-{NN}-deviations`
8. **Launch background watcher** — Start message monitoring subagent
9. **Begin execution** — Start working through task instruction steps

### Musician Work Cycle

```
Execute step → Update temp/status → Check context
    │
    ├─ Context OK → Continue to next step
    │
    ├─ Checkpoint reached → Set needs_review → Send review request
    │   → Launch pause watcher (blocks) → Process response → Resume
    │
    ├─ Context warning (>65% with remaining work) → Set error
    │   → Send context warning → Launch pause watcher → Process response
    │
    └─ Error → Increment retry_count → Set error
        → Send error report → Launch pause watcher → Process response
```

### Session Handoff

Three handoff types, plus retry exhaustion as a special case:

| Type | Condition | Key Difference |
|------|-----------|----------------|
| **Clean** | HANDOFF present, context <80% | Standard: read HANDOFF, set `fix_proposed`, send handoff msg, inform user |
| **Dirty** | HANDOFF present, context >80% | Same as clean BUT handoff msg includes test verification instructions (hallucination risk) |
| **Crash** | No HANDOFF | Send msg with verification instructions for most recently completed/worked steps |
| **Retry Exhaustion** | 5th retry failure | Conductor error — escalate to user with options (retry with new instructions, skip, investigate) |

**Conductor actions on musician exit:**
1. Detect exit via monitoring (state = `exited` or stale heartbeat)
2. Read `temp/task-{NN}-HANDOFF` if present
3. Assess handoff type (clean/dirty/crash)
4. Set task state to `fix_proposed`
5. Send handoff message with recovery instructions
6. Inform user: "Task-{NN} musician exited. Please launch replacement: `claude ...`"

**worked_by succession:**
- First session: `musician-task-{NN}`
- Second session: `musician-task-{NN}-S2`
- Third session: `musician-task-{NN}-S3`

### Resumption (New Session Taking Over)

```
New session starts → Atomic claim → Check temp/task-{NN}-HANDOFF
    │
    ├─ HANDOFF exists → Clean exit. Read contents.
    │
    └─ HANDOFF missing → Crash exit. Read temp/task-{NN}-status for last known state.
    │
    ├─ Read conductor's handoff message from orchestration_messages
    ├─ Read temp/task-{NN}-deviations
    │
    ├─ ≤2 deviations, no mismatches → Continue from last checkpoint
    │
    └─ >2 deviations OR status/comms mismatch → Report to conductor, wait for guidance
    │
    ├─ Re-run verification tests (ALWAYS, even if conductor approved)
    ├─ Write verbose resumption status to temp/ AND orchestration_messages
    ├─ Launch background watcher
    └─ Resume execution
```

---

## Watcher Protocol

### Background Watcher (During Active Work)

- **When:** Running while musician works
- **Mode:** Background (`run_in_background=True`)
- **Poll interval:** 15 seconds
- **Monitors:** `orchestration_messages` for new messages from conductor
- **Heartbeat:** Refreshes task heartbeat if >8 minutes old
- **On message:** Returns to musician with message content

### Pause Watcher (During Conductor Wait)

- **When:** Musician sets `error` or `needs_review`, blocks until conductor responds
- **Mode:** Foreground (blocks musician)
- **Poll interval:** 10 seconds
- **Monitors:** `orchestration_tasks` for state change away from `error`/`needs_review`
- **Heartbeat:** Refreshes task heartbeat if >8 minutes old
- **Timeout:** 15 minutes, then checks conductor heartbeat before declaring timeout
- **On state change:** Reads latest message, returns state + message to musician

### Watcher Lifecycle

```
Bootstrap → Background Watcher (working)
    ↓ (checkpoint or error)
Terminate background → Exit subagents → Pause Watcher (blocked)
    ↓ (conductor responds)
Terminate pause → Background Watcher (resumed)
    ↓ (cycle repeats until completion)
Terminate all → Clean exit
```

### Timeout Handling

1. Pause watcher reaches 15-minute timeout
2. Check conductor heartbeat (`task-00`)
3. If conductor alive (heartbeat <540s): reset timeout, keep waiting
4. If conductor stale (heartbeat >=540s): return TIMEOUT
5. Musician sets `error` with `last_error = 'conductor_timeout'`, sends message
6. Relaunch pause watcher (waiting for conductor recovery)
7. Second timeout: write HANDOFF, set `exited`

---

## Hook Infrastructure

### SessionStart Hook

**File:** `tools/implementation-hook/session-start-hook.sh`
**Input:** JSON from Claude Code containing `session_id`
**Output:** `additionalContext` injecting `CLAUDE_SESSION_ID={session_id}` into system prompt

The session ID is available as a system prompt value, not a bash environment variable. Claude reads it from the system prompt and uses it in database operations.

### Stop Hook

**File:** `tools/implementation-hook/hooks/stop-hook.sh` (via `hooks.json`)
**Input:** JSON from Claude Code containing `session_id`
**Behavior:**
1. Extract `session_id` from hook input
2. Query `orchestration_tasks` for row matching this `session_id`
3. Determine preset: `task-00` → orchestration preset, else → execution preset
4. Check if task state matches preset's `exit_criteria`
5. If exit criteria met: allow exit
6. If not: inject `fallback_prompt`, increment iteration counter
7. If iterations exceed `max_iterations`: force exit

### Presets

**Orchestration (`task-00`):**
- Exit criteria: `exit_requested`, `complete`
- Max iterations: 1000

**Execution (all other tasks):**
- Exit criteria: `complete`, `exited`
- Max iterations: 500

---

## File Path Conventions

### Persistent Files (survive reboots)

| File | Path | Purpose |
|------|------|---------|
| Task instructions | `docs/plans/implementation/task-{NN}.md` | Self-contained instruction files |
| Completion report | `docs/implementation/reports/task-{NN}-completion.md` | Final report for completed task |
| Error report | `docs/implementation/reports/task-{NN}-error-retry-{N}.md` | Per-retry error analysis |
| Handoff report | `docs/implementation/reports/task-{NN}-handoff-s{N}.md` | Session handoff report |
| Proposals | `docs/implementation/proposals/{date}-{topic}.md` | Structured change requests |

### Ephemeral Files (cleared on reboot)

| File | Path | Purpose |
|------|------|---------|
| Status log | `temp/task-{NN}-status` | Append-only step progress with context % |
| Deviations log | `temp/task-{NN}-deviations` | Tracked deviations from plan |
| HANDOFF | `temp/task-{NN}-HANDOFF` | Clean exit handoff document |
| Hook iterations | `temp/hook-{session_id}.iterations` | Stop hook iteration counter |

### Status File Format

Append-only entries with context percentage:
```
step 1 started [ctx: 12%]
step 1 completed [ctx: 18%]
step 2 agent 1 launched [ctx: 21%]
step 2 agent 1 returned [ctx: 29%]
step 2 deviation: switched parsing strategy (Medium) [ctx: 41%]
step 2 self-correction: test failure in parser, rewrote tokenizer [ctx: 34%]
```

### HANDOFF File Structure

```markdown
# HANDOFF: task-{NN}

## Session Info
- Session ID: {$CLAUDE_SESSION_ID}
- worked_by: musician-task-{NN}
- Exit reason: {context exhaustion / conductor requested / etc.}
- Context at exit: XX%
- Timestamp: {datetime}

## Completed Steps
- Step 1: {description} completed
- Step 2: {description} completed
- Step 3: {description} partial (agents 1-2 done, agent 3 not started)

## Pending Steps
- Step 3 agent 3: {what remains}
- Step 4: {full description}

## Deviations
- {list from temp/task-{NN}-deviations, or "none"}

## Self-Correction
- YES/NO, with details if YES

## Pending Proposals
- {list any proposals not yet reviewed}

## Next Session Instructions
1. Claim task with worked_by = musician-task-{NN}-S2
2. Read this HANDOFF + conductor's handoff message
3. Re-run verification tests from last checkpoint
4. Continue from {where left off}
```

---

## Error Prioritization

When the conductor has multiple pending items, handle in this order:

1. **Errors** — task in `error` state (blocking musician)
2. **Reviews** — task in `needs_review` state (blocking musician)
3. **Completions** — task reporting complete (non-blocking)

---

## Self-Correction Flag

Musician reports `Self-Correction: YES/NO` in ALL messages. When YES:

- Context estimates are unreliable (~6x bloat)
- Conductor should treat remaining scope estimates with skepticism
- If context is inline with task estimates: tell musician to reset flag to false
- If context >2x task estimate: warn user they may need an additional session
- If context >40% and not at final checkpoint: set `fix_proposed`, have musician estimate context to next checkpoint

Task instructions include rough context usage estimates per step. Conductor compares actual vs estimated by selectively reading the instruction file (grep for estimate keywords).

---

## Emergency Broadcasts

For cross-cutting issues affecting multiple musicians, conductor inserts **one message per affected task_id**:

```sql
-- One INSERT per task (musician watchers only monitor their own task_id)
INSERT INTO orchestration_messages (task_id, from_session, message, message_type)
VALUES ('task-03', 'task-00', 'EMERGENCY: ...', 'emergency');

INSERT INTO orchestration_messages (task_id, from_session, message, message_type)
VALUES ('task-04', 'task-00', 'EMERGENCY: ...', 'emergency');
```

No body parsing needed — each musician's background watcher detects messages on its own `task_id`. The conductor has full discretion over when and why to broadcast.

---

## Monitoring Subagent

The conductor launches a background monitoring subagent that polls for state changes across all tasks.

**Poll cycle:**
1. Query all task states and heartbeats
2. Detect: `needs_review`, `error`, `complete`, `exited`, stale heartbeats
3. Refresh conductor heartbeat (`task-00`)
4. Check for fallback rows (`task_id LIKE 'fallback-%'`)
5. Sleep, repeat

**Fallback row handling:**
1. Extract original task_id from fallback's message
2. Compare timestamps: `fallback.last_heartbeat` vs `task.last_heartbeat`
3. Task timestamp > fallback → task was worked since collision → DELETE fallback
4. Task timestamp <= fallback → task NOT worked since collision → report to user

---

## Smoothness Scale (Review Scoring)

| Score | Meaning | Conductor Action |
|-------|---------|---------------------|
| 0 | Perfect execution | Approve |
| 1-2 | Minor clarifications, self-resolved | Approve |
| 3-4 | Some deviations, documented | Approve |
| 5 | Borderline, conductor judgment | Approve (usually) |
| 6-7 | Significant issues | Request revision (`review_failed`) |
| 8-9 | Major blockers or failure | Reject with detailed feedback |

---

## Database Access Restrictions

Each skill should only touch rows it owns:

**Conductor (task-00):**
- READ any row in `orchestration_tasks` (monitoring)
- WRITE only `task-00` state/heartbeat and task-NN states it owns (review responses, staleness cleanup)
- READ/WRITE any row in `orchestration_messages`

**Musician (task-NN):**
- READ/WRITE only its own `task-NN` row in `orchestration_tasks` (state, heartbeat)
- READ `task-00` row (conductor heartbeat check)
- READ sibling task states (parallel awareness, dependency checks)
- WRITE to `orchestration_messages` only with `task_id = own task_id`
- READ from `orchestration_messages` only where `task_id = own task_id`

**Fallback exception:** Guard-blocked sessions write `fallback-{session_id}` rows (unique task_id).

---

## Git Operation Restrictions

Both skills follow these safety rules:

**Allowed:**
- `git status`, `git diff`, `git log` (read-only inspection)
- `git add` specific files (not `git add -A` or `git add .`)
- `git commit` (with meaningful messages)
- `git checkout` / `git switch` (branch switching)
- `git stash` / `git stash pop` (temporary storage)
- `git pull` (fetch + merge)

**Forbidden (require explicit user approval):**
- `git push --force` or `git push -f`
- `git reset --hard`
- `git clean -f`
- `git branch -D` (force delete)
- `git rebase` on published branches
- `git checkout .` or `git restore .` (discard all changes)
- Any operation that destroys uncommitted work

**Musician-specific:** Musicians should not push to remote. Commits are local; conductor or user handles push decisions.

---

## Indexes

Add these indexes after table creation for query performance:

```sql
-- Fast message lookup by task + time (watcher polling)
CREATE INDEX idx_messages_task_time ON orchestration_messages(task_id, timestamp);

-- Fast message filtering by type
CREATE INDEX idx_messages_type ON orchestration_messages(message_type);

-- Fast staleness detection
CREATE INDEX idx_tasks_state_heartbeat ON orchestration_tasks(state, last_heartbeat);
```

---

## Retry Limits

- **Musician error retries:** 0-5. At retry 5, musician writes HANDOFF and self-exits. This is an conductor error requiring user intervention.
- **Conductor subagent retries:** 3 maximum. After 3, escalate to user.
- **`retry_count` tracks error retries only**, not review cycles.
