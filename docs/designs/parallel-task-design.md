# Parallel Task Design

**Version:** 1.0.0
**Date:** 2026-02-04
**Status:** Active
**Purpose:** Comprehensive specification for writing task instruction files for autonomous parallel orchestration using custom hooks and background subagents

---

## Document Overview

### What This Template Is

This is a **comprehensive specification template** for creating task instruction files that enable autonomous parallel orchestration of Claude Code sessions. It defines how to write instructions for:

1. **Conductor sessions** - Coordinate multiple execution sessions autonomously
2. **Execution sessions** - Perform work while coordinating with conductor
3. **Custom hook integration** - Enforce state-based session lifecycle management
4. **Background subagent coordination** - Monitor for messages without polluting context

### When to Use This Template

**This template is for PARALLEL multi-session orchestration.** It describes an evolved coordination pattern using custom hooks and background subagents. This is a supplemental pattern to the existing sequential coordination model, not a replacement.

---

#### Pattern Selection Guide

**Use THIS template (Autonomous Parallel Orchestration) when:**
- ✅ You need **3+ independent tasks** working in parallel
- ✅ Sessions must coordinate autonomously (no user bridge)
- ✅ Work can be divided into truly independent tasks (no shared files)
- ✅ Tasks require review checkpoints during execution
- ✅ Context efficiency is critical for long-running work
- ✅ Coordination overhead is justified by parallelism gains

**Example:** Documentation extraction with 4 parallel tasks (testing patterns, API patterns, database patterns, templates) - each task reads different source files, writes to different subdirectories, no conflicts.

---

**Use EXISTING pattern (Sequential/Subagent-Driven-Development) when:**
- ✅ Single session can complete the work
- ✅ Tasks have sequential dependencies (Task B needs Task A output)
- ✅ Tasks might conflict (edit same files)
- ✅ Work is straightforward and doesn't justify parallel overhead
- ✅ User bridge coordination is acceptable
- ✅ Task is exploratory or requires frequent user decisions

**Example:** Task-03 (extract testing patterns) as a standalone task - reads 6 files, extracts patterns, creates proposal, gets review, finalizes. Sequential workflow with manual SQL message checks before each step.

---

#### Key Differences Between Patterns

| Aspect | Sequential (Existing) | Parallel (This Template) |
|--------|----------------------|-------------------------|
| **Sessions** | 1 execution session | 1 conductor + N execution sessions |
| **Coordination** | Manual SQL queries before steps | Custom hooks + background subagents |
| **Message checking** | `SELECT FROM task_messages` before each step | Background subagent monitors continuously |
| **Hook usage** | Optional (for state enforcement) | Required (prevents premature exit) |
| **Context overhead** | Lower (1 session) | Higher (N+1 sessions, but isolated) |
| **Setup complexity** | Simple (just task instructions) | Complex (hooks, subagents, state machine) |
| **Use case** | Simple, sequential, single-task | Complex, parallel, multi-task |
| **User involvement** | Can act as bridge if needed | Minimal (autonomous coordination) |

---

**Do NOT use this template when:**
- ❌ Single session can complete the work
- ❌ Tasks have sequential dependencies
- ❌ Tasks might conflict on shared resources
- ❌ Work is trivial and doesn't justify coordination overhead
- ❌ You're unfamiliar with the pattern (start with sequential)

---

#### Shared Files Classification for Code Projects

Understanding what constitutes "shared files" is critical for determining if tasks can safely run in parallel. This section defines shared file categories for code projects.

**Read-Only Sharing (SAFE for Parallelization)**

**Definition:** Files that multiple tasks import/use but do NOT modify.

**Examples:**
- ✅ **Type definitions:** `types/User.ts` - tasks import User type but don't modify definition
- ✅ **Constants & config:** `config/constants.ts` - tasks read values, don't change
- ✅ **Utility functions:** `utils/formatting.ts` - tasks use existing functions, don't add new
- ✅ **Design tokens:** `theme/colors.ts` - tasks use colors, don't modify palette
- ✅ **Third-party packages:** `node_modules/` - managed by package.json, not by tasks
- ✅ **Database schema (existing):** Tasks query existing tables, don't run migrations

**Parallelization:** ✅ SAFE - Read-only access cannot conflict

**Example:**
```typescript
// types/User.ts (read-only shared file)
export interface User {
  id: string;
  name: string;
  email: string;
}

// task-03 imports (safe)
import { User } from '../types/User';

// task-04 also imports (safe - no conflict)
import { User } from '../types/User';
```

**Write Access Sharing (UNSAFE for Parallelization)**

**Definition:** Files that multiple tasks need to MODIFY (add content, change values).

**Examples:**
- ❌ **Barrel exports:** `components/index.ts` - both tasks add new export lines
- ❌ **Shared utility files (adding functions):** `utils/formatting.ts` - task-03 adds `formatCurrency()`, task-04 adds `formatDate()`
- ❌ **Database schema (new migrations):** task-03 adds users table, task-04 adds posts table (migration numbers conflict)
- ❌ **API route files (adding endpoints):** `routes/api.ts` - both tasks add new routes to same file
- ❌ **Configuration aggregation:** `config/index.ts` - both tasks add new config sections

**Parallelization:** ❌ UNSAFE - Write conflicts likely, merge conflicts guaranteed

**Example:**
```typescript
// components/index.ts (UNSAFE - both tasks modify)

// Initial state:
export { Button } from './Button';

// task-03 wants to add:
export { ProfileCard } from './ProfileCard'; // Line 2

// task-04 wants to add:
export { Avatar } from './Avatar'; // Also line 2

// Result: MERGE CONFLICT
```

**Case-by-Case Analysis (DEPENDS on Access Pattern)**

**Test fixtures:**
- ✅ SAFE: Tasks use different fixture files (`fixtures/task03_data.json`, `fixtures/task04_data.json`)
- ❌ UNSAFE: Tasks modify shared fixture file (`fixtures/common.json`)

**Middleware:**
- ✅ SAFE: Tasks create new middleware files (`middleware/rateLimit.ts`, `middleware/cors.ts`)
- ❌ UNSAFE: Tasks modify existing middleware file (`middleware/index.ts` to register both)

**Storybook stories:**
- ✅ SAFE: Tasks create separate story files (`Button.stories.tsx`, `Card.stories.tsx`)
- ❌ UNSAFE: Tasks modify shared story configuration (`.storybook/main.ts`)

**Database migrations:**
- ✅ SAFE: Sequential migration numbers guaranteed unique (conductor assigns numbers)
- ❌ UNSAFE: Tasks choose their own migration numbers (001_users.sql vs 001_posts.sql conflicts)

---

#### Danger-Files Protocol

For files that multiple tasks need to modify, use the danger-files protocol to coordinate updates centrally.

**Concept:** Identify "danger files" that multiple tasks might modify, handle specially.

**Step 1: Danger File Identification (During Task Planning)**

Conductor creates danger file list before launching tasks:

```markdown
## Danger Files for This Project

**Files multiple tasks might modify:**
1. `src/components/index.ts` (barrel export)
2. `src/types/index.ts` (type re-exports)
3. `src/app.ts` (middleware registration)
4. `src/db/migrations/` (migration numbering)

**Danger handling strategy:**
- **Option A:** Conductor updates these files after all tasks complete
- **Option B:** Tasks message conductor "Need to add export to components/index.ts"
- **Option C:** Create task-specific files (no shared barrel exports)

**This project uses:** Option B (message conductor for centralized updates)
```

**Step 2: Task Execution with Danger File Protocol**

Tasks detect when they need to modify danger file:

```markdown
# task-03 execution

## Step 5: Export ProfileCard component

**Danger file detected:** `src/components/index.ts` is on danger list

**Action:** Message conductor instead of direct modification

```sql
INSERT INTO task_messages VALUES (
    'task-00', 'task-03',
    'DANGER FILE UPDATE REQUEST:
    File: src/components/index.ts
    Action: Add export
    Line to add: export { ProfileCard } from "./ProfileCard";
    Rationale: Component complete, needs to be exported from barrel file'
);
```

**Continue execution:** Don't block on danger file, continue with other work
```

**Step 3: Conductor Centralizes Danger File Updates**

After all tasks complete (or at review checkpoints):

```markdown
# Conductor processes danger file requests

## Messages received:
- task-03: Add export for ProfileCard
- task-04: Add export for Avatar
- task-05: No danger file requests

## Batch update to src/components/index.ts:

```typescript
// Before:
export { Button } from './Button';

// After (conductor applies all changes):
export { Button } from './Button';
export { ProfileCard } from './ProfileCard'; // task-03
export { Avatar } from './Avatar'; // task-04
```

## Commit:
```bash
git add src/components/index.ts
git commit -m "chore: export new components from barrel file

- ProfileCard (task-03)
- Avatar (task-04)"
```
```

---

#### Real-World Parallelization Examples

**Example 1: Component Library (3 Components)**

```markdown
**task-03:** Create Button component
- WRITE: `components/Button.tsx`, `components/Button.test.tsx`
- DANGER: `components/index.ts` (export)

**task-04:** Create Card component
- WRITE: `components/Card.tsx`, `components/Card.test.tsx`
- DANGER: `components/index.ts` (export)

**task-05:** Create Avatar component
- WRITE: `components/Avatar.tsx`, `components/Avatar.test.tsx`
- DANGER: `components/index.ts` (export)

**Analysis:**
- Write-write conflicts: NONE (each task writes different files)
- Danger files: YES (`components/index.ts` needed by all)

**Decision:** ✅ PARALLEL with danger-files protocol
- All tasks execute simultaneously
- Each messages conductor for export
- Conductor batches updates to index.ts after all complete
```

**Example 2: API Endpoints (Shared Router File)**

```markdown
**task-03:** Create /users endpoint
- WRITE: `api/users.ts`, `api/users.test.ts`
- DANGER: `api/index.ts` (router registration)

**task-04:** Create /posts endpoint
- WRITE: `api/posts.ts`, `api/posts.test.ts`
- DANGER: `api/index.ts` (router registration)

**Analysis:**
- Write-write conflicts: NONE (different endpoint files)
- Danger files: YES (`api/index.ts`)

**Decision:** ✅ PARALLEL with danger-files protocol
- Tasks message conductor for router registration
- Conductor updates api/index.ts after review checkpoints
```

**Example 3: Database Migrations (Conflict)**

```markdown
**task-03:** Add users table
- WRITE: `db/migrations/001_create_users.sql` (⚠️ chooses number 001)

**task-04:** Add posts table
- WRITE: `db/migrations/001_create_posts.sql` (⚠️ also chooses 001)

**Analysis:**
- Naming conflict: Both tasks choose "001_" prefix

**Decision:** ❌ UNSAFE - Must serialize OR pre-assign numbers

**Solution A (Serialize):**
- task-03 completes migration first
- task-04 starts after, sees 001 exists, uses 002

**Solution B (Pre-assign):**
- Conductor assigns numbers before launch:
  - task-03: Use prefix "003_"
  - task-04: Use prefix "004_"
- Tasks run in parallel with guaranteed unique numbers
```

---

#### Parallelization Decision Matrix

Use this workflow to decide if tasks can run in parallel:

**For each pair of tasks (task-A, task-B):**

**Step 1: List All File Operations**

**task-A files:**
- READ: [list all files task-A will read]
- WRITE: [list all files task-A will create or modify]

**task-B files:**
- READ: [list all files task-B will read]
- WRITE: [list all files task-B will create or modify]

**Step 2: Check for Write Conflicts**

**Write-Write conflicts:** Do WRITE lists overlap?
- ❌ YES → CONFLICT → Must serialize OR use danger-files approach
- ✅ NO → SAFE → Continue to Step 3

**Read-Write conflicts:** Does task-A WRITE overlap with task-B READ?
- ⚠️ YES → DEPENDENCY → task-B must wait for task-A checkpoint
- ✅ NO → SAFE → Continue to Step 3

**Step 3: Check for Danger Files**

**Danger files:** Do either task need to modify danger files?
- ⚠️ YES → USE DANGER-FILES PROTOCOL → Parallel OK with conductor coordination
- ✅ NO → SAFE → Full parallel execution

**Step 4: Final Decision**

| Scenario | Decision |
|----------|----------|
| Write-write conflict | SERIALIZE (no parallelization) |
| Read-write dependency | CHECKPOINT DEPENDENCY (task-B waits for task-A review) |
| Danger files only | PARALLEL + DANGER-FILES PROTOCOL |
| No conflicts | FULL PARALLEL |

---

### Quality Standards

This template produces **implementation-ready** task instructions that are:

1. **Unambiguous** - Execution sessions can follow without clarification
2. **Comprehensive** - All edge cases, error scenarios, and state transitions covered
3. **Autonomous** - No user intervention needed for coordination
4. **Verifiable** - Clear success criteria and verification steps
5. **Context-efficient** - Subagent patterns minimize main session context pollution

**Expected output quality:**
- Conductor can monitor and review 3-5 parallel tasks
- Execution sessions pause and resume autonomously
- Error recovery handles 5 retry attempts without user intervention
- Review checkpoints provide quality gates
- All coordination via database + custom hook (no polling loops in main session)

---

## Part 1: Foundation

### Prerequisites

Before using this template, you must have:

**1. Database Infrastructure**
```sql
-- migration_tasks: Full lifecycle tracking
CREATE TABLE migration_tasks (
    task_id TEXT PRIMARY KEY,
    instruction_path TEXT,
    status TEXT,
    worked_by TEXT,
    started_at TEXT,
    completed_at TEXT,
    report_path TEXT
);

-- coordination_status: Lightweight state for hook monitoring
CREATE TABLE coordination_status (
    task_id TEXT PRIMARY KEY,
    state TEXT NOT NULL
);

-- task_messages: Audit trail and message passing
CREATE TABLE task_messages (
    id INTEGER PRIMARY KEY,
    task_id TEXT,
    from_session TEXT,
    message TEXT,
    timestamp TEXT DEFAULT CURRENT_TIMESTAMP
);
```

<!-- ✅ VERIFIED SCHEMA - Matches Existing Database

This schema has been verified against the deployed coordination.db database used in the documentation extraction project. The tables exist with these exact column names and types.

**Verification command (optional, for new projects):**
```sql
-- Check if tables exist and match expected schema
.schema migration_tasks
.schema coordination_status
.schema task_messages
```

**For new projects using this template:**
If starting fresh, you'll need to create these tables. Add initialization scripts to your project setup:
```bash
# Example: tools/coordination-db/init.sql
# Contains CREATE TABLE statements for all three tables
sqlite3 coordination.db < tools/coordination-db/init.sql
```

For the remindly documentation extraction project, these tables already exist and are operational.
-->

## Database Schema Validation (CHECK Constraints)

### Purpose

CHECK constraints prevent invalid data from entering coordination_status:
- Invalid state names (typos: 'complet' instead of 'complete')
- Impossible state transitions (direct jump to inconsistent state)
- Data integrity violations

### Recommended Constraints

**State enumeration (prevent typos):**
```sql
CREATE TABLE coordination_status (
    task_id TEXT PRIMARY KEY,
    state TEXT NOT NULL CHECK (state IN (
        -- Conductor states
        'watching', 'reviewing', 'complete', 'exit_requested',
        -- Execution states
        'working', 'waiting', 'needs_review',
        'review_approved', 'review_failed',
        'error', 'fix_proposed', 'exited', 'complete'
    )),
    session_id TEXT,
    last_heartbeat TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    retry_count INTEGER DEFAULT 0 CHECK (retry_count >= 0 AND retry_count <= 5),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    last_error TEXT
);
```

**Retry count validation (enforce 5-retry limit):**
```sql
CHECK (retry_count >= 0 AND retry_count <= 5)
```

**Terminal state consistency (prevent modification after completion):**
```sql
-- Cannot be enforced at schema level in SQLite
-- Enforce in application logic:
-- Before UPDATE, verify state not already terminal
```

### Trade-offs

| Aspect | With Constraints | Without Constraints |
|--------|-----------------|---------------------|
| **Safety** | ✅ High - invalid data rejected | ❌ Low - garbage in, garbage out |
| **Flexibility** | ⚠️ Limited - schema changes need migration | ✅ High - any value accepted |
| **Debugging** | ✅ Easy - errors caught immediately | ❌ Hard - invalid data causes downstream issues |
| **Performance** | ⚠️ Minimal overhead (~1% query time) | ✅ No overhead |

**Recommendation:** Use CHECK constraints - safety worth minimal performance cost

### Implementation Notes

**SQLite limitations:**
- No ENUM type (use CHECK with IN clause)
- No BEFORE UPDATE triggers for complex validation
- CHECK constraints evaluated per-row (not per-transaction)

**PostgreSQL alternative (if using):**
```sql
CREATE TYPE coordination_state AS ENUM (
    'watching', 'reviewing', 'complete', 'exit_requested',
    'working', 'waiting', 'needs_review',
    'review_approved', 'review_failed',
    'error', 'fix_proposed', 'exited'
);

CREATE TABLE coordination_status (
    task_id TEXT PRIMARY KEY,
    state coordination_state NOT NULL,
    ...
);
```

### Migration Path

**Adding constraints to existing database:**
```sql
-- Step 1: Verify existing data is valid
SELECT task_id, state FROM coordination_status
WHERE state NOT IN (
    'watching', 'reviewing', 'complete', 'exit_requested',
    'working', 'waiting', 'needs_review',
    'review_approved', 'review_failed',
    'error', 'fix_proposed', 'exited'
);

-- If any rows returned: fix invalid states before migration

-- Step 2: Create new table with constraints
CREATE TABLE coordination_status_new (
    task_id TEXT PRIMARY KEY,
    state TEXT NOT NULL CHECK (state IN (...)),
    ...
);

-- Step 3: Copy data
INSERT INTO coordination_status_new SELECT * FROM coordination_status;

-- Step 4: Rename tables
DROP TABLE coordination_status;
ALTER TABLE coordination_status_new RENAME TO coordination_status;
```


**2. Custom Hook Installation**
- Message-watcher hook installed at `tools/message-watcher/`
- Hook registered in Claude Code configuration
- Database path configured in hook script
- Preset configurations available:
  - `preset-orchestration.yaml` - For conductor sessions
  - `preset-execution.yaml` - For execution sessions

**3. Task Identification**
- **task-00**: Reserved for conductor
- **task-01 through task-NN**: Execution tasks
- Consistent naming across all databases and files

**4. Directory Structure**
```
docs/
├── plans/
│   └── implementation/     # Task instruction files
├── implementation/
│   ├── reports/            # Completion reports
│   └── proposals/          # Work proposals (optional)
├── knowledge-base/         # Final deliverables (kept clean)
└── ...

temp/
└── [task-specific temporary files]
```

### Core Concepts

#### 1. Autonomous Orchestration

**Definition:** Multiple Claude Code sessions coordinate through database state and custom hooks without user acting as message bridge.

**Key characteristics:**
- **No polling loops** - Custom hook monitors database, blocks exit until criteria met
- **Background subagents** - Isolated context for monitoring, discarded after exit
- **State-based lifecycle** - Sessions transition through defined states
- **Message passing** - Database writes for coordination, not direct communication

**Why this matters:**
- Context efficiency: Subagent polling context discarded, not added to main session
- Reliability: Database state is source of truth, survives crashes
- Scalability: Can coordinate 3-5+ parallel sessions without overwhelming conductor
- Autonomy: No user intervention for coordination (only for errors/completion)

#### 2. Custom Hooks

**Definition:** Stop hooks that intercept session exit attempts, query database for task state, and block exit unless state matches exit criteria.

**How they work:**
```
Session tries to exit
  ↓
Hook queries: SELECT state FROM coordination_status WHERE task_id = ?
  ↓
State matches exit criteria? (e.g., complete, exited)
  ↓ YES → Allow exit
  ↓ NO  → Block exit, inject fallback prompt
  ↓
Session continues with prompt
```

**Hook configurations:**

**Conductor (preset-orchestration.yaml):**
- Monitors: task-00
- Exit criteria: `exit_requested`, `complete`
- Max iterations: 1000
- Fallback prompt: "Check task-00 status. If tasks need attention, process messages from subagent. Otherwise, launch subagent to listen."

**Execution (preset-execution.yaml):**
- Monitors: task-XX (replaced at runtime)
- Exit criteria: `complete`, `exited`
- Max iterations: 500
- Fallback prompt: "Check {{TASK_ID}} status. If messages need attention, process them. Otherwise, launch subagent to listen."

**Setup:**
```bash
# Conductor
bash tools/message-watcher/setup.sh --preset orchestration

# Execution
bash tools/message-watcher/setup.sh --preset execution --task-id task-03
```

#### 3. Background Subagents

**Definition:** Agents launched with `Task` tool using `run_in_background=true`, monitor database in isolated context, exit when conditions met.

**Two modes:**

**Background mode (during work):**
```python
# Main session launches subagent
Task(
    description="Monitor for conductor messages",
    prompt="Poll coordination_status and task_messages every 8s for task-03...",
    subagent_type="general-purpose",
    run_in_background=true
)

# Main session continues working on task steps
# Subagent watches database in parallel
# Main session checks subagent between steps (non-blocking)
```

**Blocking mode (during review wait):**
```python
# Main session launches subagent
Task(
    description="Wait for review approval",
    prompt="Poll coordination_status every 8s for task-03 state changes...",
    subagent_type="general-purpose",
    run_in_background=false  # Blocking
)

# Main session PAUSES until subagent exits
# Subagent exits when state = review_approved or review_failed
# Main session resumes with subagent result
```

**Context benefits:**
- Background: Main session adds ~500 tokens (launch + result), subagent context discarded
- Blocking: Main session pauses, no context added during wait
- vs Ralph Loop: Would add 6,000+ tokens for same polling duration

#### 4. State Machine

**Conductor states (task-00):**
- `watching` - Subagent active, waiting for work
- `reviewing` - Processing reviews/errors
- `exit_requested` - Manual exit or needs user input
- `complete` - All execution tasks finished

**Execution states (task-01, 02, 03, etc.):**
- `working` - Executing steps, background subagent monitoring
- `waiting` - Blocking subagent wait for response
- `needs_review` - Review requested
- `review_approved` - Approved (transient <60s)
- `review_failed` - Quality needs improvement (transient <60s)
- `error` - Recoverable error (retries 1-4)
- `fix_proposed` - Conductor proposed error fix (transient <60s)
- `exited` - Critical error (retry 5 exhausted), session terminated
- `complete` - Task finished

**State transitions (execution):**
```
working → needs_review → approved → working → complete (happy path)
working → needs_review → review_failed → working → needs_review (retry after rejection)
working → error → fix_proposed → working (retry, attempt 1-4)
working → error (5x) → exited (terminal)
```

**Transient states:**
- `review_approved`, `review_failed`, and `fix_proposed` exist for <60 seconds
- Execution session polls every 8s, detects quickly
- Conductor ignores these in main query (avoids race conditions)
- Staleness detection: If >60s in transient state = crashed session

#### 5. Session ID Naming Convention

**Definition:** Standardized format for task IDs and session IDs ensures consistency across all coordination database operations.

**Standard format (REQUIRED):**

**Conductor session:** `task-00`
- Rationale: Consistent with execution task numbering (task-03, task-04)
- Zero-padded for lexicographic sorting
- Used in: coordination_status.task_id, task_messages.task_id

**Execution sessions:** `task-NN` where NN is zero-padded two-digit number
- Examples: `task-03`, `task-04`, `task-05`, `task-10`
- Rationale: Lexicographic sorting matches numeric order (task-03 < task-10)
- Padding: Always two digits (01-99), allows up to 99 parallel tasks

**Process session ID (for session_id column):** `[hostname]-[YYYYMMDD]-[HHMMSS]-[random]`
- Example: `kyle-ubuntu-20260205-143022-a8f3`
- Rationale: Globally unique, includes timestamp for debugging
- Used in: coordination_status.session_id (for session isolation)

**Format rules:**

**Task ID format:** `task-NN`
- NN must be zero-padded (01, 02, 03, not 1, 2, 3)
- Conductor always uses 00
- Execution tasks use 03+ (01-02 reserved for future use)

**Session ID format:** `hostname-YYYYMMDD-HHMMSS-random`
- hostname: Machine running the session (lowercase, no spaces)
- YYYYMMDD: Date in ISO format (e.g., 20260205)
- HHMMSS: Time in 24-hour format (e.g., 143022)
- random: 4-character alphanumeric (lowercase)

**Invalid formats (DO NOT USE):**

| Invalid | Why | Use Instead |
|---------|-----|-------------|
| `O` | Too terse, not greppable | `task-00` |
| `E3` | Inconsistent with task-NN format | `task-03` |
| `task-3` | Not zero-padded, sorting breaks | `task-03` |
| `conductor` | Too verbose, breaks SQL column width | `task-00` |
| `exec-3-session` | Mixed format, unclear meaning | `task-03` |
| `task-003` | Three digits unnecessary (<100 tasks) | `task-03` |

**SQL query examples with standard IDs:**

**Query conductor status:**
```sql
SELECT * FROM coordination_status WHERE task_id = 'task-00';
```

**Query execution tasks:**
```sql
SELECT * FROM coordination_status
WHERE task_id LIKE 'task-%'
  AND task_id != 'task-00'
ORDER BY task_id;  -- Sorts: task-03, task-04, task-05, task-10
```

**Query messages from conductor:**
```sql
SELECT * FROM task_messages
WHERE from_session = 'task-00'
ORDER BY created_at DESC;
```

**Find active sessions:**
```sql
SELECT DISTINCT session_id
FROM coordination_status
WHERE state IN ('working', 'watching', 'reviewing', 'needs_review');
```

**Migration from old formats:**

If you see old formats in examples, replace:
- `O` → `task-00`
- `E3` → `task-03`
- `E4` → `task-04`
- `task-3` → `task-03` (add zero padding)
- `conductor-session-id` → `task-00`

**Why this matters:**

**Consistency benefits:**
- Predictable SQL queries (no format guessing)
- Grepable logs (`grep "task-03"` always works)
- Clear sorting (task-03 < task-04 < task-10)
- No ambiguity in examples (one correct format)

**Real-world scenario:**
```bash
# Debugging: Find all messages to task-03
sqlite3 coordination.db "SELECT * FROM task_messages WHERE task_id = 'task-03';"

# With inconsistent naming, this might miss messages if task used "E3" or "exec-3"
# With standard naming, guaranteed to find all messages
```

### Semantic Distinction: review_failed vs fix_proposed

**Critical for musician skill development:**

| State | Meaning | Execution Action | Message Format |
|-------|---------|------------------|----------------|
| `review_failed` | Quality/correctness inadequate | Read feedback, revise code for quality | "REVIEW FEEDBACK: [issues to address]" |
| `fix_proposed` | Technical fix ready to apply | Read fix, apply mechanically, retry | "FIX PROPOSAL (Retry X/5): [instructions]" |

**Key difference:**
- `review_failed`: Subjective quality judgment - execution decides HOW to fix
- `fix_proposed`: Objective technical fix - execution follows instructions exactly

**Example scenarios:**

**review_failed:**
```
"REVIEW FEEDBACK: Code quality needs improvement.
- Add error handling for database connection failures
- Improve variable naming (use descriptive names, not single letters)
- Add JSDoc comments to public functions"

Execution response: Read feedback, make improvements based on judgment, re-request review
```

**fix_proposed:**
```
"FIX PROPOSAL (Retry 2/5):
Root cause: Database query timeout after 5 seconds
Fix: Update db.query() timeout parameter from 5000ms to 30000ms in api/users.ts:45
Retry: Re-run the failed API test after applying fix"

Execution response: Apply fix exactly as specified, retry operation
```

## State Machine Design Constraints

### FIXED State Names - Cannot Be Customized

**CRITICAL:** State names in this template are FIXED and cannot be changed.

**Rationale:**
1. **Custom hooks depend on exact state names** - changing breaks coordination
2. **Cross-session consistency** - all sessions must use same vocabulary
3. **Template composability** - future tasks reference these specific states

**States that CANNOT be customized:**
- Conductor: `watching`, `reviewing`, `complete`, `exit_requested`
- Execution: `working`, `waiting`, `needs_review`, `review_approved`, `review_failed`, `error`, `fix_proposed`, `exited`, `complete`

**If you need domain-specific states:**
- ❌ Do NOT rename existing states
- ❌ Do NOT add new states to coordination_status
- ✅ DO use task_messages for domain-specific state information
- ✅ DO extend metadata in separate tracking tables

**Example (WRONG):**
```sql
-- ❌ DO NOT DO THIS
UPDATE coordination_status SET state = 'testing' WHERE task_id = 'task-03';
-- Breaks: Conductor doesn't recognize 'testing', hooks fail
```

**Example (CORRECT):**
```sql
-- ✅ Use standard state + message for context
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';

INSERT INTO task_messages VALUES (
    'task-00', 'task-03',
    'STATUS UPDATE: Running integration tests (step 3 of 5)'
);

-- Conductor sees 'working' (valid state)
-- Message provides domain-specific context (testing)
```

### Why State Names Are Fixed

**Hook Exit Criteria Example:**
```bash
#!/bin/bash
# Custom stop hook - checks if work complete before allowing exit

state=$(sqlite3 coordination.db "SELECT state FROM coordination_status WHERE task_id = 'task-03'")

if [[ "$state" == "complete" ]] || [[ "$state" == "exited" ]]; then
    exit 0  # Allow exit
else
    echo "Cannot exit: Task not complete (state: $state)"
    exit 1  # Block exit
fi
```

**If state names were customizable:**
- Task uses 'finished' instead of 'complete'
- Hook checks for 'complete'
- Hook always blocks exit (never sees 'complete')
- **Result:** Session hangs forever, requires manual intervention

**Conclusion:** Fixed state names = reliable automation

### State Enumeration vs Free Text

**Current design:** TEXT column for states (not ENUM)

**Rationale:**
- SQLite doesn't have native ENUM type
- TEXT with CHECK constraint provides same safety:
  ```sql
  CREATE TABLE coordination_status (
      state TEXT NOT NULL CHECK (state IN (
          'working', 'waiting', 'needs_review', 'review_approved',
          'review_failed', 'error', 'fix_proposed', 'exited', 'complete'
      ))
  );
  ```
- Flexibility for future extensions (if absolutely needed, can add state with migration)
- Performance: Indexed TEXT is fast for <20 states

**Trade-off:**
- Lose: Compile-time type safety (could typo 'complet' instead of 'complete')
- Gain: SQLite compatibility, flexibility

**Mitigation:** Application-level validation + CHECK constraint catch typos

---

## State Naming Rationale and Design Principles

### Purpose

Explain WHY specific state names were chosen over alternatives.
Understanding rationale enables:
- Confident use of existing states
- Consistent patterns in new templates
- Evaluation of naming for different domains

### Design Principles

**1. Brevity vs Clarity**
- Prefer short names (easier in SQL, logs)
- But not at expense of clarity
- ✅ `error` (clear, 5 chars)
- ❌ `err` (too terse, unclear)
- ❌ `error_state` (redundant, all values are states)

**2. Present vs Past Tense**
- Active states: present tense or -ing
- Terminal states: past tense
- Transient states: past tense (action completed, awaiting acknowledgment)

**3. Semantic Precision**
- Each state name conveys specific meaning
- No overlap or ambiguity
- ✅ `needs_review` (clear: review not yet done)
- ❌ `reviewing` (ambiguous: who is reviewing?)

### State-by-State Rationale

#### Conductor States

**`watching`** (chosen)
- Meaning: Actively monitoring execution tasks
- Rejected alternatives:
  - `monitoring` - too formal, verbose
  - `observing` - too passive
  - `coordinating` - too vague (always coordinating)
- Rationale: -ing suffix indicates ongoing action, "watching" is natural English

**`reviewing`** (chosen)
- Meaning: Actively reviewing submitted work
- Rejected alternatives:
  - `evaluating` - too formal
  - `checking` - too informal
  - `review_in_progress` - verbose
- Rationale: Clear action state, matches common workflow term

**`complete`** (chosen)
- Meaning: All tasks finished, conductor done
- Rejected alternatives:
  - `completed` - past tense but "complete" works as adjective
  - `done` - too informal
  - `finished` - verbose
  - `success` - doesn't convey completion (could succeed at subtask)
- Rationale: Short, clear, universal

**`exit_requested`** (chosen)
- Meaning: Conductor requesting user intervention
- Rejected alternatives:
  - `escalated` - past tense, doesn't convey "request"
  - `needs_user` - ambiguous what user should do
  - `blocked` - too negative, doesn't convey choice
  - `manual_intervention` - too verbose
- Rationale: Clear intent (requesting exit), shows agency (conductor decision)

#### Execution States

**`working`** (chosen)
- Meaning: Actively executing steps
- Rejected alternatives:
  - `executing` - too formal
  - `in_progress` - verbose, less natural
  - `busy` - too informal
- Rationale: Natural English, -ing indicates ongoing

**`waiting`** (chosen)
- Meaning: Blocked, waiting for conductor message
- Rejected alternatives:
  - `blocked` - too negative
  - `paused` - implies can resume without external input
  - `awaiting_response` - verbose
- Rationale: Clear semantics, distinct from `needs_review` (different wait type)

**`needs_review`** (chosen)
- Meaning: Review explicitly requested (checkpoint)
- Rejected alternatives:
  - `review_requested` - past tense, verbose
  - `pending_review` - passive voice
  - `awaiting_review` - verbose
- Rationale: "needs" conveys requirement, commonly understood pattern

**`review_approved`** (chosen)
- Meaning: Conductor approved, execution must acknowledge
- Rejected alternatives:
  - `approved` - ambiguous (what was approved?)
  - `review_passed` - less formal
  - `accepted` - too generic
- Rationale: Parallel naming with `review_failed`, clear context

**`review_failed`** (chosen)
- Meaning: Quality needs improvement
- Rejected alternatives:
  - `rejected` - too harsh, implies unusable
  - `needs_revision` - verbose
  - `review_rejected` - redundant (review implies judgment)
- Rationale: Matches `review_approved`, conveys quality issue

**`error`** (chosen)
- Meaning: Recoverable technical error
- Rejected alternatives:
  - `error_detected` - verbose, event-focused
  - `failed` - too terminal (implies can't recover)
  - `errored` - grammatically awkward
  - `error_state` - redundant
- Rationale: Simple, universal, clear

**`fix_proposed`** (chosen)
- Meaning: Conductor proposed fix, execution must apply
- Rejected alternatives:
  - `error_fix` - ambiguous (is error fixed or being fixed?)
  - `fix_ready` - sounds complete (but needs application)
  - `proposed_fix` - works but less parallel to `review_approved`
- Rationale: Past tense + specific context, matches `review_approved` pattern

**`exited`** (chosen)
- Meaning: Terminal error, session terminated
- Rejected alternatives:
  - `terminated` - too harsh (implies force)
  - `failed_terminal` - verbose
  - `aborted` - suggests premature (but 5 retries exhausted)
  - `crashed` - too specific (could exit cleanly after retries)
- Rationale: Neutral term, past tense signals terminal

**`complete`** (chosen)
- Same rationale as conductor state
- Universal terminal state for both session types

### Consistency Patterns

**Parallel states use parallel naming:**
```
review_approved / review_failed (both: review_ prefix + past tense)
fix_proposed / error (fix proposal vs error state)
```

**Active vs Terminal:**
```
Active: -ing suffix (watching, reviewing, working)
Terminal: -ed suffix or complete (exited, complete)
Waiting: descriptive noun (needs_review, waiting)
```

**Length optimization:**
```
Prefer 1-2 words (easier in queries, logs)
Reject verbose alternatives (manual_intervention → exit_requested)
Balance brevity with clarity (error not err, complete not done)
```

### Why Consistency Matters

**Predictable querying:**
```sql
-- All active states use -ing suffix
WHERE state LIKE '%ing'

-- All terminal states can be filtered
WHERE state IN ('complete', 'exited', 'exit_requested')
```

**Clear communication:**
- Developer sees `watching` → knows conductor is active
- Developer sees `exited` → knows session terminated
- No ambiguity about state meaning

---

## Part 2: Coordination Mechanism

### State Management

#### Database Operations

**Execution session initialization:**
```sql
-- Set initial state
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';

-- Update migration_tasks
UPDATE migration_tasks
SET status = 'in_progress',
    worked_by = 'exec-03-20260204-1430',
    started_at = datetime('now')
WHERE task_id = 'task-03';

-- Log start
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'exec-03-20260204-1430', 'Started work on task-03');
```

**Execution session review request:**
```sql
-- Step 1: INSERT message FIRST (always before state update)
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'exec-03-20260204-1430',
'REVIEW REQUEST: Completed steps 1-3
Reason: Mid-point checkpoint per instruction
Changed files: [list]
Tests: 15/15 passing
Commits: abc123, def456
Ready for approval to proceed to step 4.');

-- Step 2: UPDATE state SECOND (signals conductor)
UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-03';
```

**Conductor approval:**
```sql
-- Update state to transient approved
UPDATE coordination_status SET state = 'review_approved' WHERE task_id = 'task-03';

-- Insert feedback message
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'task-00', 'Review approved. Code quality good, tests passing. Proceed with step 4.');
```

**Execution session processes approval:**
```sql
-- Check for messages
SELECT message FROM task_messages
WHERE task_id = 'task-03' AND from_session = 'task-00'
ORDER BY timestamp DESC LIMIT 1;

-- Update state back to working
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';

-- Continue with next steps
```

#### Dual-Table Rationale

**Why two tables (coordination_status + migration_tasks)?**

**Context efficiency:**
- coordination_status: 2 fields, ~10-15 tokens per query
- migration_tasks: 7 fields, ~50-70 tokens per query
- Subagent polls every 8s: 75% context reduction with dual-table

**Separation of concerns:**
- coordination_status: Real-time state machine (frequent updates)
- migration_tasks: Lifecycle events (infrequent updates)
- task_messages: Detailed context (queried on-demand only)

**Data integrity:**
- coordination_status: Transient, can be reconciled if corrupted
- migration_tasks: Source of truth, permanent record
- Inconsistency handling: Trust migration_tasks, reconcile coordination_status

### Session Isolation and Conflict Prevention

#### Problem

Multiple execution sessions could attempt to work on the same task simultaneously:
- Session A claims task-03, starts working
- Session B also tries to claim task-03 (race condition)
- Both sessions modify same files → merge conflicts
- Database state becomes inconsistent

#### Solution: Session ID Tracking with Atomic Claim

**Add session_id column to coordination_status:**
- Tracks which session currently owns each task
- Enables atomic claim (only one session can claim unclaimed task)
- Prevents multiple sessions working on same task

#### Session Lifecycle

**Phase 1: Session Initialization**

```python
import datetime

# Generate unique session ID
SESSION_ID = f"exec-03-{datetime.datetime.now().strftime('%Y%m%d-%H%M%S')}"

# Example: exec-03-20260205-143022
```

**Phase 2: Atomic Task Claim**

```sql
-- Attempt to claim task (atomic operation)
UPDATE coordination_status
SET state = 'working',
    session_id = 'exec-03-20260205-143022',
    started_at = CURRENT_TIMESTAMP
WHERE task_id = 'task-03'
  AND (session_id IS NULL OR session_id = 'exec-03-20260205-143022');

-- Check if claim succeeded
SELECT changes(); -- Returns 1 if update succeeded, 0 if task already claimed
```

**Why this works:**
- `WHERE session_id IS NULL` → task not claimed by anyone
- `OR session_id = SESSION_ID` → this session already owns the task (idempotent)
- If another session already claimed, WHERE clause fails → no update → changes() = 0

**Phase 3: Work Execution**

```python
# Verify claim succeeded before doing work
rows_updated = execute_sql("SELECT changes()")

if rows_updated == 0:
    # Another session claimed this task first
    log_error("Task already claimed by another session")
    exit(1)

# Claim succeeded, safe to proceed
log_info(f"Task claimed by session {SESSION_ID}")

# Do work...
```

**Phase 4: Session Release**

```sql
-- Release task on completion
UPDATE coordination_status
SET state = 'complete',
    session_id = NULL,  -- Release ownership
    completed_at = CURRENT_TIMESTAMP
WHERE task_id = 'task-03'
  AND session_id = 'exec-03-20260205-143022';

-- Release on error/exit
UPDATE coordination_status
SET state = 'exited',
    session_id = NULL,  -- Release ownership
    completed_at = CURRENT_TIMESTAMP
WHERE task_id = 'task-03'
  AND session_id = 'exec-03-20260205-143022';
```

#### Crash Recovery

**Problem:** Session crashes mid-work, never releases session_id

**Detection:**
- Heartbeat mechanism (Task 4) detects stale session
- Conductor identifies task stuck with old session_id

**Recovery:**
```sql
-- Conductor forcibly releases crashed session
UPDATE coordination_status
SET state = 'error',
    session_id = NULL,  -- Force release
    last_error = 'Previous session crashed, task reset for retry'
WHERE task_id = 'task-03'
  AND session_id = 'exec-03-20260205-143022'  -- Old session ID
  AND last_heartbeat < datetime('now', '-5 minutes');  -- Confirmed stale
```

#### Database Schema Addition

```sql
-- Add session_id column to coordination_status
ALTER TABLE coordination_status
ADD COLUMN session_id TEXT DEFAULT NULL;

-- Index for fast session ownership queries
CREATE INDEX idx_session_id ON coordination_status(session_id);

-- Add heartbeat column for crash detection
ALTER TABLE coordination_status
ADD COLUMN last_heartbeat TIMESTAMP DEFAULT CURRENT_TIMESTAMP;

-- Index for fast heartbeat timeout queries
CREATE INDEX idx_heartbeat ON coordination_status(last_heartbeat)
WHERE state = 'working';
```

#### Integration with Existing Patterns

**Execution task template updates:**
```python
# OLD (unsafe):
execute_sql("UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03'")

# NEW (safe):
SESSION_ID = f"exec-03-{datetime.datetime.now().strftime('%Y%m%d-%H%M%S')}"
result = execute_sql("""
    UPDATE coordination_status
    SET state = 'working', session_id = ?
    WHERE task_id = 'task-03'
      AND (session_id IS NULL OR session_id = ?)
""", (SESSION_ID, SESSION_ID))

if result.rowcount == 0:
    print("ERROR: Task already claimed by another session")
    exit(1)
```

**Conductor checks for session conflicts:**
```sql
-- Detect multiple sessions trying to work same task
SELECT task_id, session_id, state, last_heartbeat
FROM coordination_status
WHERE session_id IS NOT NULL
  AND state NOT IN ('complete', 'exited')
ORDER BY task_id;

-- Should see exactly 1 session per task
-- Multiple sessions on same task = conflict (shouldn't happen with atomic claim)
```

### Custom Hook Integration

#### Setup Workflow

**Conductor session:**
```bash
# User launches conductor session
# Session receives task instruction

# Step 1: Initialize database
UPDATE coordination_status SET state = 'watching' WHERE task_id = 'task-00';

# Step 2: Activate custom hook
bash tools/message-watcher/setup.sh --preset orchestration

# Output:
# 🔄 Message watcher activated
# Monitoring: task-00
# Exit criteria: ["exit_requested", "complete"]
# Max iterations: 1000
```

**Execution session:**
```bash
# User launches execution session
# Session receives task instruction

# Step 1: Initialize database
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';

# Step 2: Activate custom hook
bash tools/message-watcher/setup.sh --preset execution --task-id task-03

# Output:
# 🔄 Message watcher activated
# Monitoring: task-03
# Exit criteria: ["complete", "exited"]
# Max iterations: 500
```

#### Exit Criteria Logic

**Conductor exit criteria:**
1. **exit_requested** - Conductor needs to ask user questions or escalate errors
2. **complete** - All execution tasks finished successfully

**When conductor sets exit_requested:**
- Execution task encountered 5 failed retries (state=exited)
- Ambiguous situation requires user decision
- All execution tasks failed
- User manually requests conductor exit

**Execution exit criteria:**
1. **complete** - Task finished successfully, all verification passed
2. **exited** - Terminal error after 5 failed retries

**Hook behavior:**
```
Conductor tries to exit, state = 'watching'
  → Hook blocks: "Check task-00 status. Process messages or launch subagent."

Conductor tries to exit, state = 'exit_requested'
  → Hook allows: "✅ Task task-00 reached exit state: exit_requested"

Execution tries to exit, state = 'working'
  → Hook blocks: "Check task-03 status. Process messages or launch subagent."

Execution tries to exit, state = 'complete'
  → Hook allows: "✅ Task task-03 reached exit state: complete"
```

#### Max Iterations Safety

**Purpose:** Prevent infinite loops if state never changes

**Conductor:**
- Max iterations: 1000
- Hook tracks attempts in state file
- After 1000 blocks: Hook allows exit with warning
- Message: "⏱️ Max iterations (1000) reached for task task-00"

**Execution:**
- Max iterations: 500
- After 500 blocks: Hook allows exit with warning
- Session should set state=error or exited before this happens

**Best practice:**
- Sessions should update state appropriately
- Max iterations is safety net, not normal operation
- If max iterations hit: Investigate why state didn't change


## Exit Criteria Design Rationale

### Why Specific States Allow/Block Exit

**Exit criteria are safety mechanisms** - prevent premature session termination that would lose work or corrupt state.

#### Conductor Exit Criteria: ['complete', 'exit_requested']

**Allowed states:**
- ✅ `complete`: All tasks finished successfully, safe to exit
- ✅ `exit_requested`: Conductor explicitly requesting user intervention

**Blocked states** (all others):
- ❌ `watching`: Monitoring execution tasks, cannot abandon mid-coordination
- ❌ `reviewing`: Review in progress, must finish decision (approve/reject)
- ❌ `working`: [Should not occur for conductor, but if it does, block]

**Rationale:**
- Conductor coordinates multiple execution tasks
- Exiting mid-coordination orphans execution sessions
- Execution sessions wait indefinitely for conductor that no longer exists
- **Must complete coordination** (all tasks terminal) or **escalate to user** before exit

**Why 'error' NOT in conductor exit criteria:**
- Conductor doesn't enter 'error' state (only execution tasks do)
- Conductor handles errors by proposing fixes to execution
- If conductor encounters unrecoverable issue → uses 'exit_requested'

#### Execution Exit Criteria: ['complete', 'exited']

**Allowed states:**
- ✅ `complete`: Work finished successfully, safe to exit
- ✅ `exited`: Terminal error after 5 retries, cannot continue, safe to exit

**Blocked states** (all others):
- ❌ `working`: Active work in progress, exiting loses work
- ❌ `waiting`: Waiting for conductor message, must receive it first
- ❌ `needs_review`: Review requested, must wait for approval/rejection
- ❌ `review_approved`: Transient - execution MUST read approval and continue
- ❌ `review_failed`: Transient - execution MUST read feedback and revise
- ❌ `error`: Recoverable error, conductor proposing fix, must wait
- ❌ `fix_proposed`: Transient - execution MUST read fix and apply it

**Rationale:**
- Work in progress ('working', 'waiting') must not be abandoned
- Transient states require immediate handling to avoid state machine deadlock
- Only terminal states ('complete', 'exited') are safe exit points

**Why 'error' NOT in execution exit criteria:**
- 'error' is recoverable (retries 1-4 remaining)
- Execution must wait for conductor to propose fix
- If error is unrecoverable after 5 retries → transitions to 'exited' (which IS in exit criteria)

**Special case - 'exited' semantics:**
- 'exited' means "session is dead, cannot continue"
- This is post-discussion with conductor (5 retries exhausted)
- Safe to exit because all recovery attempts failed
- Conductor will handle task as failed when detecting ALL_TERMINAL

### Transient State Handling

**Why transient states BLOCK exit:**

```
Scenario without blocking:
1. Conductor sets state = 'review_approved'
2. Conductor writes approval message
3. Execution session in 'needs_review', hook allows exit because approved
4. ❌ Execution exits before reading approval message
5. ❌ Work is approved but never continued - deadlock

Scenario with blocking:
1. Conductor sets state = 'review_approved'
2. Conductor writes approval message
3. Execution session in 'needs_review', hook BLOCKS exit (not in criteria)
4. Execution subagent detects 'review_approved', returns "APPROVED"
5. Execution reads message, acknowledges, sets state = 'working'
6. ✅ Approval received and processed correctly
7. Now in 'working' (still blocked) or will complete soon
```

**60-second timeout for transient states:**
- Transient states should transition quickly (<10 seconds normally)
- 60 seconds = 6x expected duration
- If stuck >60s → likely crash or deadlock
- Conductor heartbeat detection (Task 4) identifies problem

### Decision Matrix

| State | Conductor Exit | Execution Exit | Reasoning |
|-------|------------------|----------------|-----------|
| `complete` | ✅ Allow | ✅ Allow | Work finished, safe to exit |
| `exit_requested` | ✅ Allow | ❌ Block* | Conductor escalating to user |
| `exited` | N/A | ✅ Allow | Terminal error, session dead |
| `watching` | ❌ Block | N/A | Monitoring active, must finish |
| `reviewing` | ❌ Block | N/A | Review in progress |
| `working` | ❌ Block | ❌ Block | Active work, premature exit loses work |
| `waiting` | N/A | ❌ Block | Waiting for conductor message |
| `needs_review` | N/A | ❌ Block | Review requested, must get response |
| `review_approved` | N/A | ❌ Block | Transient - must read and acknowledge |
| `review_failed` | N/A | ❌ Block | Transient - must read and revise |
| `error` | N/A | ❌ Block | Recoverable, waiting for fix proposal |
| `fix_proposed` | N/A | ❌ Block | Transient - must read fix and apply |

*Note: `exit_requested` should not appear for execution tasks (conductor-only state). If it does appear, block exit (likely configuration error).

### Default Behavior

**Any state not explicitly in exit criteria → BLOCKED**

**Reasoning:** Fail-safe design
- Better to incorrectly block exit (user notices, forces exit)
- Than to incorrectly allow exit (silent data loss, corrupted state)
- Unknown state = assume unsafe, block until investigated

### Subagent Patterns

#### Background Mode (During Work)

**Purpose:** Monitor for conductor messages while main session works on task steps

**Launch pattern:**
```python
# Execution session Step 1: Launch background subagent
subagent = Task(
    description="Monitor for messages",
    prompt="""
Watch coordination_status for task-03 and task_messages.
Check every 8 seconds. Max 75 iterations (10 minutes).

Exit when:
1. CONDUCTOR MESSAGE - New message from task-00
   → Return: "MESSAGE: [content]"
2. STATE CHANGE - Task state changed unexpectedly
   → Return: "STATE_CHANGE: [old_state] → [new_state]"

Otherwise continue watching.
""",
    subagent_type="general-purpose",
    run_in_background=true
)

# Main session continues to Step 2 immediately
# Subagent watches database in parallel
```

**Between-step checking pattern:**
```python
# After Step 2 completes, check subagent
result = TaskOutput(task_id=subagent.id, block=false, timeout=100)

if result.completed:
    # Subagent found something
    message = parse_subagent_result(result.output)
    handle_message(message)

    # Relaunch subagent for next phase
    subagent = Task(
        description="Monitor for messages",
        prompt="[same prompt]",
        run_in_background=true
    )
else:
    # Subagent still watching, continue to Step 3
    pass
```

<!-- ℹ️ DESIGN NOTE: Two Coordination Patterns for Different Use Cases

This template uses BACKGROUND SUBAGENTS with TaskOutput checks:
- Async/non-blocking monitoring
- Subagent polls database in isolated context
- Main session checks between steps
- Context efficient (subagent context discarded)
- USE FOR: Parallel orchestration (3+ independent tasks)

The existing task-03.md uses MANUAL SQL QUERIES:
- Sync/blocking check before each step
- Simple, straightforward
- Lower overhead
- USE FOR: Sequential execution (single task)

Both patterns are valid for different scenarios. This template documents the parallel orchestration pattern. The sequential pattern (task-03.md style) remains the default for simple, non-parallel work.

When to use which:
- PARALLEL PATTERN (this template): 3+ independent tasks running simultaneously
- SEQUENTIAL PATTERN (task-03.md): Single task or sequential dependencies
-->

**Default checking frequency:**
- After every numbered step in task instructions
- Override if step is very quick (<30 seconds)
- Always check before requesting review
- Always check after receiving approval

#### Blocking Mode (During Review Wait)

**Purpose:** Wait for conductor approval without adding context to main session

**Launch pattern:**
```python
# Execution session Step 4: Request review
UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (...);

# Launch blocking subagent
result = Task(
    description="Wait for review approval",
    prompt="""
Watch coordination_status for task-03 state changes.
Check every 8 seconds. Max 75 iterations (10 minutes).

Exit when:
1. APPROVED - State changed to 'review_approved'
   → Return: "APPROVED"
2. FAILED - State changed to 'review_failed'
   → Return: "FAILED"
3. FIX PROPOSED - State changed to 'fix_proposed'
   → Return: "FIX_PROPOSED"
4. MESSAGE - New message from task-00 (critical)
   → Return: "MESSAGE: [content]"

Note: Watch for state CHANGES, not specific states.
""",
    subagent_type="general-purpose",
    run_in_background=false  # BLOCKING
)

# Main session PAUSES here until subagent exits
# No context added during wait

# Subagent exits with result
if result == "APPROVED":
    # Read feedback, continue to next step
    UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';
elif result == "FAILED":
    # Read feedback, fix issues, re-request review
    UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';
```

**Timeout handling:**
```python
# If subagent reaches max iterations (10 minutes)
if subagent_timeout:
    # Check current state
    SELECT state FROM coordination_status WHERE task_id = 'task-03';

    # Likely conductor is delayed or crashed
    # Log error, wait longer or escalate
    INSERT INTO task_messages VALUES (
        'task-03', 'exec-session-id',
        'ERROR: Review timeout after 10 minutes. Conductor may need attention.'
    );

    # Set error state
    UPDATE coordination_status SET state = 'error' WHERE task_id = 'task-03';
```

#### Conductor Subagent

**Purpose:** Watch for execution session attention requests

**Launch pattern:**
```python
# Conductor main loop
while True:
    result = Task(
        description="Watch for attention needed",
        prompt="""
Watch coordination_status for execution tasks (task-03, task-04, task-05).
Check every 12 seconds. Max 600 iterations (2 hours).

Exit when:
1. REVIEW NEEDED - Any task state = 'needs_review'
   → Return: "REVIEW_NEEDED: [task-ids]"
2. ERROR DETECTED - Any task state = 'error'
   → Return: "ERROR_DETECTED: [task-ids]"
3. ALL COMPLETE - All tasks state = 'complete'
   → Return: "ALL_COMPLETE"
4. STALE STATE - Task in 'review_approved'/'review_failed'/'fix_proposed' for >60s
   → Return: "STALE_STATE: [task-id] in state [state] for [duration]s"
5. HEARTBEAT TIMEOUT - Task state = 'working' AND last_heartbeat >180s stale
   → Return: "HEARTBEAT_TIMEOUT: [task-id] (last seen [seconds]s ago, session: [session-id])"
6. CRITICAL EXIT - Any task state = 'exited'
   → Return: "ERROR_DETECTED: [task-id] (EXITED)"

Ignore states: 'working', 'complete', 'review_approved', 'review_failed', 'fix_proposed'
(These are normal/transient)
""",
        subagent_type="general-purpose",
        run_in_background=false  # BLOCKING
    )

    # Process result
    if "REVIEW_NEEDED" in result:
        handle_reviews(parse_task_ids(result))
    elif "ERROR_DETECTED" in result:
        handle_errors(parse_task_ids(result))
    elif "ALL_COMPLETE" in result:
        finalize_and_exit()
    elif "STALE_STATE" in result:
        investigate_crash(parse_details(result))
    elif "HEARTBEAT_TIMEOUT" in result:
        handle_crashed_session(parse_details(result))
```

**Why conductor ignores transient states:**
- `review_approved`, `review_failed`: Execution updates within 8s
- Including in query causes race condition:
  - Conductor sets review_approved
  - Re-enters loop before execution polls (12s > 8s)
  - Sees review_approved, exits again (false positive)
- Solution: Conductor ignores these states in main query
- Staleness detection catches crashed sessions (>60s in transient state)

## Subagent Launch Error Handling

### Problem

The Task() tool can fail during launch for multiple reasons:
- Network errors during agent spawn
- Quota exceeded (too many concurrent agents)
- Subagent launch timeout
- Task tool temporarily unavailable

Without error handling, session crashes or hangs indefinitely.

### Solution: Try-Catch Wrapper Pattern

**For blocking subagents (execution must wait for result):**

```python
# REQUIRED: Wrap Task() launch in try-catch
try:
    subagent = Task(
        subagent_type="general-purpose",
        prompt="Monitor coordination_status for task-03",
        description="Wait for review"
    )

    # Verify subagent launched successfully
    if not subagent or not hasattr(subagent, 'id'):
        raise Exception("Task launch failed - no subagent ID returned")

    subagent_id = subagent.id

    # Now safe to use subagent_id
    result = TaskOutput(task_id=subagent_id, block=True)

except Exception as e:
    # Launch failed - this is FATAL for blocking subagent
    # Execution cannot proceed without conductor communication

    log_error(f"FATAL: Blocking subagent launch failed: {e}")

    # Update coordination state to signal problem
    execute_sql("""
        UPDATE coordination_status
        SET state = 'error',
            last_error = 'Subagent launch failed - cannot communicate with conductor'
        WHERE task_id = 'task-03'
    """)

    # Exit execution - cannot continue without conductor communication
    exit(1)
```

**For background subagents (execution can continue):**

```python
# RECOMMENDED: Wrap Task() launch in try-catch
try:
    subagent = Task(
        subagent_type="general-purpose",
        prompt="Monitor for conductor messages",
        description="Background monitoring",
        run_in_background=True
    )

    if not subagent or not hasattr(subagent, 'id'):
        raise Exception("Task launch failed")

    subagent_id = subagent.id
    log_info(f"Background subagent launched: {subagent_id}")

except Exception as e:
    # Launch failed - this is NOT FATAL for background subagent
    # Execution can continue, but loses background monitoring

    log_warning(f"Background subagent launch failed: {e}")
    log_warning("Continuing without background monitoring - will check coordination_status manually")

    # Set flag to indicate no background monitoring
    has_background_monitor = False

    # Continue execution normally
    # Periodically check coordination_status manually between steps
```

### When to Use Each Pattern

| Subagent Type | Launch Failure Handling | Rationale |
|---------------|------------------------|-----------|
| **Blocking (review wait, fix wait)** | FATAL - exit execution | Cannot proceed without conductor decision |
| **Background (monitoring)** | WARNING - continue without | Can check coordination_status manually |

### Implementation Checklist

**Execution task instructions must include:**
- [ ] Try-catch wrapper around ALL Task() calls
- [ ] Verification that subagent.id exists and is valid
- [ ] Error handling appropriate to blocking vs background type
- [ ] Logging of launch failures for debugging
- [ ] Fallback behavior for background subagent failures

**Conductor task instructions must include:**
- [ ] Try-catch wrapper around Task() launches for execution tasks
- [ ] Verification that all execution subagents launched successfully
- [ ] Handling for partial launch failure (some tasks launched, others failed)

### Stale Session Recovery

**The template uses two safety mechanisms with different purposes:**
1. **Staleness detection (60s)**: Conductor detects crashed execution sessions
2. **Hook max iterations**: Safety net to prevent infinite loops (rare edge case)

#### Staleness Detection Scenario

**Problem:** Execution session crashes while in `review_approved` state
- Conductor approved review → set state to `review_approved`
- Execution session crashed before polling database
- Execution never updates state to `working`
- State stuck in transient `review_approved` for >60 seconds

#### Detection by Conductor

```python
# Conductor's monitoring subagent includes staleness check:
SELECT task_id, state,
       (julianday('now') - julianday(
           (SELECT timestamp FROM task_messages
            WHERE task_id = coordination_status.task_id
            ORDER BY timestamp DESC LIMIT 1)
       )) * 86400 as seconds_since_last_message
FROM coordination_status
WHERE state IN ('review_approved', 'review_failed')
  AND seconds_since_last_message > 60;

# Subagent exits with result:
# "STALE_STATE: task-03 in state 'review_approved' for 125s"
# (Normal: <8s for execution to poll and update)
```

#### Recovery Steps (Conductor Executes)

```sql
-- Conductor detected staleness, must decide action

-- Option A: Mark as ERROR (execution can potentially restart)
UPDATE coordination_status SET state = 'error' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'task-00',
    'ERROR: Session appears crashed. State was review_approved for 125s (expected <8s).

Likely cause: Execution session terminated unexpectedly
Action required: Restart execution session or escalate to user

Session can resume from last git commit. Review approval is still valid.'
);
-- Conductor continues monitoring
-- User can restart execution session to resume work

-- Option B: Mark as EXITED (terminal, needs user intervention)
UPDATE coordination_status SET state = 'exited' WHERE task_id = 'task-03';
UPDATE migration_tasks SET status = 'failed' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'task-00',
    'CRITICAL: Session crashed in review_approved state for 125s. Marking as terminal failure.

User intervention required to assess work and decide next steps.
Review approval was sent but execution did not process it before crash.'
);
-- Conductor sets exit_requested for user review
```

#### Decision Criteria

**When to use Option A (error) vs Option B (exited):**

| Scenario | State | Staleness Duration | Action | Rationale |
|----------|-------|-------------------|--------|-----------|
| Stale in `review_approved` | | 60-120s | ERROR | Execution may have paused, give chance to resume |
| Stale in `review_approved` | | >120s | EXITED | Likely crashed, unlikely to recover |
| Stale in `review_failed` | | 60-300s | WAIT | Execution may be slow applying complex fix |
| Stale in `review_failed` | | >300s | ERROR | Fix taking too long, needs attention |
| ANY transient state | | >600s | EXITED | Definitely crashed or stuck |

**Conductor logic:**
```python
if staleness_duration > 600:  # 10 minutes
    mark_exited()  # Definite failure
elif state == 'review_approved' and staleness_duration > 120:
    mark_exited()  # Execution should have responded by now
elif state == 'review_failed' and staleness_duration > 300:
    mark_error()  # Fix taking very long
else:
    wait_and_check_again()  # Might be slow, give more time
```

#### Hook Max Iterations Interaction

**Two mechanisms, different purposes:**

```markdown
**Staleness detection (60s+):**
- Purpose: Detect crashed/stuck execution sessions
- Trigger: Transient state >60s without database updates
- Response: Conductor takes action (mark error/exited)
- Timeline: Minutes

**Hook max iterations (500/1000):**
- Purpose: Safety net if state NEVER changes
- Trigger: Hook blocks exit 500/1000 times
- Response: Hook allows exit with warning
- Timeline: Hours (depending on polling frequency)

**Which fires first:**
- Staleness detection: 60-600 seconds
- Hook max iterations: 500 × ~8s polling = 4000s = 66 minutes

Staleness detection handles crashes MUCH faster than max iterations.

**If both mechanisms could apply:**
- Staleness fires first (minutes vs hours)
- Conductor marks session as error/exited
- Hook sees new state, allows exit
- Max iterations never reached
```

#### Manual Recovery After Stale Session Detected

```bash
# User discovers execution session crashed (conductor reported staleness)

# Option 1: Restart execution session for same task
# Launch new session with same task-XX instruction
# Session resumes from last commit
# Conductor is still monitoring - will coordinate as normal

# Option 2: Mark complete manually and move on
# If work was mostly done before crash:
sqlite3 coordination.db "UPDATE coordination_status SET state='complete' WHERE task_id='task-03'"
sqlite3 coordination.db "UPDATE migration_tasks SET status='complete' WHERE task_id='task-03'"
# Conductor will see task-03 complete, continue with other tasks

# Option 3: Investigate and escalate
# Review what work was completed (git log)
# Decide if task needs full restart or partial completion acceptable
```

### Message Passing

#### Message Format Standards

**Review request:**
```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'exec-03-20260204-1430',
'REVIEW REQUEST: Completed steps 1-3
Reason: Mid-point checkpoint per instruction
Changed files: src/auth.ts, src/api.ts, tests/auth.test.ts
Tests: 15/15 passing (output: test-results-20260204-1430.txt)
Commits: abc123 (auth flow), def456 (tests)
Deviations: None
Reports: docs/implementation/reports/task-03-checkpoint-1.md
Review focus: New authentication flow (src/auth.ts lines 45-120), test coverage');
```

**Conductor approval:**
```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'task-00',
'REVIEW APPROVED: Code quality good, tests passing, architecture sound.
Proceed to step 4. No changes needed.');
```

**Conductor rejection:**
```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'task-00',
'REVIEW FAILED (Retry 1/3):
Issues found:
1. Test test_auth_token_expiry is flaky (failed 2/5 runs)
2. Missing error handling in src/auth.ts line 67 (null pointer risk)
3. Commit message abc123 too vague: "fix auth" should explain what was fixed
Actions required:
- Fix flaky test (likely timing issue)
- Add null check at auth.ts:67
- Amend commit message or add clarifying commit
Re-submit when fixed.');
```

**Error report:**
```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'exec-03-20260204-1430',
'ERROR (Retry 1/5): Test failed after implementation
Error: test_auth_integration failed with timeout
Report: docs/implementation/reports/task-03-error-retry-1.md
Awaiting conductor fix proposal');
```

**Conductor fix proposal:**
```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'task-00',
'FIX PROPOSAL (Retry 1/5):
Root cause: Auth service timeout set to 5s, integration test expects 3s
Fix: Update auth.ts:45 timeout to 3000ms to match test expectations
Retry: Re-run tests after fix');
```

#### Message Querying

**Execution session checks for messages:**
```sql
-- Before each step, check for conductor messages
SELECT message, timestamp FROM task_messages
WHERE task_id = 'task-03' AND from_session = 'task-00'
ORDER BY timestamp DESC LIMIT 3;

-- If messages found:
-- 1. Read and process
-- 2. Update state if needed
-- 3. Continue execution
```

**Conductor reads review request:**
```sql
-- When subagent reports REVIEW_NEEDED for task-03
SELECT message, timestamp FROM task_messages
WHERE task_id = 'task-03'
  AND from_session LIKE 'exec-%'
  AND message LIKE '%REVIEW REQUEST%'
ORDER BY timestamp DESC LIMIT 1;

-- Parse message for:
-- - Changed files
-- - Test results
-- - Deviations
-- - Review focus areas
```

<!-- 📋 SESSION ID NAMING STANDARD

For consistency across parallel orchestration tasks, use these session ID formats:

**Conductor sessions:**
- Pattern: `task-00` (matches task_id for conductor)
- Reason: Clear, consistent with execution task naming
- Migration note: Existing sequential tasks may use 'O' (acceptable for those contexts)

**Execution sessions:**
- Pattern: `exec-{task-num}-{timestamp}` or `exec-{task-id}`
  - Example: `exec-03-20260204-1430` (with timestamp)
  - Example: `exec-03` (simple form)
- Reason: Clearly identifies which task + session instance
- Use timestamp form if multiple attempts on same task

**In SQL queries:**
```sql
-- Conductor messages
WHERE from_session = 'task-00'

-- Execution messages (specific)
WHERE from_session LIKE 'exec-03%'

-- Execution messages (any execution session)
WHERE from_session LIKE 'exec-%'
```

Note: The existing sequential pattern (task-03.md) uses 'O' for conductor, which is fine for that simpler context. This naming standard applies specifically to parallel orchestration scenarios.
-->

---

## Part 3: Instruction Structure

### Conductor Instructions

#### Template Structure

```markdown
# Task 00: Conductor - [Project Name]

**Dependencies:** [List prerequisite tasks]
**Execution tasks:** task-01, task-02, task-03, ... task-NN
**Max concurrent:** [Number of parallel tasks]
**Estimated duration:** [Time estimate]

---

## Objective

[Clear statement of what conductor coordinates]

**Critical responsibilities:**
1. Monitor execution task states
2. Review work at checkpoints
3. Provide feedback and approvals
4. Handle errors autonomously (5 retries)
5. Coordinate final completion

---

## Prerequisites

[Database checks, directory verification, etc.]

---

## Initialization

### Step 1: Set Up Hook

```bash
bash tools/message-watcher/setup.sh --preset orchestration
```

### Step 2: Initialize Database

```sql
UPDATE coordination_status SET state = 'watching' WHERE task_id = 'task-00';
UPDATE migration_tasks
SET status = 'in_progress', worked_by = '[session-id]', started_at = datetime('now')
WHERE task_id = 'task-00';
```

### Step 3: Initialize Execution Tasks

[Create records for task-01, 02, 03, etc.]

---

## Main Loop

### Step 4: Launch Monitoring Subagent

[Subagent prompt with exit conditions]

## ALL_TERMINAL Handling Clarification

### Problem

Original template: "Ignore complete tasks" but "detect ALL_COMPLETE"

**Ambiguity:** What does "ignore" mean?
- Don't query them at all? (Then how detect all complete?)
- Query but don't report? (Then what's "ignore"?)

### Clarified Behavior

**"Ignore" means:** Query them for terminal state detection, don't report as actionable items

### Conductor Subagent Exit Conditions (UPDATED)

```markdown
Exit when:

1. **REVIEW_NEEDED** - Any task state = 'needs_review'
   ```sql
   SELECT task_id FROM coordination_status
   WHERE task_id IN ('task-03', 'task-04', 'task-05')
     AND state = 'needs_review'
   LIMIT 1;
   ```
   Return: "REVIEW_NEEDED: task-03"

2. **ERROR_DETECTED** - Any task state = 'error'
   ```sql
   SELECT task_id FROM coordination_status
   WHERE task_id IN ('task-03', 'task-04', 'task-05')
     AND state = 'error'
   LIMIT 1;
   ```
   Return: "ERROR_DETECTED: task-04 (retry 3/5)"

3. **STALE_STATE** - Transient state >60s
   ```sql
   SELECT task_id, state,
          CAST((julianday('now') - julianday(updated_at)) * 86400 AS INTEGER) as stale_seconds
   FROM coordination_status
   WHERE task_id IN ('task-03', 'task-04', 'task-05')
     AND state IN ('review_approved', 'review_failed', 'fix_proposed')
     AND updated_at < datetime('now', '-60 seconds');
   ```
   Return: "STALE_STATE: task-03 in review_approved for 67s"

4. **HEARTBEAT_TIMEOUT** - state = 'working' AND heartbeat stale >180s
   ```sql
   SELECT task_id,
          CAST((julianday('now') - julianday(last_heartbeat)) * 86400 AS INTEGER) as seconds_stale
   FROM coordination_status
   WHERE task_id IN ('task-03', 'task-04', 'task-05')
     AND state = 'working'
     AND last_heartbeat < datetime('now', '-3 minutes');
   ```
   Return: "HEARTBEAT_TIMEOUT: task-05 (last seen 185s ago)"

5. **ALL_TERMINAL** - All tasks in terminal states ('complete' OR 'exited')
   ```sql
   SELECT COUNT(*) as terminal_count
   FROM coordination_status
   WHERE task_id IN ('task-03', 'task-04', 'task-05')
     AND state IN ('complete', 'exited');

   -- If terminal_count = 3: Return "ALL_TERMINAL"
   ```
   **CRITICAL:** Conductor MUST query individual states after receiving ALL_TERMINAL
```

### ALL_TERMINAL Processing (CRITICAL CONDUCTOR LOGIC)

**Step 1: Subagent detects all terminal**
```python
# Subagent query
result = execute_sql("""
    SELECT COUNT(*) as terminal_count
    FROM coordination_status
    WHERE task_id IN ('task-03', 'task-04', 'task-05')
      AND state IN ('complete', 'exited')
""")

if result['terminal_count'] == 3:
    return "ALL_TERMINAL"  # Signal to conductor
```

**Step 2: Conductor MUST query individual states**
```python
# Subagent returned "ALL_TERMINAL"
# Conductor CANNOT assume outcome - must check which terminal state

states = execute_sql("""
    SELECT task_id, state
    FROM coordination_status
    WHERE task_id IN ('task-03', 'task-04', 'task-05')
    ORDER BY task_id
""")

# Results example:
# [
#   {'task_id': 'task-03', 'state': 'complete'},
#   {'task_id': 'task-04', 'state': 'complete'},
#   {'task_id': 'task-05', 'state': 'exited'}
# ]
```

**Step 3: Conductor decides action based on state mix**
```python
complete_count = sum(1 for s in states if s['state'] == 'complete')
exited_count = sum(1 for s in states if s['state'] == 'exited')

if exited_count == 0:
    # 100% success - all tasks completed
    execute_sql("UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-00'")
    log_success("All tasks completed successfully")
    exit(0)

else:
    # Partial or total failure - some/all tasks exited
    execute_sql("UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-00'")

    # Report to user which tasks failed
    failed_tasks = [s['task_id'] for s in states if s['state'] == 'exited']
    log_message(f"Tasks failed after retries: {failed_tasks}")
    log_message(f"Tasks succeeded: {complete_count}/{len(states)}")

    exit(0)  # Exit for user review
```

### Examples

**Example 1: All Complete (100% success)**
```
Subagent: "ALL_TERMINAL"

Conductor query:
  task-03: complete
  task-04: complete
  task-05: complete

Decision: conductor state = 'complete' (success)
```

**Example 2: Mixed Terminal States (partial failure)**
```
Subagent: "ALL_TERMINAL"

Conductor query:
  task-03: complete
  task-04: exited
  task-05: complete

Decision: conductor state = 'exit_requested' (2/3 succeeded, user review needed)
```

**Example 3: All Exited (total failure)**
```
Subagent: "ALL_TERMINAL"

Conductor query:
  task-03: exited
  task-04: exited
  task-05: exited

Decision: conductor state = 'exit_requested' (0/3 succeeded, escalate)
```

### Why This Matters

**Problem without explicit query:**
- Subagent returns "ALL_TERMINAL"
- Conductor assumes success
- Sets conductor state = 'complete'
- User thinks all tasks succeeded
- **Reality:** Some tasks exited (failed after retries)
- **Result:** Silent failure, user doesn't know to investigate

**Solution with explicit query:**
- Subagent returns "ALL_TERMINAL"
- Conductor queries individual states
- Sees mix of 'complete' and 'exited'
- Sets conductor state = 'exit_requested'
- Reports: "2/3 tasks succeeded, 1 failed (task-04)"
- **Result:** User knows to investigate task-04

### Step 5: Process Subagent Results

**When REVIEW_NEEDED:**
1. Read review requests from task_messages
2. Review changed files/tests/reports
3. Apply review criteria
4. Send approval or feedback

**When ERROR_DETECTED:**
1. Read error reports
2. Analyze root cause
3. Propose fix in message
4. Track retry count

**When ALL_COMPLETE:**
1. Verify all tasks complete
2. Review completion reports
3. Perform final verification
4. Update conductor state to complete

---

## Review Criteria

[Specific criteria for approving/rejecting work]

---

## Error Recovery

[5-retry flow with examples]

---

## Completion

### Step N: Final Verification

[What to check]

### Step N+1: Update State

```sql
UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-00';
UPDATE migration_tasks
SET status = 'complete', completed_at = datetime('now'), report_path = '[path]'
WHERE task_id = 'task-00';
```

### Step N+2: Generate Report

[Report requirements]

---

## Success Criteria

- [ ] All execution tasks complete
- [ ] All reviews approved
- [ ] No errors remaining
- [ ] Completion report generated
```

#### Key Elements

**Subagent prompt design:**
- Exit conditions must be SPECIFIC and COMPLETE
- Include ALL states to monitor (needs_review, error, complete, exited)
- Include staleness detection (>60s in transient state)
- Include timeout (max iterations)
- Return structured messages for parsing

**Review workflow:**
- Define review criteria explicitly
- Provide examples of approval and rejection messages
- Include retry count tracking
- Document what to check for each review type

**Error handling:**
- Read error report from execution session
- Analyze root cause (not just symptoms)
- Propose concrete fix (not vague suggestion)
- Track retry count (1-5)
- Escalate after retry 5

### Execution Instructions

#### Template Structure

```markdown
# Task XX: [Task Name]

**Parallel-safe:** Yes/No (with tasks [list])
**Dependencies:** [Prerequisites]
**Estimated duration:** [Time]
**Review checkpoints:** [Number and locations]

---

## Objective

[Clear statement of what this task accomplishes]

**Critical success criteria:**
1. [Criterion 1]
2. [Criterion 2]
...

---

## Prerequisites

[Database checks, file verification]

---

## Initialization

### Step 1: Set Up Hook

```bash
bash tools/message-watcher/setup.sh --preset execution --task-id task-XX
```

### Step 2: Initialize Database

```sql
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-XX';
UPDATE migration_tasks
SET status = 'in_progress', worked_by = '[session-id]', started_at = datetime('now')
WHERE task_id = 'task-XX';
```

### Step 3: Launch Background Subagent

[Subagent prompt for message monitoring]

---

## Work Execution

### Step 4: [First work phase]

[Detailed instructions]

**Between-step check:**
```python
# Check if subagent found messages
result = TaskOutput(task_id=subagent_id, block=false, timeout=100)
if result.completed:
    process_message(result.output)
    relaunch_subagent()
```

### Step 5: [Second work phase]

[More work]

**Between-step check:**
[Same pattern]

---

## Heartbeat Updates (Crash Detection)

**Purpose:** Enable conductor to detect session crashes in 'working' state

**Pattern:** Update heartbeat between execution steps

```python
# At start of execution session
SESSION_ID = f"exec-03-{datetime.datetime.now().strftime('%Y%m%d-%H%M%S')}"

# Before each major step
def update_heartbeat():
    execute_sql("""
        UPDATE coordination_status
        SET last_heartbeat = CURRENT_TIMESTAMP
        WHERE task_id = 'task-03'
          AND session_id = ?
    """, (SESSION_ID,))

# Execution loop
for step in steps:
    update_heartbeat()  # Signal: still alive

    execute_step(step)

    update_heartbeat()  # Signal: step complete

# Total heartbeat updates: ~10-20 over full execution
# Context cost: ~50 tokens per update = 500-1000 tokens total
# Benefit: 3-minute crash detection vs 2-hour timeout
```

**When to update heartbeat:**
- Before starting each step
- After completing each step
- Before long operations (>1 minute)
- After long operations complete

**Frequency:** Every 30-120 seconds is sufficient

---

## Review Checkpoint

### Step N: Pause for Review

**Update state:**
```sql
INSERT INTO task_messages VALUES ('task-XX', '[session-id]',
'REVIEW REQUEST: Completed steps 4-5
[Details about work completed]');

UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-XX';
```

**Launch blocking subagent:**
[Wait for approval]

**Process result:**
- If APPROVED: Continue to next phase
- If FAILED: Fix issues, re-request review

---

## Completion

### Step M: Final Verification

[Verification steps]

### Step M+1: Update State

```sql
UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-XX';
UPDATE migration_tasks
SET status = 'complete', completed_at = datetime('now'), report_path = '[path]'
WHERE task_id = 'task-XX';
```

### Step M+2: Generate Report

[Report requirements]

---

## Success Criteria

- [ ] All work complete
- [ ] Reviews approved
- [ ] Verification passed
- [ ] Report generated

#### Key Elements

**Between-step checking:**
- Check after EVERY numbered step (default)
- Can be overridden for very quick steps (<30s)
- Always check before requesting review
- Always check after receiving approval
- Use non-blocking TaskOutput

**Subagent management:**
- Launch background subagent at initialization
- Check between steps
- Relaunch after processing messages
- Switch to blocking mode for review waits
- Always terminate subagent before final exit

**State transitions:**
- Document ALL state updates
- Always message BEFORE state update (dual-table pattern)
- Include retry count in error messages
- Use transient states correctly (approved/failed)

### Review Checkpoints

#### When to Include Checkpoints

**Include review checkpoints when:**
- Major phase completion (e.g., after analysis, before implementation)
- Before creating files in final destination (docs/)
- After significant changes to existing code
- When decision required (e.g., which patterns to extract)
- Every 3-5 major steps (general guideline)

**Do NOT checkpoint for:**
- Trivial operations (e.g., create directory)
- Internal work that can be fully automated
- Steps that are easily reversible

#### Checkpoint Design

**Pre-checkpoint requirements:**
```markdown
### Step N: Pre-Review Preparation

**Before requesting review:**
1. Commit all changes
   ```bash
   git add [changed files]
   git commit -m "task-XX: [description]"
   ```
2. Run verification tests
   ```bash
   [test commands]
   ```
3. Create checkpoint report (if required)
   ```bash
   # Generate docs/implementation/reports/task-XX-checkpoint-1.md
   ```
4. List changed files for review message
   ```bash
   git diff [last-reviewed-commit]..HEAD --name-only
   ```
```

**Review request:**
```markdown
### Step N+1: Request Review

**Update database:**
```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-XX', '[session-id]',
'REVIEW REQUEST: Completed steps X-Y
Reason: [Why review needed at this point]
Changed files: [list]
Tests: [passing count]
Commits: [SHAs with descriptions]
Deviations: [None or list with explanations]
Reports: [path to checkpoint report if any]
Review focus: [What conductor should focus on]');

UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-XX';
```

**Launch blocking subagent:**
[Prompt to wait for approval/rejection]
```

**Post-review actions:**
```markdown
### Step N+2: Process Review Result

**Read conductor feedback:**
```sql
SELECT message FROM task_messages
WHERE task_id = 'task-XX' AND from_session = 'task-00'
ORDER BY timestamp DESC LIMIT 1;
```

**If APPROVED:**
1. Read feedback for any notes
2. Update state back to working
   ```sql
   UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-XX';
   ```
3. Continue to next phase (Step N+3)

**If FAILED:**
1. Read rejection feedback in detail
2. Parse issues and required actions
3. Update state back to working
   ```sql
   UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-XX';
   ```
4. Fix issues (document each fix)
5. Increment retry count (track in notes)
6. Return to Step N (re-request review)

**If TIMEOUT (subagent max iterations):**
1. Check conductor status
   ```sql
   SELECT state FROM coordination_status WHERE task_id = 'task-00';
   ```
2. Log timeout error
   ```sql
   INSERT INTO task_messages VALUES (
       'task-XX', '[session-id]',
       'ERROR: Review timeout after 10 minutes. Conductor may need attention.'
   );
   UPDATE coordination_status SET state = 'error' WHERE task_id = 'task-XX';
   ```
3. Wait for conductor response
```

#### Smoothness Indicators

#### Using Smoothness for Review Decisions

**Smoothness → Review Depth:**
- 0-2 (Green): Light review (spot check 2-3 files)
- 3-5 (Yellow): Medium review (check ~50% of work)
- 6-7 (Orange): Deep review (check all work carefully)
- 8-9 (Red): Full review + error analysis

**Smoothness → Review Outcome:**

| Smoothness | Typical Outcome | Action | Reasoning |
|------------|----------------|--------|-----------|
| **0-2** | ✅ Approve immediately | Proceed to next phase | Work is solid, following plan perfectly |
| **3-5** | ✅ Approve with feedback | Proceed with notes for future | Minor issues, but good enough to continue |
| **6-7** | ⚠️ Conditional | Depends on issue type | Significant problems - assess if fixable |
| **8-9** | ❌ Reject or Escalate | Request fixes or user decision | Critical issues - cannot proceed as-is |

**Conditional outcomes (smoothness 6-7):**

```markdown
**If issues are addressable by execution:**
- → ❌ Reject with specific fix requests
- → Set review_failed, provide detailed feedback
- → Example: "Code works but has performance issues - optimize loops at lines X, Y, Z"

**If issues are informational/contextual:**
- → ✅ Approve with notes
- → Document concerns for future reference
- → Example: "Extraction good, but found edge case not in original scope - note for future task"
```

**Smoothness 8-9 handling:**
```markdown
**Not a technical error (tests pass), but execution needs guidance:**
- → ❌ Reject with conductor guidance
- → Example: "Fundamental design issue - current approach creates circular dependencies"

**Not an error, but needs user decision:**
- → Set exit_requested (escalate to user)
- → Example: "Found conflicting patterns, unclear which is authoritative"
```

#### Relationship to Error State

**Key distinction:**

| Aspect | Smoothness Score | Error State |
|--------|-----------------|-------------|
| **Purpose** | Report difficulty at review checkpoints | Signal technical failure |
| **Context** | Review requests | Error recovery |
| **Indicates** | How smooth execution was | Whether tests/code work |
| **Range** | 0-9 scale (subjective assessment) | error or exited (binary) |

**Examples showing the difference:**

```sql
-- HIGH smoothness, NO error (just complex work)
INSERT INTO task_messages VALUES (
    'task-03', 'exec-03',
    'REVIEW REQUEST: Completed steps 1-5
    Smoothness: 7 (Major deviation - found 3x more patterns than expected, required extensive analysis)
    Tests: 15/15 passing ✅
    Changed files: [list]
    Work quality: Good, just more complex than planned'
);
-- Conductor: Deep review due to smoothness 7, but work quality is fine

-- LOW smoothness, YES error (technical failure)
INSERT INTO task_messages VALUES (
    'task-04', 'exec-04',
    'ERROR (Retry 1/5): Tests failing after extraction
    Smoothness: 2 (Work was straightforward until tests failed)
    Tests: 3/15 passing ❌
    Error: Integration tests timeout
    Report: docs/implementation/reports/task-04-error-retry-1.md'
);
-- Conductor: Autonomous error recovery despite low smoothness

-- HIGH smoothness, YES error (complex AND broken)
INSERT INTO task_messages VALUES (
    'task-05', 'exec-05',
    'ERROR (Retry 2/5): Tests still failing after first fix attempt
    Smoothness: 8 (Complex patterns + test failures)
    Tests: 5/20 passing ❌
    Error: Database connection failures in 15 tests
    Previous fix: Added connection pooling (didn't work)
    Report: docs/implementation/reports/task-05-error-retry-2.md'
);
-- Conductor: Error recovery + deep analysis due to high smoothness
```

**When to include smoothness in error reports:**

✅ **DO include smoothness:**
- Helps conductor understand context of error
- Smoothness 8-9 + error = very complex situation, may need user escalation sooner
- Smoothness 0-2 + error = likely simple fix, retry should work

❌ **Don't conflate smoothness with error:**
- Don't set error state just because smoothness is high
- Don't assume low smoothness means no problems
- They measure different things

**Purpose:** Help conductor determine review depth

**Scoring (0-9 scale):**
- **0-2 (Green)** ✅ - Following plan perfectly, no issues
  - No deviations from plan
  - All tests passing first try
  - Commits are clear and incremental
  - Timeline reasonable
  - → Conductor: Light review (spot check 2-3 files)

- **3-5 (Yellow)** ⚠️ - Minor detours, still on track
  - Small deviation from plan (documented with reason)
  - Tests failed once but fixed
  - Some vague commit messages
  - Timeline slightly delayed
  - → Conductor: Medium review (check ~50% of work)

- **6-7 (Orange)** 🔶 - Significant issues, major decisions made
  - Major deviation from plan
  - Multiple test failures
  - Significant debugging required
  - Timeline substantially delayed
  - → Conductor: Deep review (check all work carefully)

- **8-9 (Red)** 🚨 - Critical errors, conductor intervention needed
  - Cannot proceed without help
  - Fundamental misunderstanding
  - Tests completely broken
  - → Conductor: Full review + error analysis

**When to update smoothness score:**
- At each review checkpoint
- When deviation from plan occurs
- When error encountered
- At task completion

<!-- 🚩 FLAG 15: ADD - Non-Error Escalation Decision Framework

TODO: Add to Part 4 (Error & Edge Cases) - "Non-Error User Escalation":

The 5-retry flow handles technical errors. But some situations need user decisions without being "errors":

**Escalation Decision Matrix:**

| Situation Type | Action | State | Example |
|----------------|--------|-------|---------|
| **Technical error** | Autonomous retry (1-4) | `error` | Tests fail, code doesn't compile |
| **Terminal error** | Set exited, escalate | `exited` | 5 retries exhausted |
| **Quality issue** | Review rejection | `review_failed` | Code works but needs improvement |
| **Ambiguous requirement** | Escalate to user | `exit_requested` | "Extract current patterns" - unclear which are current |
| **Conflicting information** | Escalate to user | `exit_requested` | Pattern A says X, Pattern B says Y |
| **Design decision** | Escalate to user | `exit_requested` | Multiple valid approaches, unclear which to use |

**When execution should set `exit_requested`:**
```sql
-- Execution discovers ambiguous requirement mid-task
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'exec-03',
    'ESCALATION: Found 50 patterns, unclear which are current. User decision needed: Should I extract all 50, or only recent ones? Source conflicts: TESTING_GUIDE.md has 2022 patterns, TESTING_QUICK_REFERENCE.md has 2025 patterns.'
);
-- Session will exit (hook allows exit_requested)
-- User reviews situation, clarifies requirement, relaunches with updated instruction
```

**When conductor should set `exit_requested`:**
- Any execution task hits terminal error (exited)
- Multiple execution tasks fail
- Execution asks for user decision (forwards request)
- Conductor uncertain how to proceed

**Key principle:** If autonomous recovery is impossible (not a technical error to fix), escalate to user immediately. Don't waste retries on non-technical decisions.
-->

**Example:**
```sql
-- Smooth execution, no issues
UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-XX';
INSERT INTO task_messages VALUES (
    'task-XX', '[session-id]',
    'REVIEW REQUEST: Completed steps 1-3
    Smoothness: 0 (Following plan perfectly, no deviations)
    [rest of message]'
);

-- Encountered issue during execution
UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-XX';
INSERT INTO task_messages VALUES (
    'task-XX', '[session-id]',
    'REVIEW REQUEST: Completed steps 1-3
    Smoothness: 5 (Found duplicate pattern, had to cross-check with RAG query, added 30min)
    [rest of message]'
);
```

---

## Part 4: Error & Edge Cases

### Non-Error User Escalation

**The 5-retry flow handles technical errors. But some situations need user decisions without being "errors".**

#### Escalation Decision Matrix

| Situation Type | Action | State | Example |
|----------------|--------|-------|---------|
| **Technical error** | Autonomous retry (1-4) | `error` | Tests fail, code doesn't compile |
| **Terminal error** | Set exited, escalate | `exited` | 5 retries exhausted |
| **Quality issue** | Review rejection | `review_failed` | Code works but needs improvement |
| **Ambiguous requirement** | Escalate to user | `exit_requested` | "Extract current patterns" - unclear which are current |
| **Conflicting information** | Escalate to user | `exit_requested` | Pattern A says X, Pattern B says Y |
| **Design decision** | Escalate to user | `exit_requested` | Multiple valid approaches, unclear which to use |

#### When Execution Should Set exit_requested

```sql
-- Execution discovers ambiguous requirement mid-task
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'exec-03',
    'ESCALATION: Found 50 patterns, unclear which are current.

User decision needed: Should I extract all 50, or only recent ones?

Source conflicts:
- TESTING_GUIDE.md has 2022 patterns
- TESTING_QUICK_REFERENCE.md has 2025 patterns

Cannot proceed autonomously - requirement clarification required.'
);
-- Session will exit (hook allows exit_requested)
-- User reviews situation, clarifies requirement, relaunches with updated instruction
```

**Other escalation scenarios:**
```sql
-- Conflicting information that cannot be resolved
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-04';
INSERT INTO task_messages VALUES (
    'task-04', 'exec-04',
    'ESCALATION: Found contradictory API patterns.

Pattern A (docs/api-v1.md): "Always use POST for mutations"
Pattern B (docs/api-v2.md): "Use PUT for updates, POST for creates"

Both documents appear current (2025 dates). Cannot determine which is authoritative.
User must decide: Extract both with version context? Choose one as canonical?'
);

-- Design decision beyond execution's scope
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-05';
INSERT INTO task_messages VALUES (
    'task-05', 'exec-05',
    'ESCALATION: Database patterns extraction requires architectural decision.

Found 3 different connection pooling approaches:
1. Manual pooling (legacy, still in use)
2. Framework-managed (recommended in 2024 docs)
3. Serverless-optimized (mentioned for future)

User must decide extraction strategy: Document all three? Focus on current recommended?'
);
```

#### When Conductor Should Set exit_requested

```sql
-- Execution task hits terminal error (exited)
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-00';
INSERT INTO task_messages VALUES (
    'task-00', 'task-00',
    'ESCALATION: task-03 reached terminal error (5 retries exhausted).

Error: Test failures could not be resolved autonomously.
See: docs/implementation/reports/task-03-error-final.md

Conductor cannot proceed - user intervention required to:
1. Review error reports
2. Decide if task-03 should be retried with different approach
3. Decide if tasks 04-05 should continue or pause'
);
```

**Other conductor escalation scenarios:**
- Multiple execution tasks fail (partial failure situation)
- Execution explicitly requests user decision (forwards request)
- Conductor uncertain how to resolve conflicting states
- All execution tasks complete but quality concerns prevent finalization

#### Key Principle

**If autonomous recovery is impossible (not a technical error to fix), escalate to user immediately.**

Don't waste retries on non-technical decisions:
- ❌ BAD: Set error state for ambiguous requirement, retry 5 times with same ambiguity
- ✅ GOOD: Set exit_requested immediately, let user clarify, then resume

Execution and conductor sessions should recognize the difference between:
- **Technical problems** (can retry) → `error` state
- **Decision problems** (need user input) → `exit_requested` state

### Non-Error Escalation Logic (Concrete Implementation)

**Complete flow from execution discovering issue → conductor decision → resolution.**

#### Scenario: Execution Discovers Better Approach

**Context:** During implementation, execution realizes current approach has limitations.

**Execution detects issue:**
```bash
#!/bin/bash
# task-03 execution discovers better approach

# Current situation
echo "Current implementation: Synchronous email sending in API request handler"
echo "Problem: Causes 2-second delay in API response time"
echo "Better approach: Background job queue for email sending"

# Decision: This is architectural, requires user approval

# Insert message to conductor
sqlite3 coordination.db <<SQL
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-00', 'task-03',
    'REQUEST USER DECISION - Better Approach Identified:

    Current: Synchronous email sending in POST /api/users endpoint
    - Problem: 2-second delay in API response
    - Works correctly but poor UX

    Alternative: Background job queue (Bull + Redis)
    - Pros: Fast API response (<100ms), scalable, retries
    - Cons: Requires Redis dependency, more complex error handling

    Question: Switch to background queue OR keep synchronous?

    Current state: Synchronous implementation complete and tested.
    Can proceed with current approach or refactor to queue.
    Awaiting guidance.');
SQL

# Set state to waiting (blocked on conductor decision)
sqlite3 coordination.db "UPDATE coordination_status SET state = 'waiting' WHERE task_id = 'task-03';"

# Launch blocking subagent to wait for response
echo "Launching blocking subagent to wait for conductor guidance..."
```

**Conductor receives message:**
```bash
#!/bin/bash
# Conductor subagent detects message

# Query messages from execution tasks
messages=$(sqlite3 coordination.db "SELECT task_id, message FROM task_messages WHERE from_session LIKE 'task-%' AND from_session != 'task-00' AND message LIKE '%REQUEST USER DECISION%' ORDER BY created_at DESC;")

if [ -n "$messages" ]; then
    echo "Non-error escalation detected:"
    echo "$messages"

    # Return to main conductor session
    exit 0  # Exits subagent, conductor handles decision
fi
```

**Conductor decision logic:**

**Step 1: Assess Urgency**

**Question:** Does current approach BLOCK progress?
- **YES** → Must decide immediately (progress blocked)
- **NO** → Can defer decision (let task complete, refactor later)

**Step 2: Assess User Availability**

**Question:** Can user respond quickly (<10 minutes)?
- **YES** → Set conductor state = 'exit_requested', present decision to user
- **NO** → Conductor makes decision based on project priorities

**Step 3: Conductor Autonomous Decision (If User Unavailable)**

**Decision matrix:**

| Factor | Keep Current | Switch to Alternative |
|--------|--------------|----------------------|
| Complexity | Current simpler ✅ | Alternative adds Redis dependency ❌ |
| Performance | 2s delay acceptable for MVP ⚠️ | <100ms much better ✅ |
| Scalability | Works for <100 users ⚠️ | Scales to 10k+ users ✅ |
| Time cost | Already complete ✅ | Refactor: +2 hours ❌ |
| Risk | Low risk (tested) ✅ | Medium risk (new complexity) ⚠️ |

**Scoring:**
- Keep current: 3 ✅, 2 ⚠️, 0 ❌ = **5 points**
- Switch: 2 ✅, 1 ⚠️, 2 ❌ = **3 points**

**Decision:** KEEP CURRENT (for MVP), add TODO for future refactor

```sql
-- Conductor sends decision to execution
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'task-00',
    'DECISION - Keep Current Approach:

    Rationale: Current synchronous approach is acceptable for MVP.
    - Implementation already complete and tested
    - 2-second delay acceptable for current scale (<100 users)
    - Lower risk than refactoring to background queue

    Action: Proceed with current implementation.
    Document TODO for future optimization when scale requires it.

    Future refactor: Add background job queue when user base > 500 users.');

-- Update execution state back to working
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';
```

**Alternative: Escalate to user:**
```sql
-- If conductor cannot decide autonomously
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-00';

INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-00', 'SYSTEM',
    'USER DECISION REQUIRED:

    task-03 identified better approach during implementation.
    Decision requires project priorities knowledge.

    [Present decision details from task-03 message]

    Please review and provide guidance.');
```

**Execution reads conductor decision:**
```bash
#!/bin/bash
# Execution blocking subagent exits when state changes from 'waiting'

# Read conductor decision
decision=$(sqlite3 coordination.db "SELECT message FROM task_messages WHERE task_id = 'task-03' AND from_session = 'task-00' AND message LIKE '%DECISION%' ORDER BY created_at DESC LIMIT 1;")

if echo "$decision" | grep -q "Keep Current Approach"; then
    echo "Conductor decided: Keep current implementation"
    echo "Action: Proceeding with synchronous email sending"

    # Add TODO comment to code
    # Continue with remaining steps

elif echo "$decision" | grep -q "Switch to Alternative"; then
    echo "Conductor decided: Refactor to background queue"
    echo "Action: Implementing Bull + Redis queue"

    # Refactor code to use background queue
    # Update tests
    # Request review after refactor
fi
```

**Complete flow timeline:**

1. **T+0min:** Execution at step 4, notices performance issue
2. **T+1min:** Execution inserts "REQUEST USER DECISION" message, sets state = 'waiting'
3. **T+2min:** Conductor subagent detects message, exits to main session
4. **T+3min:** Conductor reads message, analyzes decision matrix
5. **T+4min:** Conductor inserts "DECISION" message, updates task-03 state = 'working'
6. **T+5min:** Execution subagent detects state change, exits with decision
7. **T+5min:** Execution reads decision message, proceeds with guidance

**Total coordination time:** ~5 minutes (no user involvement required)

#### Other Non-Error Escalation Scenarios

**Scenario: Missing prerequisite discovered mid-task**

```bash
# Execution at step 5, discovers missing file
sqlite3 coordination.db <<SQL
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-00', 'task-03',
    'REQUEST USER DECISION - Missing Prerequisite:

    Cannot proceed with step 5 (database patterns extraction).
    Required file: docs_old/DB_MIGRATION_GUIDE.md (referenced in plan)
    Status: File does not exist

    Options:
    1. User provides correct file path
    2. Skip database patterns (reduce scope)
    3. Accept partial completion (steps 1-4 only)

    Current progress: 4/7 steps complete (57%)
    Quality: All completed steps tested and passing

    Awaiting guidance on how to proceed.');

UPDATE coordination_status SET state = 'waiting' WHERE task_id = 'task-03';
SQL
```

**Conductor autonomous decision:**
```sql
-- Check if other tasks depend on database patterns
SELECT task_id, dependencies FROM migration_tasks WHERE dependencies LIKE '%database-patterns%';

-- If no dependencies: Accept partial completion
-- If dependencies exist: Escalate to user (file needed)

-- Example: No dependencies found
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'task-00',
    'DECISION - Accept Partial Completion:

    Rationale: Database patterns not required for other tasks.
    - Current progress (57%) is acceptable quality
    - Missing file appears to be error in planning phase
    - No downstream impact from omitting database patterns

    Action: Complete task-03 with steps 1-4 only.
    Document database patterns as out-of-scope.
    Mark task as complete (partial scope).');

UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';
```

---

## Error vs exit_requested Decision Tree

### When to Use Each State

**Purpose:** Clarify when execution should:
- Set state = 'error' (autonomous retry possible)
- Send message requesting user decision (escalation)

### Decision Flowchart

```
┌─────────────────────────────┐
│   Problem Encountered       │
└──────────┬──────────────────┘
           │
           ▼
    ┌──────────────┐
    │ Is this a    │
    │ technical    │───Yes──▶ Can autonomous
    │ failure?     │         retry succeed?
    └──┬───────────┘
       │                    ┌────Yes───▶ state = 'error'
       │No                  │            (conductor proposes fix)
       │                    │
       ▼                    └────No────▶ Retry won't help
   ┌──────────────┐                     (architectural issue)
   │ Is this a    │                           │
   │ decision     │───Yes──▶ User approval    │
   │ point?       │         required?         │
   └──┬───────────┘              │            │
      │                          └─Yes─▶ Message conductor
      │No                              │ "REQUEST USER DECISION"
      │                                │ Conductor decides:
      ▼                                │ - Provide guidance OR
   Cannot proceed                      │ - Set exit_requested
   without input                       │
      │                                │
      └────────────────────────────────┘
```

### Decision Matrix

| Scenario | State/Action | Reasoning |
|----------|--------------|-----------|
| **TypeError in API call** | `error` | Technical - conductor can propose type fix |
| **Database connection timeout** | `error` | Technical - conductor can increase timeout |
| **API rate limit exceeded** | `error` | Technical - conductor can add retry with backoff |
| **Dependency not installed** | `error` | Technical - conductor can propose install command |
| **Test failure (correctness)** | `error` | Technical - conductor can analyze and fix logic |
| **File not found (expected)** | `error` | Technical - conductor can check path or create file |
| **Better approach identified** | Message conductor | Decision point - user input valuable |
| **Breaking change required** | Message conductor | Decision point - impacts other systems |
| **Security concern discovered** | Message conductor | Decision point - risk assessment needed |
| **Multiple valid approaches** | Message conductor | Decision point - tradeoffs require judgment |
| **Unrecoverable error (5 retries)** | `exited` | Terminal - discussed with conductor, cannot fix |
| **External blocker (API down)** | Message conductor | Cannot fix - user must decide (wait vs workaround) |

### Example Scenarios

**Scenario 1: Technical Error (use 'error' state)**
```python
# Execution encounters error
try:
    result = api_client.fetch_user(user_id)
except TypeError as e:
    # Technical error - autonomous retry possible

    # Log error details
    log_error(f"TypeError in API call: {e}")
    log_error(f"user_id type: {type(user_id)}, value: {user_id}")

    # Set error state
    execute_sql("""
        UPDATE coordination_status
        SET state = 'error',
            retry_count = retry_count + 1,
            last_error = ?
        WHERE task_id = 'task-03'
    """, (str(e),))

    # Send error report to conductor
    send_error_report(
        task_id='task-03',
        error_type='TypeError',
        error_message=str(e),
        context={'user_id': user_id, 'user_id_type': type(user_id).__name__},
        stack_trace=traceback.format_exc()
    )

    # Wait for conductor fix proposal
    # Will receive state = 'fix_proposed' with instructions
```

**Scenario 2: Decision Point (message conductor)**
```python
# Execution identifies better approach
current_approach = "Synchronous email sending in API handler"
current_performance = "2-second API response time"

alternative_approach = "Background job queue (Bull + Redis)"
alternative_benefits = "Fast API response (<100ms), scalable, retries"
alternative_costs = "Requires Redis dependency, more complex error handling"

# This requires user judgment - not a technical error

# Send decision request to conductor
send_message_to_conductor(
    from_task='task-03',
    to_task='task-00',
    message_type='REQUEST_USER_DECISION',
    subject='Better Approach Identified',
    body=f"""
REQUEST USER DECISION - Better Approach Identified:

Current implementation: {current_approach}
- Works correctly but has performance issue
- {current_performance}

Alternative: {alternative_approach}
- Benefits: {alternative_benefits}
- Costs: {alternative_costs}

Question: Switch to background queue OR keep synchronous?

Current state: Synchronous implementation complete and tested.
Can proceed with current OR refactor to queue based on guidance.
""")

# Set state to 'waiting' (blocked on conductor decision)
execute_sql("UPDATE coordination_status SET state = 'waiting' WHERE task_id = 'task-03'")

# Launch blocking subagent to wait for response
# Will receive message from conductor with decision
```

**Scenario 3: External Blocker (message conductor)**
```python
# External API is down (not a technical error, cannot fix)
try:
    result = external_api.fetch_data()
except APIUnavailableError as e:
    # External blocker - cannot fix with retry

    # Check if temporary or permanent
    status = external_api.check_status()  # Returns: 503 Service Temporarily Unavailable

    # Cannot proceed - need user decision: wait or find workaround

    send_message_to_conductor(
        message_type='EXTERNAL_BLOCKER',
        subject='External API Unavailable',
        body=f"""
EXTERNAL BLOCKER - Cannot Proceed:

External API: {external_api.base_url}
Status: {status} (503 Service Temporarily Unavailable)
Error: {e}

Options:
1. Wait for API recovery (unknown duration)
2. Use cached data (may be stale)
3. Skip this step and mark as partial completion

Question: How should we proceed?

Current state: Blocked at step 3/5 (data fetch from external API)
""")

    # Set state to 'waiting'
    execute_sql("UPDATE coordination_status SET state = 'waiting' WHERE task_id = 'task-03'")
```

### Conductor Response Patterns

**For technical errors (error state):**
```python
# Conductor receives error report
# Analyzes error, proposes fix

execute_sql("""
    UPDATE coordination_status
    SET state = 'fix_proposed'
    WHERE task_id = 'task-03'
""")

send_message_to_execution(
    task_id='task-03',
    message_type='FIX_PROPOSAL',
    retry_number=2,
    root_cause='user_id is string, API expects integer',
    fix='Convert user_id to int: user_id = int(user_id)',
    location='api/users.py:45, before api_client.fetch_user() call',
    retry_instructions='Re-run API call after applying fix'
)
```

**For decision requests:**
```python
# Conductor receives decision request
# Evaluates based on project priorities

# Option A: Conductor makes decision autonomously
send_message_to_execution(
    task_id='task-03',
    message_type='DECISION',
    decision='Keep current synchronous approach',
    rationale='Acceptable for MVP, lower risk than refactor'
)

# Option B: Conductor escalates to user
execute_sql("UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-00'")
# User reviews, provides guidance, conductor relays to execution
```

---

## Example Type Decision Matrix (Prescriptive vs Adaptive)

**Critical distinction for template users: When to follow examples exactly vs adapt to project needs.**

### Prescriptive (Exact Code) - Use When:

**Hook-related code (MUST be copy-pasteable):**
- ✅ Bash commands in hook scripts
- ✅ SQL queries for coordination database
- ✅ Subagent prompts that check state
- ✅ State transition logic

**Safety-critical code (MUST prevent data loss/corruption):**
- ✅ Error handling patterns
- ✅ Session isolation logic
- ✅ Database transaction patterns
- ✅ Retry loop structure

**Coordination patterns (MUST maintain consistency):**
- ✅ Message passing format
- ✅ State change sequences
- ✅ Subagent launch/exit logic

**Examples of prescriptive code:**

**1. Hook exit criteria check (exact code required):**
```bash
# PRESCRIPTIVE: Hook must check state exactly as shown
state=$(sqlite3 coordination.db "SELECT state FROM coordination_status WHERE task_id = 'task-03';")
if [[ "$state" == "complete" ]] || [[ "$state" == "exited" ]]; then
    exit 0  # Allow hook to pass
else
    exit 1  # Block hook (work not complete)
fi
```

**2. Subagent state monitoring (exact SQL required):**
```sql
-- PRESCRIPTIVE: Query must check exact states
SELECT state FROM coordination_status
WHERE task_id = 'task-03'
  AND state IN ('review_approved', 'review_failed', 'fix_proposed');
```

**3. Message insertion pattern (exact format required):**
```sql
-- PRESCRIPTIVE: Message format for coordination
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'task-00',
    'REVIEW FEEDBACK: [specific issues]');
```

### Adaptive (Conceptual Guidance) - Use When:

**Domain-specific implementation (varies by project):**
- ⚠️ Test patterns (language/framework-specific)
- ⚠️ API design (project architecture)
- ⚠️ UI components (design system)
- ⚠️ Code style (project conventions)

**Conductor logic (varies by quality standards):**
- ⚠️ Review criteria (project-specific standards)
- ⚠️ Smoothness assessment (subjective judgment)
- ⚠️ Completion thresholds (phase-dependent)
- ⚠️ Error fix strategies (context-dependent)

**Execution implementation (varies by task):**
- ⚠️ Specific work steps (task-specific)
- ⚠️ File organization (project structure)
- ⚠️ Testing approach (domain needs)

**Examples of adaptive guidance:**

**1. Conductor review criteria (conceptual):**
```markdown
# ADAPTIVE: Review guidance (adapt to project standards)
Review the implementation for:
- TypeScript type safety (strict mode, no `any` in public APIs)
- Test coverage (>80% for new code, critical paths tested)
- Error handling (proper exceptions, no silent failures)
- Performance (no obvious O(n²) algorithms)

Approve if quality meets project standards. Use judgment for edge cases.
```

**2. Execution test patterns (conceptual):**
```markdown
# ADAPTIVE: Testing approach (varies by framework)
For each extracted pattern:
1. Create test file using project's test framework (Jest, pytest, etc.)
2. Write unit tests for core functionality
3. Add integration tests if pattern interacts with external systems
4. Follow project's test naming conventions

Test coverage target: >80% for new code
```

**3. Execution file organization (conceptual):**
```markdown
# ADAPTIVE: File structure (varies by project)
Organize extracted patterns into:
- patterns/[category]/[pattern-name].md
- Follow existing project structure
- Use descriptive filenames
- Group related patterns

Adapt structure to match existing docs/ organization.
```

### Mixed (Concrete Framework + Adaptive Details)

**Coordination patterns: Framework is prescriptive, details adaptive**

**Example: Error recovery flow**
```bash
# CONCRETE FRAMEWORK (prescriptive):
for attempt in {1..5}; do
    if execute_operation; then
        break  # Success
    else
        # Log error (prescriptive structure)
        sqlite3 coordination.db "UPDATE coordination_status SET state = 'error', retry_count = $attempt WHERE task_id = 'task-03';"

        # ADAPTIVE DETAILS: Request fix from conductor (strategy varies by error)
        # Conductor analyzes error type:
        # - TypeError → type conversion fix
        # - Timeout → increase timeout value
        # - Missing file → check path or create file
        # Fix strategy depends on error context
    fi
done

# Terminal error after 5 retries (prescriptive)
sqlite3 coordination.db "UPDATE coordination_status SET state = 'exited' WHERE task_id = 'task-03';"
```

**Example: Review checkpoint**
```sql
-- CONCRETE FRAMEWORK (prescriptive):
UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-03';

INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'task-03',
    'REVIEW REQUEST: [checkpoint name]
    Changed files: [file list]
    Test results: [pass/fail]
    Details: [report path]');

-- ADAPTIVE DETAILS: Checkpoint criteria (varies by project phase)
-- MVP phase: Basic functionality, accept rough edges
-- Production phase: High quality, strict standards
-- Conductor adapts acceptance criteria to project phase
```

### Decision Criteria Table

| Code Category | Prescriptive | Adaptive | Example |
|---------------|-------------|----------|---------|
| **Hook scripts** | ✅ REQUIRED | ❌ No | State checks, exit criteria |
| **SQL coordination** | ✅ REQUIRED | ❌ No | State updates, message format |
| **Subagent prompts** | ✅ REQUIRED | ⚠️ Minor | State monitoring (exact), response time (flexible) |
| **Message format** | ✅ REQUIRED | ⚠️ Minor | Keywords (exact), details (adaptive) |
| **Error recovery** | ✅ Framework | ✅ Details | 5-retry loop (exact), fix strategy (adaptive) |
| **Review criteria** | ⚠️ Structure | ✅ Standards | Request format (exact), quality bar (adaptive) |
| **Test patterns** | ❌ No | ✅ REQUIRED | Framework-specific, project conventions |
| **Code implementation** | ❌ No | ✅ REQUIRED | Language/domain-specific |
| **File organization** | ⚠️ Proposals | ✅ Final | Proposal structure (exact), paths (adaptive) |

### When In Doubt: Default to Prescriptive

**If unsure whether code is prescriptive or adaptive:**

1. **Ask:** Does this code coordinate between sessions?
   - **YES** → Prescriptive (consistency critical)
   - **NO** → Continue to step 2

2. **Ask:** Does this code interact with hooks or database?
   - **YES** → Prescriptive (exact schema/state required)
   - **NO** → Continue to step 3

3. **Ask:** Does this code prevent data loss or corruption?
   - **YES** → Prescriptive (safety critical)
   - **NO** → Adaptive (project-specific)

**Examples of classification:**

| Code | Step 1 (Coordinate?) | Step 2 (Hooks/DB?) | Step 3 (Safety?) | Result |
|------|---------------------|-------------------|-----------------|--------|
| `sqlite3 "SELECT state..."` | YES | - | - | **Prescriptive** |
| Test file creation | NO | NO | NO | **Adaptive** |
| Retry loop structure | NO | YES (updates state) | - | **Prescriptive** |
| API endpoint design | NO | NO | NO | **Adaptive** |
| Message insertion SQL | YES | - | - | **Prescriptive** |
| Error fix strategy | NO | NO | NO (conductor decides) | **Adaptive** |

### Summary

**Copy exactly from template:**
- All hook code
- All SQL coordination queries
- All state transition logic
- All subagent monitoring patterns
- All message passing formats

**Adapt to your project:**
- Test patterns (use your framework)
- Code style (use your conventions)
- Review criteria (use your quality standards)
- Implementation details (use your architecture)
- File organization (use your structure)

**When template says "example":** Understand the pattern, adapt the details.
**When template says "required":** Copy exactly, no modifications.

---

### Error Recovery (5-Retry Flow)

#### Overview

**Error states:**
- `error` - Recoverable error (retries 1-4)
- `fix_proposed` - Conductor proposed error fix (transient <60s)
- `exited` - Terminal error (retry 5 exhausted)

**Who sets error states:**
- Execution session determines its own error state
- Conductor does NOT set error, only sets review_failed with fix proposals
- After 5 failed retries, execution sets exited and terminates

**Retry count tracking:**
- Stored in task_messages content
- Format: "ERROR (Retry 3/5): ..." or "FIX PROPOSAL (Retry 3/5): ..."
- No schema changes needed

#### Retry 1-4: Autonomous Recovery

**Execution session encounters error:**
```sql
-- Step 1: Set error state
UPDATE coordination_status SET state = 'error' WHERE task_id = 'task-03';

-- Step 2: Create error report
-- docs/implementation/reports/task-03-error-retry-1.md
-- Include: Error message, stack trace, what was attempted, current state

-- Step 3: Insert error message
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'exec-03-20260204-1430',
'ERROR (Retry 1/5): Test failed after implementation
Error: test_auth_integration failed with timeout
Report: docs/implementation/reports/task-03-error-retry-1.md
Awaiting conductor fix proposal');

-- Step 4: Launch blocking subagent to wait for fix
```

**Conductor detects error:**
```python
# Subagent exits with "ERROR_DETECTED: task-03"

# Step 1: Read error report
error_msg = query("SELECT message FROM task_messages WHERE task_id='task-03' ORDER BY timestamp DESC LIMIT 1")
report_path = parse_report_path(error_msg)
error_report = read_file(report_path)

# Step 2: Analyze root cause
# - Read error details
# - Review recent changes
# - Check for common patterns
# - Determine fix strategy

# Step 3: Propose fix
UPDATE coordination_status SET state = 'fix_proposed' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'task-00',
    'FIX PROPOSAL (Retry 1/5):
    Root cause: Auth service timeout set to 5s, integration test expects 3s
    Fix: Update auth.ts:45 timeout to 3000ms to match test expectations
    Retry: Re-run tests after fix'
);
```

**Execution session applies fix:**
```python
# Blocking subagent exits with "FAILED" (review_failed detected)

# Step 1: Read fix proposal
fix_msg = query("SELECT message FROM task_messages WHERE task_id='task-03' AND from_session='task-00' ORDER BY timestamp DESC LIMIT 1")

# Step 2: Parse fix instructions
# "Update auth.ts:45 timeout to 3000ms"

# Step 3: Apply fix
# Edit auth.ts:45, change timeout

# Step 4: Update state back to working
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';

# Step 5: Retry the failed operation
# Re-run tests

# Step 6: If still fails, increment retry count and repeat
# If succeeds, continue to next step
```

#### Retry 5: Terminal Error

**After 4 failed retry attempts:**
```sql
-- Execution session determines this is terminal
-- Step 1: Set exited state
UPDATE coordination_status SET state = 'exited' WHERE task_id = 'task-03';

-- Step 2: Update migration_tasks
UPDATE migration_tasks SET status = 'failed', completed_at = datetime('now')
WHERE task_id = 'task-03';

-- Step 3: Insert terminal error message
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'exec-03-20260204-1430',
'CRITICAL ERROR: Max retries (5) exhausted. Task requires user intervention.
Last error: [description]
All attempted fixes: [list of 5 fix attempts]
Session terminating.');

-- Step 4: Generate error report
-- docs/implementation/reports/task-03-error-final.md

-- Session will exit (hook allows state=exited)
```

**Conductor detects terminal error:**
```python
# Subagent exits with "ERROR_DETECTED: task-03 (EXITED)"

# Step 1: Read terminal error message
error_msg = query("SELECT message FROM task_messages WHERE task_id='task-03' ORDER BY timestamp DESC LIMIT 1")

# Step 2: Log to conductor context
# "Task-03 encountered terminal error after 5 retries"

# Step 3: Check other tasks
remaining_tasks = query("SELECT task_id FROM coordination_status WHERE task_id LIKE 'task-%' AND state NOT IN ('complete', 'exited')")

# Step 4: Decide next action
if len(remaining_tasks) == 0:
    # All tasks either complete or exited
    # Set conductor to exit_requested for user review
    UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-00';
else:
    # Other tasks still running
    # Continue monitoring, user will review exited task later
    continue_monitoring()
```

#### Error Classification

**Recoverable errors (retry 1-4):**
- Test failures (flaky tests, timing issues)
- Missing dependencies (can be installed)
- Configuration errors (can be corrected)
- Code bugs (can be fixed)
- Validation failures (can be addressed)

**Terminal errors (retry 5 or immediate exit):**
- Fundamental misunderstanding of requirements
- Cannot access required resources
- Circular dependency detected
- Corruption of critical files
- User intervention explicitly required

**Examples:**

**Recoverable:**
```
ERROR (Retry 2/5): Integration test failed
Error: Connection refused to localhost:8080
Fix: Ensure test server is running before tests
```

**Terminal (immediate):**
```
CRITICAL ERROR: Cannot proceed without user decision
Situation: Found 2 conflicting patterns for same use case
Pattern A: From TESTING_GUIDE.md (newer)
Pattern B: From old docs (conflicts with A)
User must decide which pattern is correct before extraction can continue
```

---

## Error Report Template Structure

### Purpose

Standardized error reports enable conductor to:
- Quickly understand root cause
- Propose targeted fixes
- Track error patterns across retries

### Template Format

```markdown
ERROR REPORT (Retry X/5)

**Task:** [task-id]
**Step:** [which step failed]
**Error Type:** [TypeError|DatabaseError|APIError|ValidationError|etc.]
**Error Message:** [exception message]

**Context:**
- Input values: [relevant variables and their values]
- State before error: [what was the system state]
- Expected behavior: [what should have happened]
- Actual behavior: [what actually happened]

**Stack Trace:**
```
[full stack trace]
```

**Relevant Code:**
```[language]
[code snippet showing where error occurred, ~10 lines context]
```

**Previous Retries:**
[If retry > 1, summarize what was tried before]
- Retry 1: [what fix was attempted, what happened]
- Retry 2: [what fix was attempted, what happened]

**Suggested Investigation:**
[Execution's analysis of possible causes, if any insights available]
```

### Example: TypeError Error Report

```markdown
ERROR REPORT (Retry 2/5)

**Task:** task-03
**Step:** Step 3 - Fetch user data from API
**Error Type:** TypeError
**Error Message:** fetch_user() argument 'user_id' must be int, not str

**Context:**
- Input values:
  - user_id = "12345" (type: str)
  - api_endpoint = "/api/users/{user_id}"
- State before error: Successfully connected to API, authenticated
- Expected behavior: API should return user data for ID 12345
- Actual behavior: API client raised TypeError before making request

**Stack Trace:**
```
Traceback (most recent call last):
  File "api/users.py", line 45, in get_user_profile
    result = api_client.fetch_user(user_id)
  File "lib/api_client.py", line 128, in fetch_user
    if not isinstance(user_id, int):
TypeError: fetch_user() argument 'user_id' must be int, not str
```

**Relevant Code:**
```python
# api/users.py:40-50
def get_user_profile(user_id):
    """Fetch user profile from external API"""
    # user_id comes from route parameter (always string)
    api_client = APIClient(base_url=CONFIG['api_url'])

    # ERROR OCCURS HERE (line 45)
    result = api_client.fetch_user(user_id)  # user_id is str, needs int

    return result
```

**Previous Retries:**
- Retry 1: Original attempt - no fix applied, same error

**Suggested Investigation:**
The route parameter `user_id` is extracted as a string from the URL path.
The API client expects an integer. Need type conversion before calling fetch_user().

Possible fix: `user_id = int(user_id)` before line 45
```

### Example: Database Timeout Error Report

```markdown
ERROR REPORT (Retry 3/5)

**Task:** task-04
**Step:** Step 2 - Query large dataset
**Error Type:** DatabaseTimeoutError
**Error Message:** Query exceeded timeout limit (5000ms)

**Context:**
- Input values:
  - query = "SELECT * FROM transactions WHERE user_id = 12345"
  - timeout = 5000ms
  - result_count = Unknown (query didn't complete)
- State before error: Database connection established
- Expected behavior: Query returns transaction records within 5s
- Actual behavior: Query timeout after 5 seconds

**Stack Trace:**
```
Traceback (most recent call last):
  File "db/queries.py", line 78, in get_user_transactions
    results = db.query(sql, timeout=5000)
  File "lib/database.py", line 203, in query
    raise DatabaseTimeoutError(f"Query exceeded timeout limit ({timeout}ms)")
DatabaseTimeoutError: Query exceeded timeout limit (5000ms)
```

**Relevant Code:**
```python
# db/queries.py:70-85
def get_user_transactions(user_id):
    """Fetch all transactions for a user"""
    sql = """
        SELECT * FROM transactions
        WHERE user_id = ?
        ORDER BY created_at DESC
    """

    # ERROR OCCURS HERE (line 78) - query times out
    results = db.query(sql, params=[user_id], timeout=5000)

    return [Transaction.from_row(r) for r in results]
```

**Previous Retries:**
- Retry 1: Original attempt - 5s timeout, timed out
- Retry 2: Conductor increased timeout to 10s - still timed out

**Suggested Investigation:**
The query is doing a full table scan on `transactions` table.
Likely missing index on `user_id` column.

Database analysis:
- Table size: ~5 million rows
- Query plan: SCAN transactions (no index on user_id)

Possible fixes:
1. Add index: CREATE INDEX idx_user_id ON transactions(user_id)
2. Increase timeout further (30s) if index can't be added
3. Add LIMIT clause if full history not needed
```

### Template Benefits

**Faster conductor analysis:**
- Structured format → easy to parse
- Context included → no back-and-forth questions
- Previous retries → avoid repeating failed fixes

**Pattern detection:**
- Similar errors across tasks become visible
- Common root causes identified
- Systemic issues surfaced

**Debugging efficiency:**
- Stack traces + code context
- Input values captured
- Execution's insights included

---

### Edge Case Handling

#### Session Crashes

**Execution session crashes mid-work:**

**Detection:**
```sql
-- Conductor notices task stuck
SELECT task_id, state, started_at
FROM migration_tasks
WHERE status = 'in_progress'
  AND started_at < datetime('now', '-30 minutes')
  AND task_id NOT IN (SELECT task_id FROM coordination_status WHERE state = 'complete');

-- If task-03 found: Likely crashed
```

**Recovery:**
```sql
-- Manual intervention by user or conductor
-- Option A: Mark as failed, investigate later
UPDATE migration_tasks SET status = 'failed', completed_at = datetime('now')
WHERE task_id = 'task-03';
UPDATE coordination_status SET state = 'exited' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'manual-recovery',
    'Session crashed. Marked as failed for manual recovery.'
);

-- Option B: Resume in new session (if work recoverable)
-- 1. Launch new execution session with same task-03 instruction
-- 2. Check last completed step from git commits
-- 3. Resume from last checkpoint
```

**Conductor crashes:**

**Recovery:**
```bash
# User relaunches conductor session
# Hook was removed when session exited
# Re-initialize hook
bash tools/message-watcher/setup.sh --preset orchestration

# Check current state of all tasks
SELECT task_id, state FROM coordination_status WHERE task_id LIKE 'task-%';

# Resume monitoring from current state
# Conductor picks up where it left off
```

#### Stale States

**Definition:** Task stuck in transient state (review_approved/review_failed) for >60 seconds

**Cause:** Execution session crashed after conductor set transient state but before execution polled

**Detection:**
```python
# Conductor subagent includes staleness check
SELECT task_id, state,
       (julianday('now') - julianday(
           (SELECT timestamp FROM task_messages
            WHERE task_id = coordination_status.task_id
            ORDER BY timestamp DESC LIMIT 1)
       )) * 86400 as seconds_since_last_message
FROM coordination_status
WHERE state IN ('review_approved', 'review_failed')
  AND seconds_since_last_message > 60;

# If found: Return "STALE_STATE: task-03 in state 'review_approved' for 125s"
```

**Recovery:**
```sql
-- Conductor investigates
-- Option A: Execution session crashed, mark as error
UPDATE coordination_status SET state = 'error' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'task-00',
    'ERROR: Session appears to have crashed. State was review_approved for >60s. Manual recovery needed.'
);

-- Option B: Execution session just very slow, wait longer
-- (Conductor decides based on other factors)
```

---

## Subagent Timeout Configuration

### Timeout Design Principles

**Purpose of timeouts:**
1. **Crash detection:** Identify when session has died/frozen
2. **Progress verification:** Ensure work isn't stalled indefinitely
3. **Resource cleanup:** Free up resources from dead sessions

**Trade-offs:**
- Too short: False positives (timeout while working), wasted context
- Too long: Delayed crash detection, resources tied up

**Target:** Set timeout to 1.5-2x expected duration (reasonable safety margin)

### Timeout Calculation Formula

**General formula:**
```
max_iterations = (expected_duration_minutes × safety_margin) / (check_interval_seconds / 60)

Where:
- expected_duration_minutes: Typical time for this operation
- safety_margin: 1.5-2.0x (50-100% buffer)
- check_interval_seconds: How often subagent checks status
```

### Conductor Subagent Timeout

**Configuration: 600 iterations at 12-second interval = 120 minutes**

**Calculation:**
```
Expected duration: 60-90 minutes
- Typical parallel project: 3-5 tasks, each 15-30 minutes
- Most projects complete within 90 minutes

Safety margin: 1.5x
- Allows for slower tasks, unexpected complexity

max_iterations = (90 × 1.5) / (12/60)
               = 135 / 0.2
               = 675 iterations

Configured: 600 iterations (rounded for cleaner number)
Actual timeout: 120 minutes (600 × 12s)
```

**Why 12-second interval:**
- Database query cost: ~20-50ms
- Acceptable response time: Conductor should notice review request within 12-20s
- Database load: 5 queries/minute = negligible
- Trade-off: Faster (8s) increases load by 50% with minimal benefit

**Why 500 vs 600 confusion:**
- Early template version: 500 iterations (100 minutes)
- Updated after testing: 600 iterations (120 minutes) more reliable
- **Standard:** Use 600 iterations (2-hour timeout)

### Execution Background Monitoring Timeout

**Configuration: 75 iterations at 8-second interval = 10 minutes**

**Calculation:**
```
Expected duration: 5-10 minutes between review checkpoints
- Typical step batch: 3-5 steps
- Each step: 1-3 minutes

Safety margin: 1.5x

max_iterations = (10 × 1.5) / (8/60)
               = 15 / 0.133
               = 112.5 iterations

Configured: 75 iterations (⚠️ CONSERVATIVE - might be too short)
Actual timeout: 10 minutes (75 × 8s)
```

**Problem with 75 iterations:**
- Some long-running steps (database migrations, large file processing) take >10 minutes
- Timeout before completion → false positive crash detection

**Recommended update: 120 iterations (16 minutes)** for better safety margin

**Why 8-second interval:**
- Must be more responsive than conductor (execution waits for conductor)
- Typical step duration: 30-120s → checks 4-15 times per step
- Critical message response: Execution should notice conductor fix within 8-12s
- Trade-off: Faster (5s) only improves response by 3s

### Execution Review Wait Timeout

**Configuration: 1000 iterations at 8-second interval = 133 minutes**

**Calculation:**
```
Expected duration: Varies widely by review complexity
- Simple review (approval): 2-5 minutes
- Complex review (detailed feedback): 10-20 minutes
- Error analysis: 5-15 minutes

Safety margin: 3x (reviews are unpredictable, conductor might be busy with other tasks)

max_iterations = (20 × 3) / (8/60)
               = 60 / 0.133
               = 450 iterations

Configured: 1000 iterations (VERY conservative)
Actual timeout: 133 minutes
```

**Why so conservative:**
- Conductor might be reviewing multiple tasks sequentially
- Error analysis can take significant time (read logs, analyze root cause, propose fix)
- False positive here is costly (interrupts review, wastes conductor work)
- Better to wait longer than interrupt active review

### Timeout Handling Behavior

**When max_iterations reached:**

```bash
#!/bin/bash
# Subagent timeout handling

if [ $iteration_count -ge $max_iterations ]; then
    echo "Timeout reached after ${max_iterations} iterations"
    echo "Duration: $((max_iterations * check_interval))seconds"

    # Log timeout event
    sqlite3 coordination.db <<SQL
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'SUBAGENT-TIMEOUT',
    'TIMEOUT: Subagent reached max iterations (${max_iterations}).
    Duration: $((max_iterations * check_interval))s
    Last state: ${current_state}
    Possible causes: Session crashed, stuck in infinite loop, very long operation');
SQL

    # Return timeout signal to main session
    echo "TIMEOUT: No state change after $((max_iterations * check_interval))s"
    exit 0
fi
```

**Main session response to timeout:**

**Step 1: Verify Actual State**

```bash
# Don't trust subagent timeout immediately - verify with direct query
current_state=$(sqlite3 coordination.db "SELECT state FROM coordination_status WHERE task_id = 'task-03';")

if [[ "$current_state" == "complete" ]] || [[ "$current_state" == "exited" ]]; then
    echo "False positive: Task actually complete, subagent just timed out"
    echo "Reason: Very long operation completed right before timeout"
    # Continue normally
else
    echo "Confirmed timeout: Task stuck in state $current_state"
    # Handle as potential crash
fi
```

**Step 2: Decide Action Based on State**

```bash
case "$current_state" in
    "working")
        # Likely crash - working shouldn't take >2 hours without checkpoints
        echo "LIKELY CRASH: Task in 'working' for >2 hours"
        # Escalate to conductor or user
        ;;

    "needs_review")
        # Conductor might be overloaded or crashed
        echo "REVIEW TIMEOUT: Conductor hasn't responded to review request"
        # Relaunch conductor subagent with fresh timeout
        ;;

    "error")
        # Waiting for conductor fix proposal
        echo "ERROR TIMEOUT: Conductor hasn't proposed fix"
        # Check conductor health, relaunch if needed
        ;;
esac
```

**Step 3: Relaunch Subagent (If Appropriate)**

```bash
# Give session more time - relaunch subagent with fresh iteration count
echo "Relaunching subagent with extended timeout..."

subagent_id=$(launch_subagent \
    task_id="task-03" \
    max_iterations=600 \
    check_interval=8)

# Monitor new subagent
# If times out again → escalate as confirmed crash
```

### Adaptive Timeout Guidelines

**For long-running steps (>15 minutes):**

**Execution Task Instruction Override:**

Step 5: Process 10,000 database records (estimated 20 minutes)

**Override subagent timeout for this step:**
```bash
# Launch subagent with extended timeout
subagent_id=$(launch_background_subagent \
    task_id="task-03" \
    max_iterations=200 \  # 26 minutes at 8s intervals
    check_interval=8)

# Rationale: Default 75 iterations (10min) too short for 20min operation
```

**Update heartbeat frequently within step:**
```bash
for i in {1..10000}; do
    process_record "$i"

    # Update heartbeat every 1000 records (prevents timeout)
    if (( i % 1000 == 0 )); then
        sqlite3 coordination.db "UPDATE coordination_status SET last_heartbeat = CURRENT_TIMESTAMP WHERE task_id = 'task-03';"
    fi
done
```

**For fast-iteration projects:**

**Conductor Configuration Override:**

**Project:** Microservice with small focused tasks (5-10 minute tasks)

**Override conductor timeout:**
```bash
# Shorter timeout acceptable - tasks complete quickly
max_iterations=300  # 60 minutes at 12s intervals

# Rationale: Tasks rarely exceed 30 minutes, 60-minute timeout sufficient
```

### Summary Table

| Subagent Type | Interval | Iterations | Timeout | Use Case |
|---------------|----------|------------|---------|----------|
| Conductor monitoring | 12s | 600 | 120min | Monitor all execution tasks |
| Execution background | 8s | 75 → 120* | 10-16min | Monitor between checkpoints |
| Execution review wait | 8s | 1000 | 133min | Wait for conductor review |
| Execution long operation | 8s | 200+ | 26min+ | Override for known long steps |

*Recommended update from 75 to 120 iterations for better safety margin

---

## Check Interval Configuration

### Design Principles

**Purpose of check intervals:**
1. **Responsiveness:** How quickly should sessions notice state changes?
2. **Database load:** How many queries per hour is acceptable?
3. **Context cost:** Frequent checks = more tokens used

**Trade-offs:**
- Faster checks: Better responsiveness, higher database load, more context
- Slower checks: Worse responsiveness, lower database load, less context

**Target:** Balance responsiveness with resource efficiency

### Conductor Check Interval: 12 Seconds

**Rationale:**

**Responsiveness requirement:**
- Conductor should notice review request within ~15-20 seconds (acceptable UX)
- Average detection time: interval / 2 = 12s / 2 = 6 seconds
- Worst case detection: 12 seconds (task requests review right after check)

**Database load:**
```
Queries per minute: 60s / 12s = 5 queries
Queries per hour: 5 × 60 = 300 queries
Query cost: ~20-50ms per query
Total DB time per hour: 300 × 50ms = 15 seconds
```
- 15 seconds of DB time per hour = 0.4% utilization → negligible load

**Context cost:**
- Subagent runs in background (0 main session context)
- Subagent query result: ~100 tokens per check
- Over 120 minutes: 600 checks × 100 tokens = 60k tokens
- Acceptable for autonomous coordination value

**Alternatives considered:**
- 8 seconds: Improves response by 4s (marginal), increases load by 50% (300 → 450 queries/hr)
- 20 seconds: Reduces load by 40% (300 → 180 queries/hr), degrades response to 10s average (too slow)

**Decision:** 12 seconds balances responsiveness (6s average) with efficiency (300 queries/hr)

### Execution Check Interval: 8 Seconds

**Rationale:**

**Responsiveness requirement:**
- Must be more responsive than conductor (execution waits for conductor, not vice versa)
- Critical messages (fix proposals, review feedback) should be noticed within ~10 seconds
- Average detection: 8s / 2 = 4 seconds
- Worst case detection: 8 seconds

**Step duration context:**
- Typical step: 30-120 seconds
- Checks per step: 30s/8s = 4 checks (minimum) to 120s/8s = 15 checks (maximum)
- Provides good visibility into step progress

**Database load:**
```
Queries per minute: 60s / 8s = 7.5 queries
Queries per hour: 7.5 × 60 = 450 queries
Query cost: ~20-50ms (simple query on indexed column)
Total DB time per hour: 450 × 50ms = 22.5 seconds
```
- Still negligible load (0.6% utilization)

**Why faster than conductor:**
- Execution is reactive (waits for conductor decisions)
- Conductor is proactive (monitors but not blocked)
- Faster execution response = better autonomous feel

**Alternatives considered:**
- 5 seconds: Improves response by 3s (marginal), increases load by 60% (450 → 720 queries/hr)
- 12 seconds: Reduces load by 40% (450 → 300 queries/hr), degrades response to 6s average (too slow for reactive session)

**Decision:** 8 seconds provides responsive feel (4s average) while maintaining low database load

### Transient State Timeout: 60 Seconds

**Rationale:**

**Expected transition time:**
```
Normal scenario:
T+0s:  Conductor sets state = 'review_approved'
T+4s:  Execution subagent detects (average 8s/2 = 4s)
T+5s:  Execution reads message, processes, sets state = 'working'

Total: ~5 seconds (normal case)
```

**Timeout calculation:**
- Normal transition: 5-10 seconds
- Long-running step before check: 15-30 seconds (execution finishes operation before checking subagent)
- Timeout: 60 seconds = 2x worst-case normal scenario
- Buffer: 30 seconds for unexpected delays (slow database, high load)

**False positive rate:**
- Very low: Session would need to be frozen (not working) but not crashed for >60s
- Unlikely scenario: Execution is either working (progresses) or crashed (doesn't check)

**False negative rate:**
- Low: 60 seconds is long enough to avoid false alarms on slow operations

**Why not shorter (30 seconds):**
- Risk of false positives on slow sessions (database query takes 20s, check takes 5s, processing takes 8s = 33s)
- False positive is costly: Triggers crash detection, interrupts work, wastes conductor time

**Why not longer (120 seconds):**
- Delayed crash detection: If session crashes, takes 2 minutes to notice
- Ties up resources: Database state stuck in transient for 2 minutes

**Decision:** 60 seconds = 6x expected time = reliable crash detection with low false positive rate

### Heartbeat Timeout: 180 Seconds (3 Minutes)

**Note:** This is from Tier 1 (heartbeat crash detection)

**Rationale:**

**Expected heartbeat frequency:**
- Execution updates heartbeat between steps (every 30-120 seconds)
- Short steps (<30s): Heartbeat every 30s
- Long steps (>120s): Heartbeat midway through step + at completion

**Timeout calculation:**
- Expected max time between heartbeats: 120 seconds (long step with no midway update)
- Timeout: 180 seconds = 1.5x expected max
- Buffer: 60 seconds for unexpected delays

**Why 3 minutes:**
- Long enough: Avoids false positives on long steps (no midway heartbeat)
- Short enough: Crash detected within 3 minutes (much better than 2-hour max_iterations timeout)
- Practical: Gives conductor fast feedback without overwhelming with false alarms

**Comparison to alternatives:**
- Max iterations timeout: 120 minutes (75 iterations × 8s) - too slow
- Transient state timeout: 60 seconds - only detects crashes during transient states
- Heartbeat timeout: 180 seconds - detects crashes in any state, fast enough for action

**Decision:** 3-minute heartbeat timeout provides fast crash detection (vs 2-hour fallback) while avoiding false positives

### Timing Adaptation Guidelines

**For slow database (>100ms query latency):**
```markdown
## Override for Slow Database

**Problem:** Database queries take 200ms instead of 50ms
- 450 queries/hr × 200ms = 90 seconds of DB time per hour = 2.5% utilization

**Solution:** Increase check intervals to reduce load
- Conductor: 20 seconds (180 queries/hr)
- Execution: 15 seconds (240 queries/hr)

**Trade-off:** Slightly worse responsiveness (10s average vs 6s) but manageable database load
```

**For fast critical responses:**
```markdown
## Override for Time-Critical Project

**Example:** Real-time trading system where 5-second response is critical

**Solution:** Decrease execution interval only
- Conductor: 12 seconds (reviews are not time-critical)
- Execution: 5 seconds (critical message response within 2.5s average)

**Trade-off:** Higher database load (720 queries/hr) but acceptable for critical system
```

**For very long steps (>30 minutes):**
```markdown
## Override for Long-Running Operations

**Example:** Database migration that takes 45 minutes

**Solution:** Add heartbeat updates within step (don't change intervals)
```bash
# Long-running step with heartbeat updates
for i in {1..10}; do
    run_migration_batch "$i"

    # Update heartbeat every 3 minutes within long operation
    sqlite3 coordination.db "UPDATE coordination_status SET last_heartbeat = CURRENT_TIMESTAMP WHERE task_id = 'task-03';"

    sleep 180  # 3 minutes per batch
done
```

**Rationale:** Heartbeat updates prevent false timeouts without increasing check frequency
```

### Summary Table: Timing Configuration

| Parameter | Value | Purpose | Rationale |
|-----------|-------|---------|-----------|
| Conductor check interval | 12s | Monitor execution tasks | Balance: 6s avg response, 300 queries/hr |
| Execution check interval | 8s | Monitor conductor messages | Faster than conductor (reactive), 4s avg response |
| Transient state timeout | 60s | Detect crashed transitions | 6x expected time, low false positives |
| Heartbeat timeout | 180s | Detect working state crashes | Fast detection (3min) without false alarms |
| Conductor max iterations | 600 (120min) | Prevent infinite monitoring | 1.5x expected project duration (90min) |
| Execution max iterations | 120 (16min) | Prevent infinite waiting | 1.5x expected checkpoint interval (10min) |
| Review wait max iterations | 1000 (133min) | Allow long reviews | 3x expected review time (unpredictable) |

### Performance Impact Analysis

**Database query load:**
```
Conductor: 300 queries/hr × 50ms = 15s/hr = 0.4% utilization
Execution (3 tasks): 3 × 450 queries/hr × 50ms = 67.5s/hr = 1.9% utilization
Total: 2.3% database utilization for coordination

Acceptable threshold: <5% utilization
Actual: 2.3% ✅ WELL WITHIN LIMITS
```

**Context token usage:**
```
Conductor subagent: 60k tokens over 120min (runs in background, 0 main session cost)
Execution subagents (3 tasks): 3 × 30k tokens over 60min avg = 90k tokens (background)
Heartbeat updates: 500 tokens over 120min (main session, 0.27% of context)

Total main session cost: 500 tokens for heartbeat only
Benefit: Autonomous coordination worth minimal cost
```

**Responsiveness achieved:**
```
Conductor notices review request: 6s average, 12s worst case ✅
Execution notices conductor message: 4s average, 8s worst case ✅
Crash detection (transient states): 60s ✅
Crash detection (working state): 180s (with heartbeat) ✅

User experience: "Near real-time" autonomous coordination
```

## Transient State Timing Guidance

### Expected Transition Times

**Normal scenario (typical case):**
```
T+0s:  Conductor sets state = 'review_approved'
T+4s:  Execution subagent detects change (avg 8s/2 = 4s)
T+5s:  Execution reads message, processes, sets state = 'working'

Total: ~5 seconds (normal case)
```

**Long-running step scenario (acceptable case):**
```
T+0s:  Execution starts expensive operation (30s duration)
T+0s:  Execution background subagent still checking every 8s
T+15s: Conductor completes review, sets state = 'review_approved'
T+23s: Execution subagent detects change (next check cycle)
T+30s: Execution finishes operation
T+31s: Execution reads message, processes, sets state = 'working'

Total: ~16 seconds (longer but acceptable)
```

**Crash scenario (timeout case):**
```
T+0s:  Conductor sets state = 'review_approved'
T+0s:  Execution crashes (OOM, bug, system issue)
T+60s: Stale state detector fires (timeout)

Total: 60 seconds (crash detected)
```

### Timeout Configuration Decision

**60-second timeout chosen because:**
- Normal transition: 5-10 seconds
- Long-running step: 15-30 seconds
- Timeout: 60 seconds = 2x worst-case normal + buffer
- Crash detection: 1-minute delay acceptable (not time-critical)

### Adaptation for Long-Running Steps

**If your tasks have very long steps (>30s between subagent checks):**

**Option A: Increase stale timeout**
```markdown
In conductor subagent prompt:
"STALE_STATE: >60 seconds" → "STALE_STATE: >120 seconds"

Rationale: Your execution steps take 45-60s between checks
Trade-off: Slower crash detection but fewer false positives
```

**Option B: Add heartbeat (recommended - see Task 4)**
```markdown
Update last_heartbeat every 30s within long-running steps.
Allows crash detection independent of transient state transitions.

Rationale: Better separation of concerns - heartbeat for crashes,
stale timeout for actual transient state issues
```

**Option C: Break long steps into smaller steps**
```markdown
# Instead of:
Step 5: Process 10,000 records (45 seconds)

# Write as:
Step 5a: Process records 1-5,000 (22 seconds)
Step 5b: Check subagent status
Step 5c: Process records 5,001-10,000 (22 seconds)

Benefit: Subagent checked mid-step, more responsive
Recommended: Best approach when possible
```

### Timing Categories

| Transition Type | Expected Duration | Timeout Setting | Rationale |
|-----------------|------------------|-----------------|-----------|
| **Fast transition** | 5-10s (typical) | 60s default | 6x expected = good safety margin |
| **Long step transition** | 15-30s | 60s default OR 120s | Within margin OR increase timeout |
| **Very long step** | 30-60s | 120s OR heartbeat | Need larger timeout OR better mechanism |
| **Extreme duration** | >60s | Heartbeat required | Timeout alone insufficient |

**Recommendation hierarchy:**
1. **Best:** Break into smaller steps (Option C)
2. **Good:** Add heartbeat (Option B)
3. **Acceptable:** Increase timeout (Option A)


#### Database Corruption

**Symptoms:**
- coordination_status and migration_tasks disagree
- Impossible state transitions
- Missing expected records

**Detection:**
```sql
-- Audit query
SELECT
  m.task_id,
  m.status as migration_status,
  c.state as coordination_state,
  CASE
    WHEN m.status = 'complete' AND c.state != 'complete' THEN 'MISMATCH'
    WHEN m.status = 'failed' AND c.state NOT IN ('error', 'exited') THEN 'MISMATCH'
    ELSE 'OK'
  END as consistency
FROM migration_tasks m
LEFT JOIN coordination_status c ON m.task_id = c.task_id
WHERE m.task_id LIKE 'task-%';
```

**Recovery:**
```sql
-- Trust migration_tasks as source of truth
-- Reconcile coordination_status
UPDATE coordination_status
SET state = (
  CASE
    WHEN (SELECT status FROM migration_tasks WHERE task_id = coordination_status.task_id) = 'complete'
      THEN 'complete'
    WHEN (SELECT status FROM migration_tasks WHERE task_id = coordination_status.task_id) = 'failed'
      THEN 'exited'
    WHEN (SELECT status FROM migration_tasks WHERE task_id = coordination_status.task_id) = 'in_progress'
      THEN 'working'
    ELSE 'error'
  END
)
WHERE task_id IN (SELECT task_id FROM [mismatched records]);
```

#### Partial Failures

**Scenario:** Some execution tasks complete, others fail

**Conductor handling:**
```python
# When subagent reports mixed states
completed = query("SELECT task_id FROM coordination_status WHERE state = 'complete'")
exited = query("SELECT task_id FROM coordination_status WHERE state = 'exited'")
in_progress = query("SELECT task_id FROM coordination_status WHERE state NOT IN ('complete', 'exited')")

if exited and (completed or in_progress):
    # Partial failure
    # Option A: Continue with successful tasks
    for task in completed:
        finalize_task(task)

    # Option B: Stop all tasks and escalate
    for task in in_progress:
        send_message(task, "STOP: Other task failed, halting all work")
    UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-00';

    # Decision depends on task dependencies
    # If tasks are independent: Option A
    # If tasks have dependencies: Option B
```

### Cross-Task Coordination

**Principle:** This pattern works best when tasks are TRULY independent. If tasks need to coordinate with each other (not just with conductor), that's a sign to reconsider task boundaries.

#### Handling Unavoidable Dependencies

**Scenario 1: File Conflicts (Multiple tasks updating same file)**

```markdown
**Problem:** task-03 and task-04 both need to update docs/README.md

**Design solution (preferred):**
- Split the file:
  - task-03 updates `docs/knowledge-base/testing/README.md`
  - task-04 updates `docs/knowledge-base/api/README.md`
  - No conflict - different files

**If truly unavoidable (last resort):**
```sql
-- Conductor serializes the conflicting writes
-- Step 1: Approve task-03 first
UPDATE coordination_status SET state = 'review_approved' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'task-00',
    'Approved. You may update docs/README.md'
);

-- Step 2: Wait for task-03 to complete its write
-- (Conductor monitors task-03 state)

-- Step 3: Only after task-03 completes, approve task-04
UPDATE coordination_status SET state = 'review_approved' WHERE task_id = 'task-04';
INSERT INTO task_messages VALUES (
    'task-04', 'task-00',
    'Approved. task-03 has finished with docs/README.md, you may update it now'
);
```

**Cost:** Adds conductor coordination overhead, reduces parallelism benefit. Better to redesign task boundaries.

**Scenario 2: Data Dependencies (task-04 needs task-03 output)**

```markdown
**Problem:** task-04 needs to process files created by task-03

**Design solution (preferred):**
- Don't use parallel pattern - use sequential execution
- Or ensure task-04 can work on placeholder/partial data from task-03

**If discovered mid-execution:**
```sql
-- task-04 detects missing prerequisite
UPDATE coordination_status SET state = 'error' WHERE task_id = 'task-04';
INSERT INTO task_messages VALUES (
    'task-04', 'exec-04',
    'ERROR: Missing prerequisite data from task-03.

Required: docs/knowledge-base/testing/ files
Current state: Directory empty
Cannot proceed until task-03 completes.

Recommendation: Wait for task-03, then retry task-04'
);

-- Conductor sees error, coordinates sequence
-- Checks task-03 status:
SELECT state FROM coordination_status WHERE task_id = 'task-03';
-- If task-03 still working: Tell task-04 to wait
-- If task-03 complete: Tell task-04 to retry

-- Option A: task-03 still in progress
UPDATE coordination_status SET state = 'fix_proposed' WHERE task_id = 'task-04';
INSERT INTO task_messages VALUES (
    'task-04', 'task-00',
    'FIX: task-03 is still in progress. Pause current work. Launch blocking subagent to monitor task-03 completion. Resume when task-03 state = complete.'
);

-- Option B: task-03 already complete
UPDATE coordination_status SET state = 'fix_proposed' WHERE task_id = 'task-04';
INSERT INTO task_messages VALUES (
    'task-04', 'task-00',
    'FIX: task-03 is now complete. Prerequisite data is available. Retry your current step.'
);
```

**Cost:** Execution session wastes time waiting. Better to redesign: make task-04 truly independent or run sequentially.

**Scenario 3: Information Sharing (task-03 discovers issue affecting task-04)**

```markdown
**Problem:** task-03 finds pattern conflict that task-04 should know about

**Solution: All communication routes through conductor (no direct task-to-task messaging)**

Step 1: task-03 sends FYI message to conductor
```sql
-- task-03 → conductor
INSERT INTO task_messages VALUES (
    'task-00', 'exec-03',  -- TO conductor FROM task-03
    'INFO FOR TASK-04: Found pattern conflict in source files.

Pattern "authenticate()" appears in 3 different forms:
- v1: auth.ts (deprecated 2023)
- v2: auth-v2.ts (current 2025)
- v3: new-auth.ts (experimental)

This may affect task-04 API extraction. Recommend task-04 checks which version to document.'
);
```

Step 2: Conductor forwards to task-04 when appropriate
```sql
-- Conductor → task-04
-- (Sent either proactively or during next review approval)
INSERT INTO task_messages VALUES (
    'task-04', 'task-00',  -- TO task-04 FROM conductor
    'FYI from task-03: Pattern conflict found with authenticate().

task-03 identified 3 versions. When you extract API patterns, verify which version is canonical for current documentation.

Details: [forwarded from task-03 message above]'
);

-- task-04's background subagent will detect this message
-- task-04 reads message, adjusts extraction approach
```

**No direct exec-03 → exec-04 messaging:**
- Keeps coordination simple (all through conductor)
- Conductor can filter/prioritize messages
- Conductor can decide if message is relevant
- Prevents execution sessions from overwhelming each other

#### When Tasks Aren't Truly Independent

**Signs tasks aren't independent:**
- Frequent "waiting for task-X" errors
- Conductor constantly coordinating sequencing
- Multiple file conflicts
- One task blocked by another regularly

**What to do:**
1. **Reconsider using this pattern** - Parallel orchestration may not be appropriate
2. **Fall back to sequential execution** - Simpler, more predictable
3. **Redesign task boundaries** - Make tasks truly independent:
   - Split conflicting files
   - Divide work by subdirectory (no overlaps)
   - Ensure each task has complete prerequisites at start

**When sequential is better:**
- Tasks have clear dependency chain: A → B → C
- Tasks share core resources (same files, same data)
- Coordination overhead > parallelism benefit

### Context Management

#### Context Budget Tracking

**Conductor session:**
```
Initial overhead: ~10k
  - System prompt: 4k
  - System tools: 15.8k (but only subset loaded)
  - Skills: 1.8k
  - Session setup: ~3k

Per monitoring cycle: ~1k
  - Subagent launch: ~500 tokens
  - Subagent result: ~200 tokens
  - State queries: ~200 tokens
  - Context overhead: ~100 tokens

Per review: ~5-10k (depends on work size)
  - Review request message: ~500 tokens
  - Read changed files: ~3-8k tokens
  - Analysis and decision: ~1-2k tokens
  - Response message: ~500 tokens

Total for full orchestration (3 tasks, 2 checkpoints each):
  - Setup: 10k
  - Monitoring (10 cycles): 10k
  - Reviews (6 checkpoints): 36k
  - Completion: 10k
  - Buffer: 5k
  - Total: ~71k (35% of 200k)
```

**Execution session:**
```
Initial overhead: ~10k
  - System prompt: 4k
  - System tools: 15.8k (subset)
  - Skills: 1.8k
  - Session setup: ~3k

Work phases: Varies by task
  - Read source files: ~20-40k
  - Implementation: ~10-30k
  - Testing: ~5-10k

Between-step checks: ~500 tokens each
  - TaskOutput call: ~100 tokens
  - Result processing: ~300 tokens
  - Relaunch subagent: ~100 tokens

Review wait: ~0 tokens (blocking subagent)
  - Main session paused
  - No context added during wait

Total for extraction task:
  - Setup: 10k
  - Work: 60k
  - Between-step (10 checks): 5k
  - Reviews (2 waits): 0k
  - Completion: 5k
  - Buffer: 5k
  - Total: ~85k (42% of 200k)
```

#### Checkpoint Decisions

**When to checkpoint (compact) session:**

**Conductor:**
- After completing 2-3 full review cycles
- When context reaches 160k (80% of 200k)
- Before final completion phase
- If many large files reviewed

**Execution:**
- After major phase completion (before final phase)
- When context reaches 160k (80%)
- If read many large source files
- Not needed if task completes quickly (<100k)

#### Context Budget Overrun Procedures

**When estimates are wrong and context fills faster than expected:**

**Scenario 1: Execution session hits 160k mid-task**

```markdown
### Emergency Checkpoint Mid-Task

If execution session reaches 160k (80% context) before task completion:

1. **Check current context:**
   ```bash
   /context
   # If >160k (80%), initiate emergency checkpoint
   ```

2. **Document current progress:**
   ```bash
   # Create resume state file
   cat > temp/task-XX-resume-state.md <<EOF
   # Task XX Resume State

   **Last completed step:** Step N
   **Current step status:** [In progress/Partially complete]
   **Pending work:** Steps N+1 through M

   **Critical context:**
   - [Key decisions made so far]
   - [Files created/modified]
   - [Patterns identified]

   **Resume instructions:**
   - Continue from Step N+1
   - Review commits: [SHAs]
   - Check files: [paths]
   EOF
   ```

3. **Commit all work-in-progress:**
   ```bash
   git add -A
   git commit -m "$(cat <<'EOF'
   task-XX: checkpoint at step N (context limit approaching)

   Completed: Steps 1-N
   Pending: Steps N+1 through M
   Resume state: temp/task-XX-resume-state.md

   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
   EOF
   )"
   ```

4. **Update coordination status:**
   ```sql
   INSERT INTO task_messages VALUES (
       'task-XX', 'exec-session-id',
       'Context limit approaching (160k at step N/M). Checkpointing for compact. Resume state saved to temp/task-XX-resume-state.md'
   );
   -- State remains 'working' - just checkpointing, not pausing
   ```

5. **Compact session** (user runs `/compact` or natural conversation continuation triggers it)

6. **After compact, resume:**
   ```bash
   # Reactivate hook
   bash tools/message-watcher/setup.sh --preset execution --task-id task-XX

   # Read resume state
   cat temp/task-XX-resume-state.md

   # Re-query database state
   sqlite3 coordination.db "SELECT state FROM coordination_status WHERE task_id='task-XX'"
   # Should be 'working'

   # Relaunch background subagent
   # (Launch same monitoring subagent as original initialization)

   # Continue from Step N+1
   ```
```

**Scenario 2: Conductor reviewing very large files (>50k tokens)**

```markdown
### Large File Review Strategy

**Option A: Chunked reading**
```bash
# Read first 1000 lines only
Read(file_path="docs/knowledge-base/testing/comprehensive-patterns.md", offset=1, limit=1000)
# Review this chunk
# If need more context: Read next 1000 lines
```

**Option B: Focus on changed sections**
```bash
# Get git diff for context of what changed
git diff [last-reviewed-commit]..HEAD docs/knowledge-base/testing/*.md

# Only read sections around changes
# Use line numbers from diff to target reading
```

**Option C: Request summary from execution**
- In review checkpoint instructions, require execution to provide:
  - Summary of changes (300-500 tokens)
  - Key files list with line ranges
  - Spot-check locations for conductor

**Option D: Defer full review**
- Partial review now (spot-check critical files)
- Approve conditionally
- Full review after conductor compacts (if needed)
```

**Scenario 3: Execution needs to read many large source files (>100k tokens total)**

```markdown
### Large Source Reading Strategy

**Before reading all files, plan reading approach:**

1. **Assess total content:**
   ```bash
   # Check file sizes
   wc -l docs_old/*.md
   # If total >5000 lines (~7k tokens each = 35k+ total), use chunked approach
   ```

2. **Prioritize essential content:**
   - Read table of contents / headers first (identify structure)
   - Read sections relevant to extraction target
   - Skip deprecated/archived sections (note their existence)

3. **Create summaries for large files:**
   ```bash
   # For 1000+ line files, create summary first
   cat > temp/task-XX-file-summary.md <<EOF
   # Source: docs_old/COMPREHENSIVE_GUIDE.md (1500 lines)

   ## Structure:
   - Lines 1-300: Introduction (can skip)
   - Lines 301-800: Core patterns (MUST READ)
   - Lines 801-1200: Examples (skim)
   - Lines 1201-1500: Deprecated (skip)

   ## Key patterns identified (from headers):
   - Pattern A (line 350)
   - Pattern B (line 520)
   ...
   EOF

   # Then read only essential sections with offset/limit
   ```

4. **Budget for mid-task checkpoint:**
   - If all files truly needed: Accept that checkpoint will be required
   - Plan checkpoint after reading phase, before extraction phase
   - This is acceptable for complex tasks
```

**How to checkpoint:**
```bash
# Check current context
/context

# If >80%, compact
# This preserves conversation summary, discards detailed execution
# Session can continue with fresh context
```

**Post-compact considerations:**
- Hook state persists (file-based)
- Database state persists (SQLite)
- Subagents must be relaunched (they don't survive compact)
- Need to re-query current task state to resume

#### Conductor Resume After Compact

**After conductor session compacts, resume with these steps:**

```markdown
### Conductor Resume Procedure

**Step 1: Reactivate hook**
```bash
bash tools/message-watcher/setup.sh --preset orchestration
# Output: 🔄 Message watcher activated
#         Monitoring: task-00
#         Exit criteria: ["exit_requested", "complete"]
```

**Step 2: Query current state of ALL execution tasks**
```sql
SELECT task_id, state FROM coordination_status
WHERE task_id LIKE 'task-%' AND task_id != 'task-00'
ORDER BY task_id ASC;

-- Example output:
-- task-03 | complete
-- task-04 | needs_review
-- task-05 | working
```

**Step 3: Determine what needs immediate attention**

Check the query results and prioritize:

```python
# Priority order (handle in this sequence):
1. Any tasks in `exited`? → Escalate immediately (terminal error)
2. Any tasks in `error`? → Process error recovery
3. Any tasks in `needs_review`? → Process reviews
4. All tasks in `working` or `complete`? → Resume monitoring
```

**Examples:**

```markdown
**Scenario A: Task needs review**
- Query shows: task-04 = needs_review
- Action: Process task-04 review immediately
- Don't relaunch monitoring subagent yet - handle pending review first
- After review complete, relaunch monitoring

**Scenario B: All tasks working or complete**
- Query shows: All tasks in 'working' or 'complete'
- Action: Relaunch monitoring subagent immediately
- No pending work to process
```

**Step 4: Handle pending items (if any)**

```sql
-- If task was mid-review when compact happened:
-- Execution is STILL WAITING (blocking subagent running)
-- Execution state is STILL 'needs_review'
-- Conductor will see it in query above

-- Read review request
SELECT message FROM task_messages
WHERE task_id = 'task-04'
  AND message LIKE '%REVIEW REQUEST%'
ORDER BY timestamp DESC LIMIT 1;

-- Process review as normal
-- (Execution's blocking subagent will detect approval and exit)
```

**Key insight:** The compact didn't affect execution sessions - they're still running and waiting. Conductor just lost conversation history, but database state is perfect source of truth.

**Step 5: Relaunch monitoring subagent**

```python
# After handling any immediate pending items, resume monitoring
result = Task(
    description="Watch for execution attention needed",
    prompt="""
Watch coordination_status for execution tasks (task-03, task-04, task-05).
Check every 12 seconds. Max 600 iterations (2 hours).

Exit when:
1. REVIEW NEEDED - Any task state = 'needs_review'
   → Return: "REVIEW_NEEDED: [task-ids]"
2. ERROR DETECTED - Any task state = 'error'
   → Return: "ERROR_DETECTED: [task-ids]"
3. ALL COMPLETE - All tasks state = 'complete'
   → Return: "ALL_COMPLETE"
4. CRITICAL EXIT - Any task state = 'exited'
   → Return: "ERROR_DETECTED: [task-id] (EXITED)"

Query:
SELECT task_id, state FROM coordination_status
WHERE task_id IN ('task-03', 'task-04', 'task-05')
  AND state NOT IN ('working', 'complete', 'review_approved', 'review_failed', 'fix_proposed');
""",
    subagent_type="general-purpose",
    run_in_background=false  # BLOCKING
)
```

**Step 6: Continue orchestration from current state**

- Process whatever subagent reports (same as before compact)
- Database state is authoritative
- Conversation history compacted, but that's fine
- All coordination happens via database, not conversation

**What was lost in compact:**
- ❌ Detailed review analysis from earlier checkpoints
- ❌ Error investigation notes from earlier retries
- ❌ Conversation context about decisions made

**What was preserved:**
- ✅ All database state (coordination_status, migration_tasks, task_messages)
- ✅ All file changes (git commits)
- ✅ All reports (docs/implementation/reports/)
- ✅ Hook state (.claude/message-watcher.local.md)

**Recovery strategy:** If conductor needs context from before compact, read from:
- Database: `SELECT * FROM task_messages WHERE task_id='task-XX' ORDER BY timestamp DESC LIMIT 10`
- Reports: `cat docs/implementation/reports/task-XX-checkpoint-N.md`
- Git history: `git log --oneline --grep="task-XX"`
```

### Parsing Partial Completion Messages (Concrete Implementation)

**When execution reports partial completion, conductor must parse the message to extract completion percentage and make decision.**

#### Step 1: Read Partial Completion Message (SQL)

```sql
-- Query latest partial completion message
SELECT message, created_at
FROM task_messages
WHERE task_id = 'task-03'
  AND from_session = 'task-03'
  AND message LIKE '%PARTIAL COMPLETION%'
ORDER BY created_at DESC
LIMIT 1;

-- Example result:
-- message: "PARTIAL COMPLETION: Completed 4 of 6 steps (66%).
--           Completed work: Step 1 (extraction), Step 2 (RAG ingestion),
--           Step 3 (metadata), Step 4 (validation).
--           Blocked work: Step 5 (final review), Step 6 (commit)
--           Reason: External API timeout, cannot proceed."
-- created_at: 2026-02-05 14:32:18
```

#### Step 2: Extract Percentage with Bash Regex

```bash
#!/bin/bash

# Read message from SQL query result
message="PARTIAL COMPLETION: Completed 4 of 6 steps (66%)"

# Extract percentage using grep with Perl regex
percentage=$(echo "$message" | grep -oP '\(\K[0-9]+(?=%\))')
echo "Extracted percentage: $percentage"  # Output: 66

# Extract completed count and total count
completed=$(echo "$message" | grep -oP 'Completed \K[0-9]+(?= of)')
total=$(echo "$message" | grep -oP 'of \K[0-9]+(?= steps)')
echo "Completed: $completed of $total"  # Output: Completed 4 of 6

# Alternative: Extract using sed
percentage_alt=$(echo "$message" | sed -n 's/.*(\([0-9]\+\)%).*/\1/p')
echo "Alternative extraction: $percentage_alt"  # Output: 66
```

**Error handling:**

```bash
# Validate extraction succeeded
if [ -z "$percentage" ]; then
    echo "ERROR: Could not extract percentage from message"
    echo "Message format invalid: $message"
    exit 1
fi

# Validate percentage is in valid range
if [ "$percentage" -lt 0 ] || [ "$percentage" -gt 100 ]; then
    echo "ERROR: Invalid percentage: $percentage (must be 0-100)"
    exit 1
fi

echo "Valid percentage extracted: $percentage%"
```

#### Step 3: Decision Logic for Conductor

```bash
#!/bin/bash

# Conductor decision matrix for partial completion
percentage=66  # From Step 2 extraction

if [ "$percentage" -ge 80 ]; then
    # High completion (≥80%) - Accept partial
    decision="accept_partial"
    action="Mark task complete, document partial scope"

elif [ "$percentage" -ge 50 ]; then
    # Medium completion (50-79%) - Review required
    decision="review_partial"
    action="Conductor reviews completed work, decides if acceptable"

else
    # Low completion (<50%) - Reject partial
    decision="reject_partial"
    action="Request execution continue or report blocker"
fi

echo "Decision: $decision"
echo "Action: $action"

# Apply decision to coordination database
case "$decision" in
    accept_partial)
        sqlite3 coordination.db <<SQL
UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-03';
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'task-00',
    'PARTIAL COMPLETION ACCEPTED (${percentage}%):
    Sufficient work completed. Remaining steps deferred.
    Mark task complete with partial scope documented.');
SQL
        ;;

    review_partial)
        # Conductor reads completed work artifacts
        echo "Conductor reviewing partial completion:"
        sqlite3 coordination.db "SELECT message FROM task_messages WHERE task_id = 'task-03' ORDER BY created_at DESC LIMIT 5;"

        # Read completed files to assess quality
        # [Conductor-specific review logic here]

        # After review, choose accept_partial or reject_partial
        ;;

    reject_partial)
        sqlite3 coordination.db <<SQL
UPDATE coordination_status SET state = 'error' WHERE task_id = 'task-03';
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'task-00',
    'PARTIAL COMPLETION REJECTED (${percentage}%):
    Insufficient progress. Please complete blocked steps or report unresolvable blocker.
    Required: At least 50% completion or clear external blocker documentation.');
SQL
        ;;
esac
```

#### Step 4: Execution Reading Conductor Decision

```bash
#!/bin/bash
# Execution session reads conductor decision

# Read latest message from conductor
message=$(sqlite3 coordination.db "SELECT message FROM task_messages WHERE task_id = 'task-03' AND from_session = 'task-00' ORDER BY created_at DESC LIMIT 1;")

if echo "$message" | grep -q "PARTIAL COMPLETION ACCEPTED"; then
    echo "Conductor accepted partial completion. Wrapping up task."
    # Set state to complete
    sqlite3 coordination.db "UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-03';"
    exit 0

elif echo "$message" | grep -q "PARTIAL COMPLETION REJECTED"; then
    echo "Conductor rejected partial completion. Continuing work or escalating blocker."
    # Read reason from message
    # Decide: continue work OR request user intervention

else
    echo "No partial completion decision yet. Continuing to wait."
fi
```

### Partial Completion Decision Matrix

**Conductor uses this matrix to decide on partial completion requests:**

| Completion % | Quality | Blocker | Decision | Rationale |
|--------------|---------|---------|----------|-----------|
| ≥80% | Good | Any | **Accept** | Substantial work complete, high quality |
| ≥80% | Poor | Any | **Review** | High percentage but quality concerns |
| 50-79% | Good | Valid | **Accept** | Meaningful progress, external blocker |
| 50-79% | Good | None | **Review** | Should have completed more |
| 50-79% | Poor | Any | **Reject** | Insufficient quality and progress |
| <50% | Good | Critical | **Review** | Low progress but valid blocker |
| <50% | Good | Minor | **Reject** | Should have overcome minor blocker |
| <50% | Poor | Any | **Reject** | Insufficient progress and quality |

**Decision definitions:**
- **Accept:** Mark task as complete with partial scope documented
- **Review:** Conductor reads artifacts, makes final decision
- **Reject:** Execution must continue or escalate blocker to user

**Quality assessment criteria:**
- **Good:** Tests pass, code follows standards, deliverables functional
- **Poor:** Tests fail, code quality issues, incomplete deliverables

**Blocker validity:**
- **Valid:** External dependency, missing file, API timeout
- **Critical:** Cannot proceed without user input
- **Minor:** Could work around with more effort

**Example decision scenarios:**

**Scenario 1: Accept (85%, Good, External API timeout)**
```sql
-- Decision: Accept
-- Rationale: Substantial work complete (85%), high quality, valid external blocker
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'task-00',
    'PARTIAL COMPLETION ACCEPTED (85%): High quality work completed.
    External API timeout is valid blocker. Mark as complete, document deferred work.');
```

**Scenario 2: Review (72%, Good, No blocker)**
```sql
-- Decision: Review
-- Rationale: Medium progress (72%), good quality, but no clear blocker
-- Conductor reads artifacts to decide if acceptable
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'task-00',
    'REVIEWING PARTIAL COMPLETION (72%): Quality appears good, but investigating why remaining 28% not completed.');
```

**Scenario 3: Reject (45%, Poor, Minor issue)**
```sql
-- Decision: Reject
-- Rationale: Low progress (45%), poor quality, blocker seems surmountable
INSERT INTO task_messages (task_id, from_session, message) VALUES (
    'task-03', 'task-00',
    'PARTIAL COMPLETION REJECTED (45%): Insufficient progress and quality issues.
    Please resolve quality concerns and attempt to work around blocker.');
```

### Partial Completion Scenarios

**When execution session cannot reach 100% completion, document progress and escalate appropriately.**

#### Scenario 1: Max Iterations Reached at 80% Complete

```sql
-- Session approaching max iterations (hook will allow exit soon)
-- Hook has blocked 480/500 times, only 20 iterations remaining

-- Document what's complete and what's pending
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-03';

INSERT INTO task_messages VALUES (
    'task-03', 'exec-03',
    'PARTIAL COMPLETION: Reached max iterations at step 8/10 (80% complete).

COMPLETED WORK:
- Steps 1-8 fully finished
- Commits: abc123, def456, ghi789 (see git log --grep="task-03")
- Files created: 12 patterns in docs/knowledge-base/testing/
- Tests: 15/15 passing
- Checkpoint reports: task-03-checkpoint-1.md, task-03-checkpoint-2.md

PENDING WORK:
- Step 9: Final cross-reference verification
- Step 10: Completion report generation

REASON FOR PARTIAL COMPLETION:
- Context limit exceeded estimates (165k at step 8)
- Complex source files required more analysis than budgeted
- Max iterations safety triggered before full completion

RECOMMENDATION:
1. Review completed work (steps 1-8) for quality
2. If acceptable: Launch new session for steps 9-10 only
3. New session can resume from current git commits
4. Estimated remaining work: 30-45 minutes'
);

-- Session will exit (hook allows exit_requested)
```

**Conductor Response to Partial Completion:**

```sql
-- Conductor sees exit_requested from task-03
-- Reads partial completion message

-- Option A: Accept partial completion
UPDATE migration_tasks
SET status = 'partial_complete',
    completed_at = datetime('now'),
    report_path = 'docs/implementation/reports/task-03-partial-report.md'
WHERE task_id = 'task-03';

INSERT INTO task_messages VALUES (
    'task-00', 'task-00',
    'Task-03 partial completion ACCEPTED.

Completed: Steps 1-8 (80%) - quality verified
Pending: Steps 9-10 (20%) - deferred to follow-up task

Will proceed with other tasks. User can relaunch for completion later if needed.'
);

-- Option B: Relaunch for remainder
-- (User manually launches new execution session with modified instruction)
-- Instruction: "Complete task-03 steps 9-10 only, starting from current state"
```

#### Scenario 2: Requirements Changed Mid-Task

```sql
-- Execution discovers at step 6 that remaining work is obsolete

UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-04';

INSERT INTO task_messages VALUES (
    'task-04', 'exec-04',
    'REQUIREMENT CHANGE DETECTED: Partial completion recommended.

COMPLETED WORK:
- Steps 1-5 successfully finished
- Extracted 8 API patterns from docs_old/API_GUIDE_V1.md
- All patterns validated and committed

DISCOVERY:
Found docs_old/API_GUIDE_V2.md (dated 2025-11) that supersedes V1.
Remaining steps 6-10 were based on V1 approach.

ANALYSIS:
- V2 guide has completely different architecture
- Original task scope assumed only V1 existed
- V2 patterns should be extracted differently

RECOMMENDATION:
Option A: Accept steps 1-5 as partial completion (V1 patterns historical reference)
Option B: Redefine task scope to include V2 patterns (requires new instruction)
Option C: Create new task-04-v2 for V2 patterns (keep task-04 as V1 only)

User decision required - cannot proceed autonomously.'
);
```

#### Scenario 3: Blocked by External Dependency

```sql
-- Execution reaches step where cannot proceed without external input

UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-05';

INSERT INTO task_messages VALUES (
    'task-05', 'exec-05',
    'PARTIAL COMPLETION: Blocked by missing prerequisite.

COMPLETED WORK:
- Steps 1-6 (60%) fully finished
- Database patterns extracted from 4 of 6 source files

BLOCKING ISSUE:
- Step 7 requires docs_old/DB_MIGRATION_GUIDE.md
- File referenced in IMPLEMENTATION_PLAN.md but does NOT exist
- Cannot complete extraction without this source file

ATTEMPTED:
- Checked all docs_old/ subdirectories: file not found
- Searched for alternative filenames: no matches
- Cannot infer content from other sources

OPTIONS:
1. Accept partial completion (4/6 files extracted)
2. User locates missing file, provides path, relaunch
3. Modify scope to skip migration patterns (only extract from 4 available files)

User decision required.'
);
```

#### Conductor Handling of Partial Completion

**When conductor receives partial completion notification:**

**Step 1: Read partial completion message**
- Understand what's done, what's pending, why partial

**Step 2: Verify completed work quality:**
```bash
# Review commits from completed steps
git log --grep="task-03" --oneline

# Check test results
[test command results from checkpoint reports]

# Spot-check deliverables
ls -la docs/knowledge-base/testing/

# Quality may be perfect even if incomplete
```

**Step 3: Assess situation:**
```python
if completion_percentage >= 80% and quality_good:
    # Substantial work done
    recommendation = "Accept partial, optional follow-up for remainder"
elif completion_percentage >= 50% and reason_valid:
    # Meaningful progress
    recommendation = "Accept partial, plan follow-up task"
elif completion_percentage < 50%:
    # Minimal progress
    recommendation = "Investigate - may need different approach"
```

**Step 4: Update migration_tasks:**
```sql
UPDATE migration_tasks
SET status = 'partial_complete',
    completed_at = datetime('now'),
    report_path = 'docs/implementation/reports/task-XX-partial-report.md'
WHERE task_id = 'task-XX';
```

**Step 5: Document in conductor report:**
```markdown
### Task-03: Partial Completion

**Status:** 80% complete (8/10 steps)
**Quality:** Good (all deliverables meet standards)
**Reason:** Max iterations (context overrun)
**Decision:** Accepted as partial
**Follow-up:** Optional task-03-continuation for steps 9-10
```

**Step 6: Decide on coordination:**
- Continue with other tasks (task-04, task-05)?
- Pause all tasks pending user review?
- Depends on: Is partial completion blocking for other tasks?

#### Key Principles for Partial Completion

**✅ Partial completion is NOT a failure**
- It's an intermediate state requiring user decision
- Work completed may be valuable even if task incomplete
- Better to preserve partial work than force completion

**✅ Document thoroughly**
- What WAS completed (with evidence: commits, files, tests)
- What is PENDING (specific steps)
- WHY partial (root cause: context, requirements, blocking)
- RECOMMENDATION (accept, relaunch, redefine)

**✅ Quality over quantity**
- 60% complete with high quality > 100% complete with poor quality
- Conductor should verify completed portion meets standards
- Partial completion with perfect quality is success

**❌ Don't conflate with error**
- Partial completion ≠ failure
- state = `exit_requested` (needs user decision)
- NOT state = `error` (technical problem)
- NOT state = `exited` (terminal failure)

---

## Part 5: Quality & Verification

### Verification Requirements

#### Checkpoint Verification

**Before requesting review:**

**Code changes:**
```bash
# 1. All changes committed
git status
# Expected: "nothing to commit, working tree clean"

# 2. Tests passing (if applicable)
[test command]
# Expected: All tests pass

# 3. No build errors
[build command]
# Expected: Clean build

# 4. Changed files list generated
git diff [last-reviewed-commit]..HEAD --name-only > /tmp/changed-files.txt
```

**Documentation changes:**
```bash
# 1. Metadata complete (if RAG files)
for file in docs/knowledge-base/[category]/*.md; do
    grep -q "<!-- metadata" "$file" || echo "Missing metadata: $file"
done
# Expected: No output (all files have metadata)

# 2. Cross-references valid
# Check that all referenced files exist
grep -h "\`[a-z-]*/.*\.md\`" docs/knowledge-base/**/*.md | while read ref; do
    file=$(echo "$ref" | tr -d '`')
    test -f "docs/knowledge-base/$file" || echo "Broken reference: $ref"
done
# Expected: No output

# 3. No loose files in docs/ root
ls -1 docs/*.md | grep -v README.md
# Expected: Empty (only README.md in root)
```

**Proposal verification (if proposal-first workflow):**

#### When Proposals Are Mandatory vs Optional

**Proposal-first workflow decision matrix:**

| Task Characteristic | Use Proposals? | Rationale |
|---------------------|----------------|-----------|
| **RAG ingestion involved** | ✅ MANDATORY | Prevents pollution - rejected files never indexed |
| **Creating 10+ files** | ✅ RECOMMENDED | Easy to review in batch, easy to rollback |
| **Creating files in shared location** | ✅ RECOMMENDED | Review before polluting common directory |
| **Modifying existing files** | ⚠️ OPTIONAL | Can edit in place, git tracks changes anyway |
| **Exploratory/uncertain output** | ✅ RECOMMENDED | Iteration easier with proposals |
| **High confidence in quality** | ⚠️ OPTIONAL | Low rejection risk |
| **Non-RAG documentation** | ⚠️ OPTIONAL | Pollution not a concern |
| **Single file creation** | ⚠️ OPTIONAL | Overhead not justified |
| **Code changes** | ⚠️ OPTIONAL | Git history sufficient |

**For the documentation extraction base case (tasks 03-06):**
```markdown
**All characteristics apply:**
- ✅ RAG ingestion involved
- ✅ Creating 10+ files per task
- ✅ Uncertain output (pattern extraction)

**Therefore: Proposals MANDATORY for tasks 03-06**

Benefits:
1. Prevents bad patterns from being indexed to RAG
2. Easy to iterate on extraction without RAG pollution
3. Conductor can review entire batch before ingestion
4. Simple rollback (delete proposal directory)
5. Clear separation: proposals = draft, docs/ = approved
```

**How to implement proposal-first workflow:**

```markdown
### Execution Session: Create Proposals

**Step N: Create Extraction Proposals**
```bash
# Create proposal directory for this task
mkdir -p docs/implementation/proposals/rag-files/task-03

# Extract patterns to proposal location (NOT final location)
for pattern in [patterns]; do
    create_file "docs/implementation/proposals/rag-files/task-03/${pattern}.md"
done

# List created proposals
ls -1 docs/implementation/proposals/rag-files/task-03/*.md > temp/task-03-proposals.txt
proposal_count=$(wc -l < temp/task-03-proposals.txt)
echo "Created $proposal_count proposal files"
```

**Step N+1: Request Proposal Review**
```sql
INSERT INTO task_messages VALUES (
    'task-03', 'exec-session-id',
    'REVIEW REQUEST: Extraction proposals ready

Proposal location: docs/implementation/proposals/rag-files/task-03/
File count: 12 files
File list: temp/task-03-proposals.txt

Review focus: Pattern accuracy, metadata completeness, no duplicates

If approved: Will move to docs/knowledge-base/testing/ and ingest to RAG'
);
UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-03';
```

### Conductor: Review Proposals

**Review Process:**
```bash
# Read proposal files (not final files - they don't exist yet)
ls docs/implementation/proposals/rag-files/task-03/*.md

# Spot check quality
# Check metadata completeness
# Verify no duplicates

# Decision: Approve or Reject
```

**If Approved:**
```sql
UPDATE coordination_status SET state = 'review_approved' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'task-00',
    'Proposals APPROVED. Proceed to move files to final location and ingest to RAG.'
);
```

**If Rejected:**
```sql
UPDATE coordination_status SET state = 'review_failed' WHERE task_id = 'task-03';
INSERT INTO task_messages VALUES (
    'task-03', 'task-00',
    'Proposals REJECTED. Issues found:
    1. Pattern "X" has incomplete metadata
    2. Duplicates detected: pattern-a.md and pattern-a-v2.md
    3. Missing source attribution in 3 files

    Fix these issues in proposals/ and re-request review. DO NOT move to final location yet.'
);
```

### Execution Session: Move Approved Proposals

**Step N+2: Extract Approved RAG Files (Only After Conductor Approves Proposals)**
```bash
# Received approval - extract RAG content from proposals to final location
for proposal in docs/implementation/proposals/task-03-rag-*.md; do
    # Extract content between <!-- BEGIN RAG FILE --> and <!-- END RAG FILE -->
    # Determine target filename from frontmatter (target_filename field)
    # Write to docs/knowledge-base/testing/[target_filename]

    # Example (manual):
    # grep -A999 "<!-- BEGIN RAG FILE -->" "$proposal" | grep -B999 "<!-- END RAG FILE -->" | sed '1d;$d' > "docs/knowledge-base/testing/pattern.md"
done

# Verify files created
ls docs/knowledge-base/testing/*.md | wc -l
# Expected: Number matching approved proposals

# Track files for ingestion
ls -1 docs/knowledge-base/testing/*.md > temp/task-03-rag-files.txt
```
```

**RAG Proposal Workflow is MANDATORY for Knowledge-Base Extraction:**

```markdown
**When RAG proposals ARE required:**
- Creating RAG files in `docs/knowledge-base/` (prevents duplication, enables KB overlap checking)
- Extracting patterns to knowledge base (requires conductor review before ingestion)
- Any documentation that will be ingested into local-rag MCP

**RAG Proposal Workflow:**
1. Create proposals in `docs/implementation/proposals/[task-id]-rag-*.md` (one per RAG file)
2. Each proposal contains: frontmatter, reasoning, RAG Match List (KB overlap check), RAG file content with delimiters
3. Pre-screen with KB: `query_documents("[topic]", limit=10)` at 0.4 threshold
4. Request conductor review (needs_review state)
5. After approval: extract RAG content to `docs/knowledge-base/[category]/`
6. Ingest with `tools/bulk-ingest-rag.sh`

**For non-RAG tasks** (backend refactoring, component implementation):
1. Can create files directly in final location (no proposal step)
2. Still require review checkpoint if changes need approval
3. Follow same verification and commit patterns
```

```bash
# 1. All proposals in correct location
ls docs/implementation/proposals/[proposal-type]/task-XX/
# Expected: List of proposal files

# 2. No files in final destination yet
ls docs/knowledge-base/[category]/*.md
# Expected: Empty or only pre-existing files

# 3. Proposal count matches expectation
proposal_count=$(ls docs/implementation/proposals/[proposal-type]/task-XX/*.md | wc -l)
# Compare to expected count from extraction analysis
```

#### Final Verification

**Before marking complete:**

**All deliverables present:**
```bash
# 1. Final files in correct location
test -d docs/knowledge-base/[category]
ls docs/knowledge-base/[category]/*.md | wc -l
# Expected: [expected count]

# 2. Completion report exists
test -f docs/implementation/reports/task-XX-report.md
# Expected: Exit code 0

# 3. All temporary files cleaned
find tmp/ -name "*task-XX*" -type f
# Expected: Only tracking files (if any)

# 4. Proposals cleaned (if used)
find docs/implementation/proposals/ -name "*task-XX*" -type f
# Expected: Empty (proposals moved to final location)
```

**Quality checks:**
```bash
# 1. No duplicate files
# Check for files with same content hash
find docs/knowledge-base/ -type f -name "*.md" -exec md5sum {} \; | \
    sort | uniq -w32 -D
# Expected: Empty (no duplicates)

# 2. All metadata fields present
for file in docs/knowledge-base/[category]/*.md; do
    for field in extracted source category; do
        grep -q "$field:" "$file" || echo "Missing $field in $file"
    done
done
# Expected: No output

# 3. Git status clean
git status
# Expected: "nothing to commit, working tree clean"

# 4. Commit count reasonable
git log --oneline --grep="task-XX" | wc -l
# Expected: 3-10 commits (not 100s of micro-commits)
```

**Integration checks (if applicable):**
```bash
# 1. RAG ingestion successful (if RAG task)
[query RAG to verify files indexed]

# 2. Cross-references between tasks valid
# If task-XX references files created by task-YY
for ref in [list of cross-task references]; do
    test -f "docs/knowledge-base/$ref" || echo "Missing: $ref"
done

# 3. No conflicts with other tasks
# Check for files with same name in different task outputs
find docs/knowledge-base/ -type f -name "*.md" -exec basename {} \; | \
    sort | uniq -d
# Expected: Empty (no filename conflicts)
```

### Completion Criteria

#### Execution Session Completion

**Session can mark complete when:**

**All work finished:**
- [ ] All steps in instruction completed
- [ ] All deliverables created and in correct location
- [ ] All temporary work cleaned up
- [ ] All proposals processed (if proposal-first workflow)

<!-- 🚩 FLAG 14: ADD - Partial Completion Handling

TODO: Add subsection "Handling Partial Completion":

**When session cannot reach 100% completion:**

**Scenario: Max iterations reached at 80% complete**
```sql
-- Session approaching max iterations (hook will allow exit soon)
-- Document what's complete and what's pending
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-XX';
INSERT INTO task_messages VALUES (
    'task-XX', 'exec-session-id',
    'PARTIAL COMPLETION: Reached max iterations at step 8/10 (80% complete).
    COMPLETED: Steps 1-8 (see commits abc123-def456)
    PENDING: Steps 9-10 (final verification and report generation)
    REASON: Context limit + complex source files exceeded estimates
    RECOMMENDATION: Review completed work, then relaunch for steps 9-10 with updated context budget'
);
-- Session exits, user reviews and decides: accept partial or relaunch
```

**Scenario: Requirements changed mid-task**
```sql
-- Execution discovers at step 6 that steps 7-10 are now obsolete
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-XX';
INSERT INTO task_messages VALUES (
    'task-XX', 'exec-session-id',
    'REQUIREMENT CHANGE: Steps 1-5 completed successfully.
    Steps 6-10 obsolete based on new information discovered in source files.
    RECOMMENDATION: User should review completed work and decide:
    - Accept steps 1-5 as partial completion OR
    - Redefine steps 6-10 with updated requirements'
);
```

**Conductor handling of partial completion:**
- Review what WAS completed (quality check on finished portions)
- Decide: Accept partial, relaunch for remainder, or abandon task
- Update migration_tasks with partial completion note
- Document in conductor completion report

**Key principle:** Partial completion is NOT a failure - it's an intermediate state requiring user decision.
-->

**All verification passed:**
- [ ] Final verification checks passed
- [ ] Tests passing (if applicable)
- [ ] No build errors
- [ ] Git status clean

**All reviews approved:**
- [ ] Checkpoint reviews approved (if any)
- [ ] Final review approved
- [ ] All feedback addressed
- [ ] No pending rejections

**Completion report generated:**
- [ ] Report at docs/implementation/reports/task-XX-report.md
- [ ] Report includes all required sections
- [ ] Report references all deliverables
- [ ] Report documents any issues/learnings

**Database updated:**
```sql
-- Mark complete in both tables
UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-XX';
UPDATE migration_tasks
SET status = 'complete',
    completed_at = datetime('now'),
    report_path = 'docs/implementation/reports/task-XX-report.md'
WHERE task_id = 'task-XX';

-- Final completion message
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-XX', '[session-id]',
'Task complete. All verification passed. Report at docs/implementation/reports/task-XX-report.md');
```

**Hook allows exit:**
- State = 'complete' matches execution preset exit criteria
- Session will exit when user submits next message or closes terminal

#### Handling Partial Completion

**When execution session cannot reach 100% completion, document progress and escalate appropriately.**

See Part 4 (Error & Edge Cases) for detailed partial completion procedures including:
- Scenario 1: Max iterations reached at 80% complete
- Scenario 2: Requirements changed mid-task
- Scenario 3: Blocked by external dependency
- Conductor response options
- Quality assessment of partial work
- Documentation requirements

**Quick reference - Execution session partial completion:**
```sql
UPDATE coordination_status SET state = 'exit_requested' WHERE task_id = 'task-XX';
INSERT INTO task_messages VALUES (
    'task-XX', 'exec-session-id',
    'PARTIAL COMPLETION: [percentage]% complete at step X/Y.
    COMPLETED: [what was finished with evidence]
    PENDING: [what remains]
    REASON: [why partial: context/requirements/blocking]
    RECOMMENDATION: [accept/relaunch/redefine]'
);
```

**Key principle:** Partial completion is NOT a failure - it's an intermediate state requiring user decision.

---

####  Conductor Completion

**Conductor can mark complete when:**

**All execution tasks complete:**
- [ ] All execution tasks state = 'complete' or 'exited'
- [ ] All 'exited' tasks reviewed and documented
- [ ] Decision made on partial failures (if any)

**All reviews completed:**
- [ ] All review requests processed
- [ ] All approvals/rejections sent
- [ ] No pending review requests

**Final verification:**
- [ ] Conductor completion report generated
- [ ] All execution reports reviewed
- [ ] Loose file check passed (docs/ clean)
- [ ] Integration verified (if applicable)

**Database updated:**
```sql
-- Mark complete
UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-00';
UPDATE migration_tasks
SET status = 'complete',
    completed_at = datetime('now'),
    report_path = 'docs/implementation/reports/task-00-report.md'
WHERE task_id = 'task-00';
```

**User notified:**
- [ ] Completion message output to user
- [ ] Summary of all tasks (complete/failed)
- [ ] Next steps documented (if any)

### Completion Report Requirements

#### Execution Task Report Template

```markdown
# Task XX Completion Report

**Task ID:** task-XX
**Task Name:** [Task name]
**Session:** [session-id]
**Started:** [timestamp]
**Completed:** [timestamp]
**Duration:** [duration]

---

## Summary

[1-2 sentence overview of what was accomplished]

---

## Work Completed

**Deliverables created:**
- [List of files created with paths]
- [Count and locations]

**Changes made:**
- [List of significant changes]
- [Files modified]

**Features implemented:**
- [If applicable]

---

## Coordination

**Review checkpoints:**
- Checkpoint 1: Requested at [time], approved at [time]
  - Feedback: [Summary of conductor feedback]
- Checkpoint 2: Requested at [time], approved at [time]
  - Feedback: [Summary]

**Messages received:**
- [Count] messages from conductor
- [Summary of critical messages]

**Retries:**
- [None if no errors]
- [If errors: Retry 1-N with descriptions]

---

## Verification Results

**Tests:**
- Total: [count]
- Passing: [count]
- Failing: 0
- [Test output location]

**Quality checks:**
- Metadata complete: ✅ / ❌
- Cross-references valid: ✅ / ❌
- No duplicates: ✅ / ❌
- Git clean: ✅ / ❌

**Files created:**
- docs/knowledge-base/[category]/: [count] files
- docs/implementation/reports/: [count] files
- [Other locations]

**Commits:**
- Total commits: [count]
- Commit SHAs: [list]
- Commit messages follow format: ✅ / ❌

---

## Issues Encountered

**Issues and resolutions:**
- [List each issue]
- [How it was resolved]
- [Time impact]

**Deviations from plan:**
- [None if no deviations]
- [If deviations: describe and justify]

---

## Learnings

**What went well:**
- [Positive learnings]
- [Patterns that worked]

**What could be improved:**
- [Areas for improvement]
- [Suggested optimizations]

**Insights for future tasks:**
- [Applicable to other tasks]
- [Process improvements]

---

## Artifacts

**Code:**
- Git commits: [SHAs]
- Changed files: [count]
- Lines added: [count]
- Lines removed: [count]

**Documentation:**
- RAG files: [count and paths]
- Reports: [list]

**Temporary files:**
- [List files in tmp/ related to this task]
- [Note: can be deleted after conductor review]

---

## Status

✅ **COMPLETE** - All verification passed, all deliverables created

**Next steps:**
- [If any follow-up needed]
- [Otherwise: None - task complete]
```

#### Conductor Report Template

```markdown
# Conductor Completion Report

**Task ID:** task-00
**Project:** [Project name]
**Session:** [session-id]
**Started:** [timestamp]
**Completed:** [timestamp]
**Duration:** [duration]

---

## Summary

[Brief overview of orchestration: how many tasks, what accomplished]

---

## Execution Tasks Status

| Task ID | Status | Duration | Review Checkpoints | Retries |
|---------|--------|----------|-------------------|---------|
| task-01 | ✅ Complete | 2h 15m | 2 | 0 |
| task-02 | ✅ Complete | 1h 45m | 1 | 1 |
| task-03 | ✅ Complete | 3h 10m | 2 | 0 |
| task-04 | ❌ Exited | 45m | 1 | 5 |

**Summary:**
- Total tasks: 4
- Completed: 3
- Failed: 1
- Success rate: 75%

---

## Review Activity

**Total reviews conducted:** [count]

**Checkpoint reviews:**
- task-01: [count] checkpoints, [approved/rejected]
- task-02: [count] checkpoints, [approved/rejected]
- task-03: [count] checkpoints, [approved/rejected]

**Review outcomes:**
- Approved first time: [count] ([percentage]%)
- Required iteration: [count] ([percentage]%)
- Average iteration count: [number]

**Review depth:**
- Light reviews (smoothness 0-2): [count]
- Medium reviews (smoothness 3-5): [count]
- Deep reviews (smoothness 6+): [count]

---

## Error Recovery

**Errors encountered:** [count]

**By task:**
- task-01: [count] errors, [count] retries, outcome: [resolved/exited]
- task-02: [count] errors, [count] retries, outcome: [resolved/exited]
- ...

**Error types:**
- Test failures: [count]
- Configuration errors: [count]
- Validation failures: [count]
- [Other categories]

**Recovery success rate:**
- Resolved autonomously: [count] ([percentage]%)
- Required user intervention: [count] ([percentage]%)

---

## Coordination Statistics

**Monitoring cycles:** [count]
**Average cycle duration:** [time]

**Message passing:**
- Messages sent: [count]
- Messages received: [count]
- Average response time: [duration]

**State transitions:**
- Total state changes: [count]
- Transient state duration (avg): [duration]

**Context usage:**
- Estimated context used: [tokens]
- Percentage of window: [percentage]%
- Checkpoints: [count]

---

## Final Verification

**Loose file check:**
```bash
ls -la docs/*.md
# Result: Only README.md (✅)
```

**Integration verification:**
- [What was verified]
- [Results]

**Quality metrics:**
- Total deliverables: [count] files
- All metadata complete: ✅ / ❌
- No duplicates: ✅ / ❌
- All cross-references valid: ✅ / ❌

---

## Learnings

**Orchestration patterns:**
- [What worked well]
- [What could be improved]

**Review effectiveness:**
- [Were checkpoints at right places?]
- [Was review depth appropriate?]

**Error recovery:**
- [How well did autonomous recovery work?]
- [What types of errors were hardest to recover?]

---

## Recommendations

**For future orchestrations:**
- [Process improvements]
- [Timing adjustments]
- [Review checkpoint placement]

**For failed tasks:**
- task-04: [Why it failed, recommendations for retry]

---

## Status

✅ **COMPLETE** - 3/4 tasks successful, 1 task requires user intervention

**Next steps:**
- Review task-04 failure and decide on retry strategy
- [Any other follow-up]
```

### Git Strategy

#### Commit Patterns

**Format:**
```
task-XX: [clear description]

[Optional: Additional context]

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Examples:**

**Good commits:**
```bash
git commit -m "task-03: extract testing patterns from TESTING_GUIDE.md

Extracted 12 granular patterns to knowledge-base/testing/
Excluded 3 outdated patterns per conductor guidance

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"

git commit -m "task-03: add unit tests for auth flow

15 tests covering authentication, token refresh, logout
All tests passing

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

**Bad commits (avoid):**
```bash
git commit -m "fix"
git commit -m "wip"
git commit -m "update files"
git commit -m "task-03"  # Too vague
```

#### Commit Frequency

**Guidelines:**
- Commit after each logical unit of work
- Commit before requesting review
- Commit after applying review feedback
- Commit at checkpoint completion
- Do NOT commit after every single file (too granular)
- Do NOT wait until task complete (too coarse)

**Recommended frequency:**
- 3-10 commits per task (typical)
- More for large tasks (>5 hours work)
- Fewer for small tasks (<2 hours work)

#### Branch Strategy

<!-- ℹ️ GIT STRATEGY: Main Branch for Documentation, Worktrees for Code

**For this template's primary use case (documentation extraction):**
- All work on **main branch**
- No worktrees needed - tasks work on different files

**Worktrees mentioned in other contexts (not a contradiction):**
- The `using-git-worktrees` skill exists for feature development
- Useful for code changes requiring isolation
- Different pattern for different scenarios

**This template uses main branch** for parallel orchestration because:
- Documentation tasks are non-destructive
- Each task works on different files (by design)
- No merge conflicts expected
- Simpler than worktrees (less overhead)

For code feature development with conflicts, use worktrees. For documentation extraction (this pattern), main branch is correct.
-->

**For autonomous parallel orchestration:**
- **All work on main branch**
- No feature branches needed (tasks are isolated)
- Conductor coordinates to prevent conflicts

**Rationale:**
- Documentation tasks are non-destructive
- Each task works on different files (by design)
- No merge conflicts expected
- Simpler git history
- Easier to track progress

**Exception:**
- If tasks might conflict (same files), use branches or worktrees
- But better to redesign task boundaries to avoid conflicts

#### Co-Authored-By Tag

**Required for all commits:**
```
Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Why:**
- Attribution (Claude Code contributed to commit)
- Tracking (identify AI-assisted work)
- Transparency (clear about collaboration)

**Implementation:**
```bash
# Every commit must include this tag
git commit -m "task-03: [description]

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"

# Use heredoc for multi-line messages
git commit -m "$(cat <<'EOF'
task-03: extract testing patterns

Extracted 12 patterns from TESTING_GUIDE.md
All patterns verified with conductor

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

### Smoothness Scale Calibration Examples

#### Purpose of Smoothness Scale

**Smoothness** measures how close code is to production-ready quality:
- **10/10:** Production-ready, exceptional quality, no improvements needed
- **8/10:** Production-ready, good quality, minor improvements optional
- **6/10:** Needs revision, several issues must be fixed before production
- **4/10:** Major issues, significant rework required

**Why calibration matters:**
- Consistent review standards across sessions
- Actionable feedback (specific issues, not vague scores)
- Domain-appropriate expectations (docs vs frontend vs backend)

#### Backend Feature Calibration

**Example:** Rate Limiting Middleware for Express API

##### 10/10 - Exceptional (Production-Ready+)

**Code quality:**
- ✅ TypeScript: Strict types, no `any`, full interfaces (RateLimitConfig, RateLimitOptions, RateLimitError)
- ✅ Security: Input validation, no injection risks, proper sanitization
- ✅ Performance: O(1) lookups, efficient Redis operations, no memory leaks
- ✅ Error handling: Specific error types (RateLimitExceededError, StorageError), proper HTTP status codes

**Testing:**
- ✅ Unit tests: 100% coverage, all edge cases tested (boundary values, concurrent requests)
- ✅ Integration tests: Real Redis, timeout scenarios, connection failures
- ✅ Load tests: Verified at 1000 req/s, performance benchmarks documented

**Documentation:**
- ✅ JSDoc: All public functions, parameters, return types, examples
- ✅ README: Updated with configuration guide, usage examples
- ✅ OpenAPI/Swagger: Endpoint documentation updated with 429 response

**Monitoring & Operations:**
- ✅ Logging: Structured logs with context (user ID, IP, endpoint, rate limit status)
- ✅ Metrics: Prometheus metrics exported (rate_limit_hits, rate_limit_errors, latency)
- ✅ Alerts: Configured for anomalous patterns (sudden spike in rate limits)

**Example code snippet (10/10 quality):**
```typescript
/**
 * Rate limiting middleware with configurable limits and Redis backend.
 *
 * @param options - Configuration for rate limiting
 * @returns Express middleware function
 *
 * @example
 * ```typescript
 * app.use(rateLimitMiddleware({
 *   windowMs: 60000,
 *   maxRequests: 100,
 *   keyGenerator: (req) => req.ip
 * }));
 * ```
 */
export function rateLimitMiddleware(options: RateLimitOptions): RequestHandler {
  const store = new RedisRateLimitStore(options.redis);

  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const key = options.keyGenerator(req);

    try {
      const { isAllowed, remaining, resetTime } = await store.consume(key, options);

      // Set rate limit headers (RFC standard)
      res.setHeader('X-RateLimit-Limit', options.maxRequests.toString());
      res.setHeader('X-RateLimit-Remaining', remaining.toString());
      res.setHeader('X-RateLimit-Reset', resetTime.toISOString());

      if (!isAllowed) {
        logger.warn('Rate limit exceeded', {
          key,
          endpoint: req.path,
          remaining: 0,
          resetTime
        });

        metrics.rateLimitHits.inc({ endpoint: req.path });

        throw new RateLimitExceededError(
          'Too many requests, please try again later',
          resetTime
        );
      }

      next();
    } catch (error) {
      if (error instanceof RateLimitExceededError) {
        res.status(429).json({
          error: 'Too Many Requests',
          message: error.message,
          retryAfter: error.resetTime
        });
      } else {
        logger.error('Rate limit middleware error', { error, key });
        // Fail open: Don't block requests on storage errors
        next();
      }
    }
  };
}
```

##### 8/10 - Good (Approve with Minor Notes)

**Code quality:**
- ✅ TypeScript: Proper types, minor edge cases might use `any` (acceptable)
- ✅ Security: Input validation, no obvious vulnerabilities
- ✅ Performance: No obvious bottlenecks, efficient operations
- ⚠️ Error handling: Correct status codes, basic error types (could be more specific)

**Testing:**
- ✅ Unit tests: 80-90% coverage, main scenarios tested
- ✅ Integration tests: Happy path + 2-3 error scenarios
- ⚠️ Load tests: Not performed (acceptable for 8/10, can validate in staging)

**Documentation:**
- ✅ JSDoc: Present on public functions (might be brief)
- ⚠️ README: Basic usage documented, could be more detailed
- ⚠️ OpenAPI: Not updated (can be follow-up task)

**Monitoring:**
- ✅ Logging: Basic logs, covers main paths
- ⚠️ Metrics: Not yet integrated (acceptable for 8/10, can add later)

**Review feedback (8/10):**
```
APPROVED (8/10 smoothness) - Good work, ready for merge with minor follow-ups

Strengths:
- TypeScript types are solid
- Error handling is correct (429 status code ✅)
- Unit tests cover main scenarios (85% coverage)
- Integration tests include error cases

Minor improvements for follow-up:
- Add JSDoc example for keyGenerator parameter
- Consider adding metrics collection (Prometheus)
- Load testing recommended in staging before production

Action: Approve for merge, create follow-up tasks for metrics + load testing
```

##### 6/10 - Needs Revision (Block Merge)

**Code quality:**
- ✅ TypeScript: Types defined (some incomplete or using `any`)
- ⚠️ Security: Basic validation but potential edge case issues
- ⚠️ Performance: Acceptable but not optimized
- ❌ Error handling: Wrong status code (500 instead of 429) OR missing error cases

**Testing:**
- ⚠️ Unit tests: 60-70% coverage, missing edge cases
- ❌ Integration tests: Only happy path tested (no error scenarios)

**Documentation:**
- ❌ JSDoc: Missing on several functions
- ⚠️ README: Incomplete or unclear

**Monitoring:**
- ❌ Logging: Inconsistent (some paths log, others don't)

**Review feedback (6/10):**
```
NEEDS REVISION (6/10 smoothness) - Several issues must be fixed

Critical issues:
1. Error handling: Using HTTP 500 for rate limit exceeded (should be 429)
   - File: middleware/rateLimit.ts:45
   - Fix: Change status code to 429

2. Integration tests: Missing error scenarios
   - Add test for Redis connection failure
   - Add test for concurrent requests hitting limit

3. Logging: Inconsistent across error paths
   - Add structured logging to all catch blocks
   - Include key/endpoint context in all logs

Non-critical (but recommended):
4. JSDoc: Add documentation to rateLimitMiddleware() and consume() functions
5. Type safety: Replace `any` type for options parameter with proper interface

Action: Fix critical issues 1-3, re-request review
```

##### 4/10 - Major Issues (Reject)

**Code quality:**
- ⚠️ TypeScript: Many `any` types, weak typing
- ❌ Security: Potential SQL injection or XSS vulnerabilities
- ❌ Performance: O(n²) operations or memory leaks
- ❌ Error handling: Crashes instead of returning errors

**Testing:**
- ❌ Unit tests: <40% coverage, critical paths untested
- ❌ Integration tests: None

**Documentation:**
- ❌ JSDoc: Missing entirely or minimal
- ❌ README: Not updated

**Review feedback (4/10):**
```
REJECTED (4/10 smoothness) - Major rework required

Critical blockers:
1. Security: Rate limit key not sanitized, potential Redis injection
   - Example: User-controlled key "test'; DEL *; GET test"
   - Impact: HIGH - Could delete all rate limit data
   - Fix: Sanitize key or use hashing (SHA-256)

2. Error handling: Middleware crashes on Redis errors
   - Current: Unhandled promise rejection crashes server
   - Fix: Wrap in try-catch, fail open on storage errors

3. Testing: No integration tests for Redis failures
   - Critical scenario: What happens when Redis is down?
   - Current behavior: Unknown (likely crash)
   - Fix: Add integration test, implement graceful degradation

4. Performance: Calling Redis for every request (even cached)
   - Problem: Every request does network roundtrip
   - Fix: Add local cache (LRU) for recent checks

5. Documentation: No JSDoc, unclear how to configure

Action: Major rework required. Address all 5 blockers before re-review.
Estimated time: 3-4 hours of focused work.
```

#### Frontend Feature Calibration

**Example:** User Profile Component with Avatar Upload

##### 10/10 - Exceptional (Production-Ready+)

**Code quality:**
- ✅ TypeScript: Strict prop types, no `any`, full interfaces
- ✅ Accessibility: ARIA labels, keyboard navigation, screen reader tested with NVDA/JAWS
- ✅ Responsive: Mobile/tablet/desktop tested, flexbox/grid proper usage, no horizontal scroll
- ✅ Performance: Memoized correctly, no unnecessary re-renders (React DevTools profiled)

**Design & UX:**
- ✅ Design system: Uses design tokens consistently (spacing, colors, typography from theme)
- ✅ Error states: Loading, error, empty states all handled with proper UX
- ✅ Animations: Smooth transitions (60fps), respects prefers-reduced-motion
- ✅ Polish: Proper focus management, loading skeletons, optimistic UI updates

**Testing:**
- ✅ Unit tests: Render tests, interaction tests, prop validation (>90% coverage)
- ✅ Integration tests: Full user flows (upload → display → edit → delete)
- ✅ Visual regression: Snapshots for all variants, Chromatic or Percy
- ✅ Accessibility: Axe-core tests pass, manual keyboard navigation tested

**Documentation:**
- ✅ Storybook: Stories for all variants (default, loading, error, empty, different sizes)
- ✅ JSDoc: Prop types documented with examples
- ✅ Usage guide: When to use this component, accessibility notes

**Example code snippet (10/10 quality):**
```typescript
export interface ProfileCardProps {
  /** User data to display */
  user: User;
  /** Avatar size variant */
  size?: 'small' | 'medium' | 'large';
  /** Callback when avatar upload completes */
  onAvatarChange?: (avatarUrl: string) => void;
  /** Whether the profile is editable */
  editable?: boolean;
}

/**
 * ProfileCard displays user information with optional avatar upload.
 *
 * @example
 * ```tsx
 * <ProfileCard
 *   user={currentUser}
 *   size="large"
 *   editable={true}
 *   onAvatarChange={(url) => updateUser({ avatarUrl: url })}
 * />
 * ```
 */
export const ProfileCard = memo<ProfileCardProps>(({
  user,
  size = 'medium',
  onAvatarChange,
  editable = false
}) => {
  const { upload, isUploading, error } = useAvatarUpload();
  const theme = useTheme();

  const handleAvatarSelect = useCallback(async (file: File) => {
    try {
      const url = await upload(file);
      onAvatarChange?.(url);
      toast.success('Avatar updated successfully');
    } catch (err) {
      toast.error('Failed to upload avatar');
    }
  }, [upload, onAvatarChange]);

  return (
    <Card
      role="article"
      aria-label={`Profile for ${user.name}`}
      sx={{ padding: theme.spacing.lg }}
    >
      <Stack direction="row" spacing={theme.spacing.md} align="center">
        <Avatar
          src={user.avatarUrl}
          alt={`${user.name}'s avatar`}
          size={size}
          loading={isUploading}
          editable={editable}
          onFileSelect={handleAvatarSelect}
        />

        <Stack spacing={theme.spacing.sm}>
          <Text variant="heading-sm">{user.name}</Text>
          <Text variant="body-sm" color="text-secondary">{user.email}</Text>
        </Stack>
      </Stack>

      {error && (
        <Alert
          variant="error"
          role="alert"
          sx={{ marginTop: theme.spacing.md }}
        >
          {error.message}
        </Alert>
      )}
    </Card>
  );
});

ProfileCard.displayName = 'ProfileCard';
```

##### 8/10 - Good (Approve with Minor Notes)

**Code quality:**
- ✅ TypeScript: Proper prop types, minor `any` acceptable
- ✅ Accessibility: ARIA labels, keyboard nav works
- ⚠️ Responsive: Works on mobile/desktop, minor tablet issues acceptable
- ✅ Performance: No obvious issues

**Design & UX:**
- ✅ Design system: Mostly consistent, minor deviations acceptable
- ✅ Error states: Main states covered (loading, error)
- ⚠️ Animations: Basic transitions, might not be fully polished

**Testing:**
- ✅ Unit tests: Main scenarios tested (70-85% coverage)
- ⚠️ Integration tests: Happy path + 1-2 error scenarios
- ⚠️ Visual regression: Basic Storybook stories, might miss some variants

**Documentation:**
- ⚠️ Storybook: Basic stories, could be more comprehensive
- ✅ JSDoc: Props documented

**Review feedback (8/10):**
```
APPROVED (8/10 smoothness) - Solid component, ready for production

Strengths:
- TypeScript types are clean
- Accessibility is good (keyboard nav, ARIA labels)
- Unit tests cover main scenarios (78% coverage)
- Design system usage is consistent

Minor improvements for polish phase (optional):
- Add visual regression test for "error" state
- Tablet layout could use refinement (avatar size on iPad)
- Consider adding loading skeleton instead of spinner

Action: Approve for merge, create polish task for visual refinements
```

##### 6/10 - Needs Revision

**Code quality:**
- ⚠️ TypeScript: Some `any` types on event handlers
- ❌ Accessibility: Missing ARIA labels or keyboard navigation broken
- ❌ Responsive: Mobile layout broken (avatar cut off, text overflow)
- ⚠️ Performance: Unnecessary re-renders on every keystroke

**Design & UX:**
- ⚠️ Design system: Inconsistent spacing (hardcoded values instead of tokens)
- ❌ Error states: Missing loading state OR error state not displayed

**Testing:**
- ⚠️ Unit tests: 50-60% coverage, missing edge cases
- ❌ Integration tests: Only happy path

**Review feedback (6/10):**
```
NEEDS REVISION (6/10 smoothness) - Fix accessibility and responsive issues

Critical issues:
1. Accessibility: Avatar upload button missing ARIA label
   - Screen reader announces "button" with no context
   - Fix: Add aria-label="Upload avatar" to button

2. Responsive: Mobile layout broken on screens <768px
   - Avatar image overflows container
   - Text wraps incorrectly, username truncated
   - Fix: Add responsive breakpoints, use flexbox wrapping

3. Error state: Error message not displayed to user
   - Upload fails silently (only logs to console)
   - Fix: Show error Alert component with message

4. Design system: Hardcoded spacing values
   - Using "16px" instead of theme.spacing.md
   - Inconsistent with rest of app
   - Fix: Replace all hardcoded values with theme tokens

Action: Fix issues 1-4, re-request review
```

#### Documentation Task Calibration

**Example:** Testing Patterns Extraction from docs_old/

##### 10/10 - Exceptional

**Content quality:**
- ✅ Extraction accuracy: Patterns match source files (100% spot-check verification)
- ✅ Completeness: All major patterns extracted, no significant gaps
- ✅ Organization: Logical structure, easy to navigate
- ✅ Clarity: Well-written, clear examples, proper markdown formatting

**Metadata:**
- ✅ Comprehensive: All required fields present (source file, confidence, related patterns)
- ✅ Accurate: Tags and categories correctly assigned
- ✅ RAG-ready: Optimal chunk sizes, clear section boundaries

**Verification:**
- ✅ Links work: All internal links tested, external links checked
- ✅ Code examples: Syntax-highlighted, runnable (where applicable)
- ✅ Formatting: Consistent markdown style, proper heading hierarchy

**Example output (10/10 quality):**
```markdown
# Testing Patterns: React Component Testing

**Source:** docs_old/guidelines/testing/component-testing.md
**Confidence:** High (Direct extraction from documented patterns)
**Related:** integration-testing.md, mocking-patterns.md

## Pattern: Render + Interaction Testing

### Description

Test React components by rendering them and simulating user interactions,
verifying both visual output and behavior.

### When to Use

- Testing user-facing components (buttons, forms, modals)
- Verifying interaction flows (click → state change → re-render)
- Ensuring accessibility (keyboard navigation, ARIA attributes)

### Implementation

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { ProfileCard } from './ProfileCard';

describe('ProfileCard', () => {
  it('should display user information', () => {
    const user = { name: 'Alice', email: 'alice@example.com' };
    render(<ProfileCard user={user} />);

    expect(screen.getByText('Alice')).toBeInTheDocument();
    expect(screen.getByText('alice@example.com')).toBeInTheDocument();
  });

  it('should handle avatar upload on click', async () => {
    const onAvatarChange = jest.fn();
    render(<ProfileCard user={user} editable onAvatarChange={onAvatarChange} />);

    const uploadButton = screen.getByLabelText('Upload avatar');
    const file = new File(['avatar'], 'avatar.png', { type: 'image/png' });

    fireEvent.change(uploadButton, { target: { files: [file] } });

    await waitFor(() => {
      expect(onAvatarChange).toHaveBeenCalledWith(expect.stringContaining('http'));
    });
  });
});
```

### Trade-offs

**Pros:**
- Tests user perspective (not implementation details)
- Catches accessibility issues
- Resilient to refactoring (tests behavior, not structure)

**Cons:**
- Slower than unit tests (full render required)
- Requires mocking for external dependencies (API calls)

### References

- [React Testing Library docs](https://testing-library.com/react)
- [Common testing patterns](../common-patterns/testing.md)
- [Accessibility testing guide](./accessibility-testing.md)

---
**Extracted:** 2026-02-05
**Reviewed:** ✅ Yes (Conductor verified against source)
```

##### 8/10 - Good (Approve)

**Content quality:**
- ✅ Extraction accuracy: Patterns generally match, minor interpretation (acceptable)
- ✅ Completeness: Major patterns extracted (might miss 1-2 minor patterns)
- ⚠️ Organization: Good structure, could be slightly clearer
- ✅ Clarity: Well-written, clear enough

**Metadata:**
- ✅ Present: Required fields filled in
- ⚠️ Completeness: Might miss 1-2 tags

**Verification:**
- ✅ Links: Main links work (might have 1 broken link)
- ✅ Formatting: Consistent

**Review feedback (8/10):**
```
APPROVED (8/10 smoothness) - Good extraction, ready for RAG ingestion

Strengths:
- Major patterns extracted accurately
- Clear examples and descriptions
- Metadata is complete
- Markdown formatting is consistent

Minor improvements (optional):
- Add reference to accessibility-testing.md pattern (mentioned but not linked)
- Consider adding "Anti-patterns" section (source mentions common mistakes)

Action: Approve for RAG ingestion, note improvements for future iterations
```

##### 6/10 - Needs Revision

**Content quality:**
- ⚠️ Extraction accuracy: Some patterns misinterpreted or incomplete
- ❌ Completeness: Missing 3+ significant patterns from source
- ❌ Organization: Structure unclear, hard to navigate
- ⚠️ Clarity: Some descriptions vague or confusing

**Metadata:**
- ❌ Incomplete: Missing required fields (confidence, related patterns)
- ⚠️ Inaccurate: Tags don't match content

**Review feedback (6/10):**
```
NEEDS REVISION (6/10 smoothness) - Fix completeness and metadata

Critical issues:
1. Completeness: Missing patterns for "Test Fixtures" and "Mocking External APIs"
   - Source: docs_old/guidelines/testing/component-testing.md sections 4-5
   - Fix: Extract these patterns, add to document

2. Metadata: Missing "Confidence" and "Related" fields
   - All pattern entries need confidence level (High/Medium/Low)
   - Add references to related patterns
   - Fix: Add metadata to all 8 pattern entries

3. Organization: "When to Use" section inconsistent across patterns
   - Some patterns have it, others don't
   - Fix: Standardize structure (Description → When to Use → Implementation → Trade-offs)

4. Clarity: "Interaction Testing" description is vague
   - Current: "Test components by rendering them"
   - Better: "Test React components by rendering in test environment and simulating user interactions (clicks, typing, form submission), verifying both rendered output and behavior"

Action: Fix issues 1-4, re-request review
```

#### Cross-Domain Smoothness Comparison

**Same score, different standards:**

| Score | Backend (Rate Limit) | Frontend (Profile) | Documentation (Patterns) |
|-------|---------------------|-------------------|-------------------------|
| **10/10** | Security audit-ready, load tested, metrics | WCAG AAA, 60fps animations, visual regression | 100% extraction accuracy, verified links |
| **8/10** | Secure, basic monitoring, main tests | WCAG AA, accessible, main tests | 90% accuracy, complete metadata |
| **6/10** | Security gaps, missing tests | Accessibility issues, responsive broken | Missing patterns, incomplete metadata |
| **4/10** | Injection vulnerabilities, crashes | Inaccessible, broken on mobile | Wrong patterns extracted, poor organization |

**Key insight:** Same score doesn't mean same criteria - expectations vary by domain.

### Review Depth Guidelines by Domain

#### Purpose

Different domains have different quality bars:
- **Documentation:** Emphasis on clarity, completeness
- **Frontend:** Emphasis on accessibility, responsiveness, UX
- **Backend:** Emphasis on security, performance, correctness

**Approval thresholds:**
- Documentation: 80% = good enough (iteratively improved)
- Frontend: 85% = production-ready (visual quality matters)
- Backend: 90% = production-ready (correctness and security critical)

#### Backend Feature Review Checklist

##### Functionality (CRITICAL - Must verify)

- [ ] **Core functionality works:** Execute all code paths (happy path + error scenarios)
- [ ] **API contracts match:** Request/response match specification
- [ ] **Database operations correct:** Queries return expected results, no N+1 queries
- [ ] **Integration points work:** External services (Redis, third-party APIs) handled correctly

##### Security (CRITICAL - Must verify)

- [ ] **Input validation:** All user inputs validated (type, length, format, range)
- [ ] **No injection vulnerabilities:** SQL injection, NoSQL injection, command injection checked
- [ ] **No XSS vulnerabilities:** Output properly escaped (if rendering HTML)
- [ ] **Authentication/authorization:** Proper checks on protected endpoints
- [ ] **Secrets management:** No hardcoded credentials, use environment variables
- [ ] **Rate limiting:** Prevent abuse (if applicable)

##### Error Handling (HIGH PRIORITY)

- [ ] **Proper HTTP status codes:** 200/201 (success), 400 (bad request), 401/403 (auth), 404 (not found), 429 (rate limit), 500 (server error)
- [ ] **Specific error types:** Custom error classes instead of generic Error
- [ ] **No silent failures:** All errors logged with context
- [ ] **Graceful degradation:** Failures in non-critical services don't crash app

##### Performance (HIGH PRIORITY)

- [ ] **Efficient algorithms:** No O(n²) or worse in hot paths
- [ ] **Database queries optimized:** Proper indexes, no N+1 queries
- [ ] **Caching used appropriately:** Redis/memory cache for expensive operations
- [ ] **Connection pooling:** Database/Redis connections reused
- [ ] **No memory leaks:** Event listeners cleaned up, large objects released

##### Testing (HIGH PRIORITY)

- [ ] **Unit tests: >80% coverage:** All public functions tested
- [ ] **Integration tests:** Real database/Redis, test full request/response flow
- [ ] **Edge case tests:** Boundary values (0, MAX_INT), concurrent requests, timeouts
- [ ] **Error scenario tests:** What happens when database is down? Redis unavailable? API timeout?

##### Code Quality (MEDIUM PRIORITY - Nice to have)

- [ ] **TypeScript strict mode:** No `any` types in public APIs (internal `any` acceptable)
- [ ] **Clear naming:** Functions/variables have descriptive names
- [ ] **No code duplication:** Shared logic extracted to utilities
- [ ] **Reasonable complexity:** Functions <50 lines, clear responsibility

##### Documentation (MEDIUM PRIORITY)

- [ ] **JSDoc on public functions:** Parameters, return types, examples
- [ ] **README updated:** Usage instructions, configuration options
- [ ] **OpenAPI/Swagger:** Endpoints documented (if REST API)

##### Monitoring (NICE TO HAVE - Can defer)

- [ ] **Structured logging:** JSON logs with context (request ID, user ID, endpoint)
- [ ] **Metrics exported:** Prometheus/StatsD metrics for key operations
- [ ] **Alerts configured:** Anomaly detection (error rate spikes, latency increases)

**Approval threshold:** ✅ All CRITICAL items + ✅ Most HIGH PRIORITY items = 9-10/10 smoothness

#### Frontend Feature Review Checklist

##### Functionality (CRITICAL - Must verify)

- [ ] **Core functionality works:** All user interactions behave as expected
- [ ] **State management correct:** State updates properly, no stale data
- [ ] **Props/types correct:** TypeScript types match actual usage
- [ ] **Integration with backend:** API calls work, data displayed correctly

##### Accessibility (CRITICAL - Must verify)

- [ ] **Keyboard navigation:** All interactive elements reachable and operable with keyboard
- [ ] **ARIA labels:** Screen reader announces all elements correctly
- [ ] **Focus management:** Focus visible, focus trap in modals, logical tab order
- [ ] **Color contrast:** WCAG AA minimum (4.5:1 for text, 3:1 for UI elements)
- [ ] **Semantic HTML:** Proper elements (<button>, <input>, <label>, <nav>, etc.)
- [ ] **Screen reader tested:** NVDA/JAWS/VoiceOver manual test (spot-check major flows)

##### Responsive Design (HIGH PRIORITY)

- [ ] **Mobile layout works:** Tested on iPhone SE (small), iPad (medium), desktop (large)
- [ ] **No horizontal scroll:** All breakpoints fit in viewport
- [ ] **Touch targets:** Buttons/links ≥44px (iOS guideline)
- [ ] **Responsive images:** Proper sizes/srcset for different screen densities
- [ ] **Flexbox/Grid used correctly:** No magic numbers, uses responsive units

##### Error States & Loading (HIGH PRIORITY)

- [ ] **Loading states:** Spinner or skeleton while fetching data
- [ ] **Error states:** User-friendly error messages (not stack traces)
- [ ] **Empty states:** Helpful message when no data ("No results found", "Add your first item")
- [ ] **Validation feedback:** Form errors displayed clearly, inline validation

##### Performance (HIGH PRIORITY)

- [ ] **No unnecessary re-renders:** React.memo, useMemo, useCallback used appropriately
- [ ] **Code splitting:** Large dependencies lazy-loaded
- [ ] **Image optimization:** WebP/AVIF format, proper sizing, lazy loading
- [ ] **Bundle size:** <100KB gzipped for main bundle (check with webpack-bundle-analyzer)

##### Design System Consistency (MEDIUM PRIORITY)

- [ ] **Uses design tokens:** theme.spacing, theme.colors, theme.typography (no hardcoded values)
- [ ] **Follows component patterns:** Matches existing components in style/structure
- [ ] **Consistent spacing:** Proper use of spacing scale (4px, 8px, 16px, 24px, 32px)
- [ ] **Typography scale:** Uses defined sizes (heading-lg, body-md, caption-sm)

##### Testing (MEDIUM PRIORITY)

- [ ] **Unit tests: >70% coverage:** All major interactions tested
- [ ] **Integration tests:** User flows tested (login → dashboard → action)
- [ ] **Visual regression:** Storybook snapshots for all variants (Chromatic/Percy ideal but not required)

##### Documentation (NICE TO HAVE)

- [ ] **Storybook stories:** Examples of all prop variants
- [ ] **JSDoc on props:** Type descriptions, when to use
- [ ] **Usage guide:** Accessibility notes, responsive behavior

**Approval threshold:** ✅ All CRITICAL items + ✅ Most HIGH PRIORITY items = 8.5-10/10 smoothness

#### Documentation Task Review Checklist

##### Content Quality (CRITICAL)

- [ ] **Extraction accuracy:** Spot-check 20% of patterns against source files
- [ ] **Completeness:** All major patterns extracted (no significant gaps)
- [ ] **Correct interpretation:** Patterns accurately represent source intent
- [ ] **No hallucination:** All information comes from source (not invented)

##### Organization (HIGH PRIORITY)

- [ ] **Logical structure:** Clear hierarchy, easy to navigate
- [ ] **Consistent formatting:** All sections use same markdown style
- [ ] **Proper chunking:** Sections 200-500 words (optimal for RAG retrieval)
- [ ] **Clear headings:** Descriptive, search-friendly titles

##### Metadata (HIGH PRIORITY)

- [ ] **Source attribution:** All content links back to source file
- [ ] **Confidence levels:** Each pattern tagged with extraction confidence (High/Medium/Low)
- [ ] **Related patterns:** Cross-references to related content
- [ ] **Tags/categories:** Proper classification for search

##### Verification (HIGH PRIORITY)

- [ ] **Links work:** All internal/external links tested (no 404s)
- [ ] **Code examples:** Syntax valid, properly formatted
- [ ] **Markdown valid:** No broken formatting, images load

##### RAG Optimization (MEDIUM PRIORITY)

- [ ] **Chunk boundaries:** Clear section breaks for embedding
- [ ] **Keywords present:** Important terms mentioned for search
- [ ] **Context sufficient:** Each section standalone (doesn't require reading entire doc)

##### Clarity (MEDIUM PRIORITY)

- [ ] **Well-written:** Clear, concise, proper grammar
- [ ] **Examples provided:** Concepts illustrated with code/diagrams
- [ ] **Jargon explained:** Technical terms defined on first use

**Approval threshold:** ✅ All CRITICAL items + ✅ Most HIGH PRIORITY items = 8-9/10 smoothness

#### Review Workflow by Domain

**Backend feature review:**
```markdown
## Conductor Review Process

### Phase 1: Security & Correctness (MUST PASS)

1. Read implementation code (focus on input validation, SQL queries, error handling)
2. Check for common vulnerabilities (SQL injection, XSS, command injection)
3. Verify error handling (proper status codes, no crashes)
4. Review test coverage (must have integration tests for error scenarios)

**Gate:** If security issue or correctness bug → REJECT (6/10 or lower), request fix

### Phase 2: Quality Assessment (IF PHASE 1 PASSES)

1. Check performance (query optimization, algorithm complexity)
2. Check monitoring (logging, metrics)
3. Check documentation (JSDoc, README)

**Scoring:**
- Perfect: 10/10 (all items checked)
- Missing monitoring: 9/10 (approve, note for follow-up)
- Missing documentation: 8/10 (approve, note for follow-up)
- Missing performance checks: 7/10 (needs revision, run benchmarks)

### Phase 3: Approval Decision

- **9-10/10:** Approve immediately
- **8/10:** Approve with follow-up tasks (monitoring, documentation)
- **7/10:** Needs revision (performance issues)
- **<7/10:** Reject (security/correctness issues)
```

**Frontend feature review:**
```markdown
## Conductor Review Process

### Phase 1: Accessibility & Functionality (MUST PASS)

1. Test keyboard navigation (can all interactive elements be reached with Tab?)
2. Check ARIA labels (screen reader would understand?)
3. Test responsive design (mobile, tablet, desktop)
4. Verify functionality (all interactions work as expected)

**Gate:** If accessibility issue or broken functionality → REJECT (6/10 or lower), request fix

### Phase 2: Design & Performance (IF PHASE 1 PASSES)

1. Check design system usage (tokens, consistency)
2. Check error/loading states
3. Check performance (unnecessary re-renders?)

**Scoring:**
- Perfect: 10/10 (exceptional design + performance)
- Good design, acceptable performance: 8-9/10 (approve)
- Inconsistent design or performance issues: 7/10 (needs polish)

### Phase 3: Approval Decision

- **9-10/10:** Approve immediately
- **8/10:** Approve (minor design inconsistencies acceptable)
- **7/10:** Needs revision (fix design system or performance)
- **<7/10:** Reject (accessibility or functionality broken)
```

**Documentation review:**
```markdown
## Conductor Review Process

### Phase 1: Accuracy & Completeness (MUST PASS)

1. Spot-check 20% of extracted patterns against source files
2. Verify no major patterns missing
3. Check metadata completeness (source, confidence, tags)

**Gate:** If inaccurate or incomplete → REJECT (6/10 or lower), request fixes

### Phase 2: Quality Assessment (IF PHASE 1 PASSES)

1. Check organization (logical structure?)
2. Check clarity (well-written, clear examples?)
3. Check verification (links work, formatting correct?)

**Scoring:**
- Perfect accuracy, excellent organization: 10/10
- Accurate, good organization: 8-9/10 (approve)
- Accurate, needs organization work: 7/10 (needs revision)

### Phase 3: Approval Decision

- **8-10/10:** Approve for RAG ingestion
- **7/10:** Needs revision (reorganize or add missing links)
- **<7/10:** Reject (inaccurate or incomplete)
```

#### Key Differences Summarized

| Domain | Critical Focus | Approval Threshold | Tolerance for "Good Enough" |
|--------|---------------|-------------------|----------------------------|
| Backend | Security, correctness, performance | 90% (9/10) | Low - bugs in production are costly |
| Frontend | Accessibility, responsiveness, UX | 85% (8.5/10) | Medium - polish can iterate |
| Documentation | Accuracy, completeness | 80% (8/10) | High - docs can be iteratively improved |

**Rationale:**
- Backend errors are hardest to fix (database migrations, API changes break clients)
- Frontend can iterate quickly (deploy updates without breaking changes)
- Documentation is continuously improved (RAG system allows easy updates)

---

## Part 6: Reference

### Document Organization

#### Directory Structure Standards

**docs/ - Final deliverables only:**
```
docs/
├── knowledge-base/          # RAG files (production)
│   ├── conductor/       # Orchestration patterns
│   ├── implementation/     # Implementation patterns
│   ├── reference/          # Shared knowledge
│   ├── testing/            # Testing patterns (created by task-03)
│   ├── api/                # API patterns (created by task-04)
│   ├── database/           # DB patterns (created by task-05)
│   ├── templates/          # Templates (created by task-06)
│   └── README.md           # Index
├── reference/              # Human-readable guides (compiled)
│   └── [category].md       # One guide per category
├── plans/                  # Design documents
│   ├── designs/
│   ├── implementation/
│   └── (see docs/reference/designs/ for task design templates)
├── specs/                  # Specifications
└── archive/                # Historical documents

**CRITICAL: Keep docs/ root clean!**
- Only README.md in docs/
- No loose .md files
- All content in subdirectories
```

**docs/plans/implementation/ - Task instructions:**
```
docs/plans/implementation/
├── task-00.md
├── task-01.md
└── ...
```

**docs/implementation/ - Reports and proposals:**
```
docs/implementation/
├── reports/                # Completion reports
│   ├── task-00-report.md
│   ├── task-01-report.md
│   ├── task-03-checkpoint-1.md
│   └── ...
├── proposals/              # Work proposals (optional)
│   ├── extractions/
│   │   └── task-XX-extraction-proposal.md
│   └── rag-files/
│       └── task-XX/
│           └── [proposal files].md
└── [other temporary directories]
```

**temp/ - Ephemeral files (symlink to /tmp/remindly, cleared on reboot):**
```
temp/
├── task-03-rag-files.txt   # Tracking files
├── task-04-rag-files.txt
└── [other temporary data]

**Can be deleted after conductor review**
```

#### File Creation Rules

**Execution sessions creating files:**

**1. Proposals (if proposal-first workflow):**
```bash
# CORRECT: Create in proposals directory
for file in [files]; do
    # Create in docs/implementation/proposals/rag-files/task-03/
    echo "content" > "docs/implementation/proposals/task-03-rag-pattern-name.md"
done

# WRONG: Create directly in docs/ or in subdirectories
# DO NOT DO THESE BEFORE CONDUCTOR REVIEW:
echo "content" > "docs/knowledge-base/testing/$file"  # ❌
echo "content" > "docs/implementation/proposals/rag-files/task-03/$file"  # ❌ (old pattern)
```

**2. Proposal structure - embed RAG content with delimiters:**
```bash
# Each proposal contains:
# - YAML frontmatter (type: rag-addition, task_id, target_category, target_filename)
# - Reasoning section (why this belongs in KB)
# - RAG Match List (KB overlap check results at 0.4 threshold)
# - <!-- BEGIN RAG FILE --> to <!-- END RAG FILE --> with RAG file content

# One proposal file per RAG file (not grouped):
docs/implementation/proposals/task-03-rag-pattern-1.md
docs/implementation/proposals/task-03-rag-pattern-2.md
```

**3. After conductor approves proposals, extract RAG files:**
```bash
# Extract RAG file content from each proposal
for proposal in docs/implementation/proposals/task-03-rag-*.md; do
    # Extract content between <!-- BEGIN RAG FILE --> and <!-- END RAG FILE -->
    # Write to docs/knowledge-base/testing/[target_filename].md
done

# Verify files created
ls docs/knowledge-base/testing/*.md
# Expected: One .md per approved proposal
```

**3. Reports always in docs/implementation/reports/:**
```bash
# Completion reports
echo "report content" > docs/implementation/reports/task-03-report.md

# Checkpoint reports
echo "checkpoint" > docs/implementation/reports/task-03-checkpoint-1.md

# Error reports
echo "error" > docs/implementation/reports/task-03-error-retry-1.md
```

**4. Tracking files in tmp/:**
```bash
# List of created files (for RAG ingestion tracking)
ls docs/knowledge-base/testing/*.md > temp/task-03-rag-files.txt

# Temporary working files
echo "notes" > temp/task-03-notes.txt
```

#### Conductor Verification

**Loose file check (after each task completion):**
```bash
# Check docs/ root for loose files
ls -1 docs/*.md | grep -v README.md
# Expected: Empty (no output)

# Check knowledge-base/ root for loose files
find docs/knowledge-base/ -maxdepth 1 -type f -name "*.md"
# Expected: Only README.md

# If loose files found:
echo "⚠️  WARNING: Loose files found in docs/"
echo "Files must be in subdirectories, not root"
echo "Please move files to appropriate subdirectories"
```

**Why this matters:**
- RAG queries work better with organized subdirectories
- Easier to maintain and navigate
- Prevents documentation sprawl
- Clear separation of concerns

### Complete Examples

<!-- 🚩 FLAG 6: TODO - Complete Examples Need Full Implementation
ISSUE: This section is titled "Complete Examples" but the conductor example explicitly says "(Abbreviated)". For this template to be implementation-ready, users need full working examples.

RECOMMENDATION: After template review and refinement, create companion files:
- `docs/plans/examples/conductor-complete-example.md` (full 500-800 line example)
- `docs/plans/examples/execution-complete-example.md` (full 400-600 line example)
- Reference these files from this section

These should be actual task instruction files for a real scenario (e.g., the documentation extraction project with tasks 03, 04, 05) that can be copied and adapted.

For now, the abbreviated examples provide the structure. Plan to flesh out complete examples during Stage 2 (Task Instruction Creation) of the implementation path.
-->

#### Conductor Example (Abbreviated)

```markdown
# Task 00: Conductor - Documentation Extraction Project

**Execution tasks:** task-03, task-04, task-05
**Max concurrent:** 3 tasks
**Estimated duration:** 6-8 hours

---

## Objective

Coordinate extraction of documentation patterns from docs_old/ into knowledge-base/ RAG structure. Monitor 3 parallel execution sessions, review work at checkpoints, handle errors autonomously, and ensure all deliverables meet quality standards.

---

## Initialization

### Step 1: Set Up Hook

```bash
bash tools/message-watcher/setup.sh --preset orchestration
```

### Step 2: Initialize Database

```sql
UPDATE coordination_status SET state = 'watching' WHERE task_id = 'task-00';
UPDATE migration_tasks
SET status = 'in_progress', worked_by = 'orch-20260204-1430', started_at = datetime('now')
WHERE task_id = 'task-00';
```

### Step 3: Verify Execution Tasks Initialized

```sql
SELECT task_id, status FROM migration_tasks
WHERE task_id IN ('task-03', 'task-04', 'task-05');
-- Expected: All status = 'pending'
```

---

## Main Loop

### Step 4: Launch Monitoring Subagent

```python
result = Task(
    description="Watch for execution attention needed",
    prompt="""
Watch coordination_status for execution tasks (task-03, task-04, task-05).
Check every 12 seconds. Max 600 iterations (2 hours).

Exit when:
1. REVIEW NEEDED - Any task state = 'needs_review'
   → Return: "REVIEW_NEEDED: [comma-separated task-ids]"
2. ERROR DETECTED - Any task state = 'error'
   → Return: "ERROR_DETECTED: [comma-separated task-ids]"
3. ALL COMPLETE - All tasks state = 'complete'
   → Return: "ALL_COMPLETE"
4. STALE STATE - Task in 'review_approved'/'review_failed'/'fix_proposed' for >60s
   → Return: "STALE_STATE: [task-id] in state [state] for [duration]s"
5. CRITICAL EXIT - Any task state = 'exited'
   → Return: "ERROR_DETECTED: [task-id] (EXITED)"

Query:
SELECT task_id, state FROM coordination_status
WHERE task_id IN ('task-03', 'task-04', 'task-05')
  AND state NOT IN ('working', 'complete', 'review_approved', 'review_failed', 'fix_proposed')
ORDER BY task_id ASC;

Ignore states: working, complete, review_approved, review_failed, fix_proposed
(These are normal/transient states)
""",
    subagent_type="general-purpose",
    run_in_background=false
)
```

### Step 5: Process Subagent Result

**If "REVIEW_NEEDED" in result:**
```python
task_ids = parse_task_ids(result)  # ['task-03', 'task-05']

for task_id in task_ids:
    # Read review request
    review_msg = query(f"SELECT message FROM task_messages WHERE task_id='{task_id}' AND message LIKE '%REVIEW REQUEST%' ORDER BY timestamp DESC LIMIT 1")

    # Parse details
    changed_files = parse_changed_files(review_msg)
    test_results = parse_test_results(review_msg)
    smoothness = parse_smoothness(review_msg)

    # Determine review depth based on smoothness
    if smoothness <= 2:
        # Light review (spot check)
        review_depth = "light"
        files_to_review = changed_files[:3]  # First 3 files
    elif smoothness <= 5:
        # Medium review
        review_depth = "medium"
        files_to_review = changed_files[:len(changed_files)//2]  # 50%
    else:
        # Deep review
        review_depth = "deep"
        files_to_review = changed_files  # All files

    # Review files
    for file in files_to_review:
        content = read_file(file)
        # Check: code quality, tests, architecture

    # Make decision
    if all_checks_passed:
        # Approve
        execute(f"UPDATE coordination_status SET state = 'review_approved' WHERE task_id = '{task_id}'")
        insert_message(task_id, 'task-00', f'REVIEW APPROVED: {review_depth} review passed. Proceed to next step.')
    else:
        # Reject
        execute(f"UPDATE coordination_status SET state = 'review_failed' WHERE task_id = '{task_id}'")
        insert_message(task_id, 'task-00', f'REVIEW FAILED: Issues found: [details]. Fix and re-submit.')

# Return to Step 4 (relaunch subagent)
```

**If "ERROR_DETECTED" in result:**
```python
task_ids = parse_task_ids(result)

for task_id in task_ids:
    # Read error message
    error_msg = query(f"SELECT message FROM task_messages WHERE task_id='{task_id}' AND message LIKE '%ERROR%' ORDER BY timestamp DESC LIMIT 1")

    # Parse retry count
    retry_match = re.search(r'Retry (\d+)/5', error_msg)
    retry_num = int(retry_match.group(1)) if retry_match else 1

    if retry_num >= 5:
        # Terminal error - task exited
        log(f"Task {task_id} hit max retries, marked as exited")
        # Check if should continue or stop all
        continue

    # Read error report
    report_path = parse_report_path(error_msg)
    error_report = read_file(report_path)

    # Analyze and propose fix
    fix_proposal = analyze_error_and_propose_fix(error_report)

    # Send fix
    execute(f"UPDATE coordination_status SET state = 'fix_proposed' WHERE task_id = '{task_id}'")
    insert_message(task_id, 'task-00', f'FIX PROPOSAL (Retry {retry_num}/5): {fix_proposal}')

# Return to Step 4
```

**If "ALL_COMPLETE" in result:**
```python
# All execution tasks finished
# Proceed to completion (Step 6)
```

---

## Completion

### Step 6: Final Verification

[Verify all tasks, read reports, check loose files]

### Step 7: Update State

```sql
UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-00';
UPDATE migration_tasks
SET status = 'complete', completed_at = datetime('now'),
    report_path = 'docs/implementation/reports/task-00-report.md'
WHERE task_id = 'task-00';
```

### Step 8: Generate Report

[Create conductor completion report]

---

## Success Criteria

- [ ] All 3 execution tasks complete or properly handled
- [ ] All reviews processed
- [ ] Completion report generated
- [ ] docs/ clean (no loose files)
```

#### Execution Example (Abbreviated)

```markdown
# Task 03: Extract Testing Patterns

**Parallel-safe:** Yes (with task-04, task-05)
**Dependencies:** Task 1, 2 complete
**Review checkpoints:** 2 (after extraction proposal, after RAG file creation)

---

## Objective

Extract testing patterns from TESTING_GUIDE.md and related files into granular RAG files in knowledge-base/testing/.

---

## Initialization

### Step 1: Set Up Hook

```bash
bash tools/message-watcher/setup.sh --preset execution --task-id task-03
```

### Step 2: Initialize Database

```sql
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03';
UPDATE migration_tasks
SET status = 'in_progress', worked_by = 'exec-03-20260204-1435', started_at = datetime('now')
WHERE task_id = 'task-03';
```

### Step 3: Launch Background Subagent

```python
subagent = Task(
    description="Monitor for conductor messages",
    prompt="""
Watch coordination_status for task-03 and task_messages.
Check every 8 seconds. Max 75 iterations (10 minutes).

Exit when:
1. CONDUCTOR MESSAGE - New message from task-00
   → Return: "MESSAGE: [content]"
2. STATE CHANGE - Unexpected state change
   → Return: "STATE_CHANGE: [old] → [new]"

Query:
SELECT state FROM coordination_status WHERE task_id = 'task-03';
SELECT COUNT(*) as msg_count FROM task_messages
WHERE task_id = 'task-03' AND from_session = 'task-00'
  AND timestamp > datetime('now', '-10 seconds');
""",
    subagent_type="general-purpose",
    run_in_background=true
)
```

---

## Work Execution

### Step 4: Read Source Files

[Read TESTING_GUIDE.md and related files]

**Between-step check:**
```python
result = TaskOutput(task_id=subagent.id, block=false, timeout=100)
if result.completed:
    handle_message(result.output)
    subagent = relaunch_subagent()
```

### Step 5: Analyze Patterns

[Identify valid vs outdated patterns]

**Between-step check:**
[Same as Step 4]

### Step 6: Create Extraction Proposal

[Document what will be extracted]

**Between-step check:**
[Same as Step 4]

---

## Review Checkpoint 1

### Step 7: Pause for Extraction Proposal Review

```sql
INSERT INTO task_messages VALUES ('task-03', 'exec-03-20260204-1435',
'REVIEW REQUEST: Extraction proposal ready
Reason: Checkpoint before extraction
Identified: 12 valid patterns, 3 outdated (excluded)
Proposal: docs/implementation/proposals/extractions/task-03-extraction-proposal.md
Smoothness: 0 (Following plan perfectly)
Review focus: Pattern selection accuracy');

UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-03';
```

**Launch blocking subagent:**
```python
result = Task(
    description="Wait for extraction proposal approval",
    prompt="""
Watch coordination_status for task-03 state changes.
Check every 8 seconds. Max 75 iterations.

Exit when:
1. APPROVED - State = 'review_approved'
   → Return: "APPROVED"
2. FAILED - State = 'review_failed'
   → Return: "FAILED"
""",
    subagent_type="general-purpose",
    run_in_background=false
)
```

### Step 8: Process Review Result

```python
if result == "APPROVED":
    # Read feedback
    feedback = query("SELECT message FROM task_messages WHERE task_id='task-03' AND from_session='task-00' ORDER BY timestamp DESC LIMIT 1")

    # Update state
    execute("UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03'")

    # Continue to Step 9

elif result == "FAILED":
    # Read rejection
    rejection = query("...")

    # Update state
    execute("UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-03'")

    # Fix issues
    # Return to Step 6 (recreate proposal)
```

---

## Continue Work

### Step 9: Create RAG Proposals (One Per File)

**CRITICAL:** Do NOT create RAG files directly in `docs/knowledge-base/`. Instead, create PROPOSALS in `docs/implementation/proposals/` that contain the RAG file content.

For each approved pattern from checkpoint 1, create a proposal file:
- `docs/implementation/proposals/task-03-rag-{pattern-name}.md`

Each proposal contains:
- YAML frontmatter: `type: rag-addition`, `task_id`, `target_category`, `target_filename`
- Reasoning section (why this belongs in KB)
- RAG Match List (KB overlap check at 0.4 threshold)
- `<!-- BEGIN RAG FILE -->` ... `<!-- END RAG FILE -->` with the actual RAG file content including YAML frontmatter

**Pre-screening:** Query KB for each pattern using `query_documents("[topic]", limit=10)` at 0.4 threshold before creating proposal.

**Between-step check:**
```python
# After creating proposals, relaunch background subagent
subagent = relaunch_subagent()

# Check after proposal creation
result = TaskOutput(task_id=subagent.id, block=false, timeout=100)
if result.completed:
    handle_message(result.output)
    subagent = relaunch_subagent()
```

---

## Review Checkpoint 2

### Step 10: Request Proposal Review

Commit proposals and request conductor review:
```bash
git add docs/implementation/proposals/task-03-rag-*.md
git commit -m "task-03: RAG proposals ready for review"
```

Update task state to `needs_review` with proposal count and KB overlap summary.

---

## Finalization

### Step 11: Extract Approved RAG Files to Final Location

After conductor approval, extract RAG content from proposals:
```bash
for proposal in docs/implementation/proposals/task-03-rag-*.md; do
    # Extract content between <!-- BEGIN RAG FILE --> and <!-- END RAG FILE -->
    # Determine target filename from frontmatter (target_filename field)
    # Write to docs/knowledge-base/testing/[target_filename]
done

# Verify files created
ls docs/knowledge-base/testing/*.md | wc -l
```

### Step 12: Track Files for RAG Ingestion

```bash
ls docs/knowledge-base/testing/*.md > temp/task-03-rag-files.txt
```

### Step 13: Final Verification

[Run all verification checks]

### Step 14: Update State

```sql
UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-03';
UPDATE migration_tasks
SET status = 'complete', completed_at = datetime('now'),
    report_path = 'docs/implementation/reports/task-03-report.md'
WHERE task_id = 'task-03';
```

### Step 15: Generate Report

[Create completion report]

---

## Success Criteria

- [ ] All patterns extracted
- [ ] All reviews approved
- [ ] Files in correct location
- [ ] Verification passed
- [ ] Report generated
```

---

### Multi-Domain Examples

The previous examples demonstrated parallel orchestration for documentation extraction projects. The following examples prove this template works for frontend, backend, and mixed code projects.

#### Frontend Feature Example: User Profile Component with Avatar Upload

```markdown
### Example: Frontend Feature - User Profile Component with Avatar Upload

**Project:** React + TypeScript + Vite
**Feature:** Add user profile component with avatar upload functionality
**Parallel tasks:** 3 execution tasks (component, upload handler, integration tests)

#### Task Breakdown

**task-03: ProfileCard Component**
- Read: Existing design system in `src/theme/`, component patterns in `src/components/`
- Implementation: Create ProfileCard component with avatar display
- Output: React component + Storybook stories + unit tests
- Tests: Unit tests (render, props, responsive behavior)
- Review checkpoint: Component API, accessibility (ARIA labels, keyboard nav), responsive design

**task-04: Avatar Upload Handler**
- Read: Existing upload utilities in `src/utils/`, API service patterns
- Implementation: Create `useAvatarUpload.ts` hook with image validation
- Output: Custom hook + API integration
- Tests: Unit tests (upload flow, error handling, progress)
- Review checkpoint: Error handling, security (file type validation), performance

**task-05: Integration Tests**
- Read: Existing test patterns in `tests/`, component outputs from task-03/04
- Implementation: Integration tests for full user flow
- Output: E2E tests (upload → display → error scenarios)
- Tests: Visual regression tests, accessibility tests
- Review checkpoint: Test coverage (>80%), edge case coverage

#### Parallelization Analysis

**Shared files (SAFE - read-only):**
- ✅ `src/theme/tokens.ts` (design tokens)
- ✅ `src/types/user.ts` (User type definition, tasks only import)
- ✅ `src/utils/api.ts` (API client, tasks only use)

**Independent files (SAFE - no overlap):**
- ✅ task-03 writes: `src/components/ProfileCard.tsx`, `src/components/ProfileCard.stories.tsx`
- ✅ task-04 writes: `src/hooks/useAvatarUpload.ts`, `src/api/avatar.ts`
- ✅ task-05 writes: `tests/integration/profile.test.tsx`, `tests/visual/profile.spec.ts`

**Potential conflicts (REVIEW):**
- ⚠️ `src/components/index.ts` (barrel export) - both task-03 and task-05 might update
  - **Solution:** task-03 exports component, task-05 only imports (no conflict)
- ⚠️ `src/types/index.ts` - task-04 might add `AvatarUploadResult` type
  - **Solution:** Create new file `src/types/avatar.ts` to avoid conflict

**Parallelization decision:** ✅ SAFE - proceed with parallel execution

#### Domain-Specific Smoothness Indicators (Frontend)

**10/10 - Exceptional:**
- TypeScript: Strict types, no `any`, full prop interfaces
- Accessibility: ARIA labels, keyboard navigation, screen reader tested
- Responsive: Mobile/tablet/desktop tested, flexbox/grid properly used
- Error states: Loading, error, empty states all handled with proper UX
- Performance: Memoization where needed, no unnecessary re-renders
- Design system: Uses tokens consistently, follows component patterns
- Tests: Unit + integration + visual regression, >90% coverage
- Documentation: Storybook stories with all variants, JSDoc on props

**8/10 - Good (Approve):**
- TypeScript: Proper types, minor `any` acceptable in edge cases
- Accessibility: ARIA labels, keyboard nav works, basic testing
- Responsive: Works on mobile/desktop, minor tablet issues acceptable
- Error states: Main states covered (loading, error)
- Performance: No obvious issues, might not be fully optimized
- Design system: Mostly consistent, minor deviations acceptable
- Tests: Unit tests for main scenarios (70-80% coverage)
- Documentation: Basic Storybook story, prop types documented

**6/10 - Needs Revision:**
- TypeScript: Some `any` types, missing prop interfaces
- Accessibility: Missing ARIA labels or keyboard navigation issues
- Responsive: Works on desktop only, mobile broken
- Error states: Missing loading or error state
- Performance: Unnecessary re-renders, inefficient hooks
- Design system: Inconsistent spacing/colors, not using tokens
- Tests: <60% coverage, missing edge cases
- Documentation: No Storybook stories or incomplete

**Feedback for 6/10:**
```
NEEDS REVISION (6/10 smoothness):
- Accessibility: Add ARIA label to avatar upload button, ensure keyboard navigation
- Responsive: Fix mobile layout - avatar is cut off on screens <768px
- Error states: Add loading spinner during upload, show error message on failure
- Tests: Add unit test for upload progress callback, integration test for retry logic
- Design system: Use theme.spacing.md instead of hardcoded "16px"
```

#### Review Flow Example

**Checkpoint 1: Component Design Review**

task-03 requests review after completing ProfileCard component:

```sql
-- task-03 sets state
UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-03';

-- task-03 sends message
INSERT INTO task_messages VALUES (
    'task-00', 'task-03',
    'REVIEW REQUEST (Checkpoint 1/2):
    Completed ProfileCard component with avatar display.

    Files created:
    - src/components/ProfileCard.tsx (125 lines)
    - src/components/ProfileCard.stories.tsx (8 stories)
    - src/components/__tests__/ProfileCard.test.tsx (15 test cases)

    Smoothness self-assessment: 8/10
    - TypeScript types complete
    - Responsive design tested (mobile/tablet/desktop)
    - Accessibility: ARIA labels, keyboard navigation
    - Unit tests: 85% coverage
    - Storybook: All variants covered

    Ready for review: Component API, accessibility patterns, responsive approach'
);
```

Conductor reviews code:

```markdown
# Conductor Review (task-03 checkpoint 1)

## Smoothness Assessment: 8/10 ✅ APPROVE

**Strengths:**
- TypeScript types are clean and complete
- Responsive design works well across breakpoints
- Accessibility is solid (ARIA, keyboard nav, semantic HTML)
- Test coverage exceeds requirement (85% > 80% threshold)
- Storybook stories comprehensive

**Minor issues (acceptable for 8/10):**
- Avatar size prop could use TypeScript union type ('sm' | 'md' | 'lg') instead of string
- One test case missing: What happens when avatar URL fails to load?

**Decision:** APPROVE
**Rationale:** Minor issues don't block progress. Can be addressed in polish phase.
**Action:** Continue to next phase (upload handler)
```

```sql
-- Conductor approves
UPDATE coordination_status SET state = 'review_approved' WHERE task_id = 'task-03';

INSERT INTO task_messages VALUES (
    'task-03', 'task-00',
    'REVIEW APPROVED (Checkpoint 1/2):
    Component API design is solid. Continue to avatar upload implementation.

    Optional improvements for polish phase:
    - Add size prop type safety (union type)
    - Add test case for failed avatar URL loading'
);
```

#### Final Review Example

After all tasks complete:

```markdown
# Conductor Final Review (All tasks complete)

## Overall Smoothness: 8.5/10 ✅ APPROVE FOR MERGE

**task-03 (ProfileCard):** 8/10 - Clean component, good tests
**task-04 (Upload Handler):** 9/10 - Excellent error handling, secure validation
**task-05 (Integration Tests):** 8/10 - Good coverage, visual regression solid

**Integration check:**
- ProfileCard properly uses useAvatarUpload hook ✅
- API contracts match between task-04 and task-03 ✅
- Integration tests cover full user flow ✅
- No merge conflicts detected ✅

**Decision:** Feature ready for merge to main branch
```
```

---

#### Backend Feature Example: Rate Limiting Middleware for Express API

```markdown
### Example: Backend Feature - Rate Limiting Middleware for Express API

**Project:** Express + TypeScript + PostgreSQL
**Feature:** Add rate limiting middleware to prevent API abuse
**Parallel tasks:** 3 execution tasks (middleware, storage, tests)

#### Task Breakdown

**task-03: Rate Limit Middleware Core**
- Read: Existing middleware in `src/middleware/`, Express patterns
- Implementation: Create `rateLimitMiddleware.ts` with configurable limits
- Output: Middleware + TypeScript types
- Tests: Unit tests (limit enforcement, key generation)
- Review checkpoint: Algorithm correctness, configuration API, error handling

**task-04: Redis Storage Backend**
- Read: Existing database utilities in `src/db/`, Redis patterns
- Implementation: Create `RedisRateLimitStore.ts` for distributed rate limiting
- Output: Storage adapter + connection management
- Tests: Unit tests (Redis operations, TTL, cleanup)
- Review checkpoint: Connection handling, retry logic, performance

**task-05: Integration Tests & Monitoring**
- Read: Existing test patterns, middleware from task-03, storage from task-04
- Implementation: Integration tests + monitoring metrics
- Output: API tests, load tests, metric collection
- Tests: Load testing (1000 req/s), edge case scenarios
- Review checkpoint: Test coverage, monitoring completeness, load performance

#### Parallelization Analysis

**Shared files (SAFE - read-only):**
- ✅ `src/types/request.ts` (Express request types)
- ✅ `src/config/index.ts` (Configuration loader)
- ✅ `src/utils/logger.ts` (Logging utility)

**Independent files (SAFE - no overlap):**
- ✅ task-03 writes: `src/middleware/rateLimit.ts`, `src/types/rateLimit.ts`
- ✅ task-04 writes: `src/storage/RedisRateLimitStore.ts`, `src/storage/RateLimitStore.interface.ts`
- ✅ task-05 writes: `tests/integration/rateLimit.test.ts`, `tests/load/rateLimit.load.ts`

**Potential conflicts (REVIEW):**
- ⚠️ `src/middleware/index.ts` (barrel export) - task-03 adds export, task-05 imports
  - **Solution:** task-03 exports first, task-05 depends on task-03 completion
  - **Decision:** Add task dependency: task-05 waits for task-03 review approval
- ⚠️ `src/app.ts` (Express app setup) - might need middleware registration
  - **Solution:** Defer app.ts changes to conductor after all tasks complete

**Parallelization decision:** ✅ SAFE with dependency - task-03 & task-04 parallel, task-05 waits for checkpoint

#### Domain-Specific Smoothness Indicators (Backend)

**10/10 - Exceptional:**
- TypeScript: Full type coverage, no `any`, proper error types
- Error handling: Specific error classes, proper HTTP status codes (429)
- Security: Input validation, no injection vulnerabilities, rate key hashing
- Performance: O(1) lookups, efficient Redis operations, no memory leaks
- Monitoring: Metrics exported (Prometheus format), structured logging
- Tests: Unit (90%+) + integration + load tests, edge cases covered
- Documentation: OpenAPI/Swagger updated, JSDoc on all functions
- Database: Connection pooling, retry logic, graceful degradation

**8/10 - Good (Approve):**
- TypeScript: Proper types, minor edge cases might use `any`
- Error handling: Correct status codes, basic error types
- Security: Input validation, no obvious vulnerabilities
- Performance: No obvious bottlenecks, efficient operations
- Monitoring: Basic metrics, structured logging on main paths
- Tests: Unit (80%+) + integration, main scenarios covered
- Documentation: JSDoc on public functions, basic README
- Database: Connection handling works, basic retry

**6/10 - Needs Revision:**
- TypeScript: Multiple `any` types, weak error types
- Error handling: Wrong status codes (500 instead of 429)
- Security: Missing input validation on some endpoints
- Performance: Potential N+1 queries or inefficient operations
- Monitoring: Inconsistent logging, missing metrics
- Tests: <60% coverage, missing error scenario tests
- Documentation: Missing JSDoc on several functions
- Database: No retry logic, connection leaks possible

**Feedback for 6/10:**
```
NEEDS REVISION (6/10 smoothness):
- Error handling: Use 429 status for rate limit exceeded, not 500
- Security: Add input validation for custom rate limit keys (prevent injection)
- Performance: Redis SETEX is more efficient than SET + EXPIRE (single operation)
- Monitoring: Add metric for rate limit hits by endpoint
- Tests: Add integration test for Redis connection failure scenario
- Documentation: Add JSDoc to rateLimitMiddleware() explaining all parameters
```
```

---

#### Mixed Feature Example: Real-Time Notifications

```markdown
### Example: Mixed Feature - Real-Time Notifications (WebSocket Backend + React Frontend)

**Project:** Express + React + WebSocket (socket.io)
**Feature:** Add real-time notification system for user actions
**Parallel tasks:** 4 execution tasks (backend WebSocket, frontend hook, UI component, tests)

#### Task Breakdown

**task-03: WebSocket Server (Backend)**
- Read: Existing Express setup, authentication middleware
- Implementation: Socket.io server with notification events
- Output: WebSocket handlers, event types, authentication integration
- Tests: Unit tests (connection, auth, room management)
- Review checkpoint: Event schema, security (auth verification), connection handling

**task-04: Notification Storage & API (Backend)**
- Read: Database schema, existing API patterns
- Implementation: Notification CRUD endpoints + PostgreSQL storage
- Output: REST API endpoints, database migrations, types
- Tests: Unit tests (CRUD operations, queries)
- Review checkpoint: API design, database schema, query performance

**task-05: Notification Hook & Context (Frontend)**
- Read: Existing React context patterns, WebSocket client setup
- Implementation: `useNotifications` hook + React context provider
- Output: Custom hook, context provider, TypeScript types
- Tests: Unit tests (hook behavior, context state)
- Review checkpoint: Hook API design, connection management, error handling

**task-06: Notification UI Component (Frontend)**
- Read: Design system, existing notification patterns
- Implementation: NotificationToast component with animations
- Output: React component, styles, Storybook stories
- Tests: Unit tests (render, interactions, accessibility)
- Review checkpoint: UX design, accessibility, animation performance

#### Parallelization Analysis

**Shared files (SAFE - read-only):**
- ✅ `shared/types/notification.ts` (notification schema, shared between backend/frontend)
- ✅ `frontend/src/theme/` (design tokens)
- ✅ `backend/src/config/` (server configuration)

**Independent files (PARALLEL GROUP 1 - Backend):**
- ✅ task-03 writes: `backend/src/websocket/notificationHandler.ts`, `backend/src/websocket/auth.ts`
- ✅ task-04 writes: `backend/src/api/notifications.ts`, `backend/src/db/migrations/002_notifications.sql`

**Independent files (PARALLEL GROUP 2 - Frontend):**
- ✅ task-05 writes: `frontend/src/hooks/useNotifications.ts`, `frontend/src/context/NotificationContext.tsx`
- ✅ task-06 writes: `frontend/src/components/NotificationToast.tsx`, `frontend/src/components/NotificationToast.stories.tsx`

**Potential conflicts (REVIEW):**
- ⚠️ `shared/types/notification.ts` (shared schema) - ALL tasks might update
  - **Solution:** Conductor creates schema first, tasks only read
  - **Decision:** Pre-task setup by conductor, lock file during execution
- ⚠️ Integration point: task-05 (hook) uses task-03 (WebSocket) events
  - **Solution:** task-05 implements against agreed schema, integration tested in task-06
  - **Decision:** SAFE - schema is shared/types (read-only)

**Parallelization decision:** ✅ SAFE - Two parallel groups (backend tasks 03+04, frontend tasks 05+06)

#### Coordination Flow

**Phase 1: Schema Agreement (Conductor)**

Before launching tasks, conductor creates shared schema:

```typescript
// shared/types/notification.ts (created by conductor)
export interface Notification {
  id: string;
  userId: string;
  type: 'info' | 'warning' | 'error' | 'success';
  title: string;
  message: string;
  createdAt: Date;
  read: boolean;
}

export interface NotificationEvent {
  event: 'notification:new' | 'notification:read' | 'notification:delete';
  data: Notification;
}
```

**Phase 2: Parallel Execution**

Backend group (task-03, task-04) and frontend group (task-05, task-06) execute in parallel.

**Phase 3: Integration Review**

Conductor verifies contracts match:

```markdown
# Integration Review Checklist

## Backend → Frontend Contract Verification

**WebSocket Events (task-03):**
- ✅ Emits: `notification:new`, `notification:read`, `notification:delete`
- ✅ Event payload matches `NotificationEvent` type
- ✅ Authentication required before subscribing

**API Endpoints (task-04):**
- ✅ GET /api/notifications - returns `Notification[]`
- ✅ PATCH /api/notifications/:id/read - marks notification read
- ✅ DELETE /api/notifications/:id - deletes notification

**Frontend Hook (task-05):**
- ✅ Listens for: `notification:new`, `notification:read`, `notification:delete`
- ✅ Expects payload matching `NotificationEvent` type
- ✅ Uses API endpoints from task-04 for initial fetch

**Frontend Component (task-06):**
- ✅ Uses `useNotifications` hook from task-05
- ✅ Renders `Notification` type correctly
- ✅ Handles all notification types (info, warning, error, success)

## Integration Gaps: NONE ✅

All contracts match. Ready for end-to-end testing.
```
```

---

#### Domain-Specific Concerns Summary

This section summarizes domain-specific concerns that impact smoothness assessments and parallelization safety across different project types.

**Frontend Concerns:**
- **Tests:** Jest + React Testing Library configuration, mocking hooks/context
- **Design system:** Token adherence (spacing, colors, typography), component patterns
- **Accessibility:** ARIA labels, keyboard navigation, screen reader compatibility
- **State management:** React context, hooks, or Redux patterns
- **Bundler:** Vite/Webpack configuration, code splitting

**Backend Concerns:**
- **Security:** SQL injection prevention, input validation, authentication/authorization
- **Performance:** Database query optimization (N+1 problems), caching strategies
- **Error handling:** HTTP status codes, error types, structured logging
- **Testing:** Unit tests (services, middleware), integration tests (API endpoints)
- **Monitoring:** Metrics (Prometheus), logging (structured JSON), tracing

**Mixed Concerns (Frontend + Backend):**
- **API contracts:** Request/response types match on both sides
- **Shared types:** TypeScript types consistent between frontend/backend
- **Integration points:** WebSocket events, REST API endpoints, authentication flows
- **Testing strategies:** E2E tests covering full stack, contract testing
- **Deployment:** Build ordering (shared types first, then backend, then frontend)

**Testing Strategies by Domain:**

| Domain | Primary Test Types | Coverage Target | Critical Areas |
|--------|-------------------|-----------------|----------------|
| Frontend | Unit (Jest/RTL) + Visual Regression | 80%+ | Components, hooks, accessibility |
| Backend | Unit (Jest) + Integration (Supertest) | 85%+ | API endpoints, middleware, security |
| Mixed | E2E (Playwright) + Contract Tests | 70%+ | Full user flows, API contracts |

**When to serialize vs parallelize:**
- **Frontend:** Parallelize by component/page (if no shared barrel exports)
- **Backend:** Parallelize by endpoint/service (if no shared middleware registration)
- **Mixed:** Parallelize backend + frontend separately, integrate at checkpoints

---

### Decision Log

#### State Naming

**Decision:** Use `watching`, `complete`, `exited` (not `listening`, `coordination_complete`, `error`)

**Rationale:**
- `watching` is more accurate than `listening` for conductor monitoring
- `complete` is simpler and more consistent than `coordination_complete` or `task_completed`
- `exited` clearly indicates terminal error state (distinguishes from recoverable `error`)
- Custom hook design session is authoritative (continued iterations after tmp/ files)

**Impact:**
- All state references must use these names
- Hook presets use these exit criteria
- Database queries check for these specific states

### Error Recovery Retry Count: 5 Attempts

#### Design Rationale

**Goal:** Maximize autonomous recovery while minimizing user wait time

**Approach:** Calculate cumulative success probability across retry attempts

#### Success Rate Model

**Assumption:** Each retry has different success probability based on fix complexity

| Retry | Fix Type | Example | Success Rate | Cumulative Success |
|-------|----------|---------|--------------|-------------------|
| 1 | Transient issue | Network timeout, race condition | 30% | 30% |
| 2 | Simple fix | Typo, missing import, wrong variable name | 40% | 30% + (70% × 40%) = 58% |
| 3 | Moderate fix | Logic error, wrong algorithm approach | 20% | 58% + (42% × 20%) = 66.4% |
| 4 | Complex fix | Architecture change, different library | 7% | 66.4% + (33.6% × 7%) = 68.75% |
| 5 | Last-ditch | Alternative approach, workaround | 2% | 68.75% + (31.25% × 2%) = 69.4% |

**Cumulative success after 5 retries: ~69%**

**Why not higher?**
- Retry 6-10: Success rate <1% each (diminishing returns)
- Most errors either: (a) Fixed in first 3 retries, OR (b) Require architectural change (can't be fixed autonomously)

#### Time to Exhaustion

**Average time per retry:**
```
- Error detection: 10-20 seconds (execution notices failure)
- Error report: 30-60 seconds (execution generates report with logs, context)
- Conductor analysis: 60-120 seconds (reads report, analyzes root cause)
- Fix proposal: 30-60 seconds (conductor proposes fix, updates state)
- Fix application: 30-90 seconds (execution reads fix, applies changes, retries operation)

Total per retry: 2.5-5.5 minutes
Average: ~4 minutes per retry
```

**5 retries: 20 minutes maximum** (4 min/retry × 5 retries)

**Acceptable delay:** 20 minutes is reasonable before escalating to user
- Short enough: User doesn't wait hours
- Long enough: Gives autonomous system fair chance to recover

#### Alternative Approaches Considered

##### Approach A: 3 Retries (Faster Escalation)

**Pros:**
- Faster user escalation (12 minutes)
- Less wasted time on unrecoverable errors

**Cons:**
- Lower cumulative success: ~66% (vs 69% with 5 retries)
- Misses 3% of recoverable errors
- More user interruptions (more frequent escalations)

**Rejected because:** 3% success rate difference = ~30 unnecessary escalations per 1000 errors

##### Approach B: 7 Retries (Higher Success)

**Pros:**
- Higher cumulative success: ~70% (marginal 1% improvement)

**Cons:**
- Longer user wait: 28 minutes (vs 20 minutes)
- Diminishing returns: Retries 6-7 have <1% success each
- More wasted time: Extra 8 minutes for 1% improvement

**Rejected because:** 8 minutes of extra wait for 1% success improvement = poor trade-off

##### Approach C: 10 Retries (Exhaustive)

**Pros:**
- Slightly higher success: ~71%

**Cons:**
- Very long wait: 40 minutes
- Extremely diminishing returns: Retries 6-10 contribute <2% total
- Poor UX: User wonders if system is stuck

**Rejected because:** 20 extra minutes for 2% success = unacceptable user experience

#### Decision Matrix

| Retry Count | Cumulative Success | Time to Exhaustion | User Escalations (per 1000 errors) | Recommendation |
|-------------|-------------------|-------------------|-------------------------------------|----------------|
| 3 | 66% | 12 min | 340 | ❌ Too many escalations |
| **5** | **69%** | **20 min** | **310** | **✅ Optimal balance** |
| 7 | 70% | 28 min | 300 | ⚠️ Marginal gain, significant time cost |
| 10 | 71% | 40 min | 290 | ❌ Poor UX, minimal gain |

#### Adaptive Retry Configuration

**For time-critical projects:**
```markdown
## Override: Faster Escalation

**Context:** Real-time trading system where 20-minute delay unacceptable

**Configuration:**
- Max retries: 3 (12 minutes to exhaustion)
- Trade-off: Accept 3% more user escalations for faster response

```python
# Execution error recovery (override)
MAX_RETRIES = 3  # Override default 5
```

**Rationale:** Time-critical system values fast escalation over autonomous recovery rate
```

**For low-stakes projects:**
```markdown
## Override: Extended Retries

**Context:** Batch processing job (non-interactive, runs overnight)

**Configuration:**
- Max retries: 7 (28 minutes to exhaustion)
- Trade-off: Extra time acceptable (job runs overnight anyway), maximize autonomous success

```python
# Execution error recovery (override)
MAX_RETRIES = 7  # Override default 5
```

**Rationale:** Non-interactive context values autonomous recovery over fast escalation
```

#### Real-World Retry Distribution (Expected)

**Based on model assumptions:**
```
Out of 1000 errors:
- 300 fixed on retry 1 (30%)
- 280 fixed on retry 2 (28%, cumulative 58%)
- 84 fixed on retry 3 (8.4%, cumulative 66.4%)
- 24 fixed on retry 4 (2.35%, cumulative 68.75%)
- 6 fixed on retry 5 (0.65%, cumulative 69.4%)
- 306 escalated to user after retry 5 (30.6%)

Result: 694 errors resolved autonomously, 306 user escalations
```

**Key insight:** Most successful retries happen in first 2 attempts (58%), retries 3-5 catch edge cases (11%).

#### Conclusion

**5 retries balances:**
- ✅ Reasonable autonomous success rate (69%)
- ✅ Acceptable user wait time (20 minutes)
- ✅ Efficient use of time (diminishing returns after retry 5)
- ✅ Good UX (not too fast to give up, not too slow to frustrate)

**Standard:** Use 5 retries unless project context requires override (time-critical: 3, low-stakes: 7)

#### Retry Limit

**Decision:** 5 retries (not 3)

**Rationale:**
- More opportunities for autonomous recovery
- Reduces user intervention frequency
- Complex issues may need multiple fix attempts
- Still has terminal limit (not infinite)

**Impact:**
- Error messages format: "ERROR (Retry 1/5): ..."
- Conductor tracks retry count (1-5)
- After retry 5: state = exited, session terminates

#### Dual-Table Architecture

**Decision:** Keep both coordination_status and migration_tasks

**Rationale:**
- Context efficiency: coordination_status queries 75% cheaper (2 fields vs 7)
- Separation of concerns: real-time state vs lifecycle tracking
- Data integrity: migration_tasks is source of truth
- Subagent polls coordination_status hundreds of times (huge savings)

**Impact:**
- Dual updates required (message THEN state)
- Reconciliation procedures if inconsistency
- Both tables used for different purposes

#### Between-Step Checking

**Decision:** Check after every numbered step (default, overridable)

**Rationale:**
- Allows critical message handling (30s-2min response time)
- Prevents sessions from missing conductor messages
- Minimal overhead (~500 tokens per check)
- Can be overridden for very quick steps

**Impact:**
- Task instructions include between-step checks
- Background subagent runs throughout work
- Non-blocking TaskOutput calls between steps

#### Subagent Context Isolation

**Decision:** Subagent context is discarded, not added to main session

**Rationale:**
- Main session context stays clean
- Polling iterations don't accumulate
- 10x reduction vs Ralph Loop (which added all iterations to main session)
- Enables long monitoring without context pollution

**Impact:**
- Launch subagent with Task tool
- Main session only adds launch (~500 tokens) and result (~200 tokens)
- Subagent's 75 polling iterations (~11k tokens) completely discarded

#### Transient State Handling

**Decision:** Conductor ignores review_approved/review_failed in main query, but includes staleness detection

**Rationale:**
- Avoids race condition (conductor re-enters before execution polls)
- Execution updates within 8 seconds (fast enough)
- Staleness detection catches crashed sessions (>60s in transient state)
- Simpler than complex synchronization

**Impact:**
- Conductor query excludes these states
- Separate staleness check for >60s duration
- Execution session must update quickly after detecting transient state

#### Proposal-First Workflow

**Decision:** Optional (depends on project requirements)

**Rationale:**
- Enables review before files enter final location
- Prevents pollution of docs/ with bad files
- Provides rollback capability
- Useful for learning/testing orchestration patterns
- Can be overkill for simple projects

**Impact:**
- Adds Step N+1 (move files) after RAG file review
- Adds docs/implementation/proposals/ directory
- Requires conductor to review proposals before final files
- 2 additional steps per execution task

#### Heartbeat Crash Detection

**Decision:** Implement heartbeat updates between execution steps

**Problem:** Sessions can crash while in 'working' state
- Without heartbeat: 2-hour timeout (max_iterations × check_interval)
- With heartbeat: 3-minute detection (last_heartbeat timeout)

**Trade-offs:**

| Aspect | Cost | Benefit |
|--------|------|---------|
| **Context usage** | ~1,000 tokens (0.5%) | 40x faster crash detection (3min vs 2hr) |
| **Implementation complexity** | Update heartbeat in 2-3 places | Simple: single SQL UPDATE |
| **False positive rate** | Very low (3min timeout) | Only triggers on actual crashes |

**Why 3-minute timeout:**
- Typical step duration: 30-120 seconds
- Between-step heartbeat: Max 120s gap
- Timeout: 180s = 1.5x max gap
- Buffer: 60s for slow operations

**Rationale:**
- 0.5% context cost justified by 40x faster crash detection
- Enables conductor to detect and recover from crashes quickly
- Complements session_id isolation (prevents conflicts, detects crashes)
- Simple implementation pattern (one UPDATE per execution step)

**Impact:**
- Database schema includes last_heartbeat column
- Execution tasks update heartbeat between steps
- Conductor monitors for heartbeat timeouts (>180s)
- Crashed sessions detected in 3 minutes instead of 2 hours

### Adaptation Guide

#### Customizing for Your Project

**Step 1: Define your tasks**
```markdown
Questions to answer:
1. What work needs to be done?
2. Can it be divided into independent tasks?
3. How many tasks? (3-5 is typical, 5+ gets complex)
4. What are dependencies between tasks?
5. Where do tasks need review checkpoints?
```

**Step 2: Adapt state machine**
```markdown
Questions:
1. Do you need additional states?
   - Example: 'paused' if tasks can pause for external events
2. Do you need fewer states?
   - Example: No review checkpoints = no need for needs_review
3. What are your terminal states?
   - Always need: complete, exited
   - Optional: cancelled, abandoned
```

**Step 3: Design review checkpoints**
```markdown
Questions:
1. What requires human (conductor) approval?
   - Major decisions?
   - File creation?
   - Pattern selection?
2. How many checkpoints per task?
   - Too many = slow, too few = risky
   - 1-3 checkpoints typical
3. What criteria for approval/rejection?
   - Tests passing?
   - Quality checks?
   - Architecture compliance?
```

**Step 4: Customize subagent prompts**
```markdown
Conductor subagent:
- Exit conditions: Add any project-specific states
- Polling rate: Adjust based on expected task duration
- Max iterations: Longer projects = higher max

Execution subagent:
- Exit conditions: Match your state machine
- Polling rate: Keep at 8s (good balance)
- Max iterations: 75 = 10 minutes (usually sufficient)
```

**Step 5: Adapt verification requirements**
```markdown
Questions:
1. What tests must pass?
2. What quality checks are required?
3. What artifacts must be created?
4. What git commit standards?
5. What report contents are needed?
```

**Step 6: Customize error recovery**
```markdown
Questions:
1. What errors are recoverable?
2. What errors require immediate escalation?
3. How many retries? (5 is default, adjust if needed)
4. What fix strategies for common errors?
```

#### Common Customizations

**Fewer review checkpoints:**
```markdown
# Original: 2 checkpoints per task
# Customized: 1 final review only

Changes:
- Remove mid-task review request (Step 7-8 in example)
- Keep only final review before completion
- Adjust step numbering
- Update conductor to expect 1 checkpoint per task
```

**More parallel tasks:**
```markdown
# Original: 3 execution tasks
# Customized: 7 execution tasks

Changes:
- Update conductor subagent query to include all task IDs
- Increase conductor max iterations (more tasks = longer monitoring)
- Adjust context budget estimates
- Consider batch review strategies (review 3-4 at a time)
```

**No proposal-first workflow:**
```markdown
# Original: Create proposals, review, then move to final
# Customized: Create directly in final location

Changes:
- Remove proposal creation step
- Remove proposal review checkpoint
- Remove file move step
- Files created directly in docs/knowledge-base/
- Only one review checkpoint (final verification)
```

**Different database schema:**
```markdown
# Original: dual-table (coordination_status + migration_tasks)
# Customized: Single table with more fields

Changes:
- Update all SQL queries
- Adjust context estimates (single table = more tokens per query)
- Update hook monitoring query
- Keep same state machine, just different storage
```

#### When NOT to Use This Template

**Do NOT use autonomous parallel orchestration if:**

1. **Tasks are sequential:**
   - Task B depends on Task A output
   - Use subagent-driven-development instead
   - Single session, multiple subagents for phases

2. **Work is trivial:**
   - < 2 hours total work
   - Coordination overhead > actual work
   - Just do it in one session

3. **Context is not a concern:**
   - Tasks are small
   - User can act as bridge
   - Simpler coordination acceptable

4. **Tasks interfere with each other:**
   - Edit same files
   - Conflict on resources
   - Better to serialize or redesign task boundaries

5. **Errors need user decisions:**
   - Ambiguous requirements
   - Business logic decisions needed
   - Technical decisions beyond conductor capability
   - User should be in the loop from start

**Alternative patterns:**
- **Subagent-driven-development:** Single session, multiple phases
- **User-bridge coordination:** User relays messages between sessions
- **Sequential execution:** One task at a time, simpler coordination
- **Manual orchestration:** User launches and monitors tasks manually

---

## Appendix A: Quick Reference

### State Machine Summary

**Conductor (task-00):**
- `watching` → `reviewing` → `watching` → `complete`
- `watching` → `exit_requested` (for errors/questions)

**Execution (task-XX):**
- `working` → `needs_review` → `review_approved` → `working` → `complete`
- `working` → `error` → `review_failed` → `working` (retry 1-4)
- `working` → `error` (5x) → `exited` (terminal)

**Transient (<60s):** review_approved, review_failed
**Terminal:** complete (success), exited (failure)

### Hook Setup Commands

```bash
# Conductor
bash tools/message-watcher/setup.sh --preset orchestration

# Execution
bash tools/message-watcher/setup.sh --preset execution --task-id task-XX

# Check hook status
cat .claude/message-watcher.local.md

# Manual stop (remove hook)
rm .claude/message-watcher.local.md
```

### Common SQL Patterns

```sql
-- Initialize execution task
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-XX';
UPDATE migration_tasks
SET status = 'in_progress', worked_by = '[session-id]', started_at = datetime('now')
WHERE task_id = 'task-XX';

-- Request review
INSERT INTO task_messages VALUES ('task-XX', '[session-id]', '[review request]');
UPDATE coordination_status SET state = 'needs_review' WHERE task_id = 'task-XX';

-- Approve review
UPDATE coordination_status SET state = 'review_approved' WHERE task_id = 'task-XX';
INSERT INTO task_messages VALUES ('task-XX', 'task-00', '[approval message]');

-- Process approval
UPDATE coordination_status SET state = 'working' WHERE task_id = 'task-XX';

-- Mark complete
UPDATE coordination_status SET state = 'complete' WHERE task_id = 'task-XX';
UPDATE migration_tasks
SET status = 'complete', completed_at = datetime('now'), report_path = '[path]'
WHERE task_id = 'task-XX';

-- Mark exited (terminal error)
UPDATE coordination_status SET state = 'exited' WHERE task_id = 'task-XX';
UPDATE migration_tasks SET status = 'failed', completed_at = datetime('now')
WHERE task_id = 'task-XX';
```

### Subagent Launch Patterns

```python
# Background (during work)
subagent = Task(
    description="Monitor for messages",
    prompt="[watch coordination_status + task_messages]",
    subagent_type="general-purpose",
    run_in_background=true
)

# Between-step check
result = TaskOutput(task_id=subagent.id, block=false, timeout=100)
if result.completed:
    handle_message(result.output)
    subagent = relaunch()

# Blocking (during review wait)
result = Task(
    description="Wait for approval",
    prompt="[watch for state changes]",
    subagent_type="general-purpose",
    run_in_background=false  # BLOCKING
)

if result == "APPROVED":
    continue_work()
elif result == "FAILED":
    fix_and_retry()
```

### Directory Checklist

```bash
# Before task complete
ls -1 docs/*.md | grep -v README.md  # Should be empty
find docs/knowledge-base/ -maxdepth 1 -name "*.md"  # Only README.md
find docs/implementation/proposals/rag-files/task-XX/  # Should be empty (after files moved)
test -f docs/implementation/reports/task-XX-report.md  # Should exist

# Verification passed
git status  # Clean
[test command]  # All pass
```

---

## Appendix B: Troubleshooting

### Hook Not Blocking Exit

**Symptoms:** Session exits even though state != exit criteria

**Check:**
```bash
# Hook state file exists?
test -f .claude/message-watcher.local.md && echo "File exists" || echo "Missing"

# Hook monitoring correct task?
grep "task_id:" .claude/message-watcher.local.md

# Current state matches exit criteria?
# Query coordination_status and compare to exit_criteria in state file
```

**Fix:**
```bash
# Reactivate hook
bash tools/message-watcher/setup.sh --preset [orchestration|execution] [--task-id task-XX]

# Verify state file created
cat .claude/message-watcher.local.md
```

### Subagent Not Exiting

**Symptoms:** Subagent runs to max iterations, never detects condition

**Check:**
```bash
# Read subagent output log (if available)
# Check if query is returning expected data

# Manually run subagent query
[subagent SQL query]
# Does it return the expected state/messages?
```

**Fix:**
- Review subagent prompt query syntax
- Check task_id matches
- Verify state changes are actually happening in database
- Increase max iterations if timeout too short

### Review Timeout

**Symptoms:** Execution session waiting for review >10 minutes

**Check:**
```sql
-- Is conductor still running?
SELECT state FROM coordination_status WHERE task_id = 'task-00';

-- Did conductor see the review request?
SELECT timestamp FROM task_messages
WHERE task_id = 'task-XX' AND message LIKE '%REVIEW REQUEST%';

-- Has conductor sent any messages?
SELECT message, timestamp FROM task_messages
WHERE task_id = 'task-XX' AND from_session = 'task-00'
ORDER BY timestamp DESC LIMIT 3;
```

**Fix:**
- Check conductor terminal (still running?)
- Check conductor subagent (still monitoring?)
- Manual approval if conductor crashed:
  ```sql
  UPDATE coordination_status SET state = 'review_approved' WHERE task_id = 'task-XX';
  INSERT INTO task_messages VALUES ('task-XX', 'manual', 'Manual approval due to conductor timeout');
  ```

### Database Corruption

**Symptoms:** Inconsistent states between tables

**Check:**
```sql
-- Audit query
SELECT
  m.task_id,
  m.status as migration_status,
  c.state as coordination_state
FROM migration_tasks m
LEFT JOIN coordination_status c ON m.task_id = c.task_id
WHERE m.task_id LIKE 'task-%';
```

**Fix:**
```sql
-- Reconcile (trust migration_tasks)
UPDATE coordination_status
SET state = (
  CASE
    WHEN (SELECT status FROM migration_tasks WHERE task_id = coordination_status.task_id) = 'complete'
      THEN 'complete'
    WHEN (SELECT status FROM migration_tasks WHERE task_id = coordination_status.task_id) = 'failed'
      THEN 'exited'
    ELSE 'working'
  END
)
WHERE task_id IN ([affected tasks]);
```

### Context Exhaustion

**Symptoms:** Session approaching 200k token limit

**Check:**
```bash
/context
# Look at token usage breakdown
```

**Fix:**
```bash
# Compact session
# This preserves summary, discards detailed history
# Hook and database persist

# After compact:
# - Relaunch subagents (they don't survive compact)
# - Re-query current state from database
# - Continue from current step
```

---

## Document Revision History

**v1.0.0 (2026-02-04):**
- Initial comprehensive template
- All sections complete
- Based on custom hook + subagent coordination design
- Incorporates learnings from previous sessions
- Ready for review and refinement

---

**END OF GOLDEN TEMPLATE**
