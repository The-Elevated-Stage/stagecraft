# Sequential Task Design

**Version:** 1.1.0
**Date:** 2026-02-08
**Status:** Active
**Purpose:** Comprehensive specification for writing task instruction files for sequential execution — single-session tasks with linear step progression, review checkpoints, and optional database coordination.

---

## Document Overview

### What This Document Is

This is a **comprehensive specification** for creating task instruction files that follow a sequential execution model. It defines:

1. **Task instruction file format** — The reusable skeleton for any sequential task
2. **Verification patterns** — How to validate work at each step
3. **Review checkpoints** — How and when to pause for conductor/user review
4. **Context budget management** — How to estimate, monitor, and split work when context runs low
5. **Optional database coordination** — Lightweight comms-link patterns for orchestrated workflows

### When to Use Sequential Execution

**Use sequential execution when:**
- A single session can complete all the work
- Steps have linear dependencies (Step B needs Step A's output)
- Tasks might edit the same files across steps
- Work is straightforward and doesn't justify parallel coordination overhead
- User/conductor bridge coordination is acceptable
- The task is exploratory or requires frequent user decisions

**Examples:**
- Extract documentation patterns from 6 source files into RAG knowledge base
- Reorganize directory structure with verification at each stage
- Interactive file categorization with user decisions at key points
- Database migration with sequential schema changes

### Pattern Selection Guide

Choose between sequential and parallel based on task characteristics:

| Characteristic | Sequential (This Document) | Parallel ([parallel-task-design.md](parallel-task-design.md)) |
|----------------|---------------------------|--------------------------------------------------------------|
| Session count | 1 execution session | 1 conductor + N execution sessions |
| Coordination | Manual SQL queries or PAUSE points | Custom hooks + background subagents |
| Message checking | `SELECT FROM task_messages` between steps | Background subagent monitors continuously |
| Hook usage | Optional (state enforcement) | Required (prevents premature exit) |
| Context overhead | Lower (1 session) | Higher (N+1 sessions, but isolated) |
| Setup complexity | Simple (just task instructions) | Complex (hooks, subagents, state machine) |
| Use case | Linear, single-task, sequential | Complex, multi-task, parallel |
| User involvement | Can act as bridge if needed | Minimal (autonomous coordination) |
| Error handling | Stop, document, escalate to user | 5-retry state machine with autonomous recovery |

**Rule of thumb:** Start with sequential. Only move to parallel when you have 3+ truly independent tasks with no shared file conflicts and the coordination overhead is justified by parallelism gains.

### Key Differences from Parallel Design

The parallel design introduces significant infrastructure (custom hooks, background subagent message watchers, a coordination_status state machine, autonomous retry loops). Sequential execution strips all of that away:

- **No custom hooks required** — Session lifecycle is simple (start → work → finish)
- **No background subagents** — No message watcher polling in the background
- **No state machine** — No coordination_status table with 10+ states and transitions
- **No autonomous retry loops** — Errors stop execution; user/conductor decides next steps
- **Database is optional** — Core template works with just file-based task instructions

### Quality Standards

This specification produces **implementation-ready** task instructions that are:

1. **Unambiguous** — Execution sessions can follow without clarification
2. **Comprehensive** — All edge cases, error scenarios, and recovery steps covered
3. **Verifiable** — Clear success criteria and verification steps at every stage
4. **Context-efficient** — Token budgets estimated and monitored throughout
5. **Self-contained** — All context references included; no assumed external knowledge

**Expected output quality:**
- A single session can complete the task without user intervention (between PAUSE points)
- Verification gates catch errors before they propagate to later steps
- Completion reports provide full historical record
- Error documentation enables efficient debugging and recovery

---

## Part 1: Foundation

### Prerequisites

#### Directory Structure

Sequential tasks use the standard project documentation directories:

```
docs/plans/implementation/    # Task instruction files
docs/implementation/reports/  # Completion reports
docs/implementation/proposals/ # Learnings and proposals
temp/                         # Scratch files (symlink to /tmp/remindly, cleared on reboot)
```

#### Optional: Database Infrastructure

If the task is part of an orchestrated workflow, the conductor may set up comms-link tables:

```sql
-- migration_tasks: Track task status
CREATE TABLE IF NOT EXISTS migration_tasks (
  task_id TEXT PRIMARY KEY,
  title TEXT,
  status TEXT DEFAULT 'pending',     -- pending | in_progress | complete | blocked
  worked_by TEXT,
  started_at TEXT,
  completed_at TEXT,
  report_path TEXT
);

-- task_messages: Communication between sessions
CREATE TABLE IF NOT EXISTS task_messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  task_id TEXT,
  from_session TEXT,                 -- 'conductor' or 'I' (execution session)
  message TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);
```

These are optional. Many sequential tasks work perfectly with just task instruction files and file-based outputs.

#### Task Identification Conventions

```
task-<number>-<description>.md
```

Examples:
- `task-01-directory-setup.md`
- `task-1.5-cross-reference-verification.md`
- `task-07-plan-extraction.md`

Use decimal numbering (task-1.5) for tasks inserted between existing numbered tasks. The number indicates execution order, not importance.

---

### Core Concepts

#### 1. Sequential Execution Model

A single session completes all work in linear step order:

```
Step 1 → Verify → Step 2 → Verify → [PAUSE] → Step 3 → Verify → ... → Commit → Report
```

**Key properties:**
- **Linear progression** — Steps execute in order (1, 2, 3, ...)
- **Verification gates** — Each step includes a verification check before proceeding
- **PAUSE points** — Defined stops where the session waits for conductor/user review
- **Single session** — All work happens in one context window (with checkpoint commits if context runs low)

#### 2. Context Budget Management

Every task must estimate its context usage and plan for the possibility of running out.

**Token estimation methodology:**
- Count source files to read (each file ≈ tokens × 4 characters per token)
- Add instruction overhead (~5-10k tokens for the task file itself)
- Add verification/SQL overhead (~2-5k tokens)
- Add output generation (reports, created files)
- Apply 1.3× safety multiplier

**Example budget:**
```
Task 03: Extract Testing Patterns
- Source files (6 files): ~45k tokens
- Task instructions: ~8k tokens
- Verification/SQL: ~3k tokens
- Output (RAG files): ~15k tokens
- Buffer (1.3×): ~92k tokens total
- Context window: 1M → Usage: 9%
- Risk level: Low
```

**Risk levels:**
| Usage | Risk | Action |
|-------|------|--------|
| < 50% | Low | No special monitoring needed |
| 50-70% | Medium | Monitor after each major step |
| 70-85% | High | Checkpoint commit ready, consider splitting |
| > 85% | Critical | Execute checkpoint commit, create continuation task |

**Checkpoint commit pattern:**
When context exceeds 85%, commit partial work and create a continuation task:

```bash
git add [partial work files]
git commit -m "checkpoint: task-XX partial - processed N/M items

Completed steps 1-3 of 5.
Continuing in task-XX-2.

Task: task-XX (partial)
Checkpoint at step 3/5"
```

Reference: `docs/knowledge-base/implementation/checkpoint-commit-pattern.md`

#### 3. Review Checkpoints

PAUSE points are strategic stops where the session waits for external review. They exist at decision points where conductor/user context is needed.

**PAUSE-and-tell-user pattern:**

```markdown
### PAUSE Point: [Description of what needs review]

**What was completed:** [Summary of work done so far]
**What needs review:** [Specific question or decision needed]
**Options (if applicable):**
1. [Option A with implications]
2. [Option B with implications]

**Action:** STOP WORK. Tell user: "[Specific message for conductor]"
```

**When to use PAUSE points:**
- After creating a proposal that needs approval before extraction
- When encountering an ambiguous requirement
- When a decision affects downstream steps
- At natural phase boundaries (analysis → extraction → verification)

**When NOT to use PAUSE points:**
- Between every step (too granular)
- For routine verification checks (these are inline, not pauses)
- When the task instructions provide clear guidance for the decision

#### 4. Error Handling

Sequential tasks use a simple error model: **stop, document, escalate.**

```
Error encountered
  → Document the error (what happened, what was attempted)
  → Document proposed fix (if known)
  → STOP WORK
  → Tell user/conductor: "[Error description + proposed fix]"
  → Wait for guidance before continuing
```

This is deliberately simpler than the parallel design's 5-retry autonomous recovery loop. In sequential execution, the user is available and can make decisions quickly. Autonomous retry is not needed.

**Common error categories:**
| Error Type | Response |
|-----------|----------|
| File not found | Check path, search for alternatives, escalate if not found |
| Pre-existing test failures | Document which tests fail, proceed if unrelated to task |
| Unclear classification | Present decision tree, escalate to user |
| Context exhaustion | Checkpoint commit, create continuation task |
| SQL/database error | Document query and error, escalate |

---

## Part 2: Task Instruction File Format

### Template Structure

This is the reusable skeleton for any sequential task instruction file:

```markdown
# Task [N]: [Clear Imperative Title]

**Parallel-safe:** Yes/No (whether this can run concurrently with other tasks)
**Token estimate:** ~XXk tokens (XX% of 1M context window)
**Dependencies:** [List of prerequisite tasks or conditions]

## Objective

[1-2 paragraph description of what this task accomplishes]

**Deliverables:**
1. [Specific deliverable 1]
2. [Specific deliverable 2]
3. [Specific deliverable 3]

**Key deviations from design:** [Any deliberate departures from the design doc]

## Context Reference

- **Implementation Plan:** [path with section reference]
- **Design Document:** [path with section reference]
- **Templates Used:** [list of template numbers/names]

## Prerequisites

[Verification checks that must pass before starting]

## Actions

### Step 1: [Imperative Action Title]

[Description and instructions]

**Verify:**
[How to check this step succeeded]

### Step 2: [Imperative Action Title]

[Description and instructions]

**Verify:**
[How to check this step succeeded]

### PAUSE Point: [Review Description]

[What needs review and what to tell the user]

### Step N: [Final Step]

[Description and instructions]

## Verification Checklist

[Complete checklist using Verify/Expected/If failed format]

## Commit

[Commit message template with conventional commit format]

## Completion Report

[Report template to fill out]

## Context Budget

[Token estimates with monitoring checkpoints]

## Troubleshooting

[Common issues and resolutions]

## Notes

[Additional context, warnings, or tips]
```

---

### Detailed Section Specifications

#### Header

**Purpose:** Immediate clarity on scope, constraints, and prerequisites.

**Format:**
```markdown
# Task [N]: [Clear Imperative Title]

**Parallel-safe:** Yes/No
**Token estimate:** ~XXk tokens (XX% of 1M context window)
**Dependencies:** Task X complete, [other conditions]
```

**Guidelines:**
- Title uses imperative mood: "Extract Testing Patterns", "Create Directory Structure"
- Token estimate includes all reading + writing + verification
- Dependencies list every prerequisite — don't assume context from other tasks
- Parallel-safe indicates whether this task can run alongside others without file conflicts

**Example:**
```markdown
# Task 03: Extract Testing Patterns from Source Files

**Parallel-safe:** Yes (reads from docs2/, writes to docs/knowledge-base/testing/)
**Token estimate:** ~92k tokens (9% of 1M context window)
**Dependencies:** Task 01 complete (directory structure exists), Task 1.5 complete (cross-references verified)
```

#### Objective

**Purpose:** Explain what the task accomplishes and why.

**Format:**
- 1-2 paragraph description
- Numbered deliverables list
- Key deviations from design (if any)
- Critical success criteria

**Example:**
```markdown
## Objective

Extract testing patterns, TDD guidelines, and coverage requirements from legacy
documentation files into granular RAG-optimized knowledge base files. Each source
pattern becomes a standalone file with YAML frontmatter for MCP server queries.

**Deliverables:**
1. 8-12 RAG files in `docs/knowledge-base/testing/`
2. Extraction proposal reviewed and approved
3. All files ingested into local-rag MCP server
4. Completion report with verification results

**Key deviations from design:** Simplified YAML frontmatter — using `status` field
instead of separate `lifecycle` field per RAG template standardization.
```

#### Context Reference

**Purpose:** Point to source documents so the session has full context.

**Format:**
```markdown
## Context Reference

- **Implementation Plan:** `docs/plans/implementation/2026-02-02-plan.md`
  - Section: Task 3 (detailed instructions)
  - Master Templates section (verification formats)
- **Design Document:** `docs/plans/designs/docs-reorganization-design.md`
  - Section: RAG Knowledge Base (target structure)
- **Templates Used:** Template 3 (verification), Template 4 (RAG metadata),
  Template 5 (commit format), Template 7 (proposals)
```

**Guidelines:**
- Always include specific section references, not just file paths
- List which numbered templates are used (for projects with template systems)
- Include path to previous task's completion report if there's a dependency

#### Prerequisites

**Purpose:** Gate-check before allowing work to start. Fail fast if preconditions aren't met.

**Format:**
```markdown
## Prerequisites

### Verify directory structure
```bash
test -d docs/knowledge-base/testing || echo "ERROR: testing/ directory missing"
# Expected: No output (directory exists)
```

### Verify dependencies complete
```sql
SELECT task_id, status FROM migration_tasks
WHERE task_id IN ('task-01', 'task-1.5')
AND status != 'complete';
-- Expected: Empty result set (both complete)
```

### Verify source files exist
```bash
ls docs2/guidelines/implementation/testing-patterns-planning.md
# Expected: File listed (source material exists)
```
```

**Guidelines:**
- Include bash checks for file/directory existence
- Include SQL checks for database state (if using comms-link)
- Show expected output in comments
- If any check fails, STOP and escalate before starting work

#### Actions (Steps)

**Purpose:** The core work instructions, broken into numbered steps.

**Format per step:**
```markdown
### Step N: [Imperative Action Title]

[1-2 sentence description of what this step accomplishes]

[Detailed instructions — may include code blocks, file paths, specific commands]

**Verify:**
```bash
[command to check completion]
# Expected: [what success looks like]
```
```

**Guidelines:**
- Steps numbered sequentially (Step 1, Step 2, Step 3...)
- Sub-steps use decimal numbering (Step 2.1, 2.2, 2.3) for complex steps
- Each step includes inline verification — don't defer all checking to the end
- Keep steps atomic — each step should accomplish one logical unit of work
- Include specific file paths, not vague references

**Example:**
```markdown
### Step 3: Create RAG Files from Approved Patterns

For each pattern approved in the extraction proposal, create a standalone
RAG file in `docs/knowledge-base/testing/`.

**File format:**
```markdown
---
id: testing-[pattern-name]
created: 2026-02-03
category: testing
parent_topic: [grouping]
related: []
status: valid
---

# [Pattern Title]

[Pattern content extracted from source]
```

**Verify:**
```bash
ls docs/knowledge-base/testing/*.md | wc -l
# Expected: 8-12 files (matching approved proposal count)
```
```

#### PAUSE Points

**Purpose:** Strategic stops for conductor/user review.

**Format:**
```markdown
### PAUSE Point: Review Extraction Proposal

**Completed so far:**
- Read 6 source files (~45k tokens)
- Identified 11 patterns across 4 categories
- Created extraction proposal at `temp/task-03-proposal.md`

**Needs review:**
- Are all 11 patterns correctly categorized?
- Should any patterns be merged or split?
- Are the proposed RAG file paths correct?

**Action:** STOP WORK. Tell user:
"Task 03 extraction proposal ready for review at temp/task-03-proposal.md.
Found 11 patterns in 4 categories. Please review and approve before extraction."
```

**With optional SQL (if using comms-link):**
```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'I', 'Extraction proposal ready at temp/task-03-proposal.md. Found 11 patterns in 4 categories. Awaiting review.');
-- Then: STOP WORK and tell user to alert orchestration
```

#### Verification Checklist

**Purpose:** Final gate before marking the task complete. All checks must pass.

**Format (Verify/Expected/If failed):**
```markdown
## Verification Checklist

- [ ] RAG proposals created with correct structure
      **Verify:** `head -20 docs/implementation/proposals/[task-id]-rag-*.md`
      **Expected:** Each proposal has frontmatter (type: rag-addition, task_id, target_category) and <!-- BEGIN/END RAG FILE --> delimiters
      **If failed:** Review proposal template and regenerate

- [ ] KB overlap checking completed
      **Verify:** `grep -A3 "## RAG Match List" docs/implementation/proposals/[task-id]-rag-*.md`
      **Expected:** Each proposal includes match list (either "No matches" or list with scores)
      **If failed:** Run KB queries before proceeding

- [ ] RAG files extracted and ingested
      **Verify:** `ls docs/knowledge-base/testing/*.md | wc -l` AND `query_documents("pattern", limit=5)`
      **Expected:** File count matches approved proposal count, retrieval scores < 0.3
      **If failed:** Re-extract from proposals, re-run ingestion with tools/bulk-ingest-rag.sh

- [ ] No broken cross-references
      **Verify:** `grep -r "docs2/" docs/knowledge-base/testing/`
      **Expected:** No results (no references to old location)
      **If failed:** Update references to use new paths

- [ ] Conductor approval received
      **Verify:** Check task_messages for approval message with `review_approved` state
      **Expected:** Proposals approved before extracting to final location
      **If failed:** Request conductor review of proposal content and overlap checks
```

**Guidelines:**
- Checkbox format (`- [ ]`) for copy-paste tracking
- Three-part structure is mandatory: Verify, Expected, If failed
- Bash commands shown exactly as typed
- Expected results must be specific and testable (not "looks correct")
- Include recovery instructions in "If failed"
- ALL checks must pass before marking task complete

#### Commit Section

**Purpose:** Pre-formatted commit message following conventional commit style.

**Format:**
```markdown
## Commit

```bash
git add [specific files]
git commit -m "$(cat <<'EOF'
feat(docs): extract testing patterns into RAG knowledge base

- Created 11 RAG files in docs/knowledge-base/testing/
- Extracted patterns from 6 source files in docs2/
- Added YAML frontmatter with category metadata
- Ingested all files into local-rag MCP server
- Verified cross-references and file integrity

Task: task-03
Files: 11 created, 0 modified, 0 deleted
Report: docs/implementation/reports/task-03-report.md

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```
```

**Guidelines:**
- Use conventional commit type (feat/fix/refactor/docs)
- Scope in parentheses reflects the area of change
- Imperative verb in present tense for the subject line
- Bullet list of all changes in the body
- Include task ID, file counts, and report path
- Co-Authored-By footer with the model that did the work

**Commit frequency:**
- Commit after each logical unit of work
- Commit before requesting review (at PAUSE points)
- Commit at checkpoint completion
- Do NOT commit after every single file (too granular)
- Do NOT wait until task complete (too coarse)
- Typical: 3-10 commits per task

**Good commits:**
```bash
git commit -m "task-03: extract testing patterns from source files

Extracted 12 granular patterns to knowledge-base/testing/
Excluded 3 outdated patterns per review guidance

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

**Bad commits (avoid):**
```bash
git commit -m "fix"
git commit -m "wip"
git commit -m "update files"
git commit -m "task-03"  # Too vague — describe what was done
```

#### Completion Report

**Purpose:** Structured record of what was done, for historical reference and future planning.

**Location:** `docs/implementation/reports/task-XX-report.md`

**Format:**
```markdown
## Completion Report

Create `docs/implementation/reports/task-XX-report.md`:

```markdown
# Task XX Completion Report

**Task ID:** task-XX
**Task Name:** [Task name]
**Date:** YYYY-MM-DD
**Status:** Complete
**Duration:** [start time → end time, or approximate duration]

---

## Summary

[1-2 sentence summary of what was accomplished]

---

## Work Completed

**Deliverables created:**
- [List of files created with paths]

**Changes made:**
- [List of significant modifications]

**Features implemented:**
- [If applicable, or omit section]

---

## Verification Results

**Quality checks:**
- [x] RAG files created with correct frontmatter ✅
- [x] All files ingested into local-rag ✅
- [x] No broken cross-references ✅
- [x] File count matches proposal ✅

**Tests:**
- Total: [count] | Passing: [count] | Failing: 0
- [Test output location if applicable]

**Commits:**
- Total commits: [count]
- Commit SHAs: [list]

---

## Files Created/Modified

**Created:** [list with line counts]
**Modified:** [list]
**Moved:** [list]
**Deleted:** [list]

---

## Issues Encountered

**Issues and resolutions:**
- [None, or: Issue description + resolution + time impact]

**Deviations from plan:**
- [None, or: Describe deviation and justify]

---

## Learnings

**What went well:**
- [Patterns that worked, efficient approaches]

**What could be improved:**
- [Areas for improvement, suggested optimizations]

**Proposals:**
- [Memory proposals, CLAUDE.md additions, or "None"]

---

## Context Budget

**Estimated:** ~XXk tokens
**Actual:** ~XXk tokens
**Assessment:** [On target / Under budget / Over budget — explanation]

---

## Artifacts

**Code:**
- Git commits: [SHAs]
- Lines added: [count]
- Lines removed: [count]

**Documentation:**
- Reports: [list]
- [Other documentation created]

**Temporary files:**
- [Files in temp/ related to this task — can be deleted after review]

---

## Next Steps

[What happens after this task, or "None — final task"]
```
```

#### Context Budget Section

**Purpose:** Token estimates with monitoring checkpoints so the session knows when to worry.

**Format:**
```markdown
## Context Budget

**Estimated total:** ~92k tokens (9% of 1M context window)

**Breakdown:**
- Source file reads: ~45k tokens
- Task instructions: ~8k tokens
- Verification/SQL: ~3k tokens
- Output generation: ~15k tokens
- Buffer (1.3×): ~21k tokens

**Monitoring checkpoints:**
- After Step 2 (read source files): ~53k tokens (27%)
- After Step 4 (create RAG files): ~75k tokens (38%)
- After Step 6 (verification): ~85k tokens (43%)

**Risk level:** Low

**If context > 170k (85%):**
Use checkpoint commit pattern from `docs/knowledge-base/implementation/checkpoint-commit-pattern.md`.
Create continuation task (task-XX-2) with remaining steps.
```

#### Troubleshooting Section

**Purpose:** Pre-document common issues the session might encounter.

**Format:**
```markdown
## Troubleshooting

### Issue: Source file not found at expected path

**Symptom:** `ls` returns "No such file or directory" for a source file path
**Resolution:**
1. Search for the file: `find docs2/ -name "*pattern*"`
2. Check if file was moved in a previous task
3. If genuinely missing, escalate to conductor

### Issue: Context exhaustion mid-task

**Symptom:** Context window approaching 85% before all steps complete
**Resolution:**
1. Checkpoint commit current progress
2. Create continuation task file (task-XX-2.md)
3. Document which steps are complete in the checkpoint commit
4. Mark task as partially complete in database (if using comms-link)

### Issue: Pre-existing test failures

**Symptom:** Tests fail that are unrelated to current task
**Resolution:**
1. Document which tests fail and their error messages
2. Verify failures exist on the base branch (not introduced by this task)
3. Proceed if failures are pre-existing and unrelated
4. Note pre-existing failures in completion report
```

---

## Part 3: Reusable Patterns

### Verification Checklist Pattern

The Verify/Expected/If failed triple is the standard verification format across both sequential and parallel task designs.

```markdown
- [ ] [What to check — specific, testable statement]
      **Verify:** `[exact bash command or specific instruction]`
      **Expected:** [concrete expected result — numbers, specific output, absence of errors]
      **If failed:** [recovery steps — what to do, where to look, who to escalate to]
```

**Rules:**
- Every verification item must be independently checkable
- "Expected" must be specific enough that pass/fail is unambiguous
- "If failed" must provide actionable recovery, not just "check the output"
- All items must pass before the task is marked complete

### Commit Message Pattern

Conventional commits with task metadata:

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <imperative subject line>

- [Change 1]
- [Change 2]
- [Summary statistics]

Task: task-XX
Files: N created, M modified, X deleted
Report: docs/implementation/reports/task-XX-report.md

Co-Authored-By: Claude <model> <noreply@anthropic.com>
EOF
)"
```

**Types:** feat, fix, refactor, docs, test, chore
**Scope:** Area of change (docs, api, frontend, backend, testing)

### Completion Report Pattern

Every task generates a completion report at `docs/implementation/reports/task-XX-report.md` with these mandatory sections:

1. **Header** — Task ID, name, date, status, duration
2. **Summary** — 1-2 sentences
3. **Work Completed** — Deliverables, changes, features (structured sub-sections)
4. **Verification Results** — Quality checks with ✅/❌, test counts, commit SHAs
5. **Files Created/Modified** — With counts
6. **Issues Encountered** — Resolutions + deviations from plan
7. **Learnings** — What went well, what could improve, proposals
8. **Context Budget** — Estimated vs actual
9. **Artifacts** — Git SHAs, line counts, documentation, temp files
10. **Next Steps** — Or "None"

### Learnings & Proposals Pattern

When a task discovers something worth remembering, capture it in a proposal:

**Location:** `docs/implementation/proposals/task-XX-learnings.md`

**Structure:**
```markdown
# Task XX: [Feature] Learnings & Proposals

**Date:** YYYY-MM-DD
**Task:** Task XX - [Title]

## Memory Proposals

### Memory 1: [Trigger Name]

**Trigger Condition (WHEN):** [When this knowledge applies]
**Action to Take:** [What to do]
**Rationale:** [Why it matters]
**Example:** [Concrete scenario]

## CLAUDE.md Proposals

### Proposal 1: [Rule Title]

**Proposed Rule:** [Exact text]
**Target Section:** [Which section of CLAUDE.md]
**Rationale:** [Why mandatory]
**Example of Issue:** [What goes wrong without it]

## Notes

[Other findings, edge cases, observations]
```

### Context Budget Monitoring Pattern

Embed monitoring checkpoints at key points in the task:

```markdown
**Context checkpoint (after Step N):**
- Estimated usage: ~XXk tokens (XX%)
- If > 85%: Use checkpoint commit, create continuation task
- If > 70%: Monitor closely, reduce verbosity in remaining steps
- If < 50%: On track, no action needed
```

### Error Report Pattern

When an error occurs, document it with this structured template before escalating. This ensures the user/conductor has full context to propose a fix.

**Format:**
```markdown
ERROR REPORT

**Task:** [task-id]
**Step:** [which step failed]
**Error Type:** [TypeError|FileNotFound|ValidationError|DatabaseError|etc.]
**Error Message:** [exception message or description]

**Context:**
- Input values: [relevant variables and their values]
- State before error: [what was the system state]
- Expected behavior: [what should have happened]
- Actual behavior: [what actually happened]

**Stack Trace (if applicable):**
```
[full stack trace or error output]
```

**Relevant Code (if applicable):**
```[language]
[code snippet showing where error occurred, ~10 lines context]
```

**Suggested Investigation:**
[Session's analysis of possible causes, if any insights available]
```

**Example:**
```markdown
ERROR REPORT

**Task:** task-03
**Step:** Step 5 - Ingest files into local-rag
**Error Type:** FileNotFound
**Error Message:** No such file: docs/knowledge-base/testing/tdd-patterns.md

**Context:**
- Input values: file path = "docs/knowledge-base/testing/tdd-patterns.md"
- State before error: 10 of 11 files created successfully
- Expected behavior: All 11 RAG files exist for ingestion
- Actual behavior: File 11 missing — Step 3 may have failed silently

**Suggested Investigation:**
Check Step 3 output for errors during file creation. The file may have
been skipped due to a YAML frontmatter formatting issue in the source.
```

**Guidelines:**
- Use this template for ALL errors, not just code errors — file-not-found, unclear requirements, and tool failures all benefit from structured reporting
- Include enough context that someone unfamiliar with the task can understand the problem
- "Suggested Investigation" is optional but valuable — share your hypothesis even if unsure
- After documenting, STOP WORK and escalate per the error handling model

### File Creation Rules

Formalized rules for where to create files during task execution.

**1. Proposals (if proposal-first workflow):**
```bash
# CORRECT: Create in proposals directory
docs/implementation/proposals/task-XX-extraction-proposal.md

# WRONG: Create directly in final location before review
docs/knowledge-base/testing/pattern.md  # Only AFTER review approval
```

**2. After approval, move to final location:**
```bash
# After user/conductor approval of proposal
mv docs/implementation/proposals/task-XX-[file].md docs/knowledge-base/[category]/[file].md
```

**3. Reports always in docs/implementation/reports/:**
```bash
docs/implementation/reports/task-XX-report.md          # Completion reports
docs/implementation/reports/task-XX-checkpoint-1.md     # Checkpoint reports
```

**4. Tracking files in temp/:**
```bash
temp/task-XX-rag-files.txt    # Lists of created files
temp/task-XX-notes.txt        # Working notes (cleared on reboot)
```

**5. Task instruction files in docs/plans/implementation/:**
```bash
docs/plans/implementation/task-01-directory-setup.md
docs/plans/implementation/task-03-extract-patterns.md
```

### Prescriptive vs Adaptive Guidance

When using this template, some parts must be followed exactly while others should be adapted to your project.

**Prescriptive (copy exactly):**
- Verification triple format (Verify/Expected/If failed)
- Commit message structure (conventional commit + task metadata + Co-Authored-By)
- Checkpoint commit pattern (commit, create continuation task)
- Error report template structure
- PAUSE point format (Completed/Needs review/Action)
- Completion report section structure

**Adaptive (vary by project):**
- Step content (task-specific work)
- File organization (project structure)
- Review criteria (project quality standards)
- Testing approach (language/framework-specific)
- Source file paths (project-specific)
- Troubleshooting scenarios (task-specific failure modes)

**When in doubt, ask:**
1. Does this ensure consistency across tasks? → **Prescriptive** (copy exactly)
2. Does this depend on project conventions? → **Adaptive** (follow your project)
3. Does this prevent errors or data loss? → **Prescriptive** (copy exactly)

### PAUSE-for-Review Pattern

The standard pattern for requesting conductor/user review:

```markdown
### PAUSE Point: [What Needs Review]

**Completed:** [Bullet list of what's done]
**Needs review:** [Specific question or decision]
**Deliverable:** [File path to review material]

**Action:** STOP WORK. Tell user: "[Specific, actionable message]"
```

**Optional SQL variant:**
```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-XX', 'I', '[Complete message with file counts and paths]');
-- STOP WORK. Tell user to check orchestration.
```

**Rules for PAUSE messages:**
- Include complete context (file counts, paths, specific numbers)
- Never make the conductor ask follow-up questions
- State exactly what decision is needed
- Include options if applicable

---

## Part 4: Optional Database Coordination

### When to Add Database Coordination

Add comms-link database coordination when:
- The task is part of a multi-task orchestrated workflow
- An conductor session needs to track progress across tasks
- Multiple sessions need to communicate (even if tasks are sequential)
- You need persistent state across session restarts

Skip database coordination when:
- The task is standalone (no conductor)
- The user is directly supervising the work
- PAUSE-and-tell-user is sufficient for review checkpoints

### Lightweight Comms-Link Usage

Sequential tasks use only two tables (compared to the parallel design's full state machine):

1. **migration_tasks** — Track task status (pending → in_progress → complete)
2. **task_messages** — Communication between sessions

No `coordination_status` table. No hooks. No background subagents.

### SQL Patterns

#### Mark Task In-Progress (Pattern B)

```sql
UPDATE migration_tasks
SET status = 'in_progress',
    worked_by = '[session-identifier]',
    started_at = datetime('now')
WHERE task_id = 'task-XX';
```

Run at the start of the task, after prerequisites pass.

#### Send Review Request Message (Pattern C)

```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-XX', 'I',
  'Extraction proposal ready at temp/task-XX-proposal.md. '
  || 'Found 11 patterns in 4 categories. '
  || 'Awaiting review before proceeding to extraction.');
-- Then: STOP WORK and tell user to alert orchestration
```

Use at PAUSE points. Message must be self-contained — include counts, paths, and the specific decision needed.

#### Check for Conductor Messages Between Steps

```sql
SELECT message, created_at
FROM task_messages
WHERE task_id = 'task-XX'
  AND from_session = 'conductor'
ORDER BY created_at DESC
LIMIT 5;
```

Optionally check between major steps to see if the conductor has sent guidance. If no messages, proceed with the task instructions as written.

#### Mark Task Complete (Pattern D)

```sql
UPDATE migration_tasks
SET status = 'complete',
    completed_at = datetime('now'),
    report_path = 'docs/implementation/reports/task-XX-report.md'
WHERE task_id = 'task-XX';

INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-XX', 'I',
  'Task XX complete. Created 11 RAG files, all verification checks passed. '
  || 'Report at docs/implementation/reports/task-XX-report.md.');
```

Run after verification checklist passes and commit is made.

---

## Part 5: Complete Example

### Example: Sequential Documentation Extraction Task

This is a complete, working task instruction file demonstrating all patterns:

```markdown
# Task 03: Extract Testing Patterns into RAG Knowledge Base

**Parallel-safe:** Yes (reads from docs2/guidelines/, writes to docs/knowledge-base/testing/)
**Token estimate:** ~92k tokens (9% of 1M context window)
**Dependencies:** Task 01 complete (directory structure created), Task 1.5 complete (cross-references verified)

## Objective

Extract testing-related patterns, TDD guidelines, and coverage requirements from legacy
documentation into granular RAG-optimized knowledge base files. Each distinct pattern
becomes a standalone file with YAML frontmatter for MCP server queries.

**Deliverables:**
1. 8-12 RAG files in `docs/knowledge-base/testing/`
2. Extraction proposal reviewed and approved by conductor
3. All files ingested into local-rag MCP server
4. Completion report with verification results

## Context Reference

- **Implementation Plan:** `docs/plans/implementation/2026-02-02-docs-reorg-implementation.md`
  - Section: Task 3 (detailed instructions)
  - Master Templates section (verification formats)
- **Design Document:** `docs/plans/designs/docs-reorganization-design.md`
  - Section: RAG Knowledge Base (target structure)
- **Templates Used:** Template 3 (verification), Template 4 (RAG metadata),
  Template 5 (commit format), Template 7 (proposals)

## Prerequisites

### Verify directory structure
```bash
test -d docs/knowledge-base/testing || echo "ERROR: testing/ directory missing"
# Expected: No output (directory exists)
```

### Verify dependencies
```sql
SELECT task_id, status FROM migration_tasks
WHERE task_id IN ('task-01', 'task-1.5') AND status != 'complete';
-- Expected: Empty result set
```

### Verify source files
```bash
ls docs2/guidelines/implementation/testing-patterns-planning.md \
   docs2/guidelines/implementation/mandatory-testing-requirements.md \
   docs2/guidelines/implementation/failing-tests-as-specifications.md
# Expected: All 3 files listed
```

## Actions

### Step 1: Mark task in-progress

```sql
UPDATE migration_tasks
SET status = 'in_progress', worked_by = 'execution-session', started_at = datetime('now')
WHERE task_id = 'task-03';
```

### Step 2: Read and analyze source files

Read all 6 source files. For each file, note:
- What distinct patterns it contains
- Whether patterns are still valid or outdated
- Which RAG category each pattern belongs to
- Any cross-references to other patterns

Save scratch notes to `temp/task-03-analysis-notes.txt`.

**Verify:**
```bash
test -f temp/task-03-analysis-notes.txt && echo "OK"
# Expected: OK
```

### Step 3: Create Extraction Proposal

Create `docs/implementation/proposals/[task-id]-extraction-analysis.md` with:
- Each identified pattern listed with:
  - Pattern name
  - Status: Valid / Outdated / Needs Update
  - Source file and line numbers
  - Proposed RAG file path
  - KB overlap pre-check (run `query_documents("[topic]", limit=10)` at 0.4 threshold)
  - Brief rationale

**KB Pre-screening Guidance:**
For each pattern, query the knowledge base BEFORE proposing:
```bash
# Example
query_documents("TDD testing patterns", limit=10)
# Expected: No matches at 0.4 threshold, OR
# If matches: Record scores < 0.3 indicating potential consolidation with existing file
```

**Verify:**
```bash
grep -c "^### Pattern" docs/implementation/proposals/[task-id]-extraction-analysis.md
# Expected: 8-12 patterns listed
```

### PAUSE Point: Review Extraction Proposal

**Completed:**
- Read 6 source files
- Identified patterns and categorized them
- Created extraction proposal with status assessments

**Needs review:**
- Are pattern categorizations correct?
- Should any patterns be merged or split?
- Are proposed file paths appropriate?

**Action:** STOP WORK.

```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'I',
  'Extraction proposal ready at temp/task-03-extraction-proposal.md. '
  || 'Found 11 patterns (9 valid, 2 outdated). Awaiting review.');
```

Tell user: "Task 03 proposal ready. Please review temp/task-03-extraction-proposal.md
and send approval via orchestration."

### Step 4: Check for conductor feedback

```sql
SELECT message FROM task_messages
WHERE task_id = 'task-03' AND from_session = 'conductor'
ORDER BY created_at DESC LIMIT 3;
```

Apply any feedback from conductor before proceeding.

### Step 5: Create RAG Proposals (One Per File)

**CRITICAL:** Do NOT create RAG files directly in `docs/knowledge-base/`. Instead, create PROPOSALS in `docs/implementation/proposals/` that contain the RAG file content.

**Rationale:** One proposal per RAG file enables fine-grained extraction/exclusion control by conductor. Allows rejection of individual files without affecting others.

**For each [N] approved pattern:** Create [N] separate proposal files:
- `docs/implementation/proposals/[task-id]-rag-{pattern-name-1}.md`
- `docs/implementation/proposals/[task-id]-rag-{pattern-name-2}.md`
- ... (one for each RAG file)

**Proposal structure:**
```markdown
---
type: rag-addition
task_id: [task-id]
created: YYYY-MM-DD
target_category: testing
target_filename: [pattern-name].md
---

# RAG Proposal: {Pattern Title}

## Reasoning

[Why this belongs in KB. What pattern discovered, why future sessions would benefit.]

## RAG Match List (0.4 threshold)

| Existing File | Score | Relevance |
|---|---|---|
| [matching files from KB, if any] | [score] | [relationship] |

[Or: "No existing entries matched at 0.4 threshold."]

## Proposed RAG File

Target path: `docs/knowledge-base/testing/[pattern-name].md`

<!-- BEGIN RAG FILE -->
---
id: testing-[pattern-name]
created: YYYY-MM-DD
category: testing
parent_topic: [Testing Patterns | TDD | Coverage | Test Infrastructure]
related: []
status: valid
---

[Full RAG file content. Self-contained, includes examples, no external refs.]

<!-- END RAG FILE -->
```

**Pre-screening (before creating each proposal):**
1. Query KB: `query_documents("[pattern topic]", limit=10)` at 0.4 relevance threshold
2. If matches found (score < 0.3): Record in RAG Match List, consider updating existing file instead
3. If no matches (> 0.4): Proceed with new file proposal

**Verify:**
```bash
ls docs/implementation/proposals/[task-id]-rag-*.md | wc -l
# Expected: Matches approved pattern count
```

### Step 6: Request Conductor Review

Commit proposals and request review:
```bash
git add docs/implementation/proposals/[task-id]-rag-*.md
git commit -m "[task-id]: RAG proposals ready for review"
```

```sql
INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'I',
  'RAG proposals ready at docs/implementation/proposals/task-03-rag-*.md. '
  || '[N] proposals created with KB overlap checking completed. Awaiting conductor approval.');
```

### Step 7: Ingest Approved RAG Files

After conductor approves proposals, extract RAG files and ingest:

**Extract from proposals:**
```bash
for proposal in docs/implementation/proposals/[task-id]-rag-*.md; do
  # Extract content between <!-- BEGIN RAG FILE --> and <!-- END RAG FILE -->
  # Write to docs/knowledge-base/testing/[filename].md
done
```

**Ingest into local-rag:**
```bash
./tools/bulk-ingest-rag.sh
```

Verify ingestion by querying for a known pattern:
```
Query local-rag: "TDD testing pattern"
# Expected: Score < 0.3 for matching file
```

### Step 7: Run verification checklist

[See Verification Checklist section below]

### Step 8: Create completion report

Create `docs/implementation/reports/task-03-report.md` using the completion report template.

### Step 9: Commit

```bash
git add docs/implementation/proposals/task-03-rag-*.md \
        docs/knowledge-base/testing/ \
        docs/implementation/reports/task-03-report.md

git commit -m "$(cat <<'EOF'
feat(docs): extract testing patterns into RAG knowledge base

- Created 11 RAG proposals in docs/implementation/proposals/
- Each proposal includes KB overlap pre-screening (0.4 threshold)
- Conductor approved proposals and RAG extraction
- Extracted 11 RAG files to docs/knowledge-base/testing/
- Added YAML frontmatter with category metadata
- Ingested all files into local-rag MCP server
- All verification checks passed

Task: task-03
Proposals: 11 created (pre-conductor review)
Files: 11 created in knowledge-base/, 0 modified, 0 deleted
Report: docs/implementation/reports/task-03-report.md

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

### Step 10: Mark complete

```sql
UPDATE migration_tasks
SET status = 'complete', completed_at = datetime('now'),
    report_path = 'docs/implementation/reports/task-03-report.md'
WHERE task_id = 'task-03';

INSERT INTO task_messages (task_id, from_session, message)
VALUES ('task-03', 'I',
  'Task 03 complete. Created 11 RAG files in testing/. '
  || 'All verification passed. Report at docs/implementation/reports/task-03-report.md.');
```

## Verification Checklist

- [ ] RAG files have correct YAML frontmatter
      **Verify:** `head -8 docs/knowledge-base/testing/*.md`
      **Expected:** Each file has `---` block with id, created, category, parent_topic, status
      **If failed:** Regenerate frontmatter using Template 4 format

- [ ] All approved patterns extracted
      **Verify:** Compare file count against approved proposal
      **Expected:** File count matches proposal count exactly
      **If failed:** Identify missing patterns, create remaining files

- [ ] No references to old docs2/ paths
      **Verify:** `grep -r "docs2/" docs/knowledge-base/testing/`
      **Expected:** No results
      **If failed:** Update all references to new paths

- [ ] Files ingested into local-rag
      **Verify:** Query local-rag for "TDD pattern"
      **Expected:** Score < 0.3 for matching file
      **If failed:** Re-run `tools/bulk-ingest-rag.sh`

- [ ] Completion report exists
      **Verify:** `test -f docs/implementation/reports/task-03-report.md && echo "OK"`
      **Expected:** OK
      **If failed:** Create report using completion report template

## Context Budget

**Estimated total:** ~92k tokens (9% of 1M)

**Breakdown:**
- Source file reads (6 files): ~45k tokens
- Task instructions: ~8k tokens
- Proposal + verification: ~5k tokens
- RAG file generation: ~15k tokens
- Commit + report: ~5k tokens
- Buffer: ~14k tokens

**Monitoring checkpoints:**
- After Step 2 (read sources): ~53k (27%)
- After Step 5 (create RAG files): ~75k (38%)
- After Step 7 (verification): ~85k (43%)

**Risk level:** Low

## Troubleshooting

### Issue: Source file path changed

**Symptom:** File not found at expected docs2/ path
**Resolution:**
1. Search: `find docs2/ -name "*testing*"`
2. Check if file was moved by a previous task
3. Escalate to conductor if file is genuinely missing

### Issue: Pattern classification unclear

**Symptom:** A pattern could belong to testing/ or implementation/
**Resolution:**
1. Check if the pattern is primarily about *how to test* (→ testing/)
   or *how to implement* (→ implementation/)
2. If still unclear, include in extraction proposal with both options
3. Let conductor decide during PAUSE review

### Issue: Local-rag ingestion fails

**Symptom:** `bulk-ingest-rag.sh` returns errors
**Resolution:**
1. Check file encoding (must be UTF-8)
2. Verify YAML frontmatter is valid (no tabs, proper indentation)
3. Try ingesting files individually to isolate the problem
4. Check local-rag MCP server is running

## Notes

- Save scratch analysis notes to `temp/` (cleared on reboot)
- Do NOT create files in docs/ root — use subdirectories only
- RAG files should be self-contained (no required reading order)
- Each file covers ONE concept (granularity enables precise retrieval)
```

---

## Part 6: Adaptation Guide

### Customizing for Your Project

When using this template for a new task, work through these steps:

**Step 1: Define your task**
```
Questions to answer:
1. What work needs to be done? (clear deliverables)
2. How many steps? (estimate scope)
3. What dependencies exist? (prerequisite tasks or conditions)
4. Where are the source materials? (file paths, APIs, databases)
5. What files will be created or modified? (output inventory)
```

**Step 2: Design review checkpoints**
```
Questions:
1. What requires user/conductor approval?
   - Major decisions? File creation? Pattern selection?
2. How many PAUSE points?
   - Too many = slow, too few = risky
   - 0-2 PAUSE points typical for sequential tasks
3. What criteria for proceeding after review?
   - Explicit approval? Silence = proceed?
```

**Step 3: Adapt verification requirements**
```
Questions:
1. What tests must pass?
2. What quality checks are required?
3. What artifacts must be created?
4. What git commit standards apply?
5. What report contents are needed?
```

**Step 4: Customize error recovery**
```
Questions:
1. What errors are likely? (list common failure modes)
2. What can the session fix autonomously? (e.g., retry, search alternate paths)
3. What requires user decision? (e.g., ambiguous requirements)
4. What is the escalation path? (PAUSE + tell user)
```

### Common Customizations

**No database coordination:**
```
Changes:
- Remove all SQL patterns (Steps for mark in-progress, complete, messages)
- Use PAUSE-and-tell-user exclusively for review checkpoints
- Remove Prerequisites SQL checks
- Simpler but requires user to be available
```

**No PAUSE points (fully autonomous):**
```
Changes:
- Remove all PAUSE sections
- Session completes end-to-end without stopping
- Add more inline verification to compensate for no review gates
- Best for well-understood, low-risk tasks
```

**Multiple continuation tasks (large scope):**
```
Changes:
- Design checkpoint boundaries at natural phase breaks
- Each continuation task gets its own task file (task-XX-2.md, task-XX-3.md)
- Include "Resume from" section in continuation tasks
- Track completed steps in checkpoint commits
```

**Code implementation (not documentation):**
```
Changes:
- Add testing steps (unit tests, integration tests)
- Include build/compile verification
- Add linting/formatting checks
- Commit frequency: after each passing test suite
- Review criteria: tests pass, no regressions, code quality
```

---

## Part 7: Appendix

### Comparison with Parallel Design

| Aspect | Sequential (This Document) | Parallel ([parallel-task-design.md](parallel-task-design.md)) |
|--------|---------------------------|--------------------------------------------------------------|
| **Infrastructure** | Optional 2-table database | Required state machine + hooks + subagents |
| **Error recovery** | Stop + escalate to user | 5-retry autonomous loop with state transitions |
| **Message handling** | Manual SQL check between steps | Background subagent continuous monitoring |
| **Session lifecycle** | Start → work → finish | Hook-enforced state machine (pending → claimed → working → ...) |
| **Review flow** | PAUSE → tell user → wait | needs_review state → conductor checks → review_approved |
| **Context isolation** | All work in one context | Each task in isolated context window |
| **File safety** | Steps may edit same files | Tasks must NOT edit same files |
| **Coordination overhead** | Minimal | Significant (justified by parallelism) |
| **Best for** | 1-5 sequential steps, user available | 3+ independent tasks, autonomous operation |

### Migration Guide: Sequential to Parallel

If a task grows complex enough to benefit from parallel execution:

1. **Identify independent subtasks** — Which steps can run simultaneously?
2. **Check for file conflicts** — Do any subtasks write to the same files?
3. **Set up infrastructure** — Create coordination_status table, configure hooks
4. **Split task file** — Each independent subtask becomes its own task instruction file
5. **Create conductor instructions** — Define launch order, review cycles, completion criteria
6. **Add hook configuration** — Stop hooks for session lifecycle management
7. **Add message watcher** — Background subagent for continuous message checking

Reference the parallel design document for complete infrastructure setup.

### Template Checklist (Quick Reference)

When writing a new sequential task instruction file, verify:

- [ ] Header has title, parallel-safe, token estimate, dependencies
- [ ] Objective includes deliverables list
- [ ] Context reference points to plan + design + templates
- [ ] Prerequisites include testable checks (bash and/or SQL)
- [ ] Steps are numbered with inline verification
- [ ] PAUSE points are at decision boundaries (not between every step)
- [ ] Verification checklist uses Verify/Expected/If failed format
- [ ] Commit section uses conventional commit format with task metadata
- [ ] Commit frequency guidelines followed (not too granular, not too coarse)
- [ ] Completion report uses enriched template (Work Completed, Learnings, Artifacts sections)
- [ ] Error scenarios use structured Error Report template
- [ ] File creation follows rules (proposals dir first, reports in reports/, temp in temp/)
- [ ] Context budget includes breakdown and monitoring checkpoints
- [ ] Troubleshooting covers common failure modes
- [ ] All file paths are specific (no vague references)
- [ ] Prescriptive elements copied exactly (verification format, commit format, PAUSE format)
