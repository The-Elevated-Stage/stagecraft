# Arranger-Repertoire-Repetiteur Alignment Review

**Date:** 2026-02-28
**Reviewer:** reviewer-3
**Scope:** Arranger design doc vs repertoire shared contracts vs built Repetiteur skill

---

## Executive Summary

The Arranger design document is **largely well-aligned** with the repertoire shared contracts and the Repetiteur skill on substance -- the priority chain, verification rules, journal conventions, and output format are conceptually consistent across all three. However, there are several structural and naming issues that must be resolved before the Arranger is built, and a few substantive format mismatches between the Arranger's journal design and the repertoire's journal conventions that would prevent the Repetiteur from seamlessly parsing Arranger-produced journals.

The most impactful issues are:

1. **Directory naming drift** -- The Arranger design references `shared-rules.md` and `references/shared-rules.md` as a single monolithic file. The Repetiteur references `score-preparation/` as the directory. The actual directory is `repertoire/` with multiple individual files. All three names are inconsistent.
2. **Journal format mismatch** -- The Arranger specifies Tier 3 XML-wrapped journals with `<journal>`, `<section>`, and authority-differentiated entry types. The repertoire's journal-conventions.md specifies plain markdown journals with a flat entry template. The Repetiteur's journal analysis subagent is designed to parse the repertoire format. The Arranger's XML format would require different parsing.
3. **Output format Tier mismatch** -- The Arranger specifies Tier 2 hybrid format for implementation plans (YAML frontmatter, `<sections>`, `<section>` tags, authority tags). The repertoire's output-format.md specifies a simpler format with only sentinel markers and plain markdown -- no YAML frontmatter, no `<sections>` index, no `<section>` tags, no authority tags. This is a significant structural gap.

---

## Detailed Findings by Focus Area

### 1. Priority Chain

**Status: Aligned**

The Arranger design (Section 1, Operating Principles, and Decision 14 in Section 10) states:

> compatibility > reliability > efficiency > security > performance

This matches `repertoire/priority-chain.md` exactly in both ordering and definitions. The Arranger also provides the same rationale for security ranking below efficiency (personal app with narrow threat model), which aligns with the priority chain's description.

The Repetiteur references the priority chain via `score-preparation/priority-chain.md` (path issue aside) and uses it during adaptive resolution for trade-off decisions. The example consultation journal shows the chain being applied correctly ("WebSockets rank highest on the priority chain -- compatibility, reliability, efficiency").

**Issues:** None on substance. Path naming is a separate concern (see Section 7).

---

### 2. Verification Rules

**Status: Aligned on substance, minor terminology differences**

The Arranger design (Section 4) specifies four mandatory external verification categories:
1. Android and cross-device services/settings/usage patterns
2. Protocols and patterns not already implemented in the project
3. Protocols and patterns not already implemented in the project
4. Flutter frontend design guidelines
5. Specific configuration values and settings

These match `repertoire/verification-rules.md` exactly. The four categories are identical, the canonical examples match (WiFi scan interval limit, FCM crash config), and the "no exceptions" stance is consistent.

The verification-rules also include:
- Mental implementation verification (code path tracing, mocked Gemini analysis, cross-component checks)
- Tool selection guidance
- Truth hierarchy (Official docs > Gemini > Training data)
- Structural verification (sentinel markers, header-sentinel consistency, index accuracy)

All of these appear in the Arranger design (Section 4) with matching detail. The Repetiteur's `workflow-stages.md` references the same verification standards and explicitly states "Research and verification follow the same standards as the Arranger."

**Minor observation:** The verification-rules reference includes a "structural verification" section that defines what to check (sentinel markers, header-sentinel consistency, index accuracy) and defers to output-format.md for what correct looks like. The Arranger design's finalization checklist (Section 3, Phase 6) expands this structural verification significantly -- adding hybrid structure tag validation, YAML frontmatter validation, section tag validation, and large document marker tests. These are Arranger-specific additions that do not exist in the shared verification-rules. This is not a conflict, but the Arranger extends the structural verification beyond what the shared contract defines.

---

### 3. Decision Journal Format

**Status: MISALIGNED -- Critical issue**

This is the most significant misalignment. The Arranger design and the repertoire's journal-conventions.md specify fundamentally different journal formats.

#### Repertoire journal-conventions.md specifies:
- **Plain markdown** files
- **Flat entry template:**
  ```markdown
  ## [Stage/Phase]: [Topic]
  **Finding/Decision:** ...
  **Rationale:** ...
  **Alternatives considered:** ...
  **Impact:** ...
  **External input:** ...
  ---
  ```
- Entry categories are **implicit** (Goal/Use-Case, Decision, Constraint -- inferred from content)
- Title convention: `# Consultation N Journal: {Feature Name}`
- Append-only, sequential entries

#### Arranger design (Section 5) specifies:
- **Tier 3 XML-wrapped** format with `<journal>` document-level wrapper
- **Structured sections** with `<section id="...">` tags and incrementing IDs
- **Authority-differentiated entries:**
  - `<core>` for decisions
  - `<mandatory>` for user overrides (must propagate to conductor checkpoints)
  - `<context>` for deviations (informational)
- `<metadata>` block and `<sections>` index at the top
- Entry template:
  ```xml
  <section id="checkpoint-1">
  <core>
  ## Checkpoint: [Phase Name] -- [Topic]
  **Decision:** ...
  **Rationale:** ...
  **Alternatives considered:** ...
  **Impact:** ...
  </core>
  </section>
  ```
- Separate entry types for overrides and deviations with distinct header patterns

#### Impact on Repetiteur parsing:
The Repetiteur's `journal-analysis.md` reference defines subagent prompts that read "all journals in the feature's decisions directory." The bootstrap path extracts "underlying constraints vs surface choices." The subagent output format expects `Decision`, `Underlying constraint`, `Strength`, `Source`, `Revisability` fields.

The Repetiteur's journal analysis was designed for the repertoire's flat markdown format -- it processes journals by reading entries sequentially and extracting constraint information. The Arranger's XML-wrapped format would require different parsing:
- The subagent would need to handle `<section>` tag navigation
- Authority tags (`<mandatory>` vs `<core>` vs `<context>`) carry semantic meaning the flat format lacks
- The entry header patterns differ (`## Checkpoint:` vs `## [Stage/Phase]:`)
- User override entries use a completely different field set (`User's position`, `Research finding`, `User's rationale`, `Flagged for conductor`)

The Repetiteur's journal-analysis subagent treats Dramaturg and Arranger journals as "user-voice journals" where decisions are binding. The authority tag differentiation in the Arranger's XML format is actually useful information -- `<mandatory>` overrides are stronger signals than `<core>` decisions. But the parsing logic would need to understand this structure.

The repertoire's `example-consultation-journal.md` shows the flat markdown format, which is what the Repetiteur was built to produce and parse. If the Arranger produces the XML format, there is a format mismatch in the journal pipeline.

#### What needs resolution:
Either (a) the Arranger design should adopt the repertoire's flat markdown journal format, or (b) the repertoire's journal-conventions.md should be updated to accommodate the Arranger's richer format, and the Repetiteur's journal analysis should be updated to handle both. Option (a) is simpler but loses the authority-tag semantic differentiation. Option (b) is more complex but preserves valuable signal.

A third option (c) is to update journal-conventions.md to define two tiers of journal format -- a "simple" format for consultation journals and a "structured" format for Arranger/Dramaturg journals -- with the Repetiteur's journal analysis supporting both. The Repetiteur already treats these journal types differently (user-voice vs autonomous), so format differences aligned to that distinction may be natural.

---

### 4. Output Format

**Status: MISALIGNED -- Critical issue**

The Arranger design and the repertoire's output-format.md describe structurally different plan formats.

#### Repertoire output-format.md specifies:
- Pure markdown with sentinel markers (HTML comments)
- No YAML frontmatter
- No `<sections>` index
- No `<section>` tags
- No authority tags within phase sections
- Simple structure: plan-index, overview, phase-summary, phase sections, conductor-review sections
- Sentinel markers are the only structural mechanism

#### Arranger design (Section 6) specifies:
- **Tier 2 hybrid format** with:
  - YAML frontmatter (`title`, `date`, `type`, `tier`, `feature`, `design-doc`)
  - `<sections>` index listing all section IDs
  - `<section id="...">` tags wrapping each section
  - Authority tags (`<mandatory>`, `<guidance>`, `<context>`, `<core>`) within sections
- Three-layer structure: sentinels + section tags + authority tags
- Phase sections include authority-classified content:
  - `<mandatory>` for non-negotiable constraints
  - `<guidance>` for recommended approaches
  - `<core>` for primary implementation content
- Conductor review sections use `<mandatory>` for required verification items and `<guidance>` for next-phase recommendations

#### Impact on the pipeline:
The Repetiteur produces remaining plans following `repertoire/output-format.md` -- the simpler format without YAML frontmatter, `<sections>`, `<section>` tags, or authority tags. If the Arranger produces Tier 2 format and the Repetiteur produces the simpler format, the Conductor and Copyist receive structurally different plans from different sources. This breaks the pipeline contract that "both original plans (from the Arranger) and remaining plans (from the Repetiteur) use this format."

The output-format.md explicitly states: "Implementation plans follow a standardized structure that the Conductor and Copyist consume with identical logic. Both original plans (from the Arranger) and remaining plans (from the Repetiteur) use this format."

The Arranger design even acknowledges in Section 6: "The implementation plan follows Tier 2 hybrid format" and includes detailed specifications for the three-layer structure. But this was designed independently of the output-format.md contract, which defines only sentinel markers.

#### What needs resolution:
The output-format.md in repertoire must be updated to match whatever format the Arranger will actually produce. Since the Arranger's Tier 2 format is richer and carries more information (authority tags are genuinely useful for the Copyist and Conductor), the likely path is to update output-format.md to specify the Tier 2 format, then update the Repetiteur to produce that format for remaining plans.

Alternatively, the Arranger design could be simplified to match the current output-format.md, but this loses valuable authority tag information that the Arranger design explicitly argues is important for downstream consumers.

---

### 5. Repetiteur Compatibility

**Status: Aligned on workflow, misaligned on artifacts**

#### Workflow alignment (good):
- The Repetiteur reads the Arranger's plan and decision journal during ingestion (Track A and B of parallel ingestion)
- The journal analysis bootstrap path extracts constraints from all journals including the Arranger's
- The research path validates proposed approaches against journal constraints
- Impact assessment traces blocker effects across the plan
- The remaining plan preserves original phase numbers and uses task annotation markers
- The Repetiteur's 4-stage workflow (ingestion, impact assessment, adaptive resolution, verification/handoff) is compatible with the Arranger's output

#### Artifact misalignments (problematic):
As detailed in Sections 3 and 4 above, the actual artifacts the Arranger produces would differ structurally from what the Repetiteur expects:

1. **Plan reading:** The Repetiteur reads the plan's verification index for phase structure, then reads relevant phase sections via line ranges. This works with both formats -- the plan-index sentinel markers are present in both the Arranger design and the repertoire format. However, if the Arranger plan includes `<section>` tags and authority tags, the Repetiteur's subagents would encounter XML-like structure they may not be specifically prompted to handle.

2. **Journal parsing:** As detailed in Section 3, the Arranger's XML journal format differs from what the Repetiteur's journal analysis subagent expects.

3. **Remaining plan generation:** The Repetiteur generates remaining plans following output-format.md (the simpler format). If the original plan was Tier 2 format, the remaining plan would be structurally simpler. The Conductor would receive a downgrade in structural richness mid-feature. This is functionally acceptable (the simpler format works) but aesthetically inconsistent.

4. **Plan-index format compatibility:** Both formats use the same plan-index sentinel markers, so the Conductor's selective reading mechanism works identically. The plan-index format is consistent between the Arranger design and the repertoire contract.

#### Remaining plan compatibility:
The Repetiteur's remaining plan format matches the repertoire's output-format.md. Task annotation markers (`(REVISED)`, `(NEW)`, `(REMOVED)`), rollback task format, consultation context section, and self-containment rules are all consistent between the Repetiteur's output and the repertoire contract. The examples in repertoire (example-remaining-plan.md, example-consultation-journal.md) match what the Repetiteur's skill file describes.

---

### 6. Shared Vocabulary

**Status: Mostly aligned with minor inconsistencies**

#### Consistent terms across all three:
- **Phases** -- both Arranger and Repetiteur use "phase" for the major work divisions
- **Conductor checkpoint/review** -- consistent naming
- **Sentinel markers** -- identical HTML comment syntax
- **Plan-index** -- same concept and format
- **Self-containment** -- same definition ("Copyist test")
- **Verification index / verified timestamp** -- same lock indicator concept
- **Decision journal** -- same concept (though format differs, see Section 3)
- **User-voice vs autonomous journals** -- journal-conventions distinguishes these; the Repetiteur implements this distinction
- **Priority chain ordering** -- identical across all three
- **Mental implementation** -- same concept in verification-rules and the Arranger design

#### Inconsistencies:
- **"Task" vs "phase section"**: The Arranger design explicitly states it does NOT prescribe exact task boundaries -- "Task-level decomposition is the conductor's responsibility." Phase sections provide goals and boundary guidance, not tasks. But the repertoire's output-format.md shows task-level content within phase sections (e.g., `### Task 3.2: WebSocket Signaling Adapter (REVISED)`). The Repetiteur's remaining plans include task-level detail with annotation markers. This creates a vocabulary tension: the Arranger produces phase sections without task granularity, but the output-format examples show task-level detail. The Conductor performs task decomposition from the Arranger's boundary guidance, and then the Copyist produces task instructions. But the Repetiteur includes tasks in its remaining plans because it is revising existing tasks. This is likely intentional -- the original plan has phases without tasks, and remaining plans have tasks because they reference the Conductor's task decomposition. But the output-format.md examples should clarify this distinction.

- **"Checkpoint" ambiguity**: The Arranger uses "checkpoint" for journal entries ("Checkpoint: [Phase Name] -- [Topic]") and for conductor review sections ("Conductor checkpoint sections"). The Repetiteur uses "checkpoint" for journal entries and for context threshold checkpoints. The repertoire uses "checkpoint triggers" for journal writing moments. The term is overloaded but the context always disambiguates -- this is minor.

- **Entry field naming**: The Arranger journal uses `Decision` / `Rationale` / `Alternatives considered` / `Impact`. The repertoire journal-conventions use `Finding/Decision` / `Rationale` / `Alternatives considered` / `Impact` / `External input`. The `Finding/Decision` vs `Decision` difference is minor, but the Arranger omits `External input` as a standard field. The Repetiteur's consultation journals use the repertoire format.

---

### 7. Missing Contracts

**Status: Several gaps identified**

#### A. Shared reference directory naming
The Arranger design references a monolithic `shared-rules.md` file (Section 9):
```
references/
  shared-rules.md     -- verification rules, output format, sentinels, ...
```

The Repetiteur references `score-preparation/*.md` throughout (34+ occurrences).

The actual directory is `repertoire/` containing individual files (`priority-chain.md`, `verification-rules.md`, `journal-conventions.md`, `output-format.md`, plus examples).

All three naming schemes are different. This is already flagged in the Repetiteur's `docs/working/2026-02-27-readme-deep-dive.md` as known drift. It must be resolved before the Arranger is built.

#### B. Arranger-specific plan format extensions
If the Arranger's Tier 2 format is adopted (YAML frontmatter, `<sections>`, `<section>` tags, authority tags), the repertoire needs a new contract or an updated output-format.md that defines:
- YAML frontmatter schema
- `<sections>` index placement and format
- `<section>` tag conventions within sentinel-bounded areas
- Authority tag usage within phase and review sections
- How these extensions interact with the plan-index

#### C. Arranger journal format contract
If the Arranger's Tier 3 XML journal format is adopted, the repertoire's journal-conventions.md needs updating to either:
- Define the XML format as the standard for Arranger/Dramaturg journals
- Define both formats (simple markdown for consultations, XML for planning sessions)
- Specify how the Repetiteur's journal analysis should handle both

#### D. Task annotation markers in original plans
The output-format.md defines task annotation markers (`(REVISED)`, `(NEW)`, `(REMOVED)`) only for remaining plans. The Arranger's original plans do not use these. This is correct -- annotations describe changes from a prior version and have no meaning in an original plan. No new contract needed.

#### E. Dramaturg journal format
The Arranger design mentions consuming the Dramaturg's decision journal, and journal-conventions.md lists the Dramaturg as a consumer. But the Dramaturg journal format is not defined in the repertoire. The journal-conventions currently only show Arranger and consultation journal examples. If the Dramaturg uses its own format (it pre-dates the repertoire contracts), a Dramaturg journal format contract may be needed in the repertoire.

---

### 8. Existing Contract Updates Needed

#### A. output-format.md
Must be updated to reflect the final plan format the Arranger will produce. Either:
- Expand to include Tier 2 elements (frontmatter, `<sections>`, `<section>`, authority tags), OR
- Confirm the simpler format, in which case the Arranger design must be updated

#### B. journal-conventions.md
Must be updated to accommodate the Arranger's journal format:
- Define whether Arranger journals use the flat markdown format or the Tier 3 XML format
- If different formats coexist, define how the Repetiteur's journal analysis handles both
- Add `External input` to the Arranger entry template if the Arranger may record Gemini findings

#### C. verification-rules.md structural verification section
The section currently references `score-preparation/output-format.md` (line 161):
> "The canonical definitions for what these structural elements look like are in `score-preparation/output-format.md`."

This path needs updating to `repertoire/output-format.md`.

#### D. All repertoire files metadata
All repertoire files list `consumers: arranger, repetiteur` in their metadata. Some should also list `dramaturg` (journal-conventions already does) and `conductor` and `copyist` (output-format is consumed by the Conductor and Copyist, not just Arranger and Repetiteur).

---

## Categorized Issue List

### Critical

1. **Journal format mismatch between Arranger and repertoire** -- The Arranger designs Tier 3 XML journals; repertoire defines flat markdown journals; the Repetiteur parses the flat format. This will cause the Repetiteur's journal analysis to fail or produce degraded results on Arranger-produced journals.

2. **Output format Tier mismatch** -- The Arranger designs Tier 2 plans with YAML frontmatter, `<sections>`, `<section>` tags, and authority tags. The repertoire's output-format.md defines sentinel-marker-only plans. The Repetiteur produces the simpler format. The pipeline contract that "both use this format" is broken.

3. **Shared reference directory naming inconsistency** -- Three different names for the same concept: `shared-rules.md` (Arranger design), `score-preparation/` (Repetiteur skill), `repertoire/` (actual directory). Must be resolved to a single canonical name before the Arranger is built.

### Important

4. **Arranger phase sections lack task-level granularity that output-format examples show** -- The Arranger explicitly defers task decomposition to the Conductor, but the output-format.md examples include task headers within phase sections. The output-format should clarify whether original plans include task-level headers or only phase-level content, and whether remaining plans are the exception that includes tasks because they reference existing task decomposition.

5. **verification-rules.md internal path reference** -- References `score-preparation/output-format.md` which should be `repertoire/output-format.md` (or whatever the canonical path becomes).

6. **Arranger plan structural verification extends beyond shared contract** -- The Arranger's finalization checklist includes hybrid structure tag validation, YAML frontmatter validation, section tag validation, and large document marker tests not present in verification-rules.md's structural verification section. Either expand the shared contract or document these as Arranger-specific extensions.

7. **Repetiteur journal analysis not designed for authority-tag differentiated entries** -- If the Arranger's `<mandatory>` overrides carry stronger weight than `<core>` decisions, the Repetiteur's journal analysis should be aware of this signal. Currently, the journal analysis prompt templates make no reference to authority tags.

### Minor

8. **Entry field naming inconsistency** -- Arranger uses `Decision:` while repertoire uses `Finding/Decision:`. Arranger omits `External input:` field. Minor but could confuse automated parsing.

9. **Checkpoint term overload** -- "Checkpoint" means journal entries, conductor review sections, and context threshold checkpoints depending on context. Always disambiguated but could benefit from more specific terms.

10. **Repertoire consumer metadata is incomplete** -- `output-format.md` lists only `arranger, repetiteur` as consumers, but the Conductor and Copyist also consume this format.

### Observations

11. **The Arranger's authority tag system is genuinely useful** -- The distinction between `<mandatory>` (preserve verbatim), `<guidance>` (adapt freely), and `<core>` (standard content) gives the Copyist valuable signal for task instruction generation. If adopted, this enriches the pipeline. The question is whether the output-format contract should require it.

12. **The Arranger's journal authority tags are also useful** -- Distinguishing user overrides (`<mandatory>`) from standard decisions (`<core>`) from informational deviations (`<context>`) gives the Repetiteur's journal analysis stronger signals for the underlying constraint principle. If adopted, journal-conventions should define how these map to constraint strength.

13. **Example documents in repertoire are consistent with each other** -- `example-consultation-journal.md` and `example-remaining-plan.md` demonstrate the same scenario and format, providing a coherent reference. These should be updated in parallel with any format changes.

14. **The Repetiteur's `score-preparation` to `repertoire` migration is already identified** -- The `2026-02-27-readme-deep-dive.md` in Repetiteur's working docs flags this issue. Resolution should be coordinated with the Arranger naming decision.

---

## Recommendations

### Immediate (before building Arranger)

1. **Resolve directory naming** -- Pick one canonical name for the shared contracts directory. `repertoire/` is the current actual name and has a clear metaphor (the orchestra's shared musical library). Update all references in the Arranger design, Repetiteur skill files, and any other consumers.

2. **Resolve output format** -- Update `repertoire/output-format.md` to define the authoritative plan format. If adopting the Arranger's Tier 2 format with authority tags, the Repetiteur must be updated to produce remaining plans in the same format. If keeping the simpler format, the Arranger design must be simplified.

3. **Resolve journal format** -- Update `repertoire/journal-conventions.md` to accommodate the Arranger's journal format. Recommended approach: define the authority-tag differentiated format as valid for Arranger/Dramaturg journals (user-voice journals), with the flat markdown format remaining valid for consultation journals (autonomous journals). Update the Repetiteur's journal analysis prompt templates to handle both.

### Follow-up (during Arranger build)

4. **Update verification-rules.md** -- Fix the internal `score-preparation/output-format.md` path reference. Consider whether Arranger-specific structural verification items should be added to the shared contract or documented separately.

5. **Update Repetiteur journal analysis** -- If the Arranger's journal format includes authority tags, update the journal analysis prompt templates to leverage authority tag signals when extracting constraint strength.

6. **Update repertoire consumer metadata** -- Add missing consumers to each file's metadata block.

7. **Clarify task-level content in output-format.md** -- Add a note distinguishing original plans (phase-level content, no task headers) from remaining plans (task-level content with annotation markers).
