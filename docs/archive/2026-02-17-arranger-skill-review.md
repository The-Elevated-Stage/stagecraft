# Arranger Skill Design — Dual Review Document

**Date:** 2026-02-17
**Reviewed by:** Opus subagent (pipeline cross-reference), Gemini (independent analysis)
**Target:** `docs/plans/designs/2026-02-17-arranger-skill-design.md`
**Purpose:** Working document for review session. Delete after approved changes are applied.

---

## User Decisions (Updated During Session)

- **Grep contract is a specification** — conductor/copyist updates will come later. Downstream skill updates are out of scope for this review session.
- **Hybrid sentinel markers (Option C)** — HTML comment anchors alongside markdown headers. Machine greps for comments; humans read headers naturally.
- **Document audience: 75% Claude / 25% human** — Dramaturg produces human-focused design docs. Arranger and downstream focus on implementation (Claude Code) consumption. Implementation plan is primarily machine-consumed.
- **Verification index at top of file** — Machine-readable index mapping phase numbers to line ranges. Placed at top for immediate conductor access.
- **Index as lock indicator** — The index's presence means the verification checklist passed and the plan is ready for conductor consumption. Conductor's first step should verify the index exists — if absent, the plan hasn't been verified/locked.
- **Explicit user gate for finalization** — Verification is NOT automatic. User may run arranger multiple times, brainstorm refinements, then deliberately trigger verification before locking. Finalization = user confirms → checklist runs → index generated → commit. This is the lock point.
- **Plan is locked after commit** — Once committed, the implementation plan is treated as locked by the conductor (step one). The index serves as the conductor's verification that this lock has occurred.
- **Verification checklist (growing)** — The finalization checklist may include additional verification steps beyond marker validation (beneficial for other concerns identified in this review).
- **Task decomposition is Conductor's concern** — Arranger provides boundary/goal guidance in checkpoints; Conductor (1M context) determines task granularity. Copyist estimates context and can freely split. Arranger shares all knowledge but doesn't prescribe copy/paste formulas.
- **Conductor checkpoints are review checklists** — items the Conductor must acknowledge/verify before proceeding, not just context. Conductor is overseer/parent/teacher.
- **Conductor V2 is less interactive** — pipeline after conductor launch must be self-sufficient. Raises stakes on Arranger output quality.
- **Consultation mode** — separate Arranger invocation for mid-implementation blockers. Conductor triggers first, falls back to user. Produces "remaining" implementation plan (not stacking revision files). Details: see Consultation Flow section below.
- **Decision journal persistence** — journals stored in `docs/plans/designs/decisions/{feature-name}/` (subdirectory per feature). Not archived prematurely. Conductor cleans up after implementation completes. README for this directory is handled in a separate docs restructuring session.
- **Journal naming:** `dramaturg-journal.md`, `arranger-journal.md`, `consultation-1-journal.md`, etc. within the feature subdirectory.

---

## Critical Findings

### C1. Section Marker Contract Has No Current Consumer [RESOLVED — by design]

**Source:** Opus
**Sections affected:** 6, 9

The conductor and copyist don't currently implement grep-based line ranges or selective reading. The design presents this as existing infrastructure.

**Resolution:** User confirmed the design is a specification. Conductor/copyist updates are a separate session. The design doc should be updated to explicitly note these are new behaviors requiring downstream updates — not existing infrastructure.

**Action:** Add a "Pipeline Prerequisites" note in Section 9 acknowledging conductor/copyist updates needed.

---

### C2. Conductor Reads Full Plan — "Context-Efficient Consumption" Is Aspirational [RESOLVED — by design]

**Source:** Opus
**Sections affected:** 6, 9

Same root cause as C1. The dual-audience concept is sound but described as if it already exists.

**Resolution:** Same as C1 — reframe as specification, not current state.

**Action:** Adjust language in Sections 6 and 9 from "the conductor reads only..." to "the conductor will read only..." or similar future-specification framing.

---

### C3. Task Decomposition Responsibility Is Undefined [RESOLVED]

**Source:** Opus
**Section affected:** 6

The Arranger "stops short of task-level decomposition" (Section 1). The conductor's launch prompt includes `{TASK_LIST}` implying it decides which tasks exist per phase. But the Arranger's plan doesn't include task IDs or boundaries.

**Resolution:** Task decomposition is a **Conductor concern** with guidance from the Arranger. The conductor (running on 1M context) has capacity for this planning work and currently does very little on the happy path. The Arranger provides boundary/goal focused guidance in conductor checkpoint sections — goals to accomplish, integration surfaces, dependency ordering, suggested parallelism — but does NOT prescribe exact task boundaries.

**Rationale:**
- Conductor has more freedom to adapt when "boots hit the ground"
- Partially addresses C4 (mid-flight re-plan) — conductor can adjust task structure based on actual conditions
- Copyist estimates context requirements per task — loose structure lets it freely split tasks
- Arranger should not hide knowledge from the conductor, but should not dig to provide a copy/paste formula

**Checkpoint language shift:** From prescriptive task lists to boundary/goal language. Example: "Phase 2 needs to accomplish: backend service setup, database migration, API endpoints. Service depends on config loader initializing first. Suggested parallelism: migration can run alongside service scaffolding, endpoints depend on both."

**Action:** Update Sections 1, 6, and 9 to clarify this division. Reframe "stops short of task-level decomposition" to explain the Arranger provides boundary/goal guidance while the conductor determines task granularity.

---

### C4. Mid-Flight Re-Plan Gap [RESOLVED — Repetiteur skill]

**Source:** Gemini

**Resolution:** Consultation flow is a separate skill: **Repetiteur** (from opera/ballet — the person who fixes problems during rehearsal). Full design notes captured at `docs/plans/designs/2026-02-17-repetiteur-skill-notes.md` for a future design session.

**Key decisions:**
- Separate skill (not a mode within Arranger) — interaction models diverge too fundamentally
- Two skills share `references/shared-rules.md` for verification rules, output format, journals
- Repetiteur produces a "remaining plan" — full standalone document for remaining work only, completed tasks excluded
- Mostly autonomous (user interaction = emergency fallback only)
- Decision journals persist in `docs/plans/designs/decisions/{feature-name}/` — used by both Arranger and Repetiteur
- Git reversions are a tool in the consultation flow
- Trigger mechanism depends on Conductor V2 architecture (tmux external sessions)

**Impact on Arranger design doc:** Remove "last point of user contact" framing — the Repetiteur can be invoked after. Add reference to Repetiteur as the consultation path. Note shared-rules extraction in Plugin Structure section.

---

## Major Findings

### M1. Decision Journal Archive Location Contradicts Project Rules [RESOLVED]

**Source:** Opus
**Section affected:** 5

**Resolution:** Journals are NOT archived prematurely. They persist in `docs/plans/designs/decisions/{feature-name}/` throughout the pipeline lifecycle. Conductor cleans up after implementation completes. This also supports the consultation flow — consultation sessions reference these journals for context.

---

### M2. `<planning_dir>` Is Never Defined [RESOLVED]

**Source:** Opus
**Section affected:** 5

**Resolution:** `<planning_dir>` resolves to `docs/plans/designs/decisions/{feature-name}/`. Journals co-located per feature. Subdirectory approach keeps things clean for multiple features in flight. README for this directory handled in separate docs restructuring session.

---

### M3. Dramaturg Journal Not Consumed by Arranger [RESOLVED]

**Source:** Opus
**Sections affected:** 2, 3

**Resolution:** The Arranger consumes the dramaturg journal via the **subagent delegation principle** — it does NOT read the raw journal in the main session. A subagent reads the full journal and returns distilled findings:
- VERIFIED items the Arranger can skip (one-liner each)
- PARTIAL items needing follow-up with specifics
- Final decisions still in effect with rationale
- Abandoned/invalidated approaches as one-liners ("SSE abandoned for FCM due to reliability")
- Ignores: back-and-forth discussion, superseded entries, circular exploration

**Action:** Update Section 2 (Ingestion) to include dramaturg journal subagent step. Update Phase 2 to show VERIFIED/PARTIAL flags reducing audit scope. Journal located at `decisions/{feature-name}/dramaturg-journal.md`.

---

### M4. Phase 6 Commit Behavior Underspecified [RESOLVED]

**Source:** Opus
**Section affected:** 3 (Phase 6)

**Resolution:** Largely addressed by the new finalization phase design (explicit user gate, verification checklist, user confirms commit). Branch guidance: Arranger commits to whatever branch the user is on. Branch creation/management is the Conductor's first task — not the Arranger's concern. Commit includes only the implementation plan. Journal stays in decisions directory.

---

### M5. Conductor Checkpoint Sections Are New with No Consumer [RESOLVED — by design]

**Source:** Opus
**Section affected:** 6

Same category as C1/C2 — the conductor would need updates to consume checkpoint sections. This is part of the specification.

**Action:** Include in the "Pipeline Prerequisites" note from C1.

---

### M6. Frontend Reference Markers Clash with Self-Containment [RESOLVED]

**Source:** Opus
**Section affected:** 6

**Resolution:** Always inline/duplicate frontend guidance into each phase section that needs it. Self-containment is non-negotiable — copyist never reads two sections. Document size increase is acceptable GIVEN the sentinel marker system is reliable. **Critical dependency:** if section finding fails on a larger document, the result is immediate context exhaustion. This reinforces the importance of the finalization checklist's marker validation.

**Action:** Add to finalization checklist: "Large document stress test — verify sentinel markers are findable in documents exceeding [threshold] lines." Remove the standalone Frontend Reference block from the output format. Frontend guidance is always inlined.

---

### M7. No Context Budget Estimation [RESOLVED]

**Source:** Opus
**Section affected:** 3

**Resolution:** Natural session split point at the Phase 4/5 boundary (before section writing & review), same pattern as the Dramaturg. Phases 1-4 (ingestion, feasibility, discussion, structuring) are the research-heavy phases; Phase 5 (section writing) is the output phase. The subagent delegation principle significantly reduces main session context usage during Phases 2-3. The decision journal provides continuity across session splits.

**Action:** Add session split guidance to Section 3 workflow. Define the Phase 4/5 boundary as the recommended split point. Journal preserves state across the split.

---

### M8. Research Conflict Resolution — No Truth Hierarchy [RESOLVED]

**Source:** Gemini
**Section affected:** 4

**Resolution:** Truth hierarchy: **Official docs (web) > Gemini > Training data.** For extreme conflict cases, the Arranger conducts additional research focused on **user-provided discussions online** — forums, issue trackers, Stack Overflow threads where developers discuss problems with the approach. These surface real-world implementation failures (people giving up on an approach because of issues) that official docs and Gemini may not capture.

**Action:** Add truth hierarchy to Section 4. Include the "user discussions" research escalation for conflict cases.

---

### M9. User Override Protocol Missing [RESOLVED]

**Source:** Gemini

**Resolution:** Add a "User Override" journal entry type. User can force a decision against research findings, logged with rationale and flagged as a documented risk. **Critical:** the override flag must carry through to the implementation plan itself — the conductor reads the plan, not the journals. User override risks must surface in the relevant conductor checkpoint section so the conductor is aware during execution.

**Action:** Add User Override protocol to Section 4 or 8. Define journal entry format. Define how overrides propagate to implementation plan (e.g., in conductor checkpoint: "USER OVERRIDE: [setting] set to X despite research indicating Y — user has workaround, see journal entry [ref]").

---

## Minor Findings

### m1. Redundant Content Across Sections [ACKNOWLEDGED — by design]

**Source:** Opus

Reinforcement principle is deliberate. But in a design doc (vs. skill file), consider marking reinforcement instances: "Reminder from Section 4:" so readers know which is authoritative.

**Action:** Minor wording pass when applying other changes.

---

### m2. Priority Chain Placement [RESOLVED]

**Source:** Opus

**Resolution:** Move to a named "Operating Principles" subsection in Section 1. Add rationale for security ranking below efficiency — this is a personal task management app, not a multi-tenant service. Compatibility and reliability directly affect user experience; efficiency affects battery/resources; security is important but the threat model is narrow; performance is the least constrained.

---

### m3. Gemini Hard Requirement Not Stated [RESOLVED]

**Source:** Opus

**Resolution:** Gemini is a hard requirement, same as Dramaturg. The Arranger does not launch without Gemini MCP available. Brave-search is a fallback for web queries but cannot replace Gemini's analysis and verification capabilities. Add to Section 4 and shared-rules.

---

### m4. Inconsistent Enforcement Language [RESOLVED]

**Source:** Opus

**Resolution:** Apply "no exceptions, regardless of Claude's confidence level" consistently across all four mandatory verification categories: Android/cross-device, unimplemented protocols, Flutter frontend guidelines, and specific configuration values. All are mandatory — enforcement language should match.

---

### m5. Subagent Permission Model Vague [RESOLVED]

**Source:** Opus

**Resolution:** Specify the mechanism: prompt-based enforcement. Subagent launch prompts must include explicit instruction: "Do not use Write, Edit, or Bash commands that modify files. This is a read-only planning session." This goes in shared-rules reference (used by both Arranger and Repetiteur).

---

### m6. Incomplete/Malformed Design Doc Handling [RESOLVED]

**Source:** Opus

**Resolution:** Threshold-based approach during Phase 1 (Ingestion & Overview):
- **3 or fewer simple questions** would resolve all ambiguity → Arranger pauses to discuss with user during Phase 3 (Implementation Discussion). Small gaps handled inline.
- **4+ questions needed** → Arranger recommends re-engaging the Dramaturg to flesh out the design. Too much ambiguity for the Arranger to resolve — the fix is upstream, not mid-stream.

The Arranger does NOT fill design gaps itself — that's re-litigating design, explicitly not its job.

---

### m7. Phase Summary Authoring Timing [RESOLVED]

**Source:** Opus

**Resolution:** Phase Summary written during Phase 4 (Phase Structuring) — it's the conductor's map of the work, natural output of structuring decisions. Overview written at the start of Phase 5 before section writing begins. Both reviewed by user as part of the Phase 5 section review process.

---

### m8. Overview Self-Containment for Conductor [RESOLVED]

**Source:** Opus

**Resolution:** Explicitly state: the Overview must be comprehensive enough to replace the design doc for conductor purposes. The conductor reads the implementation plan — not the design doc. The Overview carries forward all relevant design context, constraints, and goals. This is a self-containment requirement parallel to the phase section self-containment for the copyist.

---

### m9. Front-Loaded Feasibility Might Waste Work [RESOLVED — keep as-is]

**Source:** Gemini

**Resolution:** Keep Phase 2 front-loaded. The subagent delegation principle means feasibility work happens in subagents — the main session only receives distilled findings. The "waste" from a killed feature is subagent context and API calls, not main session context. The front-loading benefit (Phase 3 discussions informed by reality from the start) outweighs the risk of some subagent work being discarded.

---

### m10. Context Exhaustion Strategy [RESOLVED — merged with M7]

**Source:** Gemini

**Resolution:** Addressed by M7 (session split at Phase 4/5 boundary) and the subagent delegation principle (research-heavy work in subagents, not main session).

---

### m11. Mental Implementation Prompting Strategy [RESOLVED]

**Source:** Gemini

**Resolution:** Define specific subagent prompt patterns in shared-rules reference. Part of the subagent delegation principle. Example patterns: "Trace the initialization sequence for [service]. List every file touched in order. Report potential conflicts with [new component]." and "Simulate adding [feature] to the startup sequence. What initializes before it? What depends on it? What breaks?" These are templates, not rigid scripts — adapted per context.

---

## Nitpicks

### N1. "Phase Sections" vs "Plan Sections" vs "Sections" — inconsistent terminology [RESOLVED]

**Action:** Standardize during editing pass. "Phase section" = copyist content. "Checkpoint section" = conductor content. "Section" = either (context-dependent). Avoid "plan section."

### N2. Pipeline diagram appears 3 times (Sections 1, 6, 9) — 2 is sufficient [RESOLVED]

**Action:** Keep in Sections 1 and 9. Remove implicit third from Section 6 prose.

### N3. `<planning_dir>` might resolve to temp/ given it's undefined [RESOLVED — linked to M2]

**Resolution:** M2 resolved this. `docs/plans/designs/decisions/{feature-name}/`.

---

## New Design Elements from Discussion

### General Principle: Intelligent Context Usage via Subagent Delegation

**Prominence:** This is a general operating principle — should be stated prominently in the design doc and referenced from all relevant sections (Research Strategy, Workflow Phases, Ingestion).

**The principle:** The process of acquiring and validating knowledge is what wastes context, not the knowledge itself. Subagents absorb the acquisition cost; the main session gets the findings at appropriate detail.

**What this means in practice:**

- **Subagents handle knowledge acquisition and validation:** Journal parsing, code path tracing, research that might hit dead ends, feasibility checks. The back-and-forth, dead ends, and circular exploration happen in subagent context — not the main session.
- **Subagent returns distilled findings, not process:** "SSE was abandoned for FCM on Android due to reliability concerns (user decision)" — not the 200-line discussion that led there. Abandoned/invalidated approaches included as one-liners to inform the Arranger of bad paths, not as full narratives.
- **Not everything is compressed:** This is intelligent context usage, not aggressive summarization. Subagent outputs use bulletpoints for knowledge acquisition results (journal distillation, research findings, feasibility verdicts). But discussion topics, large-scope decisions, and broad architectural ideas retain full verbosity in the main session.
- **The Arranger is not knowledge-starved:** It has access to ALL available knowledge. The principle is about acquiring that knowledge efficiently, not about limiting what the Arranger knows. Tool use (subagents) is what separates intelligent context management from simple capability.

**Applies to:**
- Journal ingestion (dramaturg journal, prior consultation journals, arranger's own journal on re-read)
- Code path tracing and architectural analysis
- Feasibility audit results
- Any historical context that might contain dead ends, abandoned approaches, or repetitive entries

**Does NOT apply to:**
- User discussion (always full verbosity in main session)
- Phase structuring and design decisions (broad, nuanced — main session work)
- Section writing and review (interactive, detail matters)

---

### Hybrid Sentinel Markers (Option C)

```markdown
<!-- phase:1 -->
## Phase 1: Backend Services
[phase content]
<!-- /phase:1 -->

<!-- conductor-review:1 -->
## Conductor Review: Post-Phase 1
[checkpoint content]
<!-- /conductor-review:1 -->
```

- HTML comments are invisible in rendered markdown
- Machine greps for comment anchors (trivial, exact-match)
- Markdown headers preserved for human readability
- If header drifts, comment anchor still works

### Verification Index (Top of File, Lock Indicator)

```markdown
<!-- plan-index:start -->
<!-- verified:2026-02-17T14:30:00 -->
<!-- phase:1 lines:45-120 title:"Backend Services" -->
<!-- conductor-review:1 lines:121-145 -->
<!-- phase:2 lines:146-230 title:"Frontend Components" -->
<!-- conductor-review:2 lines:231-260 -->
<!-- frontend-reference lines:261-290 -->
<!-- plan-index:end -->
```

**Purpose:** Dual-function — (1) line range map for conductor/copyist, (2) lock indicator confirming verification passed.

**Conductor behavior:** First step: check for `<!-- plan-index:start -->`. If absent, plan is unverified — stop and report. If present, use index for all line range operations.

**When generated:** Only during the explicit finalization phase, after user confirms readiness.

### Finalization Phase (Explicit User Gate)

The finalization phase is deliberately separate from the arranger's main workflow. The user may:
1. Run the arranger skill, produce a draft plan
2. Review, brainstorm refinements, re-run sections
3. When fully satisfied, trigger finalization

**Finalization flow:**
1. Arranger prompts: "Ready to lock the implementation plan?"
2. User confirms
3. Verification subagent launches with checklist
4. Results presented to user
5. If all pass → index generated → user confirms commit → plan is locked
6. If failures → report to user, return to editing

**The plan is NOT committed until finalization passes.** This is the lock point — once committed, the conductor treats it as immutable.

### Finalization Checklist (Accumulating)

Items to include in the pre-commit verification:
- [ ] Sentinel marker validation (all pairs present and correctly matched)
- [ ] Header-sentinel consistency check (markdown headers match their comment anchors)
- [ ] Line range index generation (accurate to current file state)
- [ ] Self-containment spot-check (sample phase sections for completeness)
- [ ] Large document stress test — verify sentinel markers findable in documents exceeding expected size
- [ ] User override flags propagated from journal to relevant conductor checkpoint sections
- [ ] (Additional items may be added as we work through remaining findings)
