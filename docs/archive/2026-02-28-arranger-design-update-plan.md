# Arranger Design Document Update — Implementation Plan

> **For Claude:** This plan updates the Arranger design document at `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`. Apply changes section-by-section. Read the target section before each task. Use the action plan at `stagecraft/docs/working/2026-02-28-arranger-design-action-plan.md` for full rationale on each change.

**Goal:** Apply all 25 action items from the consolidated action plan to the Arranger design document.

**Architecture:** Section-by-section editing pass through the design doc, applying all relevant action items to each section. Changes do not create new files — this is purely editing an existing 764-line design document.

**Source Document:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Action Plan:** `stagecraft/docs/working/2026-02-28-arranger-design-action-plan.md`

---

## Task Overview

| Task | Design Doc Section | Action Items | Impact |
|------|-------------------|--------------|--------|
| 1 | Section 1 (Problem Statement) | 3.4, 3.5, 4.4 | Light edits |
| 2 | Section 2 (Invocation & Ingestion) | 1.3, 1.4, 2.4, 3.1, 3.2, 3.3, 3.6 | Heavy — new subsections + rewrites |
| 3 | Section 3 (Workflow Phases) | 2.5 | Light — finalization checklist addition |
| 4 | Section 4 (Research Strategy) | 3.4, 3.5 | Light — rephrase + new policy |
| 5 | Section 5 (Decision Journal) | 1.1, 1.2, 1.5 | **Rewrite** — format overhaul |
| 6 | Section 6 (Output Format) | 2.1, 2.2, 2.3, 2.5, 2.6, 4.5 | Moderate — additions + updates |
| 7 | Section 9 (Plugin Structure) | 4.1, 4.2, 4.6, 4.7 | Moderate — structural refs + language fixes |
| 8 | Section 10 (Design Decisions Log) | 1.1-related | Light — update affected decision rows |
| 9 | Verification | All | Grep-based consistency check |

---

### Task 1: Section 1 — Problem Statement & Skill Identity

**File:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Lines:** 1-47

**Step 1: Remove app-specific threat model rationale (4.4)**

Find the priority chain paragraph (line 37-39):
```
**Priority chain for technical decisions:** compatibility > reliability > efficiency > security > performance

Prefer modern but not bleeding-edge approaches. Future-favoring: prefer current code, APIs, and protocols that are well-established and well-documented. Security ranks below efficiency because this is a personal task management app with a narrow threat model — compatibility and reliability directly affect user experience, efficiency affects battery/resources, while security concerns are real but bounded.
```

Replace with:
```
**Priority chain for technical decisions:** compatibility > reliability > efficiency > security > performance (see `repertoire/priority-chain.md` for full rationale)

Prefer modern but not bleeding-edge approaches. Future-favoring: prefer current code, APIs, and protocols that are well-established and well-documented.
```

**Step 2: Rephrase "no downstream research" (3.4)**

Find in line 15:
```
The Conductor and later stages should never need additional research.
```

Replace with:
```
The Conductor and later stages should not need to research protocols, configurations, or integration patterns — though the Conductor may still research decomposition ambiguities when "boots hit the ground."
```

**Step 3: Update Gemini requirement to allow degraded mode (3.5)**

Find (line 45):
```
**Gemini MCP is required.** The Arranger does not launch without Gemini MCP available. Mandatory external verification categories cannot be satisfied without it. Brave-search is a fallback for web queries but cannot replace Gemini's analysis and verification capabilities.
```

Replace with:
```
**Gemini MCP is expected.** If Gemini MCP is unavailable at launch, the Arranger presents the user with a choice: (1) proceed in degraded mode with mandatory UNRESEARCHED marking on all items that would normally require external verification, or (2) abort. The Arranger does not silently proceed without verification capability, and does not hard-stop without user input. Brave-search remains available as a web query fallback but cannot replace Gemini's analysis capabilities.
```

---

### Task 2: Section 2 — Invocation & Ingestion

**File:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Lines:** 49-87

**Step 1: Add feature-name derivation protocol (3.1)**

After the "Invocation Patterns" subsection (after line 55), add a new subsection:

```markdown
### Feature-Name Derivation

The Arranger derives `{feature-name}` from the design doc filename by stripping the date prefix and `-design` suffix:

- `2026-02-17-background-sync-design.md` → `background-sync`
- `2026-03-01-notification-system-design.md` → `notification-system`

This feature-name is used for:
- Decisions directory path: `docs/plans/designs/decisions/{feature-name}/`
- YAML frontmatter `feature` field in the implementation plan
- Journal filename: `arranger-journal.md` within the decisions directory
- Downstream branch naming by the Conductor
```

**Step 2: Update journal path and add resilience (3.2, 1.4)**

Find in "Ingestion Behavior" (line 58):
```
launches a subagent to distill the dramaturg's decision journal** (located at `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md`).
```

This path is already correct. No change needed to the path itself. Add after the subagent return list (after line 64), a resilience note:

```markdown
If the `decisions/{feature-name}/` directory does not yet exist, the Arranger creates it during ingestion. If the Dramaturg journal is not found at the expected path, the Arranger proceeds without journal distillation — the design document is the primary input.
```

**Step 3: Expand subagent distillation return format (1.3)**

Find the subagent return list (lines 59-64):
```
- VERIFIED items the Arranger can skip in feasibility audit (one-liner each)
- PARTIAL items needing follow-up with specifics
- Final decisions still in effect with rationale
- Abandoned/invalidated approaches as one-liners (e.g., "SSE abandoned for FCM due to reliability")
```

Replace with:
```
- VERIFIED items the Arranger can skip in feasibility audit (one-liner each)
- PARTIAL items needing follow-up with specifics
- UNRESEARCHED items — decisions made without research backing, requiring mandatory independent verification before the Arranger builds on them
- Final decisions still in effect with rationale
- Abandoned/invalidated approaches as one-liners (e.g., "SSE abandoned for FCM due to reliability")
- Goal/use-case entries — inviolable user-confirmed constraints that must not be revisited without user approval
- Tension entries — acknowledged design tensions treated as constraints on phase structuring, not problems to solve
- Any stale in-progress entries flagged as potentially incomplete research
```

**Step 4: Standardize plan path and auto-scan (2.4)**

Find in "Invocation Patterns" (line 55):
```
**`/arranger` (no file)** — Scans `docs/plans/designs/` for design documents. If exactly one is found, auto-selects it. If multiple are found, prompts the user to choose. If none are found, reports and stops.
```

Replace with:
```
**`/arranger` (no file)** — Scans `docs/plans/designs/` for `*-design.md` files, excluding any `superseded/` subdirectory and `*-plan.md` files. If exactly one design doc is found, auto-selects it. If multiple are found, prompts the user to choose. If none are found, reports and stops.
```

**Step 5: Overhaul design doc completeness check (3.3)**

Find the "Design Doc Completeness Check" subsection (lines 75-82):
```
### Design Doc Completeness Check

The Arranger treats the design doc as the source of truth for *what* and *why*. During ingestion, it assesses whether the design contains enough substance to plan from:

- **3 or fewer simple questions** would resolve all ambiguity → the Arranger pauses to discuss with the user during Phase 3 (Implementation Discussion). Small gaps handled inline.
- **4+ questions needed** → the Arranger recommends re-engaging the Dramaturg to flesh out the design. Too much ambiguity for the Arranger to resolve — the fix is upstream, not mid-stream.

The Arranger does NOT fill design gaps itself — that would be re-litigating design, which is explicitly not its job.
```

Replace with:
```
### Design Doc Completeness Check

The Arranger treats the design doc as the source of truth for *what* and *why*. During ingestion, it assesses whether the design contains enough substance to plan from using a concrete checklist:

- Goals section present and non-empty
- Data model specified at architecture level
- Error handling addressed for critical paths
- Integration points with existing code identified
- Arranger Notes appendix present with at least one entry
- No unresolved PARTIAL items that block phase structuring

**Assessment outcomes:**
- **3 or fewer simple questions** would resolve remaining ambiguity → the Arranger pauses to discuss with the user during Phase 3 (Implementation Discussion). Small gaps handled inline.
- **4+ questions needed, or systemic feasibility failure** → the Arranger recommends re-engaging the Dramaturg (see Upstream Referral Protocol below).

The Arranger does NOT fill design gaps itself — that would be re-litigating design, which is explicitly not its job.
```

**Step 6: Add upstream referral protocol (3.6)**

After the "Design Doc Completeness Check" subsection and before "Design Doc Consumption" (before line 84), add:

```markdown
### Upstream Referral Protocol

When the Arranger determines the design needs fundamental revision (4+ questions needed, or systemic feasibility failure during Phase 2), it presents:

- List of specific gaps or infeasible assumptions found
- Topics needing Dramaturg-level exploration
- Suggested focus areas for a follow-up session
- Exact invocation suggestion: `/dramaturg docs/plans/designs/{design-doc-name}.md`

The Arranger does not silently work around design gaps or proceed hoping for the best. The fix is upstream.
```

---

### Task 3: Section 3 — Workflow Phases

**File:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Lines:** 90-239

**Step 1: Add self-containment checklist to Phase 6 finalization (2.5)**

Find the finalization checklist (lines 228-237):
```
**Finalization checklist:**
- Sentinel marker validation — all HTML comment pairs present and correctly matched
- Header-sentinel consistency — markdown headers match their comment anchors
- Hybrid structure tag validation — authority tags (`<mandatory>`, `<guidance>`, `<context>`) and `<core>` tags present and properly nested within phase and checkpoint sections; Tier 2 conventions followed
- Section tag validation — `<section id="...">` tags present within each sentinel-bounded area, IDs match `<sections>` index
- YAML frontmatter validation — required fields present (title, date, type, tier, feature)
- Line range index generation — accurate to current file state
- Self-containment spot-check — sample phase sections for completeness, including authority tag coverage
- Large document marker test — verify sentinels and section tags are findable across the full document length
- User override flags propagated — any overrides from journal are flagged in relevant conductor checkpoint sections
```

Replace the self-containment spot-check item with a full checklist:
```
**Finalization checklist:**
- Sentinel marker validation — all HTML comment pairs present and correctly matched
- Header-sentinel consistency — markdown headers match their comment anchors
- Hybrid structure tag validation — authority tags (`<mandatory>`, `<guidance>`, `<context>`) and `<core>` tags present and properly nested within phase and checkpoint sections; Tier 2 conventions followed
- Section tag validation — `<section id="...">` tags present within each sentinel-bounded area, IDs match `<sections>` index
- YAML frontmatter validation — required fields present (title, date, type, tier, feature, design-doc)
- Line range index generation — accurate to current file state
- Self-containment verification — each phase section contains all 7 expected components:
  1. Objective
  2. Prerequisites (specific file paths and exports, not vague references)
  3. Implementation detail
  4. Integration points
  5. Frontend guidelines (when applicable, inlined)
  6. Expected outcomes
  7. Testing recommendations
- Large document marker test — verify sentinels and section tags are findable across the full document length
- User override flags propagated — any overrides from journal are flagged in relevant conductor checkpoint sections
```

---

### Task 4: Section 4 — Research & Verification Strategy

**File:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Lines:** 243-307

**Step 1: Rephrase "no downstream research" (3.4)**

Find (line 245):
```
**The Arranger is the primary research checkpoint in the pipeline — the conductor and later stages should never need additional research.** Every protocol validated, every platform limitation discovered, every configuration value checked — here, not downstream. (The Repetiteur handles research for mid-implementation blockers if they arise.)
```

Replace with:
```
**The Arranger is the primary research checkpoint in the pipeline.** The objective is to minimize downstream research: the Conductor may still research decomposition ambiguities when conditions on the ground differ from expectations, but should never need to research protocols, configurations, or integration patterns. Every protocol validated, every platform limitation discovered, every configuration value checked — here, not downstream. (The Repetiteur handles research for mid-implementation blockers if they arise.)
```

**Step 2: Add Gemini degraded mode policy (3.5)**

After the "Mandatory External Verification" subsection (after line 254), add:

```markdown
### Gemini Degraded Mode

If Gemini MCP becomes unavailable mid-session (after initially being available), the Arranger:
1. Pauses and notifies the user
2. Offers the choice to continue in degraded mode or wait/abort
3. If continuing, marks all subsequent items that would require Gemini verification as UNRESEARCHED in the journal
4. Adds a plan-level note in the Overview section flagging that some verification was degraded

This complements the launch-time check in Section 1 — that check handles initial unavailability, this handles mid-session loss.
```

---

### Task 5: Section 5 — Decision Journal (FORMAT REWRITE)

**File:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Lines:** 310-398

This is the largest single change. The entire Format subsection (lines 321-382) is replaced.

**Step 1: Replace the Format subsection (1.1, 1.2)**

Find the Format subsection (lines 321-382), from `### Format` through the closing description paragraph ending with "The journal is a log, not a living document."

Replace with:

```markdown
### Format

Flat markdown, append-only, at `docs/plans/designs/decisions/{feature-name}/arranger-journal.md`. Follows the repertoire's journal conventions (`repertoire/journal-conventions.md`) with the shared field set.

Each entry uses this template:

```markdown
## [Entry N]: [Topic]

**Finding/Decision:** [What was decided or discovered]
**Rationale:** [Why, including research findings that informed this]
**Alternatives considered:** [What was rejected and why]
**Impact:** [What this affects — phases, other decisions, integration points]
**External input:** [Gemini findings, web search results, or "None"]
**Strength:** [mandatory | core | context]
```

The `Strength` field carries semantic meaning for downstream consumers:
- **`mandatory`** — User overrides and non-negotiable constraints. Carries strongest constraint weight. The Repetiteur's journal analysis treats these as inviolable.
- **`core`** — Standard implementation decisions with rationale. Normal constraint weight, binding but potentially revisable during Repetiteur consultation.
- **`context`** — Informational deviations, alternative approaches tried, abandoned directions. Weakest constraint weight, informational only.

Entry type conventions:
- **Checkpoint entries** use `Strength: core` — standard decisions made during Phases 2-5
- **User override entries** use `Strength: mandatory` — the user disagrees with research findings and forces a decision. These MUST be propagated to the relevant conductor checkpoint section with the flag: `USER OVERRIDE: [setting] set to X despite research indicating Y — user has workaround, see journal entry [ref]`
- **Deviation entries** use `Strength: context` — record of what changed during a loop-back and why. The deviation itself is logged; subsequent checkpoint entries contain the actual new decisions.

New entries are appended at the end of the file. Entries are never edited — if a later decision supersedes an earlier one, the new entry references the old one by entry number. The journal is a log, not a living document. The final plan is compiled from the *current state* of decisions (latest wins), not from the journal directly.
```

**Step 2: Verify lifecycle section is correct (1.5)**

Read the Lifecycle subsection (lines 393-397). It should already say journals are NOT archived or deleted. Verify this text is present and accurate — no change expected. The key line is:

```
3. **Persists** after finalization — NOT archived or deleted.
```

If present, no change needed. If it mentions archival, remove that language.

---

### Task 6: Section 6 — Output Format & Section Markers

**File:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Lines:** 401-598

**Step 1: Add overview and phase-summary to plan-index example (2.1)**

Find the plan-index example (lines 435-441):
```
<!-- plan-index:start -->
<!-- verified:YYYY-MM-DDTHH:MM:SS -->
<!-- phase:1 lines:NN-NN title:"[Phase Title]" -->
<!-- conductor-review:1 lines:NN-NN -->
<!-- phase:2 lines:NN-NN title:"[Phase Title]" -->
<!-- conductor-review:2 lines:NN-NN -->
<!-- plan-index:end -->
```

Replace with:
```
<!-- plan-index:start -->
<!-- verified:YYYY-MM-DDTHH:MM:SS -->
<!-- overview lines:NN-NN -->
<!-- phase-summary lines:NN-NN -->
<!-- phase:1 lines:NN-NN title:"[Phase Title]" -->
<!-- conductor-review:1 lines:NN-NN -->
<!-- phase:2 lines:NN-NN title:"[Phase Title]" -->
<!-- conductor-review:2 lines:NN-NN -->
<!-- plan-index:end -->
```

**Step 2: Update "Pipeline Prerequisites" language (4.5)**

Find (lines 415-417):
```
### Pipeline Prerequisites

The dual-audience format, sentinel marker system, and hybrid structure tag conventions described below are **specifications for how the pipeline will work**. The conductor and copyist skills will need updates to support line-range extraction, selective reading, authority tag interpretation, and checkpoint-based verification. These updates are a separate concern from the Arranger design — they will be implemented when the Arranger skill is built.
```

Replace with:
```
### Pipeline Prerequisites

The conductor and copyist already support sentinel marker parsing, plan-index line-range extraction, selective reading, and checkpoint-based verification. Remaining deltas documented in each skill's `docs/working/` directory:
- **Conductor:** Authority tag interpretation in phase-execution reference, Tier 2 document awareness, `overview`/`phase-summary` plan-index entries
- **Copyist:** Authority tag consumption contract, `<section>` tag awareness, integration surface handling guidance
```

**Step 3: Add authority tag consumption contract (2.2)**

After the "Self-Containment Rules" subsection (after line 598, before the section divider), add a new subsection:

```markdown
### Authority Tag Consumption Contract

The Arranger uses authority tags strategically, knowing how each downstream consumer interprets them:

**In phase sections (consumed by Copyist):**
- `<mandatory>` — Non-negotiable constraints. The Copyist preserves these verbatim in task instructions. The Conductor cannot override them, even within intra-phase authority. Musicians must follow exactly.
- `<guidance>` — Recommended approaches. The Copyist can adapt these for task-level context. The Conductor can adjust based on runtime conditions.
- `<core>` — Primary implementation content. The substance to be decomposed into task steps.

**In conductor-review sections (consumed by Conductor):**
- `<mandatory>` — Hard verification gates. Must pass before the Conductor proceeds to the next phase.
- `<guidance>` — Recommendations for task decomposition and next-phase approach. Not blocking gates — the Conductor should consider them but can proceed if not fully satisfied.

This contract means the Arranger's tag choices have downstream consequences. Marking something `<mandatory>` in a phase section means no downstream agent can modify it without user involvement. Use this tag deliberately for constraints that genuinely must be preserved.
```

**Step 4: Add danger file annotation format (2.3)**

After the new "Authority Tag Consumption Contract" subsection, add:

```markdown
### Danger File Annotations

Phase sections may contain inline annotations marking known file conflicts discovered during Phase 4 (Phase Structuring):

```
<!-- danger-file: path/to/file.dart shared-with="phase:3" -->
```

The Conductor treats these as supplementary starting points for danger file identification — self-discovery during implementation remains the primary method. The Arranger adds these annotations when cross-phase file conflicts are identified during phase structuring, providing the Conductor with advance warning.
```

**Step 5: Confirm `<section>` tag role (2.6)**

Verify that the existing text at lines 546-558 and 580 already describes `<section>` tags as providing fallback navigation alongside the primary sentinel/plan-index system. The current text at line 546 says:

```
The plan-index and `<sections>` serve complementary purposes.
```

And line 554-558 describes the three-layer system. This is already correct per action item 2.6. No change needed — just verify.

---

### Task 7: Section 9 — Plugin Structure

**File:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Lines:** 686-724

**Step 1: Replace shared-rules.md with individual repertoire files (4.1)**

Find (lines 690-700):
```
### Location

```
~/.claude/skills/arranger/
  SKILL.md
  references/
    shared-rules.md     — verification rules, output format, sentinels,
                          hybrid structure conventions, journal format,
                          priority chain, mental implementation,
                          self-containment rules
```

Standalone plugin alongside the other pipeline skills. The `shared-rules.md` reference is shared with the Repetiteur skill (see below).
```

Replace with:
```
### Location

```
~/.claude/skills/arranger/
  SKILL.md
  references/
    workflow-phases.md          — phase definitions, subagent prompts, deviation detection
    research-strategy.md        — verification categories, tool selection, mental implementation
    output-format.md            — section writing rules, self-containment checklist
    conversation-style.md       — interaction patterns, review process
```

Standalone plugin alongside the other pipeline skills. Shared contracts are consumed from repertoire:
- `repertoire/priority-chain.md` — trade-off ordering
- `repertoire/verification-rules.md` — mandatory external verification categories
- `repertoire/journal-conventions.md` — decision journal format and lifecycle
- `repertoire/output-format.md` — plan structure, sentinel markers, plan-index
```

**Step 2: Remove Repetiteur notes reference (4.2)**

Find in the "Sibling — Repetiteur" paragraph (line 717):
```
See `docs/plans/designs/2026-02-17-repetiteur-skill-notes.md` for full design notes.
```

Remove this sentence. The file is archived and fully superseded by the built Repetiteur skill.

**Step 3: Fix Copyist task-split language (4.6)**

Find in the "Downstream — Copyist" paragraph (line 721):
```
the copyist estimates context requirements per task and can freely split tasks — the arranger's loose boundary structure enables this.
```

Replace with:
```
the copyist estimates context requirements per task and may propose task splits; the Conductor approves the split plan. The Arranger's loose boundary structure enables this flexibility.
```

**Step 4: Fix Conductor phase-read statement (4.7)**

Find in the "Downstream — Conductor" paragraph (line 719):
```
It will not read phase sections directly.
```

Replace with:
```
It reads phase sections for decomposition context but does not implement from them — the Copyist and Musicians are the implementation consumers.
```

Also find in Section 6, the conductor checkpoint description (line 413):
```
**it will not read phase sections.**
```

Replace with:
```
**it reads phase sections for decomposition context but does not implement from them.**
```

---

### Task 8: Section 10 — Design Decisions Log

**File:** `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
**Lines:** 727-764

**Step 1: Update journal format decision row**

Find the row (line 762):
```
| **Tier 3 journal with authority-differentiated entries** | Plain markdown journal, single entry type | `<journal>` wrapper with `<metadata>` and `<sections>`. Entry types distinguished by authority tag: `<core>` for decisions, `<mandatory>` for user overrides (must propagate), `<context>` for deviations (informational). Tag choice carries semantic meaning for downstream consumers. |
```

Replace with:
```
| **Flat markdown journal with Strength annotation** | Tier 3 XML journal, single entry type | Flat markdown entries following repertoire journal conventions. `Strength` field (`mandatory`/`core`/`context`) carries semantic meaning: `mandatory` for user overrides (must propagate), `core` for standard decisions, `context` for deviations (informational). Preserves semantic signal without breaking Repetiteur parsing of standard journal format. |
```

**Step 2: Update Repetiteur shared-rules decision row**

Find the row (line 748):
```
| **Repetiteur as separate skill** (not Arranger mode) | Consultation as Arranger mode, no consultation support | Interaction models diverge fundamentally — Arranger is interactive, Repetiteur is autonomous. Reinforcement principle makes mode-switching within one skill problematic. Shared `references/shared-rules.md` prevents drift. |
```

Replace with:
```
| **Repetiteur as separate skill** (not Arranger mode) | Consultation as Arranger mode, no consultation support | Interaction models diverge fundamentally — Arranger is interactive, Repetiteur is autonomous. Reinforcement principle makes mode-switching within one skill problematic. Shared repertoire contracts prevent drift. |
```

**Step 3: Update priority chain decision row**

Find the row (line 757):
```
| **compatibility > reliability > efficiency > security > performance** | No explicit priority chain, case-by-case | Explicit ordering prevents ambiguous trade-off decisions. Modern but not bleeding edge. Security below efficiency because personal app with narrow threat model — user experience and resource efficiency are more directly impactful. |
```

Replace with:
```
| **compatibility > reliability > efficiency > security > performance** | No explicit priority chain, case-by-case | Explicit ordering prevents ambiguous trade-off decisions. Modern but not bleeding edge. See `repertoire/priority-chain.md` for full rationale. |
```

---

### Task 9: Verification Pass

**Step 1: Grep for stale references**

Search the updated file for:
- `shared-rules.md` — should have zero occurrences
- `score-preparation/` — should have zero occurrences
- `repetiteur-skill-notes` — should have zero occurrences
- `personal task management` or `narrow threat model` — should have zero occurrences (moved to priority-chain.md)
- `will need updates` or `will be implemented when` — should have zero occurrences (replaced with current state)

**Step 2: Verify internal consistency**

- Journal format: All references to the journal should describe flat markdown, not XML/Tier 3
- Journal path: All references should use `docs/plans/designs/decisions/{feature-name}/arranger-journal.md`
- Feature-name: Derivation protocol should be referenced wherever `{feature-name}` appears
- Authority tags: Consumption contract should be consistent between Sections 6 and 9
- Conductor phase reads: Language should be consistent between Sections 6 and 9

**Step 3: Commit**

```bash
cd /home/kyle/claude/skills_staged/orchestra/stagecraft
git add docs/designs/2026-02-17-arranger-skill-design.md
git commit -m "docs: apply arranger design action plan — journal format, output format, upstream handoff, reference cleanup

Apply all 25 action items from 2026-02-28 consolidated action plan:
- Replace Tier 3 XML journal with flat markdown + Strength annotation
- Add feature-name derivation, upstream referral, Gemini degraded mode
- Add authority tag consumption contract, danger file annotations
- Expand plan-index with overview/phase-summary entries
- Replace shared-rules.md with individual repertoire file references
- Fix Conductor/Copyist language accuracy
- Remove app-specific rationale and stale references

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Post-Update

After this plan is executed:
- Archive this plan and the action plan to `stagecraft/docs/archive/`
- The downstream skill changes documented in each skill's `docs/working/2026-02-28-arranger-alignment-changes.md` can proceed independently
- Resolution order from the action plan: repertoire first, then dramaturg, repetiteur, conductor, copyist
