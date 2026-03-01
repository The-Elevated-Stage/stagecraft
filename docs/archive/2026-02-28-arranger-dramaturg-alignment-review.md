# Arranger-Dramaturg Alignment Review

**Date:** 2026-02-28
**Scope:** Comprehensive alignment analysis between the Arranger design document (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`) and the fully built Dramaturg skill (`dramaturg/skill/SKILL.md`, all reference files, design doc, README).
**Relationship to prior reviews:** This review builds on and supersedes the Dramaturg-specific sections of `ARRANGER-CHANGES.md` (2026-02-23) and the Dramaturg items in `2026-02-27-arranger-alignment-review.md`. It provides deeper analysis by reading every Dramaturg reference file in full.

---

## Executive Summary

The Arranger-Dramaturg handoff is architecturally sound but operationally broken in its current state. The two skills share a clear division of responsibility (vision vs. plan) and a well-designed signal mechanism (VERIFIED/PARTIAL/UNRESEARCHED flags). However, three categories of misalignment prevent the handoff from working as specified:

1. **Journal location and lifecycle** -- the Dramaturg writes to a different path and archives the journal, while the Arranger and downstream pipeline expect persistence at a different path. This is the single blocking issue.
2. **Journal format divergence** -- the Dramaturg uses plain markdown entry templates; the Arranger specifies Tier 3 XML wrapper format. The ingestion subagent can bridge this, but the mismatch creates unnecessary friction.
3. **Implicit assumptions** -- the Arranger makes several assumptions about Dramaturg behavior and output that are either undocumented in the Dramaturg skill or contradict its actual behavior (feature-name derivation, design doc metadata, artifact count).

The design-level boundary (vision decisions vs. implementation decisions) is well-defined and consistently respected. The VERIFIED/PARTIAL mechanism is the strongest alignment point -- both skills describe it identically and it serves a clear optimization purpose.

---

## 1. Output-Input Handoff

### What Dramaturg Produces

The Dramaturg produces two artifacts (SKILL.md:59-63):

1. **Design document** at `docs/plans/designs/YYYY-MM-DD-<topic>-design.md`
   - Required opening: Goals section (verbose narrative of user's goals)
   - Required closing: Arranger Notes appendix (new protocols, open questions, key decisions)
   - Body: open-ended narrative capturing the design vision

2. **Decision journal** at `docs/plans/designs/YYYY-MM-DD-<topic>-dramaturg-journal.md`
   - Append-only structured decision trail
   - Archived to `docs/archive/plans/designs/` on finalization (support-phases.md:255)

### What Arranger Expects to Receive

The Arranger expects to receive (Arranger design Section 2):

1. **Design document** -- read fully by the main Arranger session
2. **Decision journal** at `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md` -- distilled by a subagent, never read raw by the main session

### Mismatches

| Item | Dramaturg Actual | Arranger Expected | Severity |
|------|------------------|-------------------|----------|
| Design doc path | `docs/plans/designs/YYYY-MM-DD-<topic>-design.md` | `docs/plans/designs/[design-doc-name].md` (YAML frontmatter `design-doc` field) | **Compatible** -- Arranger auto-scans this directory |
| Journal path | `docs/plans/designs/YYYY-MM-DD-<topic>-dramaturg-journal.md` | `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md` | **Critical mismatch** |
| Journal lifecycle | Archived to `docs/archive/` on finalization | Persists in `decisions/` directory; NOT archived | **Critical mismatch** |
| Artifact count | "Two artifacts" (SKILL.md:59) | Three artifacts listed in support-phases.md:269-272 (design doc, journal, Arranger Notes appendix) | **Minor inconsistency** -- Arranger Notes is part of the design doc, so two is correct |

### Analysis

The journal path/lifecycle mismatch is the single most critical alignment issue. The Arranger expects to find the journal at a path the Dramaturg never writes to, and even if the path were corrected, the Dramaturg archives the journal while the Arranger (and downstream Repetiteur) expect it to persist. This was previously identified in ARRANGER-CHANGES.md A-C1, but reviewing the full Dramaturg skill confirms the scope: the archival instruction is a `<mandatory>` tag in support-phases.md:255, meaning a Dramaturg instance will archive the journal as a non-negotiable action.

The design doc path is compatible because the Arranger's auto-scan targets `docs/plans/designs/` which is where the Dramaturg writes.

---

## 2. Decision Journal Format

### Dramaturg Journal Format

The Dramaturg uses plain markdown entry templates with structured fields (SKILL.md:356-384, approach-loop.md:316-344):

**User-discussed entries:**
```markdown
## Decision: [Topic]
**Phase:** Phase 5 -- Approach Loop
**Category:** [goal | use-case | decision]
**Decided:** [What was decided]
**User verbatim:** [exact words]
**User context:** [additional reasoning]
**Alternatives discussed:** [rejected options]
**Status:** settled
**Supersedes:** [reference or "--"]
```

**Research-backed entries:**
```markdown
## Research: [Topic]
**Phase:** Phase 5 -- Approach Loop
**Question:** [what was being validated]
**Tools used:** [tool list]
**Findings:** [what was discovered]
**Decision:** [what was decided]
**Arranger note:** [VERIFIED | PARTIAL | UNRESEARCHED]
**Status:** settled
**Supersedes:** [reference or "--"]
```

**Tension entries:**
```markdown
## Tension: [Short description]
**Phase:** [phase]
**Requirements in tension:** [list]
**Why they conflict:** [explanation]
**Current resolution approach:** [approach]
**Status:** acknowledged
```

### Arranger's Expected Journal Format

The Arranger specifies Tier 3 XML format (Arranger design Section 5, lines 322-375):

```xml
<journal feature="[feature-name]" type="arranger">
<metadata>...</metadata>
<sections>...</sections>
<section id="checkpoint-1">
<core>
## Checkpoint: [Phase Name] -- [Topic]
**Decision:** ...
**Rationale:** ...
</core>
</section>
</journal>
```

### Mismatches

| Item | Dramaturg Actual | Arranger Expected | Severity |
|------|------------------|-------------------|----------|
| Document wrapper | None (plain markdown) | `<journal>` XML wrapper with `<metadata>` | **Important** |
| Entry structure | Flat markdown headings | `<section id="...">` with authority tags | **Important** |
| Entry field names | Category, Decided, User verbatim, Arranger note | Decision, Rationale, Alternatives considered, Impact | **Important** |
| Entry type signal | Heading prefix (Decision/Research/Tension) | Authority tag (`<core>`/`<mandatory>`/`<context>`) | **Important** |
| Section index | None | `<sections>` list of entry IDs | **Important** |

### Analysis

The format divergence is significant but not blocking. The Arranger design explicitly states the main session never reads the raw journal -- a subagent distills it (Arranger design Section 2, line 58-64). This subagent can bridge any format difference. However, the Arranger's subagent prompt and the distillation protocol are designed around the Tier 3 XML format: looking for `<section>` tags, reading the `<sections>` index, interpreting authority tags for entry type classification. A markdown journal will require a different parsing approach.

The field name differences are more concerning for the subagent. The Arranger expects to find "Decision" and "Rationale" fields; the Dramaturg writes "Decided" and "User verbatim" / "User context" fields. The subagent distillation can handle this if properly prompted, but the prompts will need to account for Dramaturg's field naming convention.

The repertoire's `journal-conventions.md` uses yet another format (plain markdown with `Finding/Decision`, `Rationale`, `Alternatives considered`, `Impact` fields). This is a three-way format inconsistency.

---

## 3. Design Document Structure

### What Dramaturg Produces

The design doc has two required structural elements (SKILL.md:59-63, support-phases.md:199-263):

1. **Goals section (opening)** -- verbose narrative of user's goals, independent of technical decisions, self-contained
2. **Arranger Notes appendix (closing)** -- structured template with:
   - New Protocols / Unimplemented Patterns
   - Open Questions
   - Key Design Decisions

Between these, the body is "intentionally open-ended -- whatever best captures the design vision" (SKILL.md:255).

The design doc also follows synthesis quality rules (support-phases.md:225-231):
- Distinguishes confirmed decisions from design-level inferences
- Notes open questions explicitly
- Captures implied vs. stated requirements
- Captures intent over casual specifics
- Handles user self-corrections (final position wins)

### What Arranger Expects

The Arranger reads the design doc fully (Arranger design Section 2, line 57). During its Design Doc Completeness Check (lines 75-82), it assesses:
- Whether the design contains "enough substance to plan from"
- 3 or fewer simple questions = inline resolution; 4+ = re-engage Dramaturg

The Arranger also expects YAML frontmatter in its output plan that references the design doc (line 430: `design-doc: docs/plans/designs/[design-doc-name].md`), implying it expects to know the design doc filename.

### Alignment

| Item | Status | Notes |
|------|--------|-------|
| Goals section presence | **Aligned** | Arranger's overview section synthesizes from design doc goals |
| Arranger Notes appendix | **Aligned** | Arranger design explicitly references these flags for scoping feasibility audit |
| Open-ended body structure | **Aligned** | Arranger treats design doc as source of truth for what/why, not expecting rigid structure |
| Design doc metadata | **Minor gap** | Dramaturg produces no YAML frontmatter; Arranger needs to derive feature-name somehow |
| Completeness Check criteria | **Gap** | No shared checklist; Dramaturg's "implementation readiness test" (support-phases.md:179-190) is subjective |

### Analysis

The design doc structure alignment is strong. The Arranger correctly expects an open-ended document with Goals opening and Arranger Notes closing. The Arranger's completeness check is reasonable but operates independently from the Dramaturg's implementation readiness test at Phase 7 exit. There is no shared definition of "ready for the Arranger," which means the Arranger might reject a design the Dramaturg considered complete.

The Dramaturg's synthesis quality guidance (support-phases.md:225-231) directly serves the Arranger's needs -- distinguishing confirmed decisions from inferences, noting open questions, separating stated from implied requirements. This is a positive alignment point that wasn't explicitly planned as a cross-skill feature but works well as one.

---

## 4. File Locations and Paths

### Complete Path Mapping

| Artifact | Dramaturg Writes To | Arranger Expects At | Match? |
|----------|---------------------|---------------------|--------|
| Design doc | `docs/plans/designs/YYYY-MM-DD-<topic>-design.md` | `docs/plans/designs/` (auto-scan) or direct path | Yes (auto-scan compatible) |
| Decision journal (working) | `docs/plans/designs/YYYY-MM-DD-<topic>-dramaturg-journal.md` | `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md` | **No** |
| Decision journal (final) | Archived to `docs/archive/plans/designs/YYYY-MM-DD-<topic>-dramaturg-journal.md` | Persists at `decisions/{feature-name}/dramaturg-journal.md` | **No** -- opposite lifecycle |

### Feature-Name Derivation

The Arranger expects a `{feature-name}` path component for the decisions directory. The Dramaturg uses `YYYY-MM-DD-<topic>` slug patterns for filenames. Neither skill defines how to derive `{feature-name}` from the design doc.

Possible derivations:
- Strip the date prefix from the design doc filename (e.g., `2026-02-17-background-sync-design.md` -> `background-sync`)
- An explicit field in the design doc or Arranger Notes
- User input during Arranger invocation

This gap was previously identified in ARRANGER-CHANGES.md A-I4 but remains unresolved.

### The `decisions/{feature-name}/` Directory

The Dramaturg has no concept of this directory structure. It writes directly to `docs/plans/designs/`. The decisions directory is an Arranger/Repetiteur concept that was designed after the Dramaturg. The Dramaturg's skill files make no reference to creating or writing to this directory.

The repertoire's `journal-conventions.md` (lines 40-47) defines this directory as the canonical location for all journals. The Dramaturg predates this convention and was never updated to conform.

---

## 5. Scope Boundaries

### Design vs. Implementation Decision Boundary

The Dramaturg defines this boundary clearly:

**Dramaturg's scope protection rule** (SKILL.md:35, conversation-style section):
> When users push for implementation details (specific file paths, function signatures, variable names, package versions), redirect to architecture-level language.

**Gray zone guidance** (approach-loop.md:79-87, added from review findings):
> "The data model needs a location field with coordinates" = design. "Use a REAL column named lat" = implementation. "Use SQLite or PostgreSQL?" = architecture. When uncertain, err toward including it.

**The Arranger's position** (Arranger design Section 1, line 33):
> Not a re-litigation of design. Design decisions were settled in the Dramaturg phase.

### Alignment Assessment

| Boundary Question | Dramaturg | Arranger | Consistent? |
|-------------------|-----------|----------|-------------|
| Who decides what to build? | Dramaturg | Respects Dramaturg's design | Yes |
| Who decides how to build it? | Redirects to Arranger | Arranger's core job | Yes |
| Who validates technology feasibility? | Both -- Dramaturg during design discussion, Arranger during audit | Correct overlap | Yes |
| Who resolves infeasible design assumptions? | N/A (design complete) | Surfaces conflict to user | Yes |
| Who decides specific settings/configs? | Redirects to Arranger | Arranger decides | Yes |

### Analysis

The scope boundary is the strongest alignment point between the two skills. Both skills consistently describe the same division:
- **Dramaturg:** What and why, validated by research at the design level
- **Arranger:** How, validated by research at the implementation level

The "err toward including it" guidance in the Dramaturg prevents under-specification that would force the Arranger to make design decisions. The Arranger's "not a re-litigation of design" principle prevents it from overriding the Dramaturg's decisions.

One tension exists: the Dramaturg's technology feasibility research (e.g., "does FCM support this delivery pattern?") overlaps with the Arranger's feasibility audit. The VERIFIED/PARTIAL mechanism handles this correctly -- VERIFIED items let the Arranger skip re-audit. But there is no guidance for the case where the Dramaturg's VERIFIED finding turns out to be wrong (e.g., the Dramaturg verified FCM feasibility 3 months ago, but the Android API changed since). The Arranger's mandatory external verification rules would catch this, but the process of overriding a VERIFIED finding is undocumented.

---

## 6. Completeness Check

### Dramaturg's Implementation Readiness Test (Phase 7 Exit Gate)

From support-phases.md:179-190:
> Could the Arranger take this design and produce an executable implementation plan without needing to make design-level decisions?

Common gaps the Dramaturg checks for:
- Vague data model
- Unexplored error handling
- Missing edge cases that would force the Arranger to make design choices
- Interaction patterns described at high level but not worked through

### Arranger's Design Doc Completeness Check (Phase 1 Ingestion)

From Arranger design Section 2, lines 75-82:
- 3 or fewer simple questions to resolve all ambiguity -> inline resolution in Phase 3
- 4+ questions needed -> recommend re-engaging the Dramaturg

### Alignment Assessment

The two checks are conceptually consistent but operationally independent:

| Aspect | Dramaturg | Arranger | Alignment |
|--------|-----------|----------|-----------|
| Trigger point | Phase 7 exit (before finalization) | Phase 1 (ingestion) | Sequential -- Arranger validates Dramaturg's judgment |
| Threshold | Subjective ("could the Arranger plan from this?") | Quantitative (question count) | **Gap** -- no shared criteria |
| Failure action | Keep probing in current session | Re-engage Dramaturg (4+) or inline resolution (<=3) | **Gap** -- different response |
| Shared checklist | None | None | **Gap** |

### Analysis

The Dramaturg's readiness test asks the right question but provides no concrete criteria. The Arranger's check provides a concrete threshold (question count) but no shared definition of what counts as a "question" vs. a "missing design element." A design doc could pass the Dramaturg's subjective test but fail the Arranger's quantitative test, creating a frustrating loop for the user.

The ARRANGER-CHANGES.md A-I1 recommendation for a shared "design readiness checklist" remains the right approach. Candidate items (from the Dramaturg's common gaps list and the Arranger's ingestion expectations):
- Goals section present and non-empty
- Data model specified at architecture level
- Error handling addressed for critical paths
- Integration points with existing code identified
- All topics from the Topic Map settled (journal verification)
- Arranger Notes appendix present with at least one entry in each subsection

---

## 7. Assumptions the Arranger Makes About the Dramaturg

### Documented Assumptions (Explicit in Arranger Design)

1. **Design doc is the source of truth for what/why** (Section 2, line 77) -- Correct; matches Dramaturg intent.
2. **Decision journal has VERIFIED/PARTIAL flags** (Section 2, line 59) -- Correct; Dramaturg implements these.
3. **Dramaturg journal is at a specific path** (Section 2, line 58) -- **Incorrect**; path mismatch.
4. **Journal persists (not archived)** (Section 5, line 396) -- **Incorrect**; Dramaturg archives.
5. **Design doc replaces itself for conductor purposes** (Section 6, line 461) -- Correct conceptual intent but relies on plan Overview being comprehensive.

### Undocumented Assumptions (Implicit in Arranger Design)

1. **Feature-name is derivable from design doc** -- The `decisions/{feature-name}/` directory requires a feature name. No protocol exists for deriving it. The Dramaturg doesn't produce a feature-name field.

2. **Journal entries have "Decision" and "Rationale" fields** -- The Arranger's journal format (Section 5) and the repertoire's journal-conventions.md use these field names. The Dramaturg uses "Decided" and "User verbatim" / "User context" instead.

3. **Category tags (goal/use-case/decision) exist** -- The Arranger design doesn't explicitly mention consuming these, but the Repetiteur relies on them heavily (treating goals and use-cases as "inviolable constraints"). ARRANGER-CHANGES.md A-I7 recommends the Arranger also respect this distinction, but the current design doesn't account for it.

4. **Journal entries have a "Supersedes" field** -- The Dramaturg's templates include this field (approach-loop.md:325, 343). The Arranger's subagent distillation mentions "Final decisions still in effect" (Section 2, line 62), implying it expects to follow supersession chains. The mechanism works but is not explicitly connected.

5. **Tension entries exist** -- The Dramaturg produces "Tension" journal entries (SKILL.md:369-384) with status "acknowledged" rather than "settled." The Arranger design makes no mention of consuming tension entries. The Dramaturg explicitly states: "The Arranger receives tensions as design context, not as problems to solve" (SKILL.md:373). But the Arranger's subagent distillation prompt doesn't include tension entries in its expected return format.

6. **UNRESEARCHED status exists** -- The Dramaturg defines UNRESEARCHED as a third Arranger note status for decisions made without research backing (SKILL.md:367, research-strategy.md:181). The Arranger's ingestion only lists VERIFIED and PARTIAL (Section 2, lines 59-60). This was identified in the 2026-02-27 review as S2.

7. **Vision Baseline and Topic Map entries exist** -- The Dramaturg writes these special journal entries (Vision Baseline at Phase 2 exit, Vision Expansion at Phase 3 exit, Topic Map at Phase 4 exit, Section Approvals at Phase 6). The Arranger's subagent distillation doesn't mention consuming these specifically. They could provide valuable context (e.g., the Topic Map shows the scope of design coverage).

8. **Research Diversion entries may exist in the journal** -- The Dramaturg writes in-progress/settled research diversion entries (SKILL.md:297-302). A stale "in-progress" entry from a crashed session would appear in the journal. The Arranger's subagent should be aware these exist and may indicate incomplete research.

---

## Categorized Issues

### Critical (3)

**CR-1. Journal path mismatch**
- Dramaturg: `docs/plans/designs/YYYY-MM-DD-<topic>-dramaturg-journal.md`
- Arranger: `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md`
- Impact: Arranger cannot find the journal at the expected path
- Resolution: Align on one path convention. The `decisions/{feature-name}/` structure is better because it supports the multi-journal lifecycle (dramaturg + arranger + consultation journals co-located). The Dramaturg should be updated to write to this path.

**CR-2. Journal lifecycle conflict**
- Dramaturg: Archives journal to `docs/archive/` on finalization (mandatory rule, support-phases.md:255)
- Arranger: Expects journal to persist for Repetiteur and conductor reference (Section 5, line 396)
- Repertoire: Lifecycle says "NOT deleted or archived" (journal-conventions.md:182)
- Impact: After Dramaturg finalizes, the journal is moved to archive. The Arranger and Repetiteur cannot find it.
- Resolution: Remove Dramaturg's archival behavior. The journal persists in the decisions directory until the Conductor cleans up after implementation completes. This aligns with the repertoire convention.

**CR-3. Feature-name derivation protocol missing**
- No protocol exists for deriving the `{feature-name}` used in `decisions/{feature-name}/`
- The Dramaturg uses `YYYY-MM-DD-<topic>` slugs; the Arranger needs `{feature-name}`
- Impact: Even if CR-1 is resolved, neither skill knows how to construct the directory path
- Resolution: Either (a) add a `feature` field to the Dramaturg's design doc that the Arranger reads, (b) have the Dramaturg write the decisions directory path in the Arranger Notes appendix, or (c) derive from the design doc filename by stripping the date prefix and `-design` suffix.

### Important (7)

**IM-1. Journal format divergence**
- Dramaturg: Plain markdown with field-value pairs
- Arranger: Tier 3 XML with `<journal>`, `<section>`, authority tags
- Repertoire: Plain markdown with different field names
- Impact: Arranger's subagent distillation is designed around XML parsing. Must be re-designed for markdown.
- Resolution: Pick one canonical format. The markdown format is already in use by the built Dramaturg; updating to XML would require Dramaturg changes. Alternatively, accept the divergence and ensure the subagent distillation prompt handles markdown journals.

**IM-2. Entry field name inconsistency**
- Dramaturg: Decided, User verbatim, User context, Arranger note
- Arranger journal: Decision, Rationale, Alternatives considered, Impact
- Repertoire: Finding/Decision, Rationale, Alternatives considered, Impact
- Impact: Subagent distillation must know which field names to look for
- Resolution: Either standardize field names across the pipeline, or ensure the subagent prompt explicitly lists the Dramaturg's field names.

**IM-3. Tension entries not consumed by Arranger**
- Dramaturg produces Tension entries (SKILL.md:369-384) explicitly for the Arranger
- Arranger design makes no mention of consuming tension entries
- The Dramaturg says: "The Arranger receives tensions as design context, not as problems to solve"
- Impact: Design tensions are lost at the handoff
- Resolution: Add tension entry consumption to the Arranger's subagent distillation return format: "Design tensions (acknowledged, not resolved -- treat as constraints on phase structuring)"

**IM-4. UNRESEARCHED status not handled by Arranger ingestion**
- Dramaturg defines UNRESEARCHED as a third Arranger note status
- Arranger ingestion lists only VERIFIED and PARTIAL
- Impact: Decisions made during Gemini outages would be missed during feasibility scoping
- Resolution: Add UNRESEARCHED handling to the Arranger's ingestion: "UNRESEARCHED items require mandatory independent verification before planning decisions"

**IM-5. No shared design readiness checklist**
- Dramaturg's implementation readiness test is subjective
- Arranger's completeness check uses question count threshold
- Impact: Potential for design docs that pass one check but fail the other
- Resolution: Define a shared checklist consumed by both skills (see Section 6 analysis above)

**IM-6. Category tag (goal/use-case/decision) distinction not consumed by Arranger**
- Dramaturg carefully categorizes journal entries by type
- Goals and use-cases are "inviolable constraints" for downstream skills
- Arranger design doesn't mention respecting this distinction
- Impact: The Arranger might re-litigate or override a goal/use-case entry thinking it's a regular decision
- Resolution: Add to Arranger's ingestion phase: "Respect goal/use-case/decision categorization from Dramaturg journal. Goals and use-cases are user-confirmed constraints that must not be revisited without user approval."

**IM-7. Dramaturg's "three artifacts" vs "two artifacts" inconsistency**
- SKILL.md preamble says two artifacts (design doc + journal)
- support-phases.md final-design-doc `<context>` block lists three artifacts
- The Arranger Notes is part of the design doc, not separate
- Impact: A Dramaturg instance might produce a separate Arranger Notes file
- Resolution: Align the count in the Dramaturg. The correct answer is two artifacts; Arranger Notes is an appendix within the design doc.

### Minor (5)

**MN-1. Research conflict resolution vocabulary differs**
- Dramaturg: "Ground truth wins" (web evidence over Gemini synthesis)
- Arranger: "Official documentation > Gemini analysis > Training data"
- Same principle, different expression. No operational impact.

**MN-2. Arranger references `shared-rules.md`; actual repertoire is split files**
- Arranger design Section 9 references `references/shared-rules.md`
- Actual repertoire: `priority-chain.md`, `verification-rules.md`, `journal-conventions.md`, `output-format.md`
- Cosmetic mismatch in the unbuilt Arranger design.

**MN-3. Arranger embeds app-specific threat model rationale**
- "Personal task management app with a narrow threat model" (Section 1, line 39)
- The priority chain rationale in `repertoire/priority-chain.md` is generic
- The Arranger design should use the generic rationale.

**MN-4. Vision Baseline and Topic Map journal entries not in Arranger consumption spec**
- These entries provide useful context (scope coverage, original vision) but aren't listed in the subagent distillation return format
- Low-impact because the subagent reads the full journal and can extract relevant context.

**MN-5. Stale "in-progress" research diversion entries in journal**
- If a Dramaturg session crashes during research, the journal may contain an in-progress diversion entry
- The Arranger's subagent should flag these as potentially incomplete research
- Edge case with low probability.

### Observations (3)

**OB-1. Scope boundary is the strongest alignment point.** Both skills consistently describe the same design/implementation division. The Dramaturg's "err toward including it" guidance and the Arranger's "not a re-litigation of design" principle are complementary and well-enforced.

**OB-2. The VERIFIED/PARTIAL mechanism is well-designed.** Both skills describe it identically. It provides a clear optimization path: the Arranger skips feasibility audit for VERIFIED items and targets follow-up for PARTIAL items. The addition of UNRESEARCHED in the Dramaturg extends this logically.

**OB-3. The Dramaturg's synthesis quality guidance serves the Arranger well.** The rules about distinguishing confirmed decisions from inferences, noting open questions, and separating stated from implied requirements (support-phases.md:225-231) directly address what the Arranger needs from the design doc, even though they weren't designed as cross-skill features.

---

## Recommendations

### Resolution Priority Order

1. **CR-1 + CR-2 + CR-3** (journal path, lifecycle, feature-name) -- These must be resolved together as a single design decision. Recommended: adopt the `decisions/{feature-name}/` convention, remove archival, define feature-name derivation protocol. All three changes apply to the Dramaturg; the Arranger design already specifies the target state.

2. **IM-1 + IM-2** (journal format, field names) -- Decide canonical format. Options: (a) update Dramaturg to Tier 3 XML (high cost, changes a working skill), (b) update Arranger design to accept markdown journals (low cost, design not yet built), (c) accept divergence and ensure subagent distillation handles both (moderate cost, runtime complexity). Recommended: option (b) -- update the Arranger design to accept the Dramaturg's markdown format since the Arranger is unbuilt.

3. **IM-3 + IM-4 + IM-6** (tension entries, UNRESEARCHED, category tags) -- Add to the Arranger's subagent distillation specification. These are additive changes to the Arranger design.

4. **IM-5** (shared readiness checklist) -- Design a shared checklist. This is a new shared artifact in the repertoire.

5. **IM-7** (artifact count) -- Fix in Dramaturg support-phases.md.

6. **MN-1 through MN-5** -- Address during Arranger skill implementation. Low urgency.
