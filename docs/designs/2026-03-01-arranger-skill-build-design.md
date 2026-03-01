# Arranger Skill — Build Design

**Date:** 2026-03-01
**Status:** Approved
**Type:** Skill Build Specification
**Supersedes:** Structural recommendations in `2026-02-17-arranger-skill-design.md` (vision and identity preserved)

---

## 1. Relationship to Original Design

The original Arranger design document (`2026-02-17-arranger-skill-design.md`) defines the Arranger's vision, identity, workflow phases, research strategy, output format, and conversation style. That document remains the source of truth for **what the Arranger is and what it does.**

This build design specifies **how the skill is constructed** — file decomposition, reference architecture, SKILL.md structure, shared file strategy, config system, and builder instructions. Where the original design's structural recommendations (Section 9: Plugin Structure) conflict with patterns established by building the Repetiteur, Souffleur, and other skills, this document takes precedence.

Key divergences from the original design's Section 9:
- 6 phase references instead of 4 thematic references
- No separate `research-strategy.md` — shared content in repertoire, Arranger-specific bits inlined
- No separate `conversation-style.md` — identity in SKILL.md preamble, interaction mechanics in phase references
- No separate `output-format.md` — shared content in `repertoire/output-format.md`
- Deviation detection inline as `<context>` within phase references, not a standalone protocol
- Context management folded into SKILL.md, not a separate reference
- Config file system aligned with Souffleur's `.orchestra_configs/` pattern

---

## 2. Directory Structure & File Inventory

```
orchestra/arranger/
  .claude/
    settings.local.json
  .gitignore
  README.md
  docs/
    README.md
    archive/
    designs/
    working/
      repertoire-audit/
        output-format-audit.md
        journal-conventions-audit.md
        verification-rules-audit.md
        priority-chain-audit.md
        new-shared-proposals.md
    plans/
  skill/
    SKILL.md
    references/
      ingestion.md                        — Phase 1
      feasibility-audit.md                — Phase 2
      implementation-discussion.md        — Phase 3
      phase-structuring.md                — Phase 4
      section-writing.md                  — Phase 5
      finalization.md                     — Phase 6
    scripts/
      arranger-config.py                  — config resolver (Python primary)
      arranger-config.sh                  — config resolver (shell fallback)
    examples/
      example-implementation-plan.md      — complete original plan in Tier 2 format
      example-arranger-journal.md         — journal with checkpoint/override/deviation entries
```

### Shared Repertoire Files Consumed (Not Duplicated)

| Repertoire File | Consumption | Phases |
|----------------|-------------|--------|
| `repertoire/output-format.md` | Plan structure, sentinel markers, self-containment rules | 4, 5, 6 |
| `repertoire/journal-conventions.md` | Journal format, entry categories, checkpoint triggers | All |
| `repertoire/verification-rules.md` | Verification categories, mental implementation, tool selection | 2, 3, 6 |
| `repertoire/priority-chain.md` | Trade-off ordering | 3 |

### Examples Rationale

Two Arranger-specific examples complement the existing Repetiteur-focused repertoire examples:

- **`example-implementation-plan.md`** — Shows what the Arranger specifically produces: original plan (not remaining plan), no revision metadata, no task annotations, phase-level content without task headers. Demonstrates full Tier 2 format with all elements.
- **`example-arranger-journal.md`** — Shows Arranger-specific entry patterns: `Strength` field usage, checkpoint entries (`core`), user override entries (`mandatory`), deviation entries (`context`), context threshold checkpoint.

---

## 3. SKILL.md Structure

The SKILL.md serves three roles: **identity definition**, **router/prompt station**, and **cross-cutting rules**. It follows the Repetiteur's hub-and-spoke pattern adapted for the Arranger's interactive nature.

### Document Format

Tier 3 — `<skill>` wrapper, `<metadata>`, `<sections>`, all text inside authority tags.

### Sections

```xml
<skill name="arranger" version="1.0">
<metadata>
type: skill
tier: 3
</metadata>
<sections>
- mandatory-rules
- preamble
- reference-map
- config-loading
- startup-protocol
- feasibility-audit
- implementation-discussion
- phase-structuring
- section-writing
- finalization
- context-management
- examples
</sections>
```

### Section Content

**`mandatory-rules`** — Front-loaded, internalize-first:
- `<mandatory>` context watching: check context usage after each phase transition, compare against thresholds
- Database operations use comms-link only
- All file creation uses `temp/` — never `/tmp/`
- Read-only subagents for all code exploration — no edits permitted
- Design doc is source of truth for vision — never silently change design direction
- Mandatory external verification for all categories in `repertoire/verification-rules.md`
- No unimplemented protocol enters the plan without external verification
- Session targets 200k context budget

**`preamble`** — Identity, tone, operating principles:
- "You are the Arranger — the fact-checker and setting-decider." Not a code-writer, not a task-writer, not a re-litigation of design.
- Pipeline position table (Arranger vs Repetiteur comparison, from the Arranger's perspective)
- Conversation identity: fact-checker tone, explicit recommendations backed by evidence, one question at a time (with bounded-batch exception), never truncate reasoning
- Output length guidance
- Reinforcement principle: critical constraints appear at every point of relevance

**`reference-map`** — Sequential phases + cross-cutting references:

Sequential workflow stages:

| Phase | Reference | Role |
|-------|-----------|------|
| 1. Startup | `references/ingestion.md` | Design doc reading, journal distillation, overview |
| 2. Feasibility | `references/feasibility-audit.md` | Subagent verification, code path tracing, Gemini queries |
| 3. Discussion | `references/implementation-discussion.md` | Decision loops, research, user overrides |
| 4. Structuring | `references/phase-structuring.md` | Parallelization, cross-task integration surfaces |
| 5. Writing | `references/section-writing.md` | Plan section authoring, self-containment, review |
| 6. Finalization | `references/finalization.md` | Verification checklist, index generation, commit |

Cross-cutting references (used throughout, not in sequence):

| Reference | When |
|-----------|------|
| `repertoire/output-format.md` | Plan structure, during Phases 4-6 |
| `repertoire/journal-conventions.md` | Journal writing, all phases |
| `repertoire/verification-rules.md` | Research and verification, Phases 2-3 |
| `repertoire/priority-chain.md` | Trade-off decisions, Phase 3 |

**`config-loading`** — Config resolution protocol:
- Run `arranger-config.py` to load `.orchestra_configs/arranger`
- Apply resolved values (starting with `USE_GEMINI`)
- If Python fails, read <Shell Fallback> sub-section for `arranger-config.sh` invocation
- Config values shape behavior in downstream phases

**Phase sections (startup through finalization)** — Each follows the same pattern:
1. Contextualizes the phase — what it accomplishes, what inputs it expects, what outputs it produces
2. Describes the workflow at high level — enough to understand what you're about to do, not enough to skip the reference
3. Dispatches with `<reference path="references/..." load="required">`
4. States what to do when returning — "After completing Phase N, return here and proceed to Phase N+1"

**`context-management`** — Cross-cutting, folded into SKILL.md:
- 200k target budget, user is present for judgment calls
- `<mandatory>`: check context after each phase transition
- At 75%, recommend `/lethe compact` or session split to the user
- Phase 4/5 boundary is the recommended session split point
- Decision journal preserves all state from Phases 1-4 for fresh session pickup at Phase 5
- If user opts into 1M extended context, still target 200k — extra headroom is safety net

**`examples`** — Pointers to example files and shared repertoire examples.

### Lazy Student Prevention — SKILL.md Crafting Rules

The SKILL.md phase sections must be crafted with deliberate information asymmetry to prevent a model from executing on SKILL.md alone without loading the required reference files.

**What SKILL.md sections MUST include (priming):**
- The phase's purpose and what it accomplishes
- What inputs the phase expects and what outputs it produces
- Key concepts the model needs to understand before entering the reference
- The consequence of the phase's work
- `<reference path="..." load="required">` dispatch

**What SKILL.md sections MUST NOT include (forcing reference loading):**
- Subagent prompt templates
- Step-by-step procedures
- Decision trees or branching logic
- Specific checklist items
- Error handling procedures
- Configuration details or thresholds

**Language pattern:** Follow the Repetiteur's exemplar — active voice commitment to do the work, explicit deferral to the reference for how. Example: "The Arranger **will** distill the dramaturg journal via subagent **by following the distillation workflow defined in** the ingestion reference." A model reading only the SKILL.md section knows it must distill journals but cannot attempt distillation without the reference's subagent prompts, output format, and integration steps.

The builder must study the Repetiteur's SKILL.md sections as exemplars and replicate this information asymmetry for every Arranger phase section.

---

## 4. Phase Reference Files — Content Architecture

### Common Structure

Each reference file follows Tier 3 format:

```xml
<skill name="arranger-{phase-name}" version="1.0">
<metadata>
type: reference
parent-skill: arranger
tier: 3
protocol: {Phase Name}
</metadata>
<sections>...</sections>
...
</skill>
```

Each reference includes:
- `<mandatory>` context watching: "Read current context usage from system message. Compare against context thresholds in SKILL.md."
- Happy-path workflow as numbered/sequenced steps
- Error/deviation sub-sections within the same file, referenced by pointer from the happy path (e.g., "If Gemini query fails, read <Gemini Errors> below")
- `<reference>` pointers to repertoire files where shared contracts apply
- Return instruction at the end: "Return to SKILL.md and proceed to [next phase]"
- Loop-backs always described as "return to SKILL.md" — SKILL.md re-contextualizes before dispatching to any phase

### `ingestion.md` — Phase 1

**Purpose:** Read design doc, distill dramaturg journal, present overview, get user go-ahead.

**Sections:**
- **Config-aware startup** — Check `USE_GEMINI` from config. Three-path resolution:
  - `USE_GEMINI=false` → skip Gemini availability check, proceed in degraded mode with UNRESEARCHED marking. No user prompt — explicit opt-out.
  - `USE_GEMINI=true` + Gemini available → normal mode.
  - `USE_GEMINI=true` + Gemini unavailable → present choice: degraded mode or abort.
- **Invocation patterns** — `/arranger @path/to/design.md` or auto-scan of `docs/plans/designs/` for `*-design.md`.
- **Feature-name derivation** — Strip date prefix and `-design` suffix from filename.
- **Design doc reading** — Full read in main session.
- **Journal distillation** — Subagent dispatched to process dramaturg journal. Returns VERIFIED/PARTIAL/UNRESEARCHED items, final decisions, abandoned approaches, goal/use-case entries, tension entries. Main session never reads raw journal.
- **Design doc completeness check** — Concrete checklist (goals present, data model specified, error handling addressed, integration points identified, arranger notes appendix). Assessment: ≤3 questions → handle inline in Phase 3; 4+ questions → upstream referral.
- **Upstream referral protocol** — `<context>` sub-section. Present gaps, suggest `/dramaturg` re-engagement.
- **Overview presentation** — Pause before autonomous work. Present summary, key feasibility areas, PARTIAL items, estimated scope. Wait for user acknowledgment.
- **Gemini unavailability** — `<context>` sub-section. Three-path handling based on config + availability.
- Return to SKILL.md → Phase 2.

### `feasibility-audit.md` — Phase 2

**Purpose:** Front-load all feasibility verification so Phase 3 discussion is informed by reality, not training-data assumptions.

**Sections:**
- **What gets audited** — Unimplemented protocols/patterns, Android/cross-device services, cross-component integration points. Items NOT needing audit: simple implementations, wholly new files, VERIFIED items from dramaturg distillation.
- **Audit execution** — Read-only subagents for code path tracing. Gemini queries for protocol/platform verification (if `USE_GEMINI=true`). Web search for current documentation. `<reference path="repertoire/verification-rules.md" load="required">` for verification categories and tool selection.
- **Subagent prompt patterns** — Templates for code path tracing, startup sequence simulation, dependency checking. All include explicit read-only instruction.
- **Mid-session Gemini loss** — `<context>` sub-section. If `USE_GEMINI=true` and Gemini becomes unavailable mid-session: pause, notify user, offer continue-degraded or wait/abort. Subsequent items marked UNRESEARCHED.
- **Journal checkpoint** — All findings written to journal before proceeding. Conflicts with design surfaced to user, not silently worked around.
- **Deviation detection** — `<context>` sub-section. If user proposes changes during audit discussion, assess scope using hierarchy (structural/implementation/detail). Surface scope, user confirms, return to SKILL.md for re-routing if needed.
- Return to SKILL.md → Phase 3.

### `implementation-discussion.md` — Phase 3

**Purpose:** The core of the Arranger. Make implementation-level decisions through research-backed discussion.

**Sections:**
- **Decision→Research→Discussion loops** — Core interaction pattern. Identify decision point → research if needed → present findings with explicit recommendation → discuss with user → settle explicitly → journal the decision. Loop as needed.
- **Research integration** — `<reference path="repertoire/verification-rules.md" load="required">` for verification standards. `<reference path="repertoire/priority-chain.md" load="on-demand">` for trade-off decisions. Arranger-specific inline vs subagent heuristics as `<guidance>`.
- **Mental implementation** — For features with external interactions. Read-only subagents trace code paths. Gemini for mocked analysis review (if available). When NOT needed: simple/isolated implementations.
- **User override protocol** — When user disagrees with feasibility findings. Override journaled as `Strength: mandatory`. Flagged in relevant conductor checkpoint section: "USER OVERRIDE: [setting] set to X despite research indicating Y — user has workaround, see journal entry [ref]". Arranger yields but ensures visibility downstream.
- **One question at a time** — Default single question. Exception: multiple tight, bounded questions expecting short answers can batch. Open-ended questions never batch.
- **Deviation detection** — `<context>` sub-section. Same hierarchy, surface scope, user confirms, return to SKILL.md. Phase 3 emphasis: implementation vs structural distinction.
- **Journal conventions** — `<reference path="repertoire/journal-conventions.md" load="required">` for format. Strength field usage: `mandatory` for user overrides, `core` for standard decisions, `context` for deviations.
- Return to SKILL.md → Phase 4.

### `phase-structuring.md` — Phase 4

**Purpose:** Arrange settled decisions into phases optimizing for parallel execution.

**Sections:**
- **Strategic decomposition** — Naive (sequential per-feature) vs smart (prep → dependent → integration). Actively design phase structure to maximize copyist parallelization.
- **Cross-task integration awareness** — Identify integration surfaces where tasks from different phases must maintain contracts. These become explicit verification items in conductor checkpoint sections.
- **Danger file identification** — Files modified by multiple phases. Annotated inline with `<!-- danger-file: path shared-with="phase:N" -->`.
- **Phase Summary drafting** — Written during this phase, reviewed by user during Phase 5.
- **Journal checkpoint** — Phase structure written to journal. Critical checkpoint before section writing.
- **Deviation detection** — `<context>` sub-section. Phase 4 emphasis: structural changes requiring Phase 2 re-audit vs implementation changes requiring Phase 3 re-discussion.
- Return to SKILL.md → Phase 5 (recommended session split point noted).

### `section-writing.md` — Phase 5

**Purpose:** Write plan sections one at a time with user review of each.

**Sections:**
- **Overview writing** — Written first. Comprehensive enough that conductor understands the full picture without reading phase sections or design doc. The plan replaces the design doc for conductor purposes.
- **Interleaved writing** — Phase section → conductor checkpoint → phase section → conductor checkpoint. One section per message for review.
- **Self-containment verification** — Before presenting each section: does it pass the Copyist test? Seven expected components: objective, prerequisites, implementation detail, integration points, frontend guidelines (when applicable, inlined), expected outcomes, testing recommendations. `<reference path="repertoire/output-format.md" load="required">` for structural conventions.
- **Authority tag usage** — `<mandatory>` for non-negotiable constraints (copyist preserves verbatim). `<guidance>` for recommended approaches (copyist can adapt). `<core>` for primary implementation content. Tag choices have downstream consequences — stated explicitly.
- **Conductor checkpoint writing** — Review checklists, not just context. Cross-task integration checks, expected outcome verification, known risks, user override flags, context management guidance (lethe compact between phases).
- **Deviation detection** — `<context>` sub-section. Phase 5 emphasis: section review revealing missed integration concerns or implementation gaps.
- Return to SKILL.md → Phase 6.

### `finalization.md` — Phase 6

**Purpose:** Lock the plan through verification, generate index, commit.

**Sections:**
- **User gate** — Explicit user confirmation before finalization begins.
- **Verification subagent** — Launched with finalization checklist: sentinel marker pair matching, header-sentinel consistency, Tier 2 structural elements (YAML frontmatter, `<sections>` index completeness, `<section>` tag presence, authority tag well-formedness), self-containment verification (7 components per phase), large document marker test, user override flag propagation. `<reference path="repertoire/verification-rules.md" load="required">` for structural verification categories.
- **Verification failure handling** — `<context>` sub-section. If failures found: report to user, return to editing. Loop back through SKILL.md if structural or implementation changes needed.
- **Index generation** — Plan-index with line ranges generated after verification passes. `verified` timestamp. Must reflect final corrected state — never generated before verification.
- **Commit protocol** — Implementation plan committed to current branch. Journal remains in `docs/plans/designs/decisions/{feature-name}/`. Branch management is conductor's concern.
- **Post-finalization state** — Plan is locked. Presence of plan-index is conductor's verification that finalization occurred.

---

## 5. Config Script Design

### Interface Contract (Shared by Python and Shell)

- **Input:** `--project-dir` (default: current directory)
- **Resolution order:**
  1. `<project-dir>/.orchestra_configs/arranger`
  2. `<project-dir>/../.orchestra_configs/arranger`
- **Per-key resolution:** Both files are read (if they exist). For each known key, the value from the most-specific (project-dir) file wins. Keys not found in any file use defaults. This allows parent configs to set organization-wide defaults while project-level configs override specific keys.
- **Config file format:** `KEY=value`, `#` comments, blank lines ignored, strict/case-sensitive key matching, duplicate keys within a file → last value wins with warning, unknown keys → warning
- **Output:** JSON to stdout, warnings to stderr

### Starting Entries

| Key | Values | Default | Validation |
|-----|--------|---------|------------|
| `USE_GEMINI` | `true` / `false` | `true` | Exact case match. Invalid → warning + default. |

### Output Schema

```json
{
  "use_gemini": true,
  "source": {
    "use_gemini": "/project/.orchestra_configs/arranger"
  },
  "warnings": []
}
```

The `source` field is a per-key map showing which file provided each resolved value. Keys using defaults have no source entry. This supports debugging multi-file resolution.

### Scripts

**`skill/scripts/arranger-config.py`** — Primary implementation aligned with Souffleur's `souffleur-config.py`. Python 3, no external dependencies, JSON output.

**`skill/scripts/arranger-config.sh`** — Shell fallback with same interface contract and JSON output. Referenced from SKILL.md's config-loading section: "If Python execution fails, read <Shell Fallback> below."

### Integration

Config loads during SKILL.md's `config-loading` section, before dispatching to `ingestion.md`. The resolved `use_gemini` value is carried into the ingestion reference's three-path Gemini check. Future config entries extend behavior without changing the loading protocol.

---

## 6. README

The README follows The Elevated Stage standard demonstrated by sibling skills (Repetiteur, Musician, Souffleur). Required sections:

1. **Title and musical metaphor** — The arranger takes a composition and creates the detailed arrangement for the orchestra: which instruments play when, what's simultaneous, what's sequential.
2. **Value proposition table** — Arranger planning vs ad-hoc conductor planning. Why front-loaded research-backed planning prevents downstream failures.
3. **Pipeline position** — ASCII diagram: `Dramaturg → Arranger → Conductor → Musician` with Copyist and Repetiteur relationships.
4. **Usage** — Invocation patterns (`/arranger @path` and `/arranger` auto-scan).
5. **Workflow lifecycle** — ASCII flow diagram of 6 phases with loop-back points. Narrative description of each phase.
6. **Shared protocols** — Repertoire consumption (output format, journal conventions, verification rules, priority chain).
7. **Context management** — 200k target, Phase 4/5 split recommendation.
8. **Configuration** — `.orchestra_configs/arranger`, `USE_GEMINI`, resolution order, per-key precedence.
9. **Outputs** — Implementation plan (Tier 2) + arranger journal.
10. **Project structure** — Directory tree.
11. **Requirements** — Gemini MCP (desired, configurable via `USE_GEMINI`), Claude Code with skill/plugin support.
12. **Known limits** — Honest assessment of current limitations.
13. **Origin** — [The Elevated Stage](https://github.com/The-Elevated-Stage) link + design doc reference.

---

## 7. Shared Files (Repertoire) Audit Mandate

### Builder Posture

The builder's default is **shared over local**. Content that could live in either an Arranger reference or a repertoire file belongs in repertoire. Drift between the Arranger and Repetiteur's outputs is the primary risk this mandate prevents.

### Per-File Audit

For every repertoire file the Arranger consumes, the builder produces a shared file audit artifact listing:

1. **What the Arranger needs from this file** — specific sections, conventions, formats consumed.
2. **Coverage assessment** — does the existing content fully cover the Arranger's needs?
3. **Language audit** — identify Repetiteur-specific wording that should be broadened to skill-agnostic language (e.g., "the Repetiteur will..." → "the producing skill will...", Repetiteur-specific examples → generalized or dual examples).
4. **Proposed expansions** — new sections, additional examples, extended specifications. Written as standalone additions the user can merge.
5. **Proposed modifications** — existing content that needs rewording, restructuring, or correction. Written as before/after pairs.

No recommendation is too small. A single word change to broaden language is worth proposing.

### New Shared File Discovery

The builder actively searches for sharing opportunities throughout construction. For every piece of Arranger-specific content, the builder asks: "Could this be shared? Is this concept already partially expressed in a repertoire file?"

Discovery outputs:
- **New file proposals** — with proposed filename, consumers list, purpose, and draft content.
- **Existing file expansion proposals** — content that belongs in an existing repertoire file but doesn't exist yet.

### Audit Artifacts

```
arranger/docs/working/repertoire-audit/
  output-format-audit.md
  journal-conventions-audit.md
  verification-rules-audit.md
  priority-chain-audit.md
  new-shared-proposals.md
```

Working documents for the user to review and merge into repertoire. Not committed to the Arranger repo permanently — they live in `docs/working/` until processed.

### Files Under Audit

| Repertoire File | Arranger Consumption | Audit Focus |
|----------------|---------------------|-------------|
| `output-format.md` | Heavy — Phases 4-6 | Original plan format vs remaining plan format distinction, Arranger-specific YAML frontmatter fields, phase-level content (no task headers) |
| `journal-conventions.md` | Moderate — all phases | Arranger checkpoint triggers, Strength field usage, user override entry pattern |
| `verification-rules.md` | Heavy — Phases 2-3 | Language broadening, Arranger's finalization checklist items |
| `priority-chain.md` | Light — Phase 3 | Consumer metadata completeness, language broadening |
| `example-remaining-plan.md` | Reference only | Whether companion examples should live in repertoire |
| `example-consultation-journal.md` | Reference only | Whether companion examples should live in repertoire |

### Sequencing

Repertoire audit artifacts are produced **during** Arranger construction, not after. As the builder writes each phase reference and encounters a shared file dependency, it adds to the relevant audit document. The user reviews and merges repertoire changes **before** the Arranger skill is published.

---

## 8. Build Process

### Build Phases

**Phase A: Repertoire Audit** — Read all repertoire files, produce audit artifacts in `arranger/docs/working/repertoire-audit/`. This happens first because shared file changes may affect reference file content.

**Phase B: Scaffold** — Create directory structure, README, `.gitignore`, `.claude/settings.local.json`, `docs/README.md`.

**Phase C: Config Scripts** — Write `arranger-config.py` and `arranger-config.sh`. Self-contained and testable independently.

**Phase D: SKILL.md** — Write the hub/router. All phase sections, mandatory rules, preamble, reference map, config loading, context management, example pointers.

**Phase E: Phase References (sequential)** — Write in phase order: ingestion → feasibility-audit → implementation-discussion → phase-structuring → section-writing → finalization. Sequential because each phase's return dispatch connects to the next phase's contextual preamble. Builder updates repertoire audit entries alongside each reference.

**Phase F: Examples** — Write the two example files last — they must accurately reflect all conventions established in the references.

### Builder Inputs

1. **This design document** — the full specification.
2. **The original Arranger design document** — vision, identity, workflow details.
3. **All repertoire files** — read access with comprehensive audit mandate.
4. **The Repetiteur skill** (SKILL.md + all references + examples) — reference implementation for Tier 3 structure, hub-and-spoke pattern, lazy-student-prevention language, `<reference load="required">` dispatch patterns.
5. **The hybrid document structure spec** — for Tier 3 conventions.

### Builder Instructions

- All reference files are Tier 3 — all text inside authority tags, no naked markdown.
- SKILL.md is Tier 3 — `<skill>` wrapper, `<metadata>`, `<sections>`, all text tagged.
- `<mandatory>` for context watching in every reference file.
- `<mandatory>` tagging used wherever appropriate throughout all files.
- Happy paths with pointers to error/deviation sub-sections within the same file.
- Loop-backs always described as "return to SKILL.md" — SKILL.md re-contextualizes before dispatching.
- Shared file audit is a first-class deliverable, not an afterthought.
- Actively mine for new shared content — default to repertoire over local.
- Language broadening: flag every instance of skill-specific wording in shared files.
- No recommendation too small.
- No GitHub work — local files only until repertoire proposals are merged.
- Study the Repetiteur's SKILL.md sections as exemplars for the lazy-student-prevention information asymmetry pattern.

### Deliverables Checklist

| Deliverable | Location |
|------------|----------|
| Directory scaffold | `orchestra/arranger/` |
| README | `README.md` |
| docs README | `docs/README.md` |
| SKILL.md | `skill/SKILL.md` |
| Phase 1 reference | `skill/references/ingestion.md` |
| Phase 2 reference | `skill/references/feasibility-audit.md` |
| Phase 3 reference | `skill/references/implementation-discussion.md` |
| Phase 4 reference | `skill/references/phase-structuring.md` |
| Phase 5 reference | `skill/references/section-writing.md` |
| Phase 6 reference | `skill/references/finalization.md` |
| Config script (Python) | `skill/scripts/arranger-config.py` |
| Config script (shell) | `skill/scripts/arranger-config.sh` |
| Example plan | `skill/examples/example-implementation-plan.md` |
| Example journal | `skill/examples/example-arranger-journal.md` |
| Repertoire audit: output-format | `docs/working/repertoire-audit/output-format-audit.md` |
| Repertoire audit: journal-conventions | `docs/working/repertoire-audit/journal-conventions-audit.md` |
| Repertoire audit: verification-rules | `docs/working/repertoire-audit/verification-rules-audit.md` |
| Repertoire audit: priority-chain | `docs/working/repertoire-audit/priority-chain-audit.md` |
| Repertoire audit: new proposals | `docs/working/repertoire-audit/new-shared-proposals.md` |

### No GitHub Work

The Arranger directory is created locally in `orchestra/arranger/`. No git repository initialization, no GitHub repo creation, no pushes until:
1. Repertoire audit artifacts are reviewed by the user
2. Repertoire changes are merged/discarded
3. The user explicitly triggers GitHub setup
