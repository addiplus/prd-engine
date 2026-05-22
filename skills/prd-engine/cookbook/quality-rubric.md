# PRD Quality Rubric v1.0

Loaded by the PRD Reviewer agent during the Quality Gate phase.
Also available to the main session for score-to-action decisions.

---

## 1. Five-Dimension Scoring (100 Points Total)

### Dimension 1: Completeness (25 points)

| Criterion | Points | How to Check |
|-----------|--------|--------------|
| All mandatory template sections present | 5 | Verify headers: Problem Statement, Requirements, Files to Modify, Code Changes, What NOT to Change, Implementation Phases, Verification, Out of Scope |
| No TBD/TODO/placeholder text | 5 | Grep for TBD, TODO, placeholder, FIXME, TBC |
| Out-of-Scope section has items with dispositions | 3 | Each item explains WHY deferred |
| Reference data inline (if data-centric PRD) | 3 | Truth tables present when PRD involves pricing, config, or business rules |
| Metadata block fully populated | 3 | Status, Date, Priority, Files Touched, Dependencies all present |
| Agent Execution Instructions present and complete | 3 | Environment, Scope, Fallback, Exit Criteria sub-sections exist |
| Field Rename Changelog populated (if applicable) | 3 | Non-empty when prior PRDs modified same codebase area |

### Dimension 2: Specificity (25 points)

| Criterion | Points | How to Check |
|-----------|--------|--------------|
| All ACs have specific expected values | 8 | No "works correctly", "looks good", "is correct" without a concrete value |
| BEFORE/AFTER code blocks for every REQ | 8 | Count REQs, count code block pairs -- must match |
| File paths are exact (repo-relative or absolute) | 5 | No "the main file", "the relevant component" |
| Line numbers provided for code references | 4 | At least 80% of code refs include line numbers |

### Dimension 3: Implementability (20 points)

| Criterion | Points | How to Check |
|-----------|--------|--------------|
| Phases target 1-2 files each | 5 | No single phase touches 3+ files |
| Each phase has a verification step | 5 | Every phase ends with how to verify it worked |
| Build command specified | 3 | Agent Execution Instructions has a build command |
| Fallback strategy exists | 4 | Non-empty instructions for when line numbers drift or fields are renamed |
| Estimated time per phase | 3 | Every phase has a duration estimate |

### Dimension 4: Consistency (15 points)

| Criterion | Points | How to Check |
|-----------|--------|--------------|
| REQ numbering is sequential and unique | 4 | No duplicate REQ IDs, no gaps |
| File list in metadata matches files in phases | 4 | Cross-reference header file count with actual files modified in body |
| ACs in Requirements match tests in Verification | 4 | Every AC has a corresponding verification entry |
| Terminology is uniform throughout | 3 | Same names for same concepts -- no field name or abbreviation shifts |

### Dimension 5: Scope Clarity (15 points)

| Criterion | Points | How to Check |
|-----------|--------|--------------|
| "What NOT to Change" has 10+ items | 5 | Count items in section |
| Out-of-scope items explain WHY deferred | 4 | Each item has a rationale or future PRD reference |
| No scope-creep signals | 3 | Grep for "also consider", "we might want", "could also", "optionally" |
| Dependencies are specific PRD IDs, not prose | 3 | Metadata deps are "PRD-XX" or "None", never "after previous work" |

---

## 2. Tier Definitions

| Tier | Score | Action | Historical First-Pass Success |
|------|-------|--------|-------------------------------|
| **A-Tier** | 80-100 | **DELIVER** as-is | >90% |
| **B-Tier** | 60-79 | **REVISE** -- apply reviewer's top 3 suggestions, re-synthesize | ~75% |
| **C-Tier** | 40-59 | **REWORK** -- spawn additional research agents for weak areas | ~50% |
| **Reject** | 0-39 | **ESCALATE** -- fundamental issues, ask user for clarification | N/A |

**Override rule**: If ANY BLOCK-level anti-pattern is detected (see Section 3), the verdict is capped at REVISE regardless of score.

---

## 3. Anti-Pattern Registry (Ranked by Severity)

Severity is a composite of frequency (from codebase history) and impact (from failure case studies).

| Rank | Anti-Pattern | Severity | Freq | Prevention Rule |
|------|-------------|----------|------|----------------|
| 1 | **Missing caller-site updates** | CRITICAL | 30% | Scan all call sites when a function signature changes. Include every caller as a numbered Change or document in Conflict Notes. |
| 2 | **Missing BEFORE/AFTER code blocks** | CRITICAL | -- | Every Change must have literal code blocks. Reject prose-only changes at validation. |
| 3 | **Implicit scope boundaries** | HIGH | 25% | Auto-generate "What NOT to Change" from adjacency analysis. Minimum 10 items enforced. |
| 4 | **Stale search patterns / field names** | HIGH | 15% | Include Field Rename Changelog. Provide 3-5 alternative search patterns per change. |
| 5 | **Optional language for required steps** | HIGH | 10% | Scan for "if needed", "check if", "consider" in mandatory changes. Rephrase as "REQUIRED." |
| 6 | **"See previous session" references** | HIGH | -- | Self-containment check: no external references, no "the file we edited earlier." |
| 7 | **Hypothetical root causes** | MEDIUM | -- | Must confirm root cause before specifying fix. Unconfirmed causes produce investigation tasks, not fix PRDs. |
| 8 | **Option A / Option B without commitment** | MEDIUM | -- | PRD author makes the architectural decision. Document rationale. Agent executes one path. |
| 9 | **PRD too long (>600 lines)** | MEDIUM | 5% | Hard split at 600 lines. Warn at 500. Budget = 200 + (REQ_count * 60), capped at 600. |
| 10 | **Overly complex single changes** | MEDIUM | 15% | Split multi-region changes into sub-steps (Change 4a, 4b). Each sub-step targets one contiguous code region. |
| 11 | **File count inaccuracy** | MEDIUM | -- | Auto-count files from Changes section and reconcile with header. Mismatch is a blocking error. |
| 12 | **Hardcoded values in prose** | LOW | -- | Numeric values go in Ground Truth tables, never in paragraph text. ACs reference the table. |
| 13 | **No delivery order for batches** | LOW | -- | Sprint batches MUST include cross-PRD conflict matrix with file-touch map. |
| 14 | **Missing edge cases (dark mode, empty state)** | LOW | -- | Edge Case Hunter agent specifically searches for dark mode, empty state, and boundary conditions. |
| 15 | **No Out of Scope section** | LOW | -- | Out of Scope is MANDATORY. Missing it allows scope creep. Each item gets a disposition. |

### Anti-Pattern Verdict Mapping

| Anti-Pattern | If Detected | Verdict Impact |
|--------------|-------------|----------------|
| Ranks 1-2 (CRITICAL) | BLOCK | Cannot be DELIVER; capped at REVISE |
| Ranks 3-6 (HIGH) | BLOCK | Cannot be DELIVER; capped at REVISE |
| Ranks 7-11 (MEDIUM) | CONCERN | Score reduced; can still DELIVER if score >= 80 |
| Ranks 12-15 (LOW) | CONCERN | Noted in review; no score cap |

---

## 4. Score-to-Action Mapping

| Score Range | Action | What Happens Next |
|-------------|--------|-------------------|
| **80-100** | DELIVER | Copy PRD_DRAFT.md to PRD.md. Report quality score and summary. |
| **60-79** | REVISE | Read reviewer's top 3 suggestions. Apply surgical edits to draft. Write PRD.md. Max 1 revision cycle. |
| **40-59** | REWORK | Notify user: "PRD scored {X}/100. Top issues: {list}." User decides: re-synthesize or deliver as-is. |
| **0-39** | ESCALATE | "PRD has fundamental gaps. Review research at `.prd-engine/{slug}/research/`." User must provide clarification. |

---

## 5. Golden Rules (Validation Checklist)

A PRD that violates any of these 16 rules cannot score A-Tier:

```
RULE-01: metadata.files_touched == count(distinct files in Changes)
RULE-02: every Change section contains BEFORE and AFTER code blocks
RULE-03: every AC contains a numeric/string literal (not just adjectives)
RULE-04: section "What NOT to Change" exists and has >= 10 items
RULE-05: total_lines <= 600 (warn at 500, budget = 200 + REQ_count * 60)
RULE-06: no occurrence of "if needed" or "consider" in Change sections
RULE-07: no occurrence of "see previous" or "earlier session"
RULE-08: every Dependency references a PRD-{ID}
RULE-09: section "Agent Execution Instructions" exists with all 8 sub-sections
RULE-10: section "Out of Scope" exists with dispositions
RULE-11: metadata.size_budget is stated
RULE-12: all file paths are absolute or project-relative
RULE-13: Field Rename Changelog present if Dependencies is non-empty
RULE-14: Exit Criteria checklist present in Agent Execution Instructions
RULE-15: every Implementation Phase targets <= 2 files
RULE-16: PRD is fully self-contained (zero prior context needed)
```
