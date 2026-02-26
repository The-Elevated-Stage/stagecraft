# Arranger Skill — Changes Identified from Dramaturg Review

**Date:** 2026-02-23
**Primary source:** `temp/dramaturg-review/05-arranger-mapping-opus.md`
**Supporting sources:** Other review files where findings have Arranger implications

---

## How to Read This Document

All findings are extracted from the Dramaturg skill review and tagged by which skill needs the change:
- **[ARRANGER]** — Change needed in Arranger only
- **[BOTH]** — Change needed in both skills (Dramaturg side documented in DRAMATURG-CHANGES.md)
- **[GAP]** — Neither skill covers this; needs a design decision on ownership

---

## CRITICAL (1)

### A-C1. Journal location/lifecycle conflict — Dramaturg archives, Arranger expects persistence

[Tag: GAP → resolution requires changes to BOTH]

**The conflict:**
- Dramaturg places journal at `docs/plans/designs/dramaturg-journal.md` and archives to `docs/archive/` on completion
- Arranger expects journal at `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md` and expects it to persist (NOT be archived) for Repetiteur use

These are incompatible. The Dramaturg was designed first with a simpler lifecycle. The Arranger and Repetiteur were designed later with a richer lifecycle. The Dramaturg was never updated to match.

**Resolution (requires changes to both):**

**Dramaturg changes:**
1. Place journal at `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md`
2. Do NOT archive the journal — it must persist for Arranger and Repetiteur
3. Create the `decisions/{feature-name}/` directory as part of Phase 7

**Arranger changes:**
1. Define the feature-name derivation protocol (see A-I4 below)
2. Confirm the journal path matches the Dramaturg's updated output location

**Design doc location unchanged:** Both skills agree the design doc goes to `docs/plans/designs/YYYY-MM-DD-<topic>-design.md`

---

## IMPORTANT (7)

### A-I1. No shared standard for design doc completeness

[Tag: GAP]

The Dramaturg has a subjective "implementation readiness gate" at Phase 6→7. The Arranger has a question-count threshold at ingestion (3 or fewer = inline, 4+ = re-engage Dramaturg). Neither defines a shared checklist.

**Recommendation:** Define a shared "design readiness checklist" that both skills reference. The Dramaturg uses it as the Phase 6→7 gate. The Arranger uses it for ingestion validation. Candidate items: goals section present, data model specified, error handling addressed, integration points identified, all topics from topic map settled, Arranger Notes present. This could live in `score-preparation/`.

---

### A-I2. No guidance for user on transitioning from Dramaturg to Arranger session

[Tag: GAP]

Neither skill defines what happens between them. Does the user manually invoke `/arranger`? Is there a suggestion? How does the user know which design doc to point to?

**Recommendation:**
- **Dramaturg:** Add "Next Steps" note at end of Phase 7: "This design is ready for the Arranger: `/arranger docs/plans/designs/YYYY-MM-DD-<topic>-design.md`"
- **Arranger:** Ensure auto-scan works for the standard design doc path

---

### A-I3. Scope protection vs research depth tension at the boundary

[Tag: BOTH]

The Dramaturg's scope protection redirects implementation details. But technology feasibility research ("does FCM support this delivery pattern?") shapes the design and is within Dramaturg scope. The two rules can conflict in practice.

**Recommendation:**
- **Dramaturg:** Add clarification: "Scope protection applies to codebase-specific implementation details (file paths, function signatures). Technology research (does FCM support this delivery pattern? does SQLite WAL handle this concurrency?) is within scope — these are feasibility questions that shape design."
- **Arranger:** No change needed — the VERIFIED/PARTIAL mechanism correctly handles the handoff.

---

### A-I4. Feature-name derivation protocol missing for `decisions/{feature-name}/` directory

[Tag: GAP]

The Arranger expects journals at `decisions/{feature-name}/` but no protocol defines how `{feature-name}` is derived. Is it the date-topic slug? The topic only? An explicit field?

**Recommendation:** Either:
- Add a `feature` field to the Dramaturg's design doc metadata that the Arranger reads
- Or have the Dramaturg place the journal path in the Arranger Notes appendix
- Or derive from the design doc filename slug (strip date prefix)

---

### A-I5. Journal conventions in `score-preparation/` need shared vs skill-specific delineation

[Tag: BOTH]

The Dramaturg journal (Category tags: goal/use-case/decision, User verbatim field, Arranger note) is fundamentally different from the Arranger journal (authority-tag-differentiated entries, override tracking). The planned `score-preparation/journal-conventions.md` is supposed to cover both but the current designs don't distinguish shared from skill-specific.

**Recommendation:** Define clearly:
- **Shared:** Tier 3 wrapper format, append-only semantics, `<sections>` index protocol, lifecycle management
- **Skill-specific:** Entry types, entry fields, category tags

---

### A-I6. No structured protocol for Arranger to formally recommend re-engaging Dramaturg

[Tag: ARRANGER]

The Arranger can surface individual infeasible assumptions to the user. But when the design has a pervasive, systemic problem requiring Dramaturg-level revision, there's no structured "upstream referral."

**Recommendation:** Add a brief "upstream referral" protocol to the Arranger's deviation detection. When the Arranger determines the design needs revision (not just clarification): present a structured referral listing what gaps exist, what topics need exploration, and a recommendation to re-engage the Dramaturg with specific focus areas.

---

### A-I7. Dramaturg category tagging is critical for Repetiteur — Arranger should respect distinction

[Tag: ARRANGER]

The Dramaturg tags journal entries as `goal`, `use-case`, or `decision`. The Repetiteur treats `goal` and `use-case` as "inviolable constraints." The Arranger should also respect this distinction when reading the Dramaturg's journal, even though the Arranger's own journal uses different entry types.

**Recommendation:** Add a note to the Arranger's journal ingestion phase: "Respect the goal/use-case/decision distinction in the Dramaturg journal. Goals and use-cases are user-confirmed constraints that should not be revisited without user approval."

---

## MINOR (5)

### A-M1. Research conflict resolution hierarchies use different vocabulary

[Tag: BOTH]

Dramaturg: "web wins over Gemini synthesis." Arranger: "official documentation > Gemini analysis > training data." Same principle, different words. When `score-preparation/` is built, reconcile into one canonical expression.

---

### A-M2. Reinforcement principle independently established in both skills

[Tag: BOTH]

Both designs define it as foundational without acknowledging the other. Add `score-preparation/reinforcement-principle.md` as a shared reference.

---

### A-M3. Session split pattern is consistently aligned

[Tag: BOTH — no action needed]

Both skills split at the same structural boundary (research-heavy → output phases). Consistent.

---

### A-M4. Phase numbering differs (7 vs 6)

[Tag: BOTH — no action needed]

Different phase counts are appropriate. Journal `**Phase:**` field distinguishes them.

---

### A-M5. Pipeline diagram — Repetiteur appropriately absent from Dramaturg SKILL.md

[Tag: BOTH — no action needed]

Dramaturg doesn't interact with Repetiteur. Correct omission.

---

## SUGGESTIONS (3)

### A-S1. Add `score-preparation/reinforcement-principle.md` as shared reference

Both SKILL.md files reference it rather than independently defining it.

---

### A-S2. Clarify scope protection at Dramaturg/Arranger boundary

Technology feasibility research is in-scope for Dramaturg; codebase-specific implementation details are out-of-scope. Add to both skills' scope protection sections.

---

### A-S3. Add structured "upstream referral" protocol for Dramaturg re-engagement

When Arranger determines design needs fundamental revision, present structured referral with gaps, topics, and recommendation for focused Dramaturg session.

---

## Statistics

| Severity | Count |
|----------|-------|
| CRITICAL | 1 |
| IMPORTANT | 7 |
| MINOR | 5 (2 need action, 3 confirmed aligned) |
| SUGGESTION | 3 |
| **Total** | **16** |

---

## Summary

The Dramaturg and Arranger are well-designed with clear separation of concerns. The handoff mechanism (three artifacts + VERIFIED/PARTIAL flags) is sound. The **one critical issue** — journal location/lifecycle conflict — must be resolved before the Arranger is built. The important gaps (shared completeness standard, transition guidance, feature-name derivation, upstream referral protocol) are all addressable additions, not architectural changes.
