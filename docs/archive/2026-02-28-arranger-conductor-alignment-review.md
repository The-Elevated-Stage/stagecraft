# Arranger-Conductor Alignment Review

**Date:** 2026-02-28
**Reviewer:** conductor-reviewer (automated alignment review)
**Arranger Design:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Conductor Skill:** `conductor/skill/SKILL.md` (v4.0) + all references and examples

---

## Executive Summary

The Arranger design and the built Conductor skill are **well-aligned on the fundamental contract**. The plan-index sentinel marker system, selective reading strategy, dual-audience document model, and the Conductor's role as task decomposer (not plan consumer) are all consistent between the two documents. The Conductor's initialization protocol explicitly expects Arranger-produced plans with plan-index verification, and its phase execution protocol reads phase sections on-demand using plan-index line ranges -- exactly as the Arranger specifies.

However, there are several areas where the alignment is incomplete or where the Arranger's design makes assumptions about Conductor behavior that are not fully reflected in the current Conductor implementation. The most significant issues involve: (1) the Conductor's handling of the `<sections>` tag and Tier 2 hybrid structure within the plan, (2) gaps in how conductor-review checkpoint sections are consumed, and (3) the absence of authority tag interpretation in the Conductor's review and phase execution logic.

**Overall Alignment Score: Strong with targeted gaps.**

---

## Detailed Findings

### 1. Plan Format Compatibility

**Status: Strongly Aligned**

The Arranger specifies that implementation plans use:
- YAML frontmatter (`title`, `date`, `type`, `tier`, `feature`, `design-doc`)
- `<!-- plan-index:start -->` / `<!-- plan-index:end -->` sentinel block at top
- `<!-- verified:YYYY-MM-DDTHH:MM:SS -->` timestamp
- `<!-- phase:N lines:NN-NN title:"Phase Title" -->` entries
- `<!-- conductor-review:N lines:NN-NN -->` entries
- Optional `<!-- revision:N -->` and `<!-- supersedes:{file} -->` for Repetiteur revisions

The Conductor's initialization protocol (`references/initialization.md`, `plan-index-verification` section) expects exactly this format:
- Checks for `<!-- plan-index:start -->` presence as the first verification step
- Parses `<!-- phase:N lines:NN-NN title:"Phase Title" -->` and `<!-- conductor-review:N lines:NN-NN -->` entries
- Checks `<!-- verified:YYYY-MM-DDTHH:MM:SS -->` timestamp
- Checks `<!-- revision:N -->` for Repetiteur consultation count

**Finding:** The sentinel marker format is fully specified in the Arranger and fully consumed by the Conductor. No format conflicts detected.

**Minor observation:** The Arranger specifies `<!-- overview -->` / `<!-- /overview -->` and `<!-- phase-summary -->` / `<!-- /phase-summary -->` sentinel pairs wrapping those sections. The Conductor's plan-bootstrap-reading section says it reads "the Overview section (via plan-index line range)" and "the Phase Summary section (via plan-index line range)." However, the plan-index format shown in the Arranger design only includes `phase:N` and `conductor-review:N` entries -- it does not include `overview` or `phase-summary` line range entries in the plan-index block itself. The Overview and Phase Summary have their own sentinel pairs but no explicit plan-index entries mapping them to line ranges. The Conductor would need to either: (a) scan for `<!-- overview -->` / `<!-- phase-summary -->` sentinels directly, or (b) the Arranger's finalization would need to add overview/phase-summary entries to the plan-index. This is a minor gap since both sections appear near the top of the file and are easy to find, but it's a deviation from the "read via plan-index line range" principle stated in the Conductor.

### 2. Phase Structure

**Status: Strongly Aligned**

The Arranger explicitly states it provides "boundary/goal guidance" for each phase but NOT task-level decomposition (Section 1: "Not a task-writer"). It produces:
- Phase sections with Objective, Prerequisites, Implementation, Integration Points, Expected Outcomes
- Sequential vs. parallel guidance at the phase level
- Cross-task integration surfaces identified
- Dependency ordering between phases

The Conductor's phase execution protocol (`references/phase-execution.md`, `task-decomposition` section) expects exactly this division of labor:
- "The Arranger provides boundary/goal guidance -- the Conductor decides the actual task boundaries"
- Decomposition steps: read phase section, identify natural task boundaries, assess parallelism, assign task IDs
- Launches Copyist to create task instructions from Conductor's decomposition

**Finding:** The responsibility boundary is clearly defined and consistent on both sides. The Conductor actively handles task decomposition from Arranger phase-level goals.

**Observation on parallel/sequential within phases:** The Arranger's Phase 4 (Phase Structuring) designs phases to "maximize what the copyist can parallelize" -- but the Arranger does not prescribe whether tasks within a phase are sequential or parallel. The Conductor's phase execution has distinct sequential and parallel execution patterns. The Conductor determines this based on its own analysis of the phase content (danger files, shared resources). The Arranger's `conductor-review:N` sections provide "guidance for the next phase" including "parallelization opportunities" -- this is the handoff point where the Arranger's parallelization intent reaches the Conductor. This is well-aligned.

### 3. Plan-Index

**Status: Aligned with Minor Gap**

The Arranger specifies the plan-index as:
- Generated during Phase 6 (Finalization) by a verification subagent
- Contains `phase:N` and `conductor-review:N` line range entries
- Serves as both a line-range map and a lock indicator
- `verified` timestamp confirms finalization occurred

The Conductor uses the plan-index:
- Initialization: verifies presence of `<!-- plan-index:start -->` (hard gate)
- Initialization: parses line ranges for bootstrap reading (Overview, Phase Summary)
- Phase execution: reads `phase:N` line ranges on-demand when entering each phase
- Phase execution: reads `conductor-review:N` line ranges for verification checklists
- Repetiteur protocol: checks `revision:N` metadata for consultation count

**Finding:** Plan-index consumption is well-aligned. The Conductor's code explicitly uses the format the Arranger generates.

**Gap (same as Finding #1 observation):** The plan-index example in the Arranger design includes entries for `phase:N` and `conductor-review:N` but not for `overview` or `phase-summary`. The Conductor's initialization says it reads these "via plan-index line range." Either the Arranger's finalization should generate overview/phase-summary entries in the plan-index, or the Conductor should use the sentinel markers directly for these sections. This needs clarification before implementation.

### 4. Task Decomposition Boundary

**Status: Aligned -- No Gap**

The Arranger explicitly addresses this boundary:
- Section 1: "Not a task-writer. Task-level decomposition is the conductor's responsibility."
- Section 9: "The conductor determines task decomposition from the arranger's boundary/goal guidance and coordinates the copyist to create task instruction files."
- Checkpoint sections provide "boundary/goal guidance for task decomposition, not prescriptive task lists."

The Conductor fully handles this:
- `references/phase-execution.md`, `task-decomposition` section: 7-step decomposition process
- Conductor reads phase section, identifies natural task boundaries, assesses parallelism, assigns task IDs
- Launches Copyist teammate with phase info, task list, line ranges, and overrides
- Validates returned instructions via `scripts/validate-instruction.sh`

**Finding:** The decomposition boundary is clearly defined and the Conductor has a well-specified process for handling it. The Conductor's mandatory rule about launching a teammate for exploration if decomposition is unclear (`"If unable to determine how to split a phase into tasks, launch a teammate"`) provides a safety net.

**Observation:** The Arranger's conductor checkpoint sections include `<guidance>` tags with "recommendations for task decomposition, parallelization opportunities, dependencies to respect." The Conductor reads these checkpoint sections during phase execution (per `plan-consumption` section: "Read `conductor-review-N` section"). This creates a clear information flow: Arranger provides decomposition guidance in checkpoints, Conductor reads checkpoints and applies that guidance during task decomposition.

### 5. Metadata & Context

**Status: Aligned with Observations**

**What the Conductor expects in the plan:**

| Metadata | Conductor Expectation | Arranger Provision |
|---|---|---|
| Plan-index with line ranges | Required (hard gate) | Generated during finalization |
| Overview section | Required (bootstrap reading) | Specified in document structure |
| Phase Summary section | Required (bootstrap reading) | Specified, written during Phase 4 |
| Phase sections | Required (on-demand per phase) | Self-contained, detailed |
| Conductor-review sections | Required (post-phase verification) | Review checklists with mandatory items |
| Feature name | Used for branch naming, decisions directory | In YAML frontmatter (`feature` field) |
| Phase count | Used for phase execution loop | Derivable from plan-index entries |
| Dependency graph | Used for phase ordering | In Phase Summary and between-phase prerequisites |
| Revision metadata | Used for Repetiteur consultation count | Optional, present in revised plans |

**Finding:** All metadata the Conductor needs is present in the Arranger's output format. The YAML frontmatter, plan-index, and section structure together provide everything the Conductor requires for initialization and phase execution.

**Observation on YAML frontmatter:** The Arranger specifies YAML frontmatter fields: `title`, `date`, `type`, `tier`, `feature`, `design-doc`. The Conductor's initialization does not explicitly parse YAML frontmatter -- it reads the plan-index and Overview/Phase Summary sections. The `feature` field in frontmatter is implicitly used for branch naming and decisions directory paths, but the Conductor doesn't document explicit YAML parsing. This is fine as long as the Overview section carries the feature name forward, which the Arranger design requires ("Comprehensive enough that the conductor understands the full picture").

### 6. Error Recovery & Re-planning

**Status: Strongly Aligned**

The escalation path is consistent across both documents:

```
Conductor (5 corrections) --> Repetiteur (3 consultations) --> User
```

**Arranger's design on Repetiteur:**
- Section 1: "The Repetiteur skill handles mid-implementation consultation if blockers arise."
- Section 9: "The Repetiteur is a separate skill that shares the arranger's `shared-rules.md` reference."
- Section 9: Repetiteur produces a "remaining plan" -- a full, standalone implementation plan
- Decision journals persist specifically to support Repetiteur consultations

**Conductor's error recovery chain:**
- `references/error-recovery.md`: 5-correction limit per Musician error, then escalate to Repetiteur
- `references/repetiteur-invocation.md`: 3-consultation limit, revision tracking via plan-index `<!-- revision:N -->`
- Repetiteur produces revised plan with task annotations (`REVISED`, `NEW`, `REMOVED`)
- Conductor performs plan changeover: verifies plan-index, reads new plan, maps task annotations

**Finding:** The escalation chain is fully consistent. The Conductor's Repetiteur protocol expects exactly the kind of output that the Arranger design says the Repetiteur would produce (since the Repetiteur shares the Arranger's output format via `shared-rules.md`).

**Observation on plan changeover:** The Conductor's plan-changeover procedure (`references/repetiteur-invocation.md`, `plan-changeover` section) expects the revised plan to have a valid plan-index, Overview, Consultation Context, and Phase Summary. The Arranger design says Repetiteur-produced plans follow the same format. The Conductor checks for `<!-- plan-index:start -->` as the first step of changeover -- this is aligned with the Arranger's lock indicator pattern.

### 7. Review Protocol

**Status: Aligned with Authority Tag Gap**

**Arranger's conductor checkpoint sections include:**
- Verification checklists with `<mandatory>` tags (items that must be verified)
- Known risks
- `<guidance>` tags for recommendations for next phase
- User override flags propagated from decision journal
- Cross-task integration checks

**Conductor's review usage:**
- Phase execution reads `conductor-review:N` section when entering a phase (`plan-consumption` section)
- Phase completion verifies against the conductor-review checklist (step 4 of `phase-completion` section: "Verify against conductor-review-N checklist -- read the conductor-review section for this phase from the plan. All checklist items must pass before proceeding.")
- The Review Protocol (`references/review-protocol.md`) handles Musician submission reviews (smoothness scale, decision thresholds)

**Finding:** There are two distinct review mechanisms in the Conductor:
1. **Musician work reviews** (Review Protocol) -- evaluating individual task submissions
2. **Phase-level verification** (conductor-review checklist) -- verifying the Arranger's checklist items after a phase completes

Both are present and functional. The Conductor reads conductor-review sections and uses them as a quality gate before proceeding to the next phase.

**Gap -- Authority tag interpretation:** The Arranger design specifies that conductor checkpoint sections use `<mandatory>` tags to flag items the conductor must verify and `<guidance>` tags for recommendations. The Conductor's current reference files do not describe how to interpret or differentiate these authority tags within conductor-review sections. The Conductor treats the entire conductor-review section as a checklist ("All checklist items must pass"), which is more conservative than the Arranger's intent (where `<guidance>` items are recommendations, not hard requirements). This is not a breaking issue -- being more conservative is safer -- but it means the Conductor may block progress on guidance items that the Arranger intended as optional.

### 8. RAG/Context Loading

**Status: Strongly Aligned**

The Arranger's selective reading approach is deeply integrated into the Conductor:

**Bootstrap (Initialization):**
- Read plan-index (tiny, always full)
- Read Overview section (via line range)
- Read Phase Summary section (via line range)
- Total bootstrap context cost: "typically 1-3k tokens" per the Conductor's `plan-bootstrap-reading` section

**On-demand (Phase Execution):**
- When entering a new phase, read `phase:N` section via plan-index line ranges
- Read `conductor-review:N` section for that phase
- "Do not read future phase sections" (mandatory rule)

**Copyist consumption:**
- Conductor passes line ranges to Copyist: "Phase section: lines {LINE_START}-{LINE_END}"
- Copyist reads only its assigned line range -- self-containment ensures this is sufficient

**Finding:** The selective reading strategy described in the Arranger design is exactly implemented in the Conductor. The line-range-based consumption model keeps both Conductor and Copyist context usage proportional to one phase at a time, not the full plan. This is a well-designed context management approach.

**Observation on RAG processing:** The Conductor has an entire RAG processing workflow (`references/review-protocol.md`, `rag-processing-workflow` section) for handling knowledge-base proposals from Musicians. This is downstream of the Arranger -- the Arranger doesn't produce RAG entries, but the Conductor's system for processing them is consistent with the pipeline model.

---

## Issue Categorization

### Critical Issues

None identified. The fundamental contract between Arranger output format and Conductor consumption is sound.

### Important Issues

**I-1: Overview/Phase-Summary line ranges missing from plan-index format**

The Conductor's initialization reads Overview and Phase Summary "via plan-index line range" (`references/initialization.md`, `plan-bootstrap-reading` section). However, the plan-index format shown in the Arranger design (Section 6) only includes `phase:N` and `conductor-review:N` entries -- no `overview` or `phase-summary` line range entries appear in the plan-index example.

**Impact:** The Conductor cannot read these sections via plan-index line ranges unless the Arranger's finalization adds them, or the Conductor falls back to scanning for `<!-- overview -->` / `<!-- phase-summary -->` sentinel markers.

**Recommendation:** Either:
- (a) Add `<!-- overview lines:NN-NN -->` and `<!-- phase-summary lines:NN-NN -->` to the plan-index format in the Arranger design, OR
- (b) Update the Conductor's `plan-bootstrap-reading` section to specify that Overview and Phase Summary are located by their sentinel markers rather than plan-index entries.

Option (a) is cleaner -- it keeps all line-range navigation in one place.

**I-2: Authority tag interpretation not specified in Conductor**

The Arranger specifies that conductor checkpoint sections use `<mandatory>` tags for items that must be verified and `<guidance>` tags for recommendations. The Conductor does not describe how it interprets these tags -- it treats all checklist items equally.

**Impact:** The Conductor may unnecessarily block on `<guidance>` items. More importantly, `<mandatory>` items with plan-level authority (the SKILL.md mandatory rule: "Items with mandatory authority tags in the implementation plan's phase sections are NOT modifiable by the Conductor, even within intra-phase authority") need to be recognized. The Conductor has this rule in its mandatory section, but its phase execution and review protocols don't describe the mechanics of detecting and respecting these tags.

**Recommendation:** Add a subsection to the Conductor's phase execution reference that describes:
- How to identify `<mandatory>` items in both phase sections and conductor-review sections
- That `<mandatory>` items in phase sections are not modifiable by the Conductor
- That `<guidance>` items in conductor-review sections are recommendations, not hard gates
- That user override flags (from the Arranger's journal) appear in conductor checkpoint sections and should be noted but not blocked on

**I-3: Hybrid document structure (Tier 2) not referenced in Conductor**

The Arranger design specifies the implementation plan follows Tier 2 of the project's hybrid document structure -- YAML frontmatter, `<sections>` index, `<section id="...">` tags with authority tags inside. The Conductor's reference files do not mention Tier 2, `<sections>` indices, or `<section>` tags in the context of reading the implementation plan. The Conductor reads the plan using sentinel markers and line ranges exclusively.

**Impact:** Low operational impact -- the sentinel markers and line ranges are sufficient for the Conductor's needs. However, the `<sections>` index in the plan could be useful for recovery scenarios or fallback navigation if plan-index line ranges become stale (e.g., after a partial edit that shifts line numbers).

**Recommendation:** Add a brief note to the Conductor's initialization or phase-execution reference acknowledging that the plan is a Tier 2 document with `<section>` tags as a fallback navigation mechanism, but that the plan-index is the primary navigation system.

### Minor Issues

**M-1: Conductor-review section naming convention**

The Arranger design uses `conductor-review:N` in sentinel markers and `conductor-review-N` as section IDs (with hyphen instead of colon). The Conductor's plan-index parsing expects `conductor-review:N` with colons. Both are present and consistent in their respective domains, but implementers should be careful about which convention is used where (colons in sentinel markers/plan-index, hyphens in section IDs).

**M-2: YAML frontmatter consumption**

The Conductor does not explicitly parse YAML frontmatter from the plan. The `feature` field in frontmatter is used implicitly (decisions directory, branch naming) but these values come from the Overview section or MEMORY.md rather than YAML parsing. If any future Conductor behavior depends on YAML fields, explicit parsing would need to be added.

**M-3: Danger file annotation format**

The Arranger design mentions that phase sections may contain "inline annotations" for known file conflicts (danger files). The Conductor's danger-file-assessment section says "check for inline annotations in the phase section -- the Arranger may mark known conflicts, but the Conductor must not rely solely on these annotations." The format of these annotations is not specified in the Arranger design. The Conductor correctly treats self-discovery as the primary method and Arranger annotations as supplementary.

**M-4: Plan path convention**

The Conductor's MEMORY.md tracking uses `docs/plans/designs/{feature}-plan.md` as the plan path format. The Arranger design doesn't specify the exact path convention for its output, though it mentions the plan lives somewhere the Conductor can find it. The example path in the Conductor's initialization example is `docs/plans/2026-02-04-docs-reorganization.md` (note: `docs/plans/`, not `docs/plans/designs/`). The MEMORY.md section shows `docs/plans/designs/{feature}-plan.md`. This inconsistency is within the Conductor's own examples -- the Arranger's output path should match whatever convention the Conductor's MEMORY.md tracking expects.

### Observations

**O-1: Self-containment validation is Arranger-side only**

The Arranger performs self-containment checks during Phase 5 (Section Writing): "Could a copyist session reading only these lines produce complete, unambiguous task instructions?" This validation happens before the plan reaches the Conductor. The Conductor does not perform its own self-containment validation -- it trusts the Arranger's finalization checklist. This is appropriate given the pipeline model (quality is checked at the source), but if self-containment ever fails, the Conductor has no defense mechanism beyond the Copyist potentially flagging incomplete context.

**O-2: User override flags in conductor checkpoints**

The Arranger design specifies that user overrides from the decision journal are flagged in conductor checkpoint sections: "USER OVERRIDE: [setting] set to X despite research indicating Y -- user has workaround, see journal entry [ref]." The Conductor's phase execution and review protocols do not describe how to handle these override flags. The Conductor would encounter them as text in the conductor-review section but has no specific behavior defined for them. This is low-risk since the Conductor treats all checklist items as verification targets, but override-flagged items might warrant different treatment (noting the risk rather than blocking on the research finding).

**O-3: Conductor does not read phase sections directly**

The Arranger design explicitly states: "The conductor will read only these checkpoints, the overview, and the phase summary -- it will not read phase sections directly." However, the Conductor's task-decomposition section says "Read the phase section (already loaded from plan-consumption)" and the plan-consumption section says "Read `phase:N` section." This appears contradictory, but it's actually correct behavior -- the Conductor reads the phase section for task decomposition purposes, not for implementation. The Arranger's statement should be read as: the Conductor does not implement from phase sections (that's the Copyist/Musician's job), but it does read them for decomposition context. The Conductor's `task-decomposition` section confirms this: it reads the phase to "identify natural task boundaries" and then passes line ranges to the Copyist for actual instruction creation.

**O-4: Arranger's testing recommendations flow**

The Arranger notes testing considerations discovered during planning and includes them in phase sections. These flow to Musicians via Copyist task instructions. The Conductor's review protocol checks test results ("Tests: {status}" in review request format) but does not specifically verify that Arranger-recommended tests were executed. This is a weak link in the testing chain but is mitigated by the Conductor's completion protocol running the full test suite.

**O-5: Decision journal directory lifecycle**

Both documents agree on the lifecycle: the decisions directory (`docs/plans/designs/decisions/{feature-name}/`) persists during implementation for Repetiteur reference, and the Conductor cleans it up during the Completion Protocol. The Conductor's completion reference (`references/completion.md`, `decisions-cleanup` section) specifies `rm -rf docs/plans/designs/decisions/{feature-name}/` with a guidance note to preserve if Repetiteur consultations occurred. This matches the Arranger's design.

---

## Recommendations Summary

| Priority | ID | Recommendation |
|---|---|---|
| Important | I-1 | Add overview/phase-summary line ranges to plan-index format, or update Conductor to use sentinel markers for these sections |
| Important | I-2 | Add authority tag interpretation mechanics to Conductor's phase execution reference |
| Important | I-3 | Acknowledge Tier 2 structure in Conductor references as fallback navigation |
| Minor | M-1 | Document sentinel marker (colon) vs section ID (hyphen) naming convention |
| Minor | M-2 | Note that YAML frontmatter is not explicitly parsed by Conductor |
| Minor | M-3 | Consider specifying danger file annotation format in Arranger design |
| Minor | M-4 | Resolve plan path convention inconsistency in Conductor examples |

---

## Conclusion

The Arranger-Conductor alignment is strong. The core contract -- plan-index sentinel markers, selective line-range reading, dual-audience sections, and the task decomposition boundary -- is well-specified on both sides and fully consistent. The Conductor was clearly designed with the Arranger's output format in mind, and the Arranger design explicitly considers what the Conductor needs.

The identified issues are refinements, not structural problems. The most impactful recommendation is I-1 (overview/phase-summary line ranges), which is a straightforward format addition. I-2 (authority tag interpretation) would improve the Conductor's ability to respect the Arranger's intent for mandatory vs. guidance items, but the current conservative approach (treating everything as required) is safe. I-3 is informational.

No changes to either the Arranger design or the Conductor skill are blocking for implementation. The Arranger can be built against the current Conductor with confidence that the output format will be correctly consumed.
