# Arranger Design — Consolidated Action Plan

**Date:** 2026-02-28
**Status:** Approved
**Supersedes:** `ARRANGER-CHANGES.md` (2026-02-23), `2026-02-27-arranger-alignment-review.md`
**Source reviews:** `2026-02-28-arranger-{dramaturg,conductor,repertoire-repetiteur,copyist}-alignment-review.md`

---

## Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Journal format | Flat markdown + `Strength` annotation field | Preserves semantic signal without breaking Repetiteur parsing |
| Plan output format | Full Tier 2 hybrid (YAML + `<sections>` + `<section>` + authority tags) | Richer downstream signaling; `output-format.md` and Repetiteur updated to match |
| Scope of this plan | Arranger design changes only; downstream changes documented per-skill | Separate passes fix downstream skills from their working docs |

---

## Section 1: Journal Format Changes

Target: Arranger design Section 5 (Decision Journal) and Section 2 (Ingestion)

### 1.1 Replace Tier 3 XML journal format with flat markdown + Strength annotation

The Arranger's decision journal currently specifies Tier 3 XML format (`<journal>`, `<section>`, authority-differentiated entries). Replace with flat markdown entries following the repertoire's template structure, adding a `**Strength:**` field:

- `mandatory` — user overrides, non-negotiable constraints
- `core` — standard decisions with rationale
- `context` — informational deviations, alternative approaches tried

### 1.2 Adopt repertoire field names

Use the repertoire's field names (`Finding/Decision`, `Rationale`, `Alternatives considered`, `Impact`) plus the `Strength` annotation and an `External input` field for Gemini findings. Drop the Arranger-specific field names (`Decision`, `Rationale` with different semantics).

### 1.3 Expand subagent distillation return format (Section 2)

Current ingestion lists VERIFIED, PARTIAL, final decisions, and abandoned approaches. Add:

- **UNRESEARCHED items** — decisions made without research backing, requiring mandatory independent verification before planning decisions
- **Tension entries** — acknowledged design tensions from the Dramaturg, treated as constraints on phase structuring, not problems to solve
- **Goal/use-case entries** — inviolable user-confirmed constraints, distinguished from regular decisions. Goals and use-cases must not be revisited without user approval.
- **Stale in-progress entries** — flag any research diversion entries with status "in-progress" as potentially incomplete research

### 1.4 Update journal path expectation

Change from `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md` assumption to match the standardized convention. The Arranger's ingestion phase should create the `decisions/{feature-name}/` directory if it doesn't exist yet (resilience during transition period).

### 1.5 Remove journal archival assumption

Remove any assumption that journals get archived. They persist in the decisions directory until Conductor cleanup. This aligns with `repertoire/journal-conventions.md` lifecycle rules.

---

## Section 2: Plan Output Format Changes

Target: Arranger design Section 6 (Output Format) and Section 3 Phase 6 (Finalization)

### 2.1 Add overview and phase-summary to plan-index

The plan-index currently only shows `phase:N` and `conductor-review:N` entries. Add `overview` and `phase-summary` line-range entries. The Conductor's bootstrap reads these "via plan-index line range."

```
<!-- plan-index:start -->
<!-- verified:YYYY-MM-DDTHH:MM:SS -->
<!-- overview lines:NN-NN -->
<!-- phase-summary lines:NN-NN -->
<!-- phase:1 lines:NN-NN title:"Phase Title" -->
<!-- conductor-review:1 lines:NN-NN -->
<!-- phase:2 lines:NN-NN title:"Phase Title" -->
<!-- conductor-review:2 lines:NN-NN -->
<!-- plan-index:end -->
```

### 2.2 Document authority tag consumption contract

The design already specifies `<mandatory>`, `<guidance>`, and `<core>` within phase sections. Add an explicit consumption contract section:

- `<mandatory>` in phase sections: not modifiable by Conductor, preserved verbatim by Copyist in task instructions
- `<guidance>` in phase sections: recommendations, Conductor can adapt, Copyist can rephrase for task-level context
- `<core>` in phase sections: primary implementation content
- `<mandatory>` in conductor-review sections: hard verification gates
- `<guidance>` in conductor-review sections: recommendations, not blocking gates

### 2.3 Define danger file annotation format

Currently unspecified. Add a lightweight convention for inline annotations in phase sections marking known file conflicts:

```
<!-- danger-file: path/to/file.dart shared-with="phase:3" -->
```

The Conductor treats these as supplementary — self-discovery is primary.

### 2.4 Standardize plan path convention

Standardize on `docs/plans/designs/{feature}-plan.md` to match the Conductor's MEMORY.md tracking format. Refine the Arranger's auto-scan for design docs to target `*-design.md` files specifically, excluding `*-plan.md` and any `superseded/` directory.

### 2.5 Explicit self-containment checklist

Strengthen the finalization verification by listing the 7 expected phase section components as a checklist the finalization subagent verifies:

1. Objective
2. Prerequisites (specific file paths and exports, not vague references)
3. Implementation detail
4. Integration points
5. Frontend guidelines (when applicable, inlined)
6. Expected outcomes
7. Testing recommendations

### 2.6 Confirm `<section>` tag role

`<sections>` index and `<section>` tags coexist with sentinel markers as a secondary navigation mechanism. Sentinels + plan-index remain the primary navigation system; `<section>` tags provide fallback navigation and structural validation.

---

## Section 3: Upstream Handoff Changes

Target: Arranger design Sections 1, 2, 3

### 3.1 Add feature-name derivation protocol

Add to Section 2 (Invocation & Ingestion): derive `{feature-name}` by stripping the date prefix and `-design` suffix from the design doc filename.

Example: `2026-02-17-background-sync-design.md` → `background-sync`

Used for: decisions directory path, YAML frontmatter `feature` field, downstream branch naming.

### 3.2 Update journal path

The Arranger expects the Dramaturg journal at `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md`. The Arranger's ingestion phase creates the directory if it doesn't exist yet.

### 3.3 Add Design Doc Completeness Checklist

Replace the vague "enough substance to plan from" assessment with a concrete checklist:

- Goals section present and non-empty
- Data model specified at architecture level
- Error handling addressed for critical paths
- Integration points with existing code identified
- Arranger Notes appendix present with at least one entry
- No unresolved PARTIAL items that block phase structuring

This checklist should eventually become a shared repertoire contract.

### 3.4 Rephrase "no downstream research" aspiration

Change from hard guarantee to objective: "Minimize downstream research; the Conductor may still research decomposition ambiguities, but should never need to research protocols, configurations, or integration patterns."

### 3.5 Add Gemini degraded mode policy

If Gemini MCP is unavailable, present the user with the choice to:
- Proceed in degraded mode with mandatory UNRESEARCHED marking on all verification-required items, or
- Abort

Don't silently proceed, don't hard-stop without user input.

### 3.6 Add upstream referral protocol

When the Arranger determines the design needs fundamental revision (4+ questions needed, or systemic feasibility failure), present:

- List of specific gaps or infeasible assumptions
- Topics needing Dramaturg-level exploration
- Suggested focus areas for a follow-up session
- Exact invocation suggestion: `/dramaturg docs/plans/designs/{design-doc-name}.md`

---

## Section 4: Reference & Naming Cleanup

Target: Throughout the Arranger design doc

### 4.1 `shared-rules.md` → individual repertoire files

Replace all references to the monolithic `references/shared-rules.md` with the actual repertoire files:
- `repertoire/priority-chain.md`
- `repertoire/verification-rules.md`
- `repertoire/journal-conventions.md`
- `repertoire/output-format.md`

Update the skill file structure section accordingly.

### 4.2 Remove Repetiteur notes reference

Section 9 references `docs/plans/designs/2026-02-17-repetiteur-skill-notes.md`. This file is archived and fully superseded by the built Repetiteur skill and its design doc. Remove the reference.

### 4.3 Resolve `hybrid-document-structure.md` reference

Referenced in Sections 1, 5, 6 but doesn't exist in the repo. Either:
- Create it in repertoire as a shared contract defining the tier system (preferred — multiple skills reference it), or
- Replace references with inline definitions of Tier 2 and Tier 3

### 4.4 Remove app-specific threat model rationale

Remove "personal task management app with a narrow threat model" from Section 1. Reference the generic rationale in `repertoire/priority-chain.md` instead.

### 4.5 Update "pending downstream changes" language

Rewrite statements about Conductor/Copyist updates being pending. Acknowledge what's already implemented, list only remaining deltas (authority tag consumption, `<section>` tag awareness, plan-index completeness).

### 4.6 Fix Copyist task-split language

Change: "Copyist can freely split tasks"
To: "Copyist may propose task splits; Conductor approves the split plan."

### 4.7 Fix Conductor phase-read statement

Change: "The Conductor will not read phase sections directly"
To: "The Conductor reads phase sections for decomposition context but does not implement from them. The Copyist and Musicians are the implementation consumers."

---

## Downstream Change Documentation

The following working docs are created alongside this action plan. Each contains full context for a separate session to execute the changes.

| Skill | Working Doc | Key Changes |
|-------|-------------|-------------|
| repertoire | `repertoire/docs/working/2026-02-28-arranger-alignment-changes.md` | `output-format.md` Tier 2 update, `journal-conventions.md` Strength field, `verification-rules.md` path fix, `hybrid-document-structure.md` creation |
| dramaturg | `dramaturg/docs/working/2026-02-28-arranger-alignment-changes.md` | Journal path/lifecycle, feature-name directory, artifact count, transition guidance, scope protection |
| conductor | `conductor/docs/working/2026-02-28-arranger-alignment-changes.md` | Authority tag interpretation, Tier 2 awareness, plan-index updates, plan path convention |
| copyist | `copyist/docs/working/2026-02-28-arranger-alignment-changes.md` | Authority tag consumption, `<section>` tag awareness, integration surfaces, example bug fixes |
| repetiteur | `repetiteur/docs/working/2026-02-28-arranger-alignment-changes.md` | `score-preparation/` → `repertoire/` migration, Tier 2 remaining plan format, journal Strength field parsing |

---

## Resolution Order

1. **Arranger design updates** (this plan) — Sections 1-4 applied to `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md`
2. **Repertoire contract updates** — `output-format.md`, `journal-conventions.md`, `verification-rules.md`, optionally `hybrid-document-structure.md`
3. **Dramaturg updates** — journal path, lifecycle, feature-name directory
4. **Repetiteur updates** — path migration, Tier 2 plan format, journal Strength parsing
5. **Conductor updates** — authority tags, Tier 2 awareness, plan-index consumption
6. **Copyist updates** — authority tag consumption, example bug fixes
7. **Cross-skill validation pass** — verify all contracts align after individual updates
