# PRD Template v1.0

This is the canonical template for the PRD Engine synthesizer agent. Fill every section.
Do NOT skip sections -- write "None identified" if a section has no applicable content.
Sections marked MANDATORY appear in every PRD. Sections marked CONDITIONAL are included
when their trigger condition is met.

Target length: scaled by complexity — 200 + (REQ_count × 60), capped at 600. Warn at 500. Force-split at 600 along phase boundaries.

---

## TEMPLATE START

````markdown
# PRD-{NUMBER}: {TITLE}

**Status**: DRAFT | READY | FINAL
**Date**: {YYYY-MM-DD}
**Author**: PRD Engine (Synthesized from {N} research reports)
**Priority**: {P0|P1|P2} -- {DOMAIN_TAG}
**Severity**: {CRITICAL|HIGH|MEDIUM|LOW}
**Bug IDs Fixed**: {BUG-01, BUG-02 | "N/A -- new feature"}
**Estimated Time**: {N hours}
**Dependencies**: {PRD-XX, PRD-YY | "None"}
**Files Touched**: {COUNT} -- {file1.ts, file2.tsx, ...}
**Size Budget**: {LINE_COUNT} lines

<!--
  METADATA RULES:
  - File count MUST match the distinct files listed in the body (Sections 9+11).
    This is a contract -- agents treat this number literally.
  - Bug IDs link to a central tracker or swarm finding ID.
  - Estimated Time comes from the implementation strategist report.
  - Size Budget = 200 + (REQ_count × 60), capped at 600. Warn at 500. Force-split at 600.
-->

---

## 1. Problem Statement                                            [MANDATORY]

### What This Addresses
<!-- 3-5 sentences. Choose the sub-pattern that fits:
     - INCIDENT-DRIVEN (bug): What happened, when, to whom. Business impact bullets.
     - DEFECT-DRIVEN (data bug): Current vs. expected behavior table. CEO/stakeholder ref.
     - FEATURE-DRIVEN (new feature): Numbered concrete problems. "The result:" summary.
     Always include specific dates, users, data values -- never generalize. -->

{Description of the problem or feature need with concrete details}

### Business Impact
- {Revenue / efficiency / user experience / compliance consequence}
- {Second impact}
- {Third impact}

### Discovery Source
<!-- Traceable origin: stakeholder annotation ID, user report date, bug ID, or research finding.
     Example: "Stakeholder annotation #2 on quarterly review document"
     Example: "User report from alice@example.com on 2026-03-05" -->

{Where this request originated -- with reference ID}

---

## 2. Root Cause Analysis / Background                             [MANDATORY*]

<!-- * MANDATORY for bug-fix PRDs. For greenfield features, rename this section to
     "Background / Technical Landscape" and describe the current state of the codebase
     in the area being modified. -->

### Failure Mode 1: {NAME}
<!-- Code walkthrough with file:line references and execution trace.
     Show the exact path from trigger to symptom. -->

{Code walkthrough: file_path:line -> function -> consequence}

### Failure Mode 2: {NAME}
<!-- Include if there are multiple independent failure modes. -->

{Second failure mode walkthrough, if applicable}

### Combined Effect
<!-- How independent failures compound into the user-visible bug. -->

{Explanation of how failures interact}

---

## 3. Agent Execution Instructions                                 [MANDATORY]

<!-- THIS IS THE KEY INNOVATION. This section guards against the documented failure
     cases the engine targets: stale field references, missed caller-site updates,
     dropped numeric units, off-by-magnitude size errors, type drift between layers. -->

### Environment
- **Repo**: {absolute path to repo root}
- **Branch**: {branch name} (commit {short_hash})
- **Build command**: `{command}` -- MUST pass with 0 errors after all changes
- **Test command**: `{command}` -- run after build to verify behavior

### Scope
- **Files to modify** (numbered, absolute paths):
  1. {/path/to/file1.ts} -- {brief description of changes}
  2. {/path/to/file2.tsx} -- {brief description of changes}
- **Files to read** (reference only):
  - {/path/to/types.ts} -- for type definitions
  - {/path/to/constants.ts} -- for reference values
- **Files NOT to modify** (explicit exclusions):
  - {/path/to/unrelated.ts} -- {reason: owned by PRD-XX, unrelated, etc.}

### Prerequisites
<!-- State every prior PRD that must be applied, and what it changed.
     If none, write "None -- this PRD can be applied to a clean checkout." -->
- PRD-{N}: {what it changed, what field renames are in effect}

### Execution Order
1. Read ALL target files before making any changes
2. Verify prerequisite state: grep for `{pattern}` in `{file}` -- must find `{expected}`
3. Apply changes in numbered order (Phase 1 -> Phase 2 -> ...)
4. Run build command after each phase
5. Run verification checklist after all phases
6. If build fails: read error message, check prerequisite chain, report

### Field Rename Changelog
<!-- Track all renames from prior PRDs that affect this area of the codebase.
     If none, write "No prior renames affect this PRD." -->

| Original Name | Current Name | Changed By |
|---------------|-------------|------------|
| {old_field}   | {new_field} | PRD-{N}    |

### Fallback Strategy
<!-- Decision tree for when the primary approach fails. -->
- Line numbers don't match -> grep for content patterns (provided in each Change section)
- Field name missing -> check the Field Rename Changelog above
- Build fails after changes -> revert last change, report which change broke the build
- Function signature changed -> read current version, adapt the change to current structure

### Checkpoint Protocol
<!-- Required for PRDs with >2 phases or estimated time >30 min. -->
- After each phase: write progress to `{checkpoint_path}/CHECKPOINT.md`
- Format: phase completed, files modified, build status, issues found
- If context budget >60%: write insurance checkpoint with full remaining plan and STOP

### Exit Criteria
- [ ] Build passes with zero errors
- [ ] All verification checklist items in Section 11 confirmed
- [ ] No stale references to renamed fields (grep confirms)
- [ ] No changes outside the allowed file list in Scope above
- [ ] No "What NOT to Change" items were modified

---

## 4. Goals                                                        [MANDATORY]

<!-- Numbered goals with measurable outcomes. Each goal maps to one or more REQs. -->

- **G1**: {Measurable outcome -- e.g., "Eliminate all row splits across page boundaries"}
- **G2**: {Measurable outcome}
- **G3**: {Measurable outcome}

---

## 5. Non-Goals                                                    [MANDATORY]

<!-- Explicit scope boundaries. What we are NOT doing in this PRD and why. -->

- {Feature/behavior explicitly excluded}: {one-sentence reason}
- {Second non-goal}: {reason}

---

## 6. Ground Truth / Data Tables                              [CONDITIONAL]

<!-- MANDATORY when the PRD involves configuration, business rules,
     or any numerical acceptance criteria. Inline the canonical data here.
     NEVER reference external files -- all data must be self-contained.
     Every numerical AC in Section 7 must trace to a value in these tables. -->

| {Column 1} | {Column 2} | {Column 3} | {Column 4} |
|-------------|-------------|-------------|-------------|
| {value}     | {value}     | {value}     | {value}     |

---

## 7. Requirements                                                 [MANDATORY]

<!-- Format: REQ-{PRD#}.{NN} with zero-padded sequence (e.g., REQ-13.01, REQ-13.02).
     Group by priority for PRDs with 10+ requirements:
       Category A: P0 (must-have)
       Category B: P1 (should-have)

     RULES:
     - Every REQ gets BEFORE/AFTER code blocks (the #1 predictor of success).
     - ACs must use specific values ("$5.00"), NEVER adjectives ("correct price").
     - Use Given/When/Then for behavioral criteria.
     - Include at least one NEGATIVE AC per requirement (edge case / failure path).
     - Required steps use MUST or REQUIRED -- never "if needed" or "optionally". -->

### REQ-{PRD#}.01: {Imperative Title}

**Description**: {One clear sentence}
**Where**: `{file_path}` lines {N-M}
**Rationale**: {Links to root cause failure mode, user story, or goal G1/G2/G3}

**BEFORE** (current code, lines {N-M}):
```{language}
// Include 3-5 lines of surrounding context for location anchoring
{EXACT current code from the codebase}
```

**AFTER** (required change):
```{language}
// Same surrounding context with the modification applied
{EXACT replacement code}
```

**Search Patterns** (if code has shifted from prior PRDs):
<!-- 3-5 alternative grep patterns to locate this code region -->
- `grep -n "{distinctive_string_1}" {file_path}`
- `grep -n "{distinctive_string_2}" {file_path}`
- `grep -n "{distinctive_string_3}" {file_path}`

**Acceptance Criteria**:
- [ ] AC-01: {Testable assertion with specific expected value}
- [ ] AC-02: {Second testable assertion}
- [ ] AC-03: {NEGATIVE case -- e.g., "Unknown input key logs warning, uses fallback (no crash)"}

---

### REQ-{PRD#}.02: {Imperative Title}

{Same structure as REQ-{PRD#}.01. Repeat for every requirement.}

---

## 8. Technical Approach                                      [CONDITIONAL]

<!-- MANDATORY for new features or architectural changes. Optional for simple bug fixes
     where the BEFORE/AFTER in Section 7 is self-explanatory.
     Include: architecture decisions, library choices, data model changes.
     For any decision, document the rationale -- prevents future "why was this done?" -->

### Architecture Decision: {Decision Title}
**Chosen approach**: {What was decided}
**Rationale**: {Why this approach, not the alternatives}
**Rejected alternatives**:
- {Alternative A}: {Why rejected}
- {Alternative B}: {Why rejected}

---

## 8a. Idempotency Strategy                                       [CONDITIONAL]

<!-- INCLUDE when the feature creates resources via external APIs where the
     creation response can be lost (network errors, timeouts, rate limits).
     SKIP for pure local operations or APIs with built-in idempotency (Stripe, AWS). -->

| Component | Value |
|-----------|-------|
| Key format | `{field1}\|{field2}\|{field3}` |
| Key properties | Deterministic, sorted inputs, escaped delimiters, no random/time components |
| Embedding mechanism | {Where the key is stored in the artifact — header, metadata field, tag} |
| Recovery query | {How to find artifacts by key after creation — API query, list+filter} |
| Collision handling | {What happens if two artifacts share a key — skip, log, error} |
| Circuit breaker | {Max time/calls for recovery query — prevents runaway API usage} |

---

## 8b. Backward Compatibility Strategy                             [CONDITIONAL]

<!-- INCLUDE when the feature changes how existing artifacts are identified,
     detected, matched, or processed. SKIP for greenfield features with no
     existing artifacts in the wild. -->

| Component | Value |
|-----------|-------|
| Existing artifacts affected | {What artifacts lack the new identifier — count, age, location} |
| Primary detection | {New mechanism — header match, schema field, API filter} |
| Fallback detection | {Old mechanism preserved for transition — string match, legacy format} |
| Fallback signal | {How operators know fallback was used — WARNING log, metric, flag} |
| Removal criteria | {When the fallback can be safely removed — all old artifacts expired/migrated} |

---

## 9. Files to Modify                                              [MANDATORY]

<!-- This table is the contract. The count of distinct files here MUST match
     the Files Touched count in the header metadata. -->

| # | File | Action | Change Description | Lines | REQ |
|---|------|--------|--------------------|-------|-----|
| 1 | `{file_path}` | EDIT | {what changes} | {N-M} | REQ-{PRD#}.01 |
| 2 | `{file_path}` | EDIT | {what changes} | {N-M} | REQ-{PRD#}.02 |

---

## 10. What NOT to Change                                          [MANDATORY]

<!-- Minimum 10 items. This section prevents the #3 anti-pattern (implicit scope
     boundaries) which causes 25% of implementation failures.
     List files, functions, and components adjacent to the change targets.
     Include the REASON for each -- agents need to know WHY, not just WHAT. -->

1. Do NOT modify `{function/file}` -- {reason, e.g., "owned by PRD-XX"}
2. Do NOT modify `{function/file}` -- {reason, e.g., "unrelated to this change"}
3. Do NOT modify `{function/file}` -- {reason}
4. Do NOT modify `{function/file}` -- {reason}
5. Do NOT modify `{function/file}` -- {reason}
6. Do NOT modify `{function/file}` -- {reason}
7. Do NOT modify `{function/file}` -- {reason}
8. Do NOT modify `{function/file}` -- {reason}
9. Do NOT modify `{function/file}` -- {reason}
10. Do NOT modify `{function/file}` -- {reason}

---

## 11. Implementation Phases                                       [MANDATORY]

<!-- Each phase: 1-2 files, 5-15 minutes of agent work, produces a testable artifact.
     Phase N+1 receives EXPLICIT named outputs from Phase N.
     No partial/broken states between phases -- the system must build after each phase.
     If a phase changes a function signature, it MUST also update all call sites. -->

### Phase 1: {Name} (REQ-{PRD#}.01, REQ-{PRD#}.02)
**Files**: `{path_1}` (EDIT), `{path_2}` (EDIT)
**Estimated Time**: {N} minutes
**Complexity**: {S|M|L} -- S=find/replace, M=logic changes, L=new architecture
**Dependencies**: None

#### Changes
1. In `{file_1}`, modify `{function/region}`:
   - BEFORE: {brief or reference to Section 7 REQ}
   - AFTER: {brief or reference to Section 7 REQ}

2. In `{file_2}`, add `{function/region}`:
   - {Description of addition}

#### Phase 1 Verification
- Build: `{build_command}` -- must pass with 0 errors
- Test: `{specific test or manual check}`
- Confirm: {observable outcome with specific value}

### Phase 2: {Name} (REQ-{PRD#}.03)
**Files**: `{path_3}` (EDIT)
**Estimated Time**: {N} minutes
**Dependencies**: Phase 1 (uses `{named_output}` from Phase 1)

#### Changes
{Numbered changes}

#### Phase 2 Verification
{Build + test + confirm}

---

## 12. Verification Checklist                                      [MANDATORY]

<!-- Three layers of verification. Use specific expected values throughout. -->

### Pre-Implementation Checks
- [ ] Verify build passes BEFORE making any changes (baseline)
- [ ] Confirm target files exist at expected paths
- [ ] Verify prerequisite PRDs have been applied (grep for expected patterns)

### Per-Requirement Tests

#### REQ-{PRD#}.01
| Test | Method | Expected Result |
|------|--------|-----------------|
| {happy path} | {steps to reproduce} | {specific expected output} |
| {edge case} | {steps} | {specific expected output} |
| {negative case} | {steps} | {specific expected output -- e.g., "warning logged, no crash"} |

#### REQ-{PRD#}.02
| Test | Method | Expected Result |
|------|--------|-----------------|
| {test} | {method} | {result} |

### Regression Tests
| Test | Expected Result |
|------|-----------------|
| {existing feature A} | No impact -- unchanged behavior |
| {existing feature B} | No impact -- unchanged behavior |
| {export function C} | No impact -- separate function |

### Manual Test Checklist
- [ ] {Step-by-step scenario 1 with expected visual/behavioral outcome}
- [ ] {Step-by-step scenario 2}
- [ ] {Step-by-step scenario 3}

---

## 13. Risk Assessment                                        [CONDITIONAL]

<!-- Included at standard+ intensity or when risks are non-trivial.
     Separate "risks of the fix" from "risks of NOT fixing" -- this
     is persuasive for stakeholder prioritization. -->

### Risks of This Change

| Risk | Severity | Likelihood | Mitigation |
|------|----------|------------|------------|
| {risk description} | LOW/MEDIUM/HIGH | LOW/MEDIUM/HIGH | {specific mitigation} |

### Risks If NOT Implemented

| Risk | Impact |
|------|--------|
| {risk description} | {business/user consequence} |

---

## 14. Out of Scope                                                [MANDATORY]

<!-- Every item explains WHY it is out of scope and gets a disposition.
     This prevents scope creep during implementation. -->

### Deferred to Future PRD
- {Item}: {Why deferred, target PRD ID if known (e.g., "PRD-{FUTURE}")}

### Separate Dependencies
- {Item}: {What external dependency blocks this work}

### Future Enhancement
- {Item}: {Why this is not needed now but may be later}

---

## 15. Open Questions                                         [CONDITIONAL]

<!-- Unresolved decisions that need stakeholder input before implementation.
     If none, write "None -- all decisions resolved during research phase." -->

- [ ] {Question requiring stakeholder decision}: {context for the decision}

---

## 16. Summary                                                     [MANDATORY]

| Metric | Value |
|--------|-------|
| Requirements | {N} (REQ-{PRD#}.01 through REQ-{PRD#}.{NN}) |
| Priority | {P0/P1/P2} |
| Estimated effort | {N} hours |
| Files touched | {N} |
| Lines added/modified | ~{N} |
| Implementation phases | {N} |
| Dependencies | {PRD-XX, PRD-YY or "None"} |
| New runtime dependencies | {package names or "None"} |

---

*PRD-{NUMBER} generated {YYYY-MM-DD} by PRD Engine v1.0*
*Source: {N} research reports ({report_names})*
````

## TEMPLATE END

---

## Synthesizer Instructions

When filling this template, follow these rules in priority order:

### Rule 1: Every REQ Gets BEFORE/AFTER Code Blocks
This is the single strongest predictor of implementation success (consensus across
all four analysis reports). Pull BEFORE code from the Codebase Archaeologist report
(Section 5). Write AFTER code by synthesizing the Requirements Analyst and
Implementation Strategist findings.

### Rule 2: Every REQ Traces to a Source
Link each requirement to: a stakeholder annotation ID, user report, bug ID, research finding,
or goal (G1/G2/G3). Untraceable requirements signal scope creep.

### Rule 3: "What NOT to Change" Has 10+ Items
Pull from the Risk & Edge Case Analyst report (Section 7). If fewer than 10 items,
add entries by listing functions and components adjacent to the change targets
from the Codebase Archaeologist report.

### Rule 4: Verification Uses Exact Expected Values
Every test in Section 12 must have a specific expected output. NEVER write "should
work correctly" or "looks right." Pull values from Ground Truth tables (Section 6)
or Acceptance Criteria (Section 7).

### Rule 5: Keep Within Size Budget
Size budget = 200 + (REQ_count × 60), capped at 600. If the PRD exceeds its
budget, split into two PRDs along phase boundaries from the Implementation Strategist report. Write both. Hard reject at 600 lines.

### Rule 6: Resolve Conflicts Between Reports
If the Domain Expert recommends an approach that conflicts with the Codebase
Archaeologist's findings about existing patterns, favor existing patterns unless
there is a strong documented reason to deviate. Record the decision in Section 8
(Technical Approach / Architecture Decisions).

### Rule 7: Fill Every Section
Write "None identified" rather than omitting a section. The quality reviewer
checks for section completeness.

### Rule 8: Agent Execution Instructions Are Non-Negotiable
Section 3 must be complete with Environment, Scope, Prerequisites, Execution Order,
Field Rename Changelog, Fallback Strategy, and Exit Criteria. This section alone
prevents the documented failure cases this engine guards against.

### Rule 9: No External References
The PRD must be fully self-contained. No "see previous session," no "the file we
edited earlier," no references to external spreadsheets or documents. All data
inline. An agent must be able to execute this PRD with zero prior context.

### Rule 10: No Hedging Language for Required Steps
Scan the PRD for "if needed," "consider," "optionally," "you may want to" in
mandatory change descriptions. Replace with "MUST," "REQUIRED," or remove the
conditional framing entirely.

---

## Validation Checklist (Post-Generation)

The synthesizer (or quality gate agent) must confirm:

```
[ ] RULE-01: metadata.files_touched == count(distinct files in Sections 9+11)
[ ] RULE-02: every REQ in Section 7 has BEFORE and AFTER code blocks
[ ] RULE-03: every AC contains a numeric/string literal (not just adjectives)
[ ] RULE-04: Section 10 "What NOT to Change" has >= 10 items
[ ] RULE-05: total lines <= 600 (warn at 500, budget = 200 + REQ_count * 60)
[ ] RULE-06: no "if needed" or "consider" in Change descriptions
[ ] RULE-07: no "see previous" or "earlier session" anywhere
[ ] RULE-08: every Dependency references a PRD-{ID} or "None"
[ ] RULE-09: Section 3 "Agent Execution Instructions" is complete
[ ] RULE-10: Section 14 "Out of Scope" exists and has items
[ ] RULE-11: metadata.size_budget is stated
[ ] RULE-12: all file paths are absolute or project-relative
[ ] RULE-13: Field Rename Changelog present if Dependencies is non-empty
[ ] RULE-14: Exit Criteria checklist present in Section 3
[ ] RULE-15: every Implementation Phase targets <= 2 files
[ ] RULE-16: PRD is fully self-contained (zero prior context needed)
```
