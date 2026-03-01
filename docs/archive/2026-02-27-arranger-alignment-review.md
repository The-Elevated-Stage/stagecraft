# Arranger Design Alignment Review

Date: 2026-02-27
Scope: Compare `stagecraft/docs/designs/2026-02-17-arranger-skill-design.md` against live Dramaturg, Conductor, and Copyist skill implementations, plus active shared repertoire contracts they rely on.

## Snapshot

- Critical mismatches: 8
- Significant mismatches: 6
- Minor/reference mismatches: 5
- Total items: 19

This review distinguishes three resolution types:
- Arranger design update: fix the Arranger design doc/spec.
- Downstream skill update: fix Dramaturg/Conductor/Copyist behavior/docs.
- Discussion required: choose direction first, then update one side or both.

## Critical Mismatches

| ID | Mismatch | Evidence | Resolution Type | Recommended Resolution |
|---|---|---|---|---|
| C1 | Dramaturg journal location does not match Arranger ingestion path. | Arranger expects `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md` (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:58`). Dramaturg writes `docs/plans/designs/YYYY-MM-DD-<topic>-dramaturg-journal.md` (`dramaturg/skill/SKILL.md:32`, `:63`, `:339`). | Discussion required | Standardize on one path contract. Strong candidate: decisions directory contract already used by Conductor/Repetiteur docs and repertoire. |
| C2 | Dramaturg archives journal, but Arranger and repertoire require persistent decisions journals. | Dramaturg archives on finalize (`dramaturg/skill/references/support-phases.md:255`). Arranger says journal remains in decisions dir (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:239`, `:396`). Repertoire lifecycle says not archived (`repertoire/journal-conventions.md:182`). | Downstream skill update | Remove Dramaturg archival behavior and keep active journal in decisions directory until Conductor cleanup. |
| C3 | Journal format contract conflict: Arranger specifies Tier 3 XML journal; live ecosystem uses append-only markdown journal entries. | Arranger XML `<journal>` format (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:322-375`). Repertoire: append-only markdown (`repertoire/journal-conventions.md:57`). Dramaturg journal templates are markdown field blocks (`dramaturg/skill/SKILL.md:356-367`). | Discussion required | Pick one canonical format. If XML is desired, migrate repertoire + Dramaturg + Repetiteur docs; otherwise simplify Arranger journal spec to markdown. |
| C4 | Plan format contract conflict: Arranger design mandates Tier 2 hybrid with YAML + `<sections>/<section>` tags; repertoire/output consumers are sentinel+markdown contract. | Arranger hybrid spec + YAML checks (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:405`, `:421`, `:228-233`). Repertoire output format sample is sentinel + markdown headers (no YAML frontmatter, no `<section>` wrappers) (`repertoire/output-format.md:36-77`). | Discussion required | Decide whether to migrate the whole pipeline to hybrid tags, or align Arranger design back to sentinel+markdown contract. |
| C5 | Arranger claims Conductor will not read phase sections; Conductor currently reads phase sections every phase and uses them for decomposition. | Arranger says Conductor reads only overview/summary/checkpoints (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:413`, `:719`). Conductor reads `phase:N` and `conductor-review-N` (`conductor/skill/references/phase-execution.md:36-43`) and decomposes from phase section (`:163-168`). | Discussion required | Either update Arranger design to match live Conductor behavior, or redesign Conductor decomposition flow to avoid phase-section reads. |
| C6 | Plan-index schema is insufficient for Conductor bootstrap as documented. Conductor expects overview/phase-summary via index ranges, but index schema only includes phase/review entries. | Conductor bootstrap expects Overview + Phase Summary via plan-index ranges (`conductor/skill/references/initialization.md:46-47`, `:131`). Arranger sample index has only `phase` and `conductor-review` entries (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:437-440`). Repertoire sample does the same (`repertoire/output-format.md:43-44`, `:90-93`). | Discussion required | Add `overview` and `phase-summary` line-range entries to plan-index schema, or update Conductor bootstrap docs to sentinel-based reading for those sections. |
| C7 | Arranger spec states Conductor/Copyist updates are pending, but those capabilities already exist in live implementations. | Arranger says downstream updates will be needed (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:417`). Conductor/Copyist already implement line-range selective reads and checkpoint workflow (`conductor/skill/references/initialization.md:44-47`, `conductor/skill/references/phase-execution.md:36-45`, `copyist/skill/SKILL.md:74`, `:90`). | Arranger design update | Rewrite as “already implemented,” then list only remaining deltas. |
| C8 | Arranger’s authority-tag contract for Copyist is under-specified in live Copyist behavior. | Arranger expects Copyist to preserve/adapt by `<mandatory>/<guidance>/<core>` (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:411`, `:595`, `:721`). Copyist expected phase-component parsing does not explicitly process plan-side authority tags (`copyist/skill/SKILL.md:92-99`, `:130-134`). | Downstream skill update | Add explicit Copyist rules for consuming phase-section authority tags into task instructions (mandatory preservation vs adaptable guidance). |

## Significant Mismatches

| ID | Mismatch | Evidence | Resolution Type | Recommended Resolution |
|---|---|---|---|---|
| S1 | Arranger says Copyist can freely split tasks; Copyist requires Conductor approval before split expansion. | Arranger “can freely split tasks” (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:721`). Copyist requires report + approval before splitting (`copyist/skill/references/launch-prompt-template.md:55`, `copyist/skill/SKILL.md:159-166`). | Arranger design update | Change wording to “Copyist may propose splits; Conductor approves split plan.” |
| S2 | Arranger ingestion does not account for `UNRESEARCHED` Dramaturg journal status. | Arranger ingestion list includes VERIFIED/PARTIAL only (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:59-62`). Dramaturg defines `UNRESEARCHED` requiring Arranger verification (`dramaturg/skill/SKILL.md:367`, `dramaturg/skill/references/research-strategy.md:181`). | Arranger design update | Add explicit `UNRESEARCHED` handling path (mandatory verification before planning decisions). |
| S3 | Arranger journal lifecycle/checkpoints differ from repertoire conventions. | Arranger starts journal at Phase 2 (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:394`) and checkpoint list omits section-complete/finalization triggers (`:386-390`). Repertoire says start at session start/Phase 1 and include section completion + finalization + context threshold checkpoints (`repertoire/journal-conventions.md:118-123`, `:147-149`, `:179`). | Discussion required | Align Arranger lifecycle and trigger definitions with shared repertoire conventions (or formally update repertoire). |
| S4 | `/arranger` auto-scan target is ambiguous because design docs and active plans share directory namespace. | Arranger auto-scan of `docs/plans/designs/` for design docs (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:54`). Repertoire says active plan also lives in `docs/plans/designs/` (`repertoire/output-format.md:258-263`). Dramaturg design docs also in same directory (`dramaturg/skill/SKILL.md:62`). | Arranger design update | Restrict scan to `*-design.md` (or frontmatter type filter) and exclude `*-plan*.md` + `superseded/`. |
| S5 | “No downstream research needed” aspiration does not match live Conductor fallback behavior. | Arranger says downstream should never need additional research (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:15`). Conductor allows teammate + Gemini fallback when decomposition is unclear (`conductor/skill/references/phase-execution.md:175`). | Arranger design update | Rephrase as objective, not hard guarantee (“minimize downstream research; fallback allowed when decomposition ambiguity remains”). |
| S6 | Arranger “Gemini required, no replacement” policy lacks parity with Dramaturg degraded-mode semantics. | Arranger non-launch without Gemini (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:45`). Dramaturg defines user-choice degraded path and explicit unresearched marking (`dramaturg/skill/SKILL.md:25`, `dramaturg/skill/references/research-strategy.md:178-183`). | Discussion required | Decide if Arranger should hard-stop always, or adopt same user-directed degraded policy with explicit risk labeling. |

## Minor / Reference Mismatches

| ID | Mismatch | Evidence | Resolution Type | Recommended Resolution |
|---|---|---|---|---|
| M1 | Arranger references missing `docs/hybrid-document-structure.md` in this repository. | Referenced repeatedly (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:43`, `:322`, `:405`); file not present in repo search. | Arranger design update | Replace with existing repertoire contract references, or add the missing file and link it from repertoire. |
| M2 | Arranger references `shared-rules.md`, but repertoire is split across dedicated files (`priority-chain`, `verification-rules`, `journal-conventions`, `output-format`). | Arranger path (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:694-700`). Repertoire split contracts are active. | Arranger design update | Update references to current repertoire files instead of `shared-rules.md`. |
| M3 | Repetiteur notes path in Arranger design is stale. | Arranger points to `docs/plans/designs/2026-02-17-repetiteur-skill-notes.md` (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:717`), file exists at `repetiteur/docs/archive/2026-02-17-repetiteur-skill-notes.md`. | Arranger design update | Correct link/path to actual location or move canonical notes to shared location. |
| M4 | Arranger embeds app-specific threat-model rationale in a cross-project orchestration contract. | Arranger rationale: “personal task management app with a narrow threat model” (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:39`). Shared priority-chain rationale is generic and reusable (`repertoire/priority-chain.md:21-33`). | Arranger design update | Remove product-specific rationale from Arranger core contract; keep generic priority-chain rationale from repertoire. |
| M5 | Shared verification rules reference a stale/nonexistent canonical format path. | `repertoire/verification-rules.md` points to `score-preparation/output-format.md` (`repertoire/verification-rules.md:161`), while active shared format is `repertoire/output-format.md`. | Downstream/shared contract update | Update verification-rules reference to the active output-format contract path. |

## Items That Already Align

- Plan-index lock semantics are aligned between Arranger design and Conductor bootstrap (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:574`; `conductor/skill/references/initialization.md:103-106`).
- Conductor checkpoint sections as required review checklists are aligned (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:413`; `conductor/skill/references/phase-execution.md:703`).
- Copyist line-range-only phase reading is aligned (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:721`; `copyist/skill/SKILL.md:74`).
- Priority ordering string is aligned (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:37`; `repertoire/priority-chain.md:24`; `dramaturg/skill/SKILL.md:27`).
- Decisions-directory cleanup ownership is aligned at high level (`stagecraft/docs/designs/2026-02-17-arranger-skill-design.md:239`; `conductor/skill/references/completion.md:139-157`).

## Recommended Resolution Order

1. Resolve contract-level decisions first: C1-C6 (journal path/format, plan format, Conductor read model, index schema).
2. Update Arranger design text after those decisions: C7, S1-S5, M1-M3.
3. Apply downstream skill changes that depend on chosen direction: C2, C8, and any resulting Dramaturg/Conductor/Copyist edits.
4. Run cross-skill contract validation pass after edits (journal path, index schema, phase-consumption assumptions, archival policy).
