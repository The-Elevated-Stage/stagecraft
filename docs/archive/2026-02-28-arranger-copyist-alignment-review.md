# Arranger-Copyist Alignment Review

**Date:** 2026-02-28
**Reviewer:** reviewer-4
**Documents reviewed:**
- Arranger design: `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
- Copyist SKILL.md: `copyist/skill/SKILL.md`
- Copyist references: `launch-prompt-template.md`, `anti-patterns.md`, `sequential-task-template.md`, `parallel-task-template.md`, `schema-and-coordination.md`
- Copyist examples: `sequential-task-example.md`, `parallel-task-example.md`
- Copyist README: `copyist/README.md`
- Copyist docs README: `copyist/docs/README.md`

---

## Executive Summary

The Arranger design and Copyist skill are **well-aligned on core architecture** — the sentinel marker system, plan-index, line-range extraction, self-containment philosophy, and dual-audience document structure all have clear, matching specifications on both sides. The Copyist was clearly built with the Arranger's output format in mind.

However, there are **several notable gaps and inconsistencies** that would cause friction or failure during actual integration:

- **Critical:** The Copyist examples use a stale guard clause pattern (`NOT IN`) that contradicts both the Copyist's own anti-patterns document and the template's `IN` pattern
- **Critical:** The Arranger design says task decomposition is the Conductor's concern, but the Copyist expects task IDs to arrive pre-assigned in the launch prompt — the handoff of decomposition responsibility between Conductor and Copyist is clear, but the Arranger must avoid prescribing task-level granularity
- **Important:** The Arranger design specifies "Testing recommendations" as a phase section component, but the Copyist's expected phase components list omits this, creating a potential parsing/coverage gap
- **Important:** The Arranger uses `<section>` tags within phases as a Tier 2 convention, but the Copyist's phase parsing documentation only references sentinel markers and markdown headers — `<section>` tags inside phase content are unmentioned
- **Minor:** Polling interval inconsistencies in the parallel example contradict the Copyist's own anti-pattern #25

Overall alignment is **strong at the architectural level** but needs **cleanup at the detail level** before the Arranger is built. Most issues are in the Copyist's examples rather than in the core specifications.

---

## Detailed Findings

### 1. Plan Parsing

**Status: Well-Aligned with Minor Gaps**

The Arranger specifies implementation plans using:
- Sentinel markers: `<!-- phase:N -->` / `<!-- /phase:N -->` and `<!-- conductor-review:N -->` / `<!-- /conductor-review:N -->`
- A plan-index at the top: `<!-- plan-index:start -->` containing `<!-- phase:N lines:NN-NN title:"..." -->` entries
- `<section>` tags coexisting with sentinels: `<section id="phase-N">`
- `<sections>` index listing all section IDs
- YAML frontmatter with required fields

The Copyist correctly expects:
- Sentinel markers `<!-- phase:N -->` / `<!-- /phase:N -->` (SKILL.md lines 84-88)
- Plan-index with `<!-- plan-index:start -->` containing `<!-- phase:N lines:NN-NN title:"..." -->` (SKILL.md line 90)
- Line-range-based reading: `Read {PLAN_PATH}, offset={LINE_START}, limit={LINE_END - LINE_START + 1}` (SKILL.md line 71)
- Verification that first/last non-blank lines match sentinel markers (SKILL.md lines 102-103)

**Gap: `<section>` tags within phase content.** The Arranger design specifies that `<section id="phase-N">` tags coexist with sentinel markers inside each phase boundary (design Section 6, line 514: `<section id="phase-1">`). The Copyist's phase parsing documentation (SKILL.md lines 79-103) only mentions sentinel markers and markdown headers (`## Phase N: [Title]`). It never references `<section>` tags as part of the phase content it will receive. This is not necessarily a problem — `<section>` tags inside the phase content won't break parsing — but the Copyist is unaware of them as a structural element, meaning it might not leverage them for navigation within a phase section.

**Gap: `<sections>` index presence.** The Arranger includes a `<sections>` index in the plan (listing all section IDs). This index would be present in the line range the Copyist receives if it falls within the phase section boundaries. The Copyist doesn't reference this index at all — it relies solely on the line range boundaries. Again, not a breaking issue, but a feature the Copyist is unaware of.

**Verdict:** Parsing will work correctly. The Copyist reads by line range, verifies sentinel markers at boundaries, and processes content. The `<section>` tags and `<sections>` index are additive structural elements the Copyist can safely ignore. No breaking incompatibility.

### 2. Phase-to-Task Translation

**Status: Well-Aligned**

The Arranger explicitly states that **task decomposition is the Conductor's concern, not the Arranger's** (design line 32: "It does not create task instruction files — the copyist does that... Task-level decomposition is the conductor's responsibility"). The Arranger provides "boundary/goal guidance" for phases but does not prescribe exact task boundaries.

The Copyist's workflow aligns with this model:
1. Conductor determines task decomposition from the Arranger's phase content
2. Conductor passes task IDs, task type, line ranges, and overrides to Copyist via launch prompt
3. Copyist reads the phase section and decomposes it into the specified tasks
4. Copyist can freely split tasks further if context estimates exceed 170k (SKILL.md line 159)

The launch prompt template (launch-prompt-template.md) carries all the necessary bridging information:
- `{N}` / `{NAME}` — phase number and name
- `{TYPE}` — sequential or parallel (determines template selection)
- `{TASK_LIST}` — specific task IDs
- `{PLAN_PATH}` / `{LINE_START}` / `{LINE_END}` — plan location and phase boundaries
- `{CONDUCTOR_NOTES}` — overrides, learnings, scope adjustments, danger file decisions

**Observation:** The Arranger design mentions that authority tags within phase sections signal how the Copyist should treat content — `<mandatory>` for verbatim preservation, `<guidance>` for adaptable approaches, `<core>` for primary substance (design Section 6, lines 411, 595). The Copyist SKILL.md does not explicitly document how it should interpret these authority tags during task decomposition. However, the Copyist's mandatory rules already enforce key behaviors (all SQL exact, self-containment, no external references) that implicitly align with the Arranger's intent. The Copyist's template `<template follow="exact">` and `<template follow="format">` conventions serve a similar purpose at the task instruction level.

**Verdict:** The translation pipeline is sound. Conductor mediates between Arranger's phase-level guidance and Copyist's task-level output. The Copyist has the information it needs from the launch prompt.

### 3. The Copyist Test (Self-Containment)

**Status: Strong Alignment**

The Arranger design makes self-containment its **primary deliverable quality** (design line 207: "self-contained phase sections are the primary purpose of the entire skill"). The Arranger checks each phase section before user review: "does this section contain everything the copyist needs? Could a copyist session reading only these lines produce complete, unambiguous task instructions?" (design lines 206-207).

The Copyist enforces self-containment from its side:
- "Every instruction file must be self-contained — never write 'see implementation plan section X'" (SKILL.md line 27)
- "Copy all relevant plan content into the instruction — musician sessions do not load the implementation plan" (SKILL.md line 202)
- Anti-pattern #1 explicitly forbids external document references (anti-patterns.md lines 22-43)
- Anti-pattern #23 forbids reading the full plan instead of the phase section (anti-patterns.md lines 433-451)

The Arranger specifies that self-contained phase sections include (design lines 586-593):
- Objective
- Prerequisites
- Implementation detail
- Integration points
- Frontend guidelines (when applicable, inlined)
- Expected outcomes
- Testing recommendations

The Copyist expects to receive in the phase section (SKILL.md lines 92-99):
- Objective
- Prerequisites
- Implementation detail
- Integration points
- Frontend guidelines (when applicable, inlined)
- Expected outcomes
- Testing recommendations

**These lists match exactly.** The Copyist documentation mirrors the Arranger's specification.

**Potential gap: Cross-phase dependencies.** The Arranger notes that prerequisites include "outputs from prior phases, specific file paths and exports created" (design line 588). This means each phase section carries its own prerequisite information, which is essential for self-containment. However, the quality of this depends entirely on Arranger execution — if a prerequisite is vaguely stated ("Phase 1 outputs available"), the Copyist won't have enough to generate meaningful prerequisite checks. The Arranger design does emphasize specificity ("specific file paths and exports created"), but this is a runtime quality concern rather than a structural misalignment.

**Verdict:** Self-containment is the strongest point of alignment between these two skills. Both sides enforce it architecturally, and the expected phase section components match precisely.

### 4. Sequential vs Parallel Templates

**Status: Aligned with a Gap in Arranger Signaling**

The Copyist has distinct templates:
- `sequential-task-template.md` — For non-hook tasks (no background subagent, no review checkpoints, simpler coordination)
- `parallel-task-template.md` — For hook-coordinated tasks (background subagent, review checkpoints, danger files, error recovery cycle)

The Copyist receives the task type from the Conductor's launch prompt: `{TYPE}` field (sequential/parallel) per launch-prompt-template.md line 69.

**How does the Arranger signal parallelism?** The Arranger design discusses phase structuring extensively (design Section 3, Phase 4, lines 178-195):
- The Arranger arranges phases to maximize parallel execution
- It identifies which tasks can run simultaneously without file conflicts
- It identifies integration surfaces between phases

However, the Arranger's output format (Section 6) does not include an explicit per-phase "parallel: true/false" flag or marker. The Phase Summary section describes "parallelization opportunities" (design line 477), and conductor checkpoint sections include "guidance for the next phase" including parallelization (design lines 536-537). But the parallel/sequential decision for individual tasks within a phase is made by the **Conductor**, not the Arranger, which aligns with the Arranger's principle that task decomposition is the Conductor's concern.

**The flow is:**
1. Arranger structures phases and identifies parallelization opportunities in Phase Summary and conductor checkpoints
2. Conductor reads checkpoint guidance and determines which tasks within a phase are parallel vs sequential
3. Conductor sets `{TYPE}` in the Copyist launch prompt

**Verdict:** This is correctly layered — the Arranger provides phase-level parallelism guidance, the Conductor makes task-level parallel/sequential decisions, and the Copyist receives the determination via the launch prompt. No misalignment.

### 5. Context Budget Awareness

**Status: Well-Aligned**

The Copyist operates within a 170k token context budget (140k file reads + 30k instruction output), documented in SKILL.md lines 152-166.

The Arranger design supports selective loading through:
- Plan-index with line ranges for each phase (design lines 571-578)
- The Conductor extracts line ranges and passes them to the Copyist: "read lines X-Y" (design line 578)
- Self-contained phase sections mean the Copyist only needs to read its assigned section

The Copyist's workflow confirms selective reading:
- "Read only the assigned lines from the implementation plan" (SKILL.md line 68)
- `Read {PLAN_PATH}, offset={LINE_START}, limit={LINE_END - LINE_START + 1}` (SKILL.md line 71)
- "Read only the assigned phase section by line range. Do not read the full plan." (SKILL.md line 74, mandatory)

**Verdict:** The plan-index system is explicitly designed for the Copyist's context budget. The Copyist reads only its assigned phase section by line range, keeping context consumption predictable. This is one of the design's strongest integration points.

### 6. Integration Surfaces

**Status: Partially Aligned — Gap in Copyist Consumption**

The Arranger identifies cross-task integration surfaces during Phase 4 (Phase Structuring) and records them in two places:
1. **Phase sections** — Integration Points subsection within each phase (design line 508: "How this phase's work connects to other phases' work. Contracts that must be maintained.")
2. **Conductor checkpoint sections** — Explicit cross-task integration verification items (design line 191, lines 526-528)

The Copyist's expected phase components include "Integration points" (SKILL.md line 97), confirming it expects to receive this information.

**How the Copyist uses integration surfaces:** The Copyist does not have a dedicated template section for "Integration Points." Looking at the templates:
- Sequential template: No explicit integration section. Integration content would be embedded in the Work Execution steps or the Objective context.
- Parallel template: Has a "Danger Files" section (`<section id="danger-files">`) which handles shared file coordination — a specific subset of integration surfaces.

The Copyist does propagate integration information through:
- Danger file tables (parallel template) — shared resource coordination
- Prerequisites section — dependency checks
- Context sections within objectives — background and connections

**Gap:** The Arranger's "Integration Points" concept is broader than just danger files. It includes contracts between phases, expected interfaces, and verification expectations. While the Copyist can embed this information in various sections, there is no explicit template section or guidance for how integration surface information should be distributed across task instruction sections. A Copyist session might embed integration points in the objective context, or in the work execution steps, or miss them entirely if the phase section's integration subsection is long.

**Verdict:** Integration surfaces will flow from Arranger to Copyist via the phase section content. The Copyist will process them, but the lack of explicit template guidance for integration surface handling means placement in task instructions is ad-hoc. The danger files mechanism handles the most critical case (shared file coordination). Less critical integration points rely on Copyist judgment.

### 7. Schema & Coordination

**Status: Aligned**

The Copyist's schema reference (`schema-and-coordination.md`) defines:
- `orchestration_tasks` table with 13 valid states
- `orchestration_messages` table with 12 valid message types
- State machine with Conductor and Musician state ownership
- SQL patterns for all lifecycle operations

The Arranger design does not directly reference the database schema — it operates upstream of the orchestration database. The Arranger produces an implementation plan; the Conductor sets up orchestration infrastructure.

**The alignment surface is indirect:** The Arranger's phase structure and conductor checkpoint system define the *work decomposition* that the Conductor later maps to `orchestration_tasks` rows. The Arranger's checkpoint-based verification maps conceptually to the Conductor's review cycle (`needs_review` -> `review_approved`/`review_failed`).

Key alignment points:
- Arranger's phase dependencies → Copyist's `dependencies` field in task instruction metadata
- Arranger's parallelism guidance → Copyist's `parallel-safe` field
- Arranger's conductor checkpoints → Copyist's review checkpoint sections
- Arranger's integration verification items → Copyist's danger files and verification checklist

**Verdict:** No schema conflicts. The Copyist's database schema is orthogonal to the Arranger's output — the Conductor bridges between them. The conceptual mapping is clean.

### 8. Launch Prompt Template

**Status: Aligned**

The Copyist's launch prompt template (`launch-prompt-template.md`) requires:
- Phase number and name (`{N}`, `{NAME}`)
- Task type (`{TYPE}`)
- Task list (`{TASK_LIST}`)
- Plan path (`{PLAN_PATH}`)
- Line range (`{LINE_START}`, `{LINE_END}`)
- Overrides & Learnings (`{CONDUCTOR_NOTES}`)

The Arranger provides all of this through its output:
- Phase numbers and names — in sentinel markers: `<!-- phase:N lines:NN-NN title:"..." -->`
- Task type — determined by Conductor from Arranger's parallelism guidance
- Task list — determined by Conductor from Arranger's boundary guidance
- Plan path — the committed implementation plan file
- Line ranges — in the plan-index
- Overrides — Conductor accumulates these during execution; Arranger's user override flags in conductor checkpoints seed the initial set

**Observation:** The Arranger design mentions that conductor checkpoint sections include "User Override flags" (design lines 237, 299) that should be propagated. The launch prompt's Overrides & Learnings field is the mechanism for this propagation. The Conductor reads the override flag from the checkpoint, includes it in the Copyist launch prompt, and the Copyist's precedence rules (launch-prompt-template.md lines 101-115) ensure the override takes priority over plan content.

**Verdict:** Complete alignment. Every field the launch prompt needs is available from the Arranger's output plus the Conductor's runtime decisions.

---

## Categorized Issues

### Critical

1. **Stale guard clause in examples.** The sequential example (`sequential-task-example.md` line 118) and parallel example (`parallel-task-example.md` line 141) both use `AND state NOT IN ('working', 'complete', 'exited')` for the task claim SQL. This contradicts:
   - Anti-pattern #24 (`anti-patterns.md` lines 456-466), which explicitly flags this as wrong
   - Both templates (`sequential-task-template.md` line 129, `parallel-task-template.md` line 172), which correctly use `AND state IN ('watching', 'fix_proposed', 'exit_requested')`
   - The schema reference (`schema-and-coordination.md` line 190), which uses the correct `IN` pattern

   **Impact:** A Copyist using the example as a reference (rather than the template) would produce task instructions with an overly permissive guard clause, potentially allowing musicians to claim tasks in unsafe states. **This is a Copyist bug, not an Arranger-Copyist misalignment, but it directly affects the quality of the Arranger-to-Musician pipeline.**

   **Recommendation:** Update both examples to use `AND state IN ('watching', 'fix_proposed', 'exit_requested')`.

2. **Parallel example uses wrong polling interval.** The parallel example's background subagent (`parallel-task-example.md` line 165) polls every 8 seconds. Anti-pattern #25 (`anti-patterns.md` lines 469-480) explicitly flags hardcoded intervals that diverge from the Musician's canonical timing: background watcher should be 15 seconds. The templates correctly use 15 seconds (parallel template line 195, schema reference line 361).

   **Impact:** Same as above — example-driven Copyist behavior would produce task instructions with incorrect polling intervals.

   **Recommendation:** Update the parallel example to use 15-second polling for the background watcher.

### Important

3. **Arranger authority tags not explicitly consumed by Copyist.** The Arranger design specifies that authority tags within phase sections (`<mandatory>`, `<guidance>`, `<core>`) signal how the Copyist should treat content — mandatory content preserved verbatim, guidance adaptable, core is primary substance (design Section 6, lines 411, 595). The Copyist SKILL.md does not document how to interpret these tags during task decomposition. The Copyist's own templates use authority tags for *output* (task instructions), but there's no rule for how to *consume* them from the Arranger's phase sections.

   **Impact:** A Copyist session might treat all Arranger phase content uniformly rather than differentiating mandatory constraints from adaptable guidance. In practice, the Copyist's self-containment rules ("copy all relevant plan content") and the templates' own `<mandatory>` blocks likely produce acceptable results, but the explicit authority-tag-as-input-signal mechanism the Arranger intends is undocumented on the Copyist side.

   **Recommendation:** Add a brief note to Copyist SKILL.md's workflow section (around line 76, after the "Expected phase components" list) explaining that Arranger phase sections use authority tags to classify content, and that `<mandatory>` content from the phase section should be preserved verbatim in task instructions while `<guidance>` content can be adapted for task-level context.

4. **Parallel example missing heartbeat in background subagent.** The parallel example's background subagent template (`parallel-task-example.md` lines 158-180) omits the heartbeat maintenance logic that is present in the canonical template (`schema-and-coordination.md` lines 358-381 and `parallel-task-template.md` lines 189-215). Specifically, the example omits step 3 of the monitoring loop: checking `last_heartbeat` and updating if older than 60 seconds.

   **Impact:** A Copyist using the example as reference would produce task instructions where the background subagent does not maintain heartbeats, potentially triggering the Conductor's staleness detection.

   **Recommendation:** Update the parallel example's background subagent to include the heartbeat maintenance step.

5. **Parallel example error report missing fields.** The parallel example's error report SQL (`parallel-task-example.md` lines 775-793) omits several fields present in the canonical error report pattern (`schema-and-coordination.md` lines 254-278). Specifically, it's missing: `Context Usage`, `Self-Correction`, `Report`, and `Key Outputs` fields.

   **Impact:** Inconsistency between example and canonical pattern could lead to incomplete error reports.

   **Recommendation:** Update the parallel example's error report to include all canonical fields.

### Minor

6. **`<section>` tags inside phases undocumented in Copyist.** The Arranger design specifies `<section id="phase-N">` tags inside phase sentinel boundaries (design Section 6). The Copyist's phase parsing documentation (SKILL.md lines 79-103) only references sentinel markers and markdown headers. The `<section>` tags won't cause problems but are an unacknowledged structural element.

   **Recommendation:** Consider adding a brief note in the Copyist's "Plan Structure" context block mentioning that phase sections may contain `<section>` tags per Tier 2 convention — these can be ignored as the Copyist works from the full phase text.

7. **Integration surface handling is ad-hoc.** The Arranger produces "Integration Points" subsections within phases. The Copyist has no explicit template section or guidance for where to place integration surface information in task instructions. It's likely embedded in objectives, prerequisites, or work execution steps based on Copyist judgment.

   **Recommendation:** Consider adding a brief note to the Copyist SKILL.md workflow section acknowledging that Arranger phase sections contain integration points and suggesting they be distributed across Prerequisites (for dependency contracts), Danger Files (for shared resources), and Work Execution steps (for implementation-level integration).

8. **Testing recommendations coverage.** The Arranger includes "Testing recommendations" as a phase section component (design line 593) for cases where the Arranger discovered testing considerations during planning. The Copyist's expected phase components list at SKILL.md line 99 includes "Testing recommendations." The Copyist templates include a Testing Requirements section. Alignment is actually present — listed as minor only because the Arranger's testing recommendations are "observations that inform the conductor's verification expectations" (design line 174), which is a lighter treatment than the Copyist's template section (which expects specific test file paths, test cases, and manual verification checklists). The Copyist will need to expand on the Arranger's observations to fill out the Testing Requirements section.

   **Recommendation:** No structural change needed. The Copyist naturally expands phase-level testing observations into task-level testing requirements. This is working as designed — the Arranger provides direction, the Copyist provides specificity.

### Observations

9. **Example documents are v2.0 but contain v1.0 patterns.** Both the sequential and parallel examples (the `NOT IN` guard clause, the 8-second polling interval, missing subagent heartbeats) contain patterns from what appears to be an earlier version of the Copyist specification. The templates and schema reference are internally consistent (v2.0 patterns), but the examples lag behind. This suggests the examples were not updated during a recent migration/revision.

10. **Arranger frontend guidelines inlining.** The Arranger design emphasizes that frontend guidelines are "always inlined" into phase sections (design line 599), never in a standalone block. This is specifically for self-containment. The Copyist doesn't explicitly mention frontend guidelines as a special category, but its self-containment rules ("copy all relevant plan content into the instruction") cover this case implicitly. No action needed.

11. **User Override propagation chain.** The Arranger journals user overrides with `<mandatory>` tags and flags them in conductor checkpoint sections. The Conductor reads the flag, includes it in the Copyist's Overrides & Learnings section, and the Copyist's precedence rules ensure overrides take priority. This three-hop propagation chain (Arranger -> Conductor -> Copyist -> Musician) is well-designed but depends on the Conductor correctly identifying and forwarding override flags. The chain has no structural gap — just a runtime execution dependency.

12. **Tier mismatch is by design.** The Arranger produces Tier 2 documents (hybrid markdown with XML tags). The Copyist produces Tier 3 documents (all text inside authority tags). This is intentional — the Arranger's output is a planning document for mixed human/machine consumption, while the Copyist's output is a pure machine-consumed instruction file. The Copyist correctly transforms Tier 2 phase content into Tier 3 task instructions.

---

## Recommendations Summary

### For Copyist (before Arranger is built)

1. **Fix examples** — Update both `sequential-task-example.md` and `parallel-task-example.md`:
   - Guard clause: `NOT IN` -> `IN ('watching', 'fix_proposed', 'exit_requested')`
   - Background subagent polling: 8 seconds -> 15 seconds
   - Background subagent: add heartbeat maintenance step
   - Error report SQL: add missing fields to match canonical pattern

2. **Document authority tag consumption** — Add a note to SKILL.md explaining how to interpret `<mandatory>`, `<guidance>`, and `<core>` tags in the Arranger's phase section content when generating task instructions.

3. **Document integration surface handling** — Add guidance for distributing Arranger's "Integration Points" content across task instruction sections.

### For Arranger (when built)

4. **No structural changes needed.** The Arranger design's output format is compatible with the Copyist's consumption model. The sentinel markers, plan-index, self-containment rules, and dual-audience structure all match.

5. **Ensure phase sections include all 7 components.** The Copyist's SKILL.md lists exactly the components the Arranger specifies. The Arranger should verify during Phase 5 (Section Writing) that each phase section contains all of: Objective, Prerequisites, Implementation detail, Integration points, Frontend guidelines (when applicable), Expected outcomes, Testing recommendations.

6. **Authority tags as an input signal.** When the Arranger is built, the SKILL.md should reference the Copyist's consumption model — `<mandatory>` content will be preserved verbatim in task instructions, `<guidance>` content may be adapted. This creates a feedback loop where the Arranger can use authority tags strategically.
