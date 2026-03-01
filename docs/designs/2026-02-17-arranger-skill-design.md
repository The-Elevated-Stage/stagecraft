# Arranger Skill — Design Document

**Date:** 2026-02-17
**Status:** Design Phase
**Type:** Standalone Claude Code Plugin Skill

---

## 1. Problem Statement & Skill Identity

The Dramaturg produces a design document — a vision of what *could* be built, validated by research but unconcerned with implementation specifics. The Conductor expects a detailed implementation plan — phases, tasks, dependencies, enough detail for autonomous copyist and musician sessions to build without further human input.

**The gap:** Nothing translates vision into executable plan. Without the Arranger, either the Dramaturg overreaches into implementation details (polluting design discussion) or the Conductor receives an underspecified design doc and makes implementation decisions it shouldn't — decisions about phase structure, parallelization strategy, protocol configurations, and integration patterns that haven't been validated.

The Arranger is a **fact-checker and setting-decider**, not a code-writer. It stress-tests the design's feasibility, makes implementation-level decisions through research-backed discussion, and structures the work into phases that maximize parallel execution. It is the **primary point of user contact** for implementation planning — every decision must be finalized here, every protocol validated, every cross-task integration surface identified. The Conductor and later stages should not need to research protocols, configurations, or integration patterns — though the Conductor may still research decomposition ambiguities when "boots hit the ground." (The Repetiteur skill handles mid-implementation consultation if blockers arise — see Section 9.)

### Why "Arranger"?

In music, the arranger takes a composition and creates the detailed arrangement for the orchestra: which instruments play when, what's simultaneous, what's sequential. The composer (dramaturg) writes the music; the arranger makes it performable; the conductor coordinates the performance; the musicians play their parts; the copyist creates the individual parts from the arranger's score.

### Pipeline Position

```
dramaturg → arranger → conductor → musician
(vision)    (plan)     (coordination)  (implementation)
                        copyist creates individual parts from arranger's score
```

### What Arranger Is NOT

- **Not a code-writer.** It validates facts and decides on specific settings and configurations. It "mentally implements" critical paths to catch issues, but never writes production code.
- **Not a task-writer.** It does not create task instruction files — the copyist does that. The Arranger produces phase sections with boundary/goal guidance for each phase. Task-level decomposition is the conductor's responsibility — the Arranger provides goals, integration surfaces, dependency ordering, and suggested parallelism, but does not prescribe exact task boundaries. This gives the conductor freedom to adjust task granularity based on actual conditions, and the copyist freedom to split tasks based on context budget estimates.
- **Not a re-litigation of design.** Design decisions were settled in the Dramaturg phase. If feasibility research reveals a design assumption is unworkable, the Arranger surfaces this as a conflict to the user — it doesn't silently change the design direction.

### Operating Principles

**Priority chain for technical decisions:** compatibility > reliability > efficiency > security > performance (see `repertoire/priority-chain.md` for full rationale)

Prefer modern but not bleeding-edge approaches. Future-favoring: prefer current code, APIs, and protocols that are well-established and well-documented.

**Intelligent context usage:** The process of acquiring and validating knowledge is what wastes context, not the knowledge itself. Subagents absorb acquisition costs (journal parsing, code path tracing, research dead ends); the main session receives distilled findings at appropriate detail. The Arranger is never knowledge-starved — it has access to all available knowledge. The principle is about acquiring that knowledge efficiently. See Section 4 (Research & Verification Strategy) for full details.

**Document audience:** The implementation plan is primarily machine-consumed (75% Claude Code / 25% human readability) and follows **Tier 2** of the project's hybrid document structure — hybrid markdown with XML authority and structural tags (see `docs/hybrid-document-structure.md`). The decision journal uses flat markdown with a `Strength` annotation field following the repertoire's journal conventions. The Dramaturg produces human-focused design documentation (Tier 1); the Arranger's outputs are Claude-consumed with human reviewability. This informs every output format decision.

**Context budget: 200k target.** The Arranger is interactive, not autonomous — the user is present to make judgment calls about context pressure. No mandatory compact or forced exit. At 75% usage, recommend `/lethe compact` or session split to the user. The Phase 4/5 boundary is always presented as the recommended split point regardless of context pressure — the decision journal preserves all state from Phases 1-4, allowing a fresh session to resume at Phase 5. If the user opts into a 1M extended context window, the Arranger should still target the 200k window to minimize cost; the extra headroom is a safety net, not a budget to fill.

**Gemini MCP is expected.** If Gemini MCP is unavailable at launch, the Arranger presents the user with a choice: (1) proceed in degraded mode with mandatory UNRESEARCHED marking on all items that would normally require external verification, or (2) abort. The Arranger does not silently proceed without verification capability, and does not hard-stop without user input. Brave-search remains available as a web query fallback but cannot replace Gemini's analysis capabilities.

---

## 2. Invocation & Ingestion

### Invocation Patterns

- **`/arranger @path/to/design.md`** — Direct file reference, begins processing immediately.
- **`/arranger` (no file)** — Scans `docs/plans/designs/` for `*-design.md` files, excluding any `superseded/` subdirectory and `*-plan.md` files. If exactly one design doc is found, auto-selects it. If multiple are found, prompts the user to choose. If none are found, reports and stops.

### Feature-Name Derivation

The Arranger derives `{feature-name}` from the design doc filename by stripping the date prefix and `-design` suffix:

- `2026-02-17-background-sync-design.md` → `background-sync`
- `2026-03-01-notification-system-design.md` → `notification-system`

This feature-name is used for:
- Decisions directory path: `docs/plans/designs/decisions/{feature-name}/`
- YAML frontmatter `feature` field in the implementation plan
- Journal filename: `arranger-journal.md` within the decisions directory
- Downstream branch naming by the Conductor

### Ingestion Behavior

The Arranger reads the design document fully, then **launches a subagent to distill the dramaturg's decision journal** (located at `docs/plans/designs/decisions/{feature-name}/dramaturg-journal.md`). The subagent returns:
- VERIFIED items the Arranger can skip in feasibility audit (one-liner each)
- PARTIAL items needing follow-up with specifics
- UNRESEARCHED items — decisions made without research backing, requiring mandatory independent verification before the Arranger builds on them
- Final decisions still in effect with rationale
- Abandoned/invalidated approaches as one-liners (e.g., "SSE abandoned for FCM due to reliability")
- Goal/use-case entries — inviolable user-confirmed constraints that must not be revisited without user approval
- Tension entries — acknowledged design tensions treated as constraints on phase structuring, not problems to solve
- Any stale in-progress entries flagged as potentially incomplete research

The main session never reads the raw journal — this is the intelligent context usage principle in action. The subagent absorbs the back-and-forth; the Arranger gets the distilled findings.

If the `decisions/{feature-name}/` directory does not yet exist, the Arranger creates it during ingestion. If the Dramaturg journal is not found at the expected path, the Arranger proceeds without journal distillation — the design document is the primary input.

After ingestion, the Arranger **pauses to present an overview** before beginning work. This overview includes:
- What the design covers (high-level summary)
- Key areas that will need feasibility verification (protocols, platform-specific features, integration points)
- Items the dramaturg flagged as PARTIAL (needing Arranger follow-up)
- Estimated complexity/scope of the planning work ahead
- Any immediate concerns or questions from the initial read

The Arranger does **not** begin autonomous work until the user acknowledges the overview. This serves two purposes: (1) confirms the right document was loaded, and (2) gives the user a chance to provide additional context or constraints before work begins.

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

### Upstream Referral Protocol

When the Arranger determines the design needs fundamental revision (4+ questions needed, or systemic feasibility failure), it presents:

- List of specific gaps or infeasible assumptions found
- Topics needing Dramaturg-level exploration
- Suggested focus areas for a follow-up session
- Exact invocation suggestion: `/dramaturg docs/plans/designs/{design-doc-name}.md`

The Arranger does not silently work around design gaps or proceed hoping for the best. The fix is upstream.

### Design Doc Consumption

The Arranger does not re-litigate design decisions — those were settled in the Dramaturg phase. If the Arranger's feasibility research reveals a design decision is unworkable, it surfaces this to the user as a conflict rather than silently changing the approach. The user decides whether to revise the design or find a workaround.

---

## 3. Workflow Phases

The Arranger follows 6 phases with loop-back points for user-driven deviations. Each phase writes to the decision journal progressively. Phases are conversational, not rigid gates — but transitions have clear triggers.

```
1. INGESTION & OVERVIEW
   Read design doc, present summary, get user go-ahead
          │
          ↓
2. FEASIBILITY AUDIT
   Subagents verify protocols, trace code paths, validate settings
   Read-only — no edits permitted, subagents launched with corresponding permissions
   All Android/cross-device services externally verified (Gemini, web search)
          │
          ↓
3. IMPLEMENTATION DISCUSSION
   Research-backed decisions on implementation specifics
   Decision→Research→Discussion→Decision loops (dramaturg-style)
   Frontend design guidelines researched externally when applicable
   Journal updated at each decision checkpoint
   ←── loops to 2 if structural deviation from user
          │
          ↓
4. PHASE STRUCTURING
   Arrange work into phases optimizing for parallelism
   Identify cross-task integration surfaces
   Define conductor checkpoint content between phases
   ←── loops to 3 if user proposes implementation changes
   ←── loops to 2 if user proposes structural changes
          │
          ↓
5. SECTION WRITING & REVIEW  ◄── recommended session split point (Phase 4/5 boundary)
   Write plan sections one at a time, user reviews each
   Interleaved: phase section → conductor checkpoint → phase section
   Sentinel markers follow strict naming convention for downstream consumption
   ←── loops to 3 for implementation deviations
   ←── loops to 2 for structural deviations
          │
          ↓
6. FINALIZATION & COMMIT  (explicit user gate)
   User confirms readiness to lock
   Verification subagent validates markers, generates line range index
   Assemble final plan, journal stays in decisions directory
   Commit implementation plan (locked for conductor consumption)
```

### Phase 1: Ingestion & Overview

Covered in Section 2 (Invocation & Ingestion). The Arranger reads, summarizes, and waits for go-ahead.

### Phase 2: Feasibility Audit

**Purpose:** Front-load all feasibility verification so the Implementation Discussion (Phase 3) is informed by current reality, not training-data assumptions.

**What gets audited:**

- **Protocols and patterns not already implemented in the project.** If the design proposes FCM and the project has never used FCM, the Arranger verifies FCM integration specifics via Gemini and web search. **No unimplemented protocol should enter the plan without external verification.** If the project already has working code for the pattern, no audit needed.
- **Android and cross-device services/settings.** All platform-specific service configurations, usage patterns, and limitations must be externally verified — **no exceptions, regardless of Claude's confidence level.** Training data is unreliable for platform specifics that change with OS versions. The WiFi scan interval limit (30 minutes on Android, not the 5 minutes the design assumed) is the canonical example.
- **Cross-component integration points.** Subagents trace existing code paths where new changes will interact with existing systems. "If we add this service, does the startup sequence initialize it in the right order?" This catches issues that individual musicians miss — they handle errors within their scope but are blind to problems between files owned by different tasks.

**How audits execute:**

- **Read-only subagents** for code path tracing and architectural analysis. **These are planning sessions — no edits of any kind permitted. Subagents must be launched with read-only permissions.** This constraint is non-negotiable.
- **Gemini queries** for protocol/platform verification. Quick, targeted questions — not open-ended exploration.
- **Web search** for current documentation, known issues, version-specific gotchas.

**What does NOT need auditing:** Simple implementations, wholly new files that don't touch existing components, and items the dramaturg journal flagged as VERIFIED (the subagent distillation from Phase 1 identifies these). PARTIAL items from the dramaturg journal get targeted follow-up — the dramaturg already narrowed the scope.

**Journal checkpoint:** All findings written to the decision journal before proceeding. If the audit reveals a design assumption is unworkable, surface it to the user as a conflict — don't silently work around it.

### Phase 3: Implementation Discussion

**Purpose:** The core of the Arranger — make implementation-level decisions through research-backed discussion. This is where "a new toggle on the New Task screen" becomes "a SwitchListTile in the Advanced section, below the priority dropdown, using the existing form field pattern."

**Execution:** Dramaturg-style decision loops:
1. **Identify a decision point** from the design doc or feasibility findings
2. **Research if needed** — Gemini for settings validation, subagents for code exploration. **All Android/cross-device settings externally verified, not assumed.** Frontend design guidelines fetched from external references when Flutter UI decisions are involved.
3. **Discuss with user** — present findings with explicit recommendations (not just options), get input
4. **Settle the decision** — explicit confirmation before moving on
5. **Journal the decision** — append to journal with rationale
6. Loop as needed

**"Mental implementation" for critical paths:** For features that touch components outside themselves, the Arranger "mentally implements" them via **read-only subagents** — tracing through the execution path to catch initialization order issues, missing dependencies, or conflicting state. This can also involve writing a pseudo-implementation and sending to Gemini for analysis ("Here's how I think the background service would initialize — what issues do you see?"). **Not writing code — walking through the logic to verify the approach works.** Simple or isolated implementations don't need this treatment; the trigger is external interaction.

**Testing recommendations:** When the Arranger discovers testing considerations naturally during discussion (e.g., "this service needs mocking for unit tests" or "integration tests should verify the startup sequence"), it notes them. These aren't exhaustive test plans — just observations that inform the conductor's verification expectations.

**Loop-back:** If the user proposes changes that deviate from the planned implementation approach, loop back to Phase 2 (structural changes requiring re-audit) or restart the relevant discussion topic (implementation changes). See Section 7 (Deviation Detection) for the full hierarchy.

### Phase 4: Phase Structuring

**Purpose:** This is where the Arranger earns its name. It takes all the settled implementation decisions and *arranges* them into phases that optimize for parallel execution while respecting dependencies.

**The strategic decomposition:**
- **Naive arrangement:** Phase 1 = full Feature A, Phase 2 = full Feature B (forces sequential execution)
- **Smart arrangement:** Phase 1 = preparation for A and B (parallelizable), Phase 2 = dependent work blocked by Phase 1 (parallelizable within phase), Phase 3 = integration tasks

The Arranger actively designs the phase structure to maximize what the copyist can parallelize. This means thinking about:
- Which tasks can run simultaneously without file conflicts
- Which tasks produce outputs that other tasks depend on
- Where the conductor needs to verify cross-task integration before proceeding

**Cross-task integration awareness:** The Arranger identifies integration surfaces — places where Task A's changes and Task B's changes must work together. These become explicit verification items in the conductor checkpoint sections. **Individual musicians can handle errors within their scope but are blind to cross-task contract violations.** The conductor checkpoint sections must make these visible.

**Phase Summary authored here.** The Phase Summary section (the conductor's map of the work) is a natural output of phase structuring decisions. It's drafted during this phase and reviewed by the user as part of Phase 5's section review process. The Overview section is written at the start of Phase 5 before individual section writing begins.

**Journal checkpoint:** Phase structure written to journal. This is a critical checkpoint — if the user disagrees with the phase arrangement, better to catch it now than during section writing.

### Phase 5: Section Writing & Review

**Purpose:** Write the implementation plan sections one at a time, with user review of each, following the interleaved format the conductor expects.

**Document structure:** See Section 6 (Output Format & Section Markers) for the full specification of the dual-audience format, section marker convention, and self-containment rules.

**Review process:** One section at a time (phase sections and checkpoint sections alike). The user reviews and approves each before the Arranger proceeds to the next. Loop-backs apply per Section 7 (Deviation Detection):
- Implementation deviation → Phase 3 (re-discuss)
- Structural deviation → Phase 2 (re-audit)

**Self-containment check:** Before presenting each phase section for review, the Arranger verifies: does this section contain everything the copyist needs? Could a copyist session reading *only* these lines produce complete, unambiguous task instructions without needing to reference the overview or other phase sections? If not, the section needs more context before review. **This is the Arranger's core deliverable quality — self-contained phase sections are the primary purpose of the entire skill.**

### Session Split Point: Phase 4/5 Boundary

Phases 1-4 (ingestion, feasibility, discussion, structuring) are research-heavy. Phase 5 (section writing) is the output phase. The Phase 4/5 boundary is the **recommended session split point** when context is running low — the decision journal preserves all state from Phases 1-4, allowing a fresh session to pick up at Phase 5 with full context for writing.

### Phase 6: Finalization & Commit

**Purpose:** Lock the implementation plan through verification, generate the line range index, and commit. This phase is gated behind **explicit user confirmation** — the user may run the Arranger multiple times, brainstorm refinements, and iterate before triggering finalization.

**Finalization flow:**
1. Arranger prompts: "Ready to lock the implementation plan?"
2. User confirms readiness
3. Verification subagent launches with the finalization checklist (see below)
4. Results presented to user
5. If all pass → line range index generated at top of file → user confirms commit
6. If failures → report to user, return to editing
7. Commit the implementation plan to the current branch (branch management is the conductor's concern, not the Arranger's)

**The plan is NOT committed until finalization passes.** This is the lock point — once committed, the conductor treats the plan as immutable. The presence of the line range index at the top of the file is the conductor's verification that finalization occurred.

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

**Decision journal:** The journal remains in `docs/plans/designs/decisions/{feature-name}/` — it is NOT archived or deleted. The journal persists for reference by the Repetiteur (consultation skill) and the conductor if needed. The conductor cleans up the decisions directory after implementation is complete.

---

## 4. Research & Verification Strategy

Research and verification are the Arranger's core competency. **The Arranger is the primary research checkpoint in the pipeline.** The objective is to minimize downstream research: the Conductor may still research decomposition ambiguities when conditions on the ground differ from expectations, but should never need to research protocols, configurations, or integration patterns. Every protocol validated, every platform limitation discovered, every configuration value checked — here, not downstream. (The Repetiteur handles research for mid-implementation blockers if they arise.)

### Mandatory External Verification

These categories **must always** be verified through Gemini or web search, never assumed from training data:

- **Android and cross-device services/settings/usage patterns.** Platform APIs change with OS versions. A setting that worked on Android 13 may be deprecated or restricted on Android 14. The WiFi scan interval limit is the canonical example — training data might say 5-minute intervals are possible, but Android actually limits scans to 30-minute intervals. **No exceptions, regardless of Claude's confidence level.**
- **Protocols and patterns not already implemented in the project.** If the project doesn't have working code for a pattern, Claude should not trust its training-data understanding. Version-specific gotchas, ecosystem compatibility issues, and platform constraints only surface through current research. The FCM configuration that would have caused consistent app crashes is the canonical example. **If it's not already in the codebase, verify it externally.**
- **Flutter frontend design guidelines.** Flutter's design ecosystem moves fast. Widget patterns, Material 3 guidelines, and recommended approaches evolve between Flutter releases. External references produce stronger frontend guidance than training data alone. **No exceptions, regardless of Claude's confidence level.**
- **Specific configuration values and settings.** When the Arranger decides on a specific timeout, interval, batch size, or protocol setting, that value's validity must be checked. "Use a 5-minute polling interval" needs verification that 5 minutes is actually achievable on the target platform. **No exceptions, regardless of Claude's confidence level.** Every concrete setting value gets verified before entering the plan.

### Gemini Degraded Mode

If Gemini MCP becomes unavailable mid-session (after initially being available), the Arranger:
1. Pauses and notifies the user
2. Offers the choice to continue in degraded mode or wait/abort
3. If continuing, marks all subsequent items that would require Gemini verification as UNRESEARCHED in the journal
4. Adds a plan-level note in the Overview section flagging that some verification was degraded

This complements the launch-time check in Section 1 — that check handles initial unavailability, this handles mid-session loss.

### "Mental Implementation" Verification

For features that touch components outside themselves, the Arranger stress-tests the approach by mentally walking through the implementation:

- **Code path tracing via read-only subagents.** "If we add this service, trace the startup sequence — does it initialize before its dependencies?" Subagents explore existing code, follow the execution path, and report potential conflicts. **No edits permitted — these are planning sessions. Subagents must be launched with read-only permissions.**
- **Mocked analysis via Gemini.** Write a pseudo-implementation of a critical integration point and send it to Gemini for review. "Here's how I think the background service would initialize — what issues do you see?" Gemini catches ordering problems, missing error handling, and platform-specific gotchas that code-path tracing alone might miss.
- **Cross-component dependency checks.** When Task A will modify a file that Task B also depends on, verify the integration contract. Will Task A's changes break Task B's assumptions? These findings feed directly into the conductor checkpoint sections as explicit verification items.

**When mental implementation is NOT needed:** Simple implementations, wholly new files, changes isolated within a single component. The trigger is **external interaction** — if the change talks to or affects something outside its own scope, verify it.

**Subagent prompt patterns for mental implementation:**
- "Trace the initialization sequence for [service]. List every file touched in order. Report potential conflicts with [new component]."
- "Simulate adding [feature] to the startup sequence. What initializes before it? What depends on it? What breaks?"
- These are templates adapted per context, not rigid scripts.

### Tool Selection

Best tool for each job, no rigid hierarchy:

- **Gemini** (`gemini-search`, `gemini-query`) — Primary for feasibility checks, setting validation, protocol verification, platform-specific questions. Also valuable for reviewing mocked implementations and suggesting alternatives.
- **Web search** (`brave-search`) — Current documentation, known issues, community discussions, version-specific changelogs. Complements Gemini for discovering what exists.
- **Explore subagents** — Codebase analysis, code path tracing, architectural understanding. **Read-only — no edits. Subagent launch prompts must include explicit instruction: "Do not use Write, Edit, or Bash commands that modify files. This is a read-only planning session."** Used for mental implementation verification and understanding existing patterns the new code must integrate with.
- **General-purpose subagents** — Multi-source synthesis when a topic needs investigation across several sources. Protects main context from dead-end research paths.
- **WebFetch** — Reading specific URLs that research points to.

### Inline vs. Subagent Decision

- **Inline (main session):** Quick Gemini queries, single-source lookups, "I just need to check this one thing." Fast, low overhead, keeps the conversation flowing.
- **Subagent (delegated):** Multi-source investigation, code path tracing, speculative research that might hit dead ends, journal parsing, any historical context that might contain dead ends or abandoned approaches. Protects main context. **All mental implementation verification runs in subagents** — these can be context-heavy and should never consume main session context.

**The intelligent context principle governs this decision.** The waste is in the process of acquiring and validating knowledge, not the knowledge itself. Subagent outputs use bulletpoints for knowledge acquisition results (journal distillation, research findings, feasibility verdicts). But discussion topics, large-scope decisions, and broad architectural ideas retain full verbosity in the main session. The Arranger is never knowledge-starved — it acquires knowledge efficiently through tools, not by limiting what it knows.

### Research Conflict Resolution

**Truth hierarchy:** Official documentation (web) > Gemini analysis > Training data.

When sources conflict, the Arranger follows this hierarchy. For extreme conflicts — where even official docs and Gemini disagree, or where the approach seems theoretically sound but practically problematic — the Arranger conducts additional research focused on **user-provided discussions online**: forums, issue trackers, Stack Overflow threads, GitHub issues. These surface real-world implementation failures — developers experiencing problems, giving up on approaches, discovering undocumented limitations — that official documentation and Gemini analysis may not capture.

### User Override Protocol

If the user disagrees with a feasibility finding ("I know the docs say that, but I have a workaround"), the Arranger allows the user to force the decision. The override is:

1. **Journaled** as a "User Override" entry with the user's rationale
2. **Flagged in the implementation plan** — the relevant conductor checkpoint section includes: "USER OVERRIDE: [setting] set to X despite research indicating Y — user has workaround, see journal entry [ref]"
3. **Not silently absorbed** — the conductor must be aware of override risks during execution

The Arranger yields to the user when overridden, but ensures the risk is visible to downstream consumers.

### Research as Context Preservation

Research findings are always journaled. If the Arranger discovers something during Phase 2 (Feasibility Audit) that's relevant to a Phase 3 discussion, the journal preserves it. If a user discussion triggers a detour that consumes significant context, the journal ensures earlier research findings survive. **The journal is the Arranger's external memory — anything important enough to act on is important enough to journal.**

---

## 5. Decision Journal

The decision journal is the Arranger's progressive external memory — it captures decisions, research findings, and draft content as the session progresses, ensuring no work is lost if user discussions trigger detours or context compaction occurs.

### Purpose

1. **Context preservation.** If a rich user discussion consumes significant context, earlier decisions and research findings survive in the journal. The Arranger can re-read the journal to restore state.
2. **Progressive persistence.** Decisions are written as they're made, not reconstructed at the end. This means the Arranger always has a recoverable checkpoint.
3. **Audit trail.** The journal shows how decisions evolved — not just the final answer but the path taken. This can be valuable if the conductor encounters issues and needs to understand the Arranger's reasoning.

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

### Checkpoint Triggers

The Arranger writes to the journal at:
- Every settled decision in Phase 3 (Implementation Discussion)
- Feasibility findings in Phase 2 that affect planning
- Phase structure decisions in Phase 4
- Any point where the Arranger has done significant autonomous work and is about to engage the user — journal first, then discuss

### Lifecycle

1. **Created** at the start of Phase 2 (Feasibility Audit) in `docs/plans/designs/decisions/{feature-name}/` — initialized with a header and the first entry
2. **Appended to** throughout Phases 2-5
3. **Persists** after finalization — NOT archived or deleted. The journal remains available for the Repetiteur (consultation skill) and conductor reference
4. **Cleaned up** by the conductor after implementation is complete, along with the rest of the feature's decisions directory

---

## 6. Output Format & Section Markers

This section defines the implementation plan's structure — the dual-audience format, the section marker convention, the hybrid document structure integration, and the self-containment rules. **This is the API contract between the Arranger's output and its downstream consumers (conductor, copyist, musician).** Getting this wrong breaks the entire pipeline's context management.

The implementation plan is a **Tier 2** document per the project's hybrid document structure (`docs/hybrid-document-structure.md`). It uses YAML frontmatter for metadata, and weaves authority tags (`<mandatory>`, `<guidance>`, `<context>`) and structural tags (`<core>`, `<section>`, `<sections>`) into the content within sentinel-bounded sections. The sentinel markers (HTML comments) and the hybrid structure's XML tags serve complementary purposes at different levels — sentinels mark section boundaries for machine-parseable line-range extraction, while `<section>` tags provide standard Tier 2 navigation and authority tags signal content classification within those boundaries.

### Dual-Audience Document

The implementation plan serves two audiences through interleaved sections:

1. **Phase sections** — Written for the copyist. Contain detailed implementation content: what to build, how components interact, specific settings and configurations, integration points, testing recommendations. Each phase section must be **self-contained** — the copyist reads only its assigned phase section and must be able to produce complete, unambiguous task instructions from that section alone. Frontend design guidelines are **always inlined** into the phase sections that need them — never in a separate standalone block. Authority tags within phase sections distinguish non-negotiable constraints (`<mandatory>`) from recommended approaches (`<guidance>`) from core implementation content (`<core>`) — the copyist uses these signals to determine what must be preserved verbatim in task instructions vs. what can be adapted for task-level context.

2. **Conductor checkpoint sections** — Written for the conductor. These are **review checklists**, not just context — the conductor must acknowledge/verify each item before proceeding to the next phase. Contain phase goals and boundaries (not prescriptive task lists), verification expectations (especially cross-task integration checks), known risks, user override flags, context management recommendations (lethe compact protocol between phases), and guidance for the next phase. `<mandatory>` tags flag items the conductor must verify; `<guidance>` tags provide recommendations for the next phase. The conductor reads these checkpoints, the overview, and the phase summary as its primary inputs. **It reads phase sections for decomposition context but does not implement from them** — the Copyist and Musicians are the implementation consumers.

### Pipeline Prerequisites

The conductor and copyist already support sentinel marker parsing, plan-index line-range extraction, selective reading, and checkpoint-based verification. Remaining deltas documented in each skill's `docs/working/` directory:
- **Conductor:** Authority tag interpretation in phase-execution reference, Tier 2 document awareness, `overview`/`phase-summary` plan-index entries
- **Copyist:** Authority tag consumption contract, `<section>` tag awareness, integration surface handling guidance

### Document Structure

The implementation plan follows Tier 2 hybrid format — YAML frontmatter for metadata, `<sections>` index and `<section>` tags for navigation, authority tags within sections, standard markdown for content. The sentinel markers (HTML comments) provide machine-parseable section boundaries for line-range extraction; the `<section>` tags provide standard Tier 2 navigation; the authority tags provide content classification within those boundaries.

```markdown
---
title: "Implementation Plan: [Feature Name]"
date: YYYY-MM-DD
type: implementation-plan
tier: 2
feature: [feature-name]
design-doc: docs/plans/designs/[design-doc-name].md
---

# Implementation Plan: [Feature Name]

<!-- plan-index:start -->
<!-- verified:YYYY-MM-DDTHH:MM:SS -->
<!-- overview lines:NN-NN -->
<!-- phase-summary lines:NN-NN -->
<!-- phase:1 lines:NN-NN title:"[Phase Title]" -->
<!-- conductor-review:1 lines:NN-NN -->
<!-- phase:2 lines:NN-NN title:"[Phase Title]" -->
<!-- conductor-review:2 lines:NN-NN -->
<!-- plan-index:end -->

<sections>
- overview
- phase-summary
- phase-1
- conductor-review-1
- phase-2
- conductor-review-2
</sections>

<!-- overview -->
<section id="overview">
## Overview

<core>
[What we're building, why, end goals, defined constraints from design
and research. Comprehensive enough that the conductor understands the
full picture without reading phase sections or the design doc. This
overview REPLACES the design doc for conductor purposes — the conductor
reads this plan, not the dramaturg's output.]
</core>

<context>
[Background — preceding design work, dramaturg decisions carried forward,
relevant project state at time of planning.]
</context>
</section>
<!-- /overview -->

<!-- phase-summary -->
<section id="phase-summary">
## Phase Summary

<core>
[Quick reference: what each phase covers, dependencies between phases,
parallelization opportunities. The conductor's map of the work.
Written during Phase 4 (Phase Structuring).]
</core>
</section>
<!-- /phase-summary -->

<!-- phase:1 -->
<section id="phase-1">
## Phase 1: [Phase Title]

<core>
### Objective
[What this phase accomplishes]

### Prerequisites
[What must be true before this phase starts — outputs from prior phases,
specific file paths and exports created]

### Implementation

<mandatory>[Non-negotiable constraint specific to this phase]</mandatory>

[Detailed implementation content — settings, file paths, component
interactions, error handling approaches. All decided by the Arranger.]

<guidance>
[Approach recommendations, patterns to follow, suggested order of work.
The copyist can adapt these for task-level context.]
</guidance>

### Integration Points
[How this phase's work connects to other phases' work.
Contracts that must be maintained.]

### Expected Outcomes
[What the conductor should see when this phase completes successfully]
</core>
</section>
<!-- /phase:1 -->

<!-- conductor-review:1 -->
<section id="conductor-review-1">
## Conductor Review: Post-Phase 1

<core>
### Verification Checklist

<mandatory>All checklist items must be verified before proceeding.</mandatory>

- [ ] [Cross-task integration check]
- [ ] [Expected outcome verification]
- [ ] [Contract verification between phases]

### Known Risks
[Issues to watch for, edge cases identified during planning]

### Guidance for Phase 2

<guidance>
[Recommendations for task decomposition, parallelization opportunities,
dependencies to respect.

Context management: Run `/lethe compact` before starting Phase 2
to compress the completed phase work and reclaim context headroom.]
</guidance>
</core>
</section>
<!-- /conductor-review:1 -->

...repeat for all phases...
```

The plan-index and `<sections>` serve complementary purposes. `<sections>` is the standard Tier 2 navigation index — it lists section IDs for targeted reading. The plan-index adds line-range specificity and serves as the lock indicator (its presence confirms finalization passed). Both are maintained: `<sections>` is authored during section writing, the plan-index is generated during finalization.

### Sentinel Markers, Section Tags, and Authority Tags

The implementation plan uses three complementary systems:

1. **Sentinel markers** (HTML comments) — Section boundary markers for machine-parseable line-range extraction. The conductor reads the plan-index, extracts line ranges, and tells the copyist "read lines X-Y." Sentinels are the boundaries that make this work. They are also human-invisible in rendered markdown.

2. **`<section>` tags** — Standard Tier 2 navigation tags with `id` attributes. These provide the hybrid document structure's section navigation convention, enabling targeted reading by section ID. The `<sections>` index near the top of the document lists all section IDs.

3. **Authority tags** (`<mandatory>`, `<guidance>`, `<context>`, `<core>`) — Content classification within sections. These signal how the copyist and conductor should treat enclosed content.

Sentinel markers and `<section>` tags coexist as the outermost boundary layer — sentinels provide line ranges for the plan-index, `<section>` tags provide ID-based navigation per Tier 2 convention. Authority tags operate inside both, classifying content within those boundaries.

**Why sentinel markers in addition to `<section>` tags:** The plan-index maps sentinel markers to line ranges — this is the conductor's primary consumption mechanism. `<section>` tags provide the standard Tier 2 navigation convention that all hybrid-format documents share. Sentinel markers also protect against LLM formatting drift — HTML comments are structurally simpler than XML tags for boundary detection. The finalization checklist validates alignment between both systems.

| Sentinel | Section Tag | Markdown Header | Audience |
|---|---|---|---|
| `<!-- phase:N -->` / `<!-- /phase:N -->` | `<section id="phase-N">` | `## Phase N: [Title]` | Copyist |
| `<!-- conductor-review:N -->` / `<!-- /conductor-review:N -->` | `<section id="conductor-review-N">` | `## Conductor Review: Post-Phase N` | Conductor |
| `<!-- overview -->` / `<!-- /overview -->` | `<section id="overview">` | `## Overview` | Conductor |
| `<!-- phase-summary -->` / `<!-- /phase-summary -->` | `<section id="phase-summary">` | `## Phase Summary` | Conductor |

### Verification Index (Top of File)

The plan begins with a machine-readable index inside `<!-- plan-index:start -->` / `<!-- plan-index:end -->` sentinels. This index maps phase numbers to line ranges and serves a dual purpose:

1. **Line range map** — the conductor reads the index first and knows exactly where to send the copyist without scanning the document
2. **Lock indicator** — the index's presence confirms the finalization checklist passed. The conductor's first step is to check for `<!-- plan-index:start -->`. If absent, the plan is unverified — stop and report.

The index — including the `verified` timestamp — is generated by the producing skill's finalization process: the Arranger's verification subagent for original plans, the Repetiteur's finalization for remaining plans. The timestamp confirms that finalization verification passed and the index is accurate to the committed file state. (Note: `repertoire/output-format.md` currently attributes the timestamp to the Conductor — this will be corrected in the repertoire update pass.)

**How the conductor uses the index:** Read the index, extract the line range for the relevant phase, tell the copyist "read lines X-Y of the implementation plan." This keeps the copyist focused on its scope and prevents the conductor from ingesting detail it doesn't need. **This line-range approach is critical for downstream context management.**

The plan-index and `<sections>` serve complementary purposes at different levels. `<sections>` is the standard Tier 2 navigation index — it lists section IDs for targeted reading by any consumer. The plan-index adds line-range specificity and serves as the lock indicator (its presence confirms finalization passed). A separate `<sections>` tag is maintained because it is a Tier 2 convention — any Claude session reading a Tier 2 document expects it, and it provides navigation even if the plan-index is not yet generated (during section writing, before finalization).

### Self-Containment Rules

Each phase section must pass this test: **Could a copyist session reading only these lines produce complete, unambiguous task instructions?**

A self-contained phase section includes:
- **Objective** — What this phase accomplishes
- **Prerequisites** — What must be true before this phase starts (outputs from prior phases, specific file paths and exports created)
- **Implementation detail** — Specific enough that the copyist doesn't need to infer or research. Settings, file paths, component interactions, error handling approaches — all decided by the Arranger, not left for the copyist to figure out.
- **Integration points** — How this phase's work connects to other phases' work. What contracts must be maintained.
- **Frontend guidelines** — When applicable, inlined directly (Material 3 patterns, widget recommendations sourced from external references — not training data)
- **Expected outcomes** — What the conductor should see when this phase completes successfully
- **Testing recommendations** — When the Arranger discovered testing considerations during planning

**Authority tags enhance self-containment.** Within each phase section, hybrid structure tags classify content by authority level. `<mandatory>` marks non-negotiable constraints the copyist must preserve verbatim in task instructions. `<guidance>` marks recommended approaches the copyist can adapt when splitting work across tasks. `<core>` wraps the primary implementation substance. This classification helps the copyist make intelligent decisions about what to carry forward literally vs. what to adjust for task-level context — without requiring the copyist to make judgment calls about which parts of an undifferentiated text block are constraints vs. suggestions.

**What self-containment means:** The *implementation instructions* within each phase section are complete. The copyist should rarely need to reference the overview section — doing so defeats the purpose of self-containment, which is **constraining the copyist's context usage.** The copyist has tasks to create for everything in its assigned phase; loading a large overview on top of that wastes the context it needs for task creation.

**Frontend guidance is always inlined.** When phases involve Flutter UI work, the externally-researched design guidelines are duplicated into each phase section that needs them. There is no standalone Frontend Reference block. This preserves self-containment — the copyist never reads two sections. Document size increase is acceptable given the sentinel marker system provides reliable section boundaries. **If section finding fails on a larger document, the result is immediate context exhaustion** — this reinforces the importance of the finalization checklist's marker validation.

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

### Danger File Annotations

Phase sections may contain inline annotations marking known file conflicts discovered during Phase 4 (Phase Structuring):

```
<!-- danger-file: path/to/file.dart shared-with="phase:3" -->
```

The Conductor treats these as supplementary starting points for danger file identification — self-discovery during implementation remains the primary method. The Arranger adds these annotations when cross-phase file conflicts are identified during phase structuring, providing the Conductor with advance warning.

---

## 7. Deviation Detection & Loop-backs

When the user proposes changes during the interactive phases (3-5), the Arranger must determine the scope of the change and loop back appropriately. This mirrors the Dramaturg's vision regression detection but adapted for implementation-level decisions.

### Deviation Hierarchy

| Deviation type | Example | Detection | Loop to |
|---|---|---|---|
| **Structural** | "Let's split this into two separate services instead of one" | Changes the phase structure — what phases exist, how work is decomposed, parallelization strategy | Phase 2 (Feasibility Audit) — the new structure needs verification |
| **Implementation** | "Use SharedPreferences instead of SQLite for that setting" | Changes the technical approach within the existing structure | Phase 3 (Implementation Discussion) — research and confirm the change |
| **Detail** | "Make that polling interval 10 minutes instead of 5" | Refinement within an already-settled approach | Revise in place — journal the change, update the section |

### Behavior When Deviation Is Detected

1. **The Arranger surfaces the scope.** "This changes how the phases are structured — we'd need to re-verify feasibility for the new approach. Should we loop back?" Or: "This is an implementation change within the existing structure — let me research it and we can discuss."
2. **The user confirms.** The user decides whether the Arranger's assessment of scope is correct. What looks structural might actually be a detail change from the user's perspective, or vice versa.
3. **Loop-back executes.** The Arranger returns to the appropriate phase. The decision journal preserves all work done before the deviation — nothing is lost. Superseded decisions are noted in the journal with the new direction.

### Why Explicit Detection Matters

Without deviation detection, two failure modes occur:
- **Under-reaction:** A structural change gets treated as a detail tweak. The phase structure is now wrong but the Arranger keeps writing sections based on it. The conductor later discovers the plan doesn't make sense.
- **Over-reaction:** A detail change triggers a full feasibility re-audit. The user's time is wasted re-verifying things that haven't changed.

The explicit hierarchy ensures the response is proportional to the change. **The user always confirms the Arranger's scope assessment** — this prevents false positives in both directions.

### Journal Continuity Through Loop-backs

When a loop-back occurs:
1. The Arranger appends a deviation entry to the journal: what changed, why, what phase it's looping to
2. Previous decisions remain in the journal — they're not deleted
3. New decisions that supersede old ones reference the original entry
4. When compiling the final plan (Phase 6), the Arranger uses the *latest* decision for each topic

This means the journal grows during deviations but the final plan remains clean. The archived journal shows the full decision evolution.

---

## 8. Conversation Style

The Arranger inherits the Dramaturg's conversational patterns, adapted for implementation-level discussion. These rules define the skill's character — they are reinforced here and throughout every section where the behavior is relevant, because **authoritative statements made once get lost during actual work.**

### Decision→Research→Discussion→Decision Loops

The core interaction pattern. When an implementation decision needs to be made:
1. Identify the decision point
2. Research if needed — **all Android/cross-device settings externally verified, all unimplemented protocols checked via Gemini, all specific configuration values validated**
3. Present findings with an explicit recommendation (not just options)
4. Discuss with the user
5. Explicitly settle: "So we're going with [approach] for [reason]. Moving on?"
6. Journal the decision

This loop repeats as many times as the discussion needs. Research and discussion are interleaved, not sequential — if discussing findings raises new questions, research those immediately.

### One Question at a Time (With the Same Exception)

**Default:** One question per message. Implementation decisions often have nuance that multi-question dumps lose.

**Exception:** Multiple tight, bounded questions in a single message when each expects a short answer. "Should the retry count be 3 or 5?" and "Exponential or linear backoff?" can be batched. "How should the error recovery work?" cannot be batched with anything — it's open-ended.

### Output Length

**Never truncate reasoning to fit an artificial limit.** If validating a protocol requires 150 lines of analysis, that analysis gets 150 lines. The ~75 line mark is a heuristic for when to split into separate messages if covering multiple topics — not a cap on single-topic depth.

### One Section at a Time for Review

During Phase 5 (Section Writing & Review), present exactly one section per message. The user reviews and approves each before the next is presented. This keeps scope focused and ensures every integration concern, every verification item, and every implementation detail gets proper attention.

### Fact-Checker Tone

The Arranger is a fact-checker and setting-decider, not a code-writer. Its conversation should reflect this identity:
- "Let me verify that setting before we commit to it" — not "Let me write that code"
- "Research shows Android limits this to X" — presenting validated facts
- "I traced through the startup sequence and found a potential ordering issue" — reporting verification findings
- "Based on the Gemini analysis, I recommend Y over Z because..." — explicit recommendations backed by evidence
- "You can override this, but I'll flag it for the conductor" — when the user disagrees with research findings (see User Override Protocol in Section 4)

### Reinforcement Principle

Critical constraints appear in every section where they're relevant, not just in their "home" section. This design document follows the same principle — mandatory external verification is mentioned in the Workflow Phases, the Research & Verification Strategy, and here in Conversation Style. **When this skill is eventually written, the same reinforcement approach must carry through to the skill file itself.** Authoritative statements made once get lost during actual work. Repeated reminders at the point of relevance keep them active.

---

## 9. Plugin Structure

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

### Pipeline Position

```
dramaturg → arranger → conductor → musician
(vision)    (plan)     (coordination)  (implementation)
                        copyist creates individual parts from arranger's score
              repetiteur handles mid-implementation consultation if blockers arise
```

The arranger consumes the dramaturg's design document and decision journal, then produces an implementation plan that the conductor consumes. The conductor determines task decomposition from the arranger's boundary/goal guidance and coordinates the copyist to create task instruction files for musicians.

### Relationship to Adjacent Skills

**Upstream — Dramaturg:** The arranger treats the design doc as settled vision. It does not re-litigate what/why decisions. The dramaturg's decision journal (with VERIFIED/PARTIAL flags) is consumed via subagent to scope the feasibility audit. If feasibility research reveals a design assumption is unworkable, the arranger surfaces this as a conflict to the user — it doesn't silently change the design direction.

**Sibling — Repetiteur:** The Repetiteur is a separate skill that consumes the same shared repertoire contracts. It handles mid-implementation consultation when the conductor hits blockers that can't be resolved autonomously. The Repetiteur produces a "remaining plan" — a full, standalone implementation plan for remaining work only. Decision journals persist in `docs/plans/designs/decisions/{feature-name}/` specifically to support Repetiteur consultations.

**Downstream — Conductor:** The conductor will read the verification index, the Overview, Phase Summary, and Conductor Review sections. It reads phase sections for decomposition context but does not implement from them — the Copyist and Musicians are the implementation consumers. It uses the line range index to pass phase boundaries to the copyist. **The arranger's sentinel marker convention, hybrid structure tag conventions, and verification index are the API contract that makes this work.** Checkpoint sections are **review checklists** — items the conductor must acknowledge/verify before proceeding. They contain boundary/goal guidance for task decomposition, not prescriptive task lists, giving the conductor freedom to determine task granularity based on actual conditions.

**Downstream — Copyist:** The copyist reads only its assigned phase section (by line range from the conductor). The phase section must be self-contained — **the copyist should rarely need to reference the overview**, as doing so defeats the context-constraining purpose of self-containment. Authority tags within phase sections (`<mandatory>`, `<guidance>`, `<core>`) signal what the copyist must preserve verbatim in task instructions vs. what can be adapted for task-level context. The copyist estimates context requirements per task and may propose task splits; the Conductor approves the split plan. The Arranger's loose boundary structure enables this flexibility.

**Downstream — Musician:** Musicians never read the arranger's plan directly. They receive task instructions from the copyist. The arranger's influence on musicians is indirect — through the quality and completeness of its phase sections, which determine the quality of task instructions the copyist produces.

---

## 10. Design Decisions Log

Decisions made during the brainstorming process and their rationale:

| Decision | Alternatives Considered | Rationale |
|---|---|---|
| **Linear pipeline with discussion gates** over iterative per-topic or front-loaded autonomous | Per-topic deep dives, heavy autonomous with late discussion | Preserves dramaturg-style discussion loops while front-loading feasibility work. Linear structure maps cleanly to decision journal checkpoints. |
| **Hybrid interactivity** (autonomous work + inline decisions + section review) | Fully interactive, checkpoint-gated, mostly autonomous | Autonomous research avoids wasting user time on verification grunt work. Inline decisions catch misunderstandings early. Section review ensures the final output matches user expectations. |
| **Decision journal as progressive external memory** | Progressive file writing with back-editing, temp file tracking | Append-only avoids inconsistent state. `decisions/{feature-name}/` directory survives reboots. Clean final plan compiled fresh from journal. Journal persists for Repetiteur consultation reference — conductor cleans up after implementation. |
| **Dual-audience document with interleaved sections** | Single-audience plan, separate conductor/copyist documents | One document, two audiences, hybrid sentinel markers. Eliminates document synchronization problems while enabling context-efficient consumption via index-based line ranges. |
| **Hybrid sentinel markers** (HTML comments + markdown headers) | Markdown headers only, XML tags, structured data format | HTML comment sentinels for machine consumption (exact match, trivial). Markdown headers for human readability. Dual markers protect against LLM formatting drift. Finalization checklist validates both. |
| **Verification index at top of file** (lock indicator) | No index (grep on demand), index at bottom | Top placement gives conductor immediate access. Index serves dual purpose: line range map AND lock indicator (presence confirms finalization passed). Generated by verification subagent during finalization. |
| **Explicit finalization phase with user gate** | Auto-commit after compilation, commit during section writing | User may iterate multiple times before locking. Finalization = user confirms → checklist runs → index generated → commit. Separates editing from locking. |
| **Self-contained phase sections with minimal overview references** | Sections that reference overview freely, fully standalone documents | Primary purpose is **constraining copyist context.** If the copyist must read a large overview alongside its phase section, self-containment is defeated. Sections carry their own context; overview references are the exception. |
| **Mandatory external verification for Android/cross-device/unimplemented protocols** | Trust training data for known patterns, verify only unknowns | Platform APIs change with OS versions. Training data is unreliable for version-specific behavior. WiFi scan limits and FCM crash configs are canonical examples of what goes wrong without verification. |
| **Read-only subagents for mental implementation** | Allow subagent edits, skip code-path verification | Mental implementation catches cross-component issues (startup ordering, contract violations) that individual musicians miss. Read-only is non-negotiable — these are planning sessions. |
| **Phase structuring as parallelization strategy** | Flag parallelizable tasks, let conductor decide structure | The arranger's phase arrangement IS the parallelization decision. Smart decomposition (prep → dependent work → integration) unlocks parallel execution. Naive per-feature phasing forces sequential work. |
| **Cross-task integration in conductor checkpoints** | Leave integration to musicians, catch in testing | Musicians handle errors within their scope but are blind to cross-task contract violations. Conductor checkpoints with explicit integration verification items catch these at the coordination layer. |
| **Frontend guidance always inlined** | Standalone frontend reference block, separate document | Self-containment is non-negotiable — copyist never reads two sections. Document size increase acceptable given reliable sentinel markers. If section finding fails on larger docs, immediate context exhaustion — finalization checklist validates. |
| **Task decomposition is conductor's concern** | Arranger prescribes exact tasks, conductor just relays | Conductor (1M context) has capacity for task decomposition. Arranger provides boundary/goal guidance in checkpoints. Copyist estimates context per task and may propose splits (Conductor approves). Loose structure enables adaptation when "boots hit the ground." |
| **Conductor checkpoints as review checklists** | Checkpoints as context-only, checkpoints as task lists | Conductor is overseer/parent/teacher — must acknowledge/verify items before proceeding. Checklists ensure active verification, not passive context. |
| **Repetiteur as separate skill** (not Arranger mode) | Consultation as Arranger mode, no consultation support | Interaction models diverge fundamentally — Arranger is interactive, Repetiteur is autonomous. Reinforcement principle makes mode-switching within one skill problematic. Shared repertoire contracts prevent drift. |
| **Decision journals persist** (not archived) | Archive after compilation, delete after compilation | Journals needed by Repetiteur for consultation context. Conductor cleans up after implementation complete. `decisions/{feature-name}/` directory per feature. |
| **Truth hierarchy** (official docs > Gemini > training data) | No explicit hierarchy, case-by-case | Explicit ordering for research conflicts. Extreme cases escalate to user discussion research (forums, issue trackers) for real-world failure evidence. |
| **User override protocol** | Silent absorption, hard refusal | User can force decisions against research. Override journaled and flagged in conductor checkpoint — risk visible to downstream. Arranger yields but ensures visibility. |
| **Intelligent context usage via subagent delegation** | All research in main session, aggressive compression | Knowledge acquisition wastes context, not knowledge itself. Subagents absorb dead ends; main session gets distilled findings. Not knowledge-starved — efficiently acquired. |
| **Incomplete design doc threshold** (≤3 questions inline, 4+ → Dramaturg) | Always handle gaps, always reject | Small gaps handled in Phase 3 discussion. Large gaps sent upstream where they belong. Arranger doesn't fill design gaps — that's re-litigation. |
| **Session split at Phase 4/5 boundary** | No split guidance, split at Phase 2/3 | Phases 1-4 are research-heavy, Phase 5 is output. Journal preserves state across split. Same pattern as Dramaturg. |
| **Deviation hierarchy** (structural → implementation → detail) | Single loop-back point, no formal detection | Proportional response to changes. Structural changes need re-audit, implementation changes need re-discussion, detail changes need a quick revision. User confirms the Arranger's scope assessment. |
| **Reinforcement of critical constraints throughout doc** | State constraints once authoritatively | Authoritative statements made once get lost during actual work. Repeated reminders at the point of relevance keep them active — both in this design doc and in the eventual skill file. |
| **compatibility > reliability > efficiency > security > performance** | No explicit priority chain, case-by-case | Explicit ordering prevents ambiguous trade-off decisions. Modern but not bleeding edge. See `repertoire/priority-chain.md` for full rationale. |
| **Plan replaces design doc for conductor** (not used alongside it) | Conductor reads both design doc and plan | The implementation plan carries forward all relevant design context. Requiring the conductor to read two documents wastes context and creates potential contradictions. |
| **Invocation: direct file or auto-scan** | Always require file path, always scan | Direct path is efficient when the user knows which design. Auto-scan with single-file auto-select and multi-file prompt handles the common cases without extra user effort. |
| **Name: "arranger"** | "blueprint," "architect," "planner" | In music, the arranger takes a composition and creates the detailed arrangement — which instruments, what's simultaneous, what's sequential. Creates a natural arts pipeline: dramaturg → arranger → conductor → musician, with copyist creating individual parts. |
| **Tier 2 hybrid format for implementation plan** | Pure markdown, Tier 3 full XML, custom format | Tier 2 is the project default for Claude-consumed documents with human reviewability. YAML frontmatter for metadata, `<sections>` and `<section>` tags for navigation, authority tags within sections. Sentinels preserved for their specific line-range purpose — complementary systems at different levels. |
| **Flat markdown journal with Strength annotation** | Tier 3 XML journal, single entry type | Flat markdown entries following repertoire journal conventions. `Strength` field (`mandatory`/`core`/`context`) carries semantic meaning: `mandatory` for user overrides (must propagate), `core` for standard decisions, `context` for deviations (informational). Preserves semantic signal without breaking Repetiteur parsing of standard journal format. |
| **Three-layer structure** (sentinels + section tags + authority tags) | Replace sentinels with `<section>` tags, use only sentinels, single system | Three complementary levels: sentinels for line-range extraction, `<section>` tags for Tier 2 navigation, authority tags for content classification. Sentinels give machine-parseable boundaries with line ranges, `<section>` tags give standard navigation per hybrid structure, authority tags classify content within. |
| **Plan-index and `<sections>` as complementary navigation** | Plan-index replaces `<sections>`, `<sections>` only, no navigation index | `<sections>` provides standard Tier 2 section ID navigation expected by any consumer. Plan-index adds line-range specificity and lock indication. Both maintained — `<sections>` authored during writing, plan-index generated during finalization. Consistent with Tier 2 conventions. |
