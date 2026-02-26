# Arranger — Integration Review Findings (Design Phase)

*Source: Conductor skill review session, 2026-02-20*
*Reviewer: Arranger integration reviewer*

These findings are against the design document at
`orchestration/docs/designs/2026-02-17-arranger-skill-design.md`, not an implemented skill.

## Critical (1)

### A-C1: Danger file annotation contract missing
- **File:** `2026-02-17-arranger-skill-design.md` (entire document)
- **Issue:** The Conductor's `phase-execution.md` (lines 60-73) expects inline danger file annotations with `[warning] Danger Files:` format from the Arranger. The Arranger design has NO concept of "danger files" — zero matches for the term. The Conductor's 3-step danger file governance workflow will find nothing to extract, silently skipping file-conflict mitigation for parallel tasks.
- **Complication:** Even if the Arranger adds file-conflict awareness, the Conductor expects annotations at the TASK level (`"Task 3: ... Danger Files: ..."`), but the Arranger operates at the PHASE level and explicitly does not do task decomposition (line 32). Task-level danger file annotations are a conceptual impossibility in the Arranger's model.
- **Fix options:**
  1. Arranger annotates file-conflict risks at PHASE level (in Integration Points subsections). Conductor maps these to tasks during its own task decomposition. Requires updates to both sides.
  2. Conductor discovers file conflicts independently during task decomposition. Requires updating the Conductor to NOT expect Arranger-produced annotations.
  Option 1 aligns with "front-load research in Arranger." Option 2 aligns with "task decomposition is Conductor's concern."

## Minor (3)

### A-m1: "Parallel-safe" marking undefined
- **Issue:** Conductor's danger file skip condition (`phase-execution.md` line 161) references an Arranger annotation: "The Arranger's plan explicitly marks the phase as parallel-safe with no shared resources." The Arranger design has no mechanism for this flag.
- **Fix:** Either add a parallel-safe annotation convention to the Arranger's phase structuring, or remove the skip condition from the Conductor.

### A-m2: Plan file output path unspecified
- **Issue:** Arranger design says plans are committed "to the current branch" (line 224) but does not specify the output file path. Conductor expects `docs/plans/designs/{feature}-plan.md`. Repetiteur naming convention (`{feature}-plan-r1.md`) also depends on this path.
- **Fix:** Specify output path convention in the Arranger's Finalization phase.

### A-m3: Arranger-to-Conductor handoff is implicit
- **Issue:** Unlike the well-specified Repetiteur-to-Conductor handoff, the Arranger-to-Conductor transition relies on the user manually bridging the gap. No explicit discovery mechanism on either side.
- **Fix:** Document the handoff convention: Arranger writes plan path to MEMORY.md (or user provides it when invoking `/conductor`).
