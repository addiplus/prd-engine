# Agent Execution Instructions Template v1.0

Standalone reference template for the "Agent Execution Instructions" PRD section.
The synthesizer reads this file and fills each sub-section using research report data.
This section guards against the documented failure cases that motivate this template:
stale field references, miswired caller sites, lost numeric units, dropped size dimensions.

---

## Template

```markdown
## Agent Execution Instructions

### Environment
- **Repo**: {absolute path to repo root}
- **Branch**: {branch name} (commit {short_hash})
- **Build command**: `{command}` -- MUST pass with 0 errors after all changes
- **Test command**: `{command}` -- run after build to verify behavior
```

**Fill instructions**: Pull repo path from Report 01 (Codebase Archaeologist). Pull branch/commit
from the current git state. Build/test commands come from Report 05 (Implementation Strategist)
or from the project's package.json / Makefile / CI config.

---

```markdown
### Scope
- **Files to modify** (numbered, absolute paths):
  1. {/path/to/file1.ts} -- {brief description of changes}
  2. {/path/to/file2.tsx} -- {brief description of changes}
- **Files to read** (reference only):
  - {/path/to/types.ts} -- for type definitions
  - {/path/to/constants.ts} -- for reference values
- **Files NOT to modify** (explicit exclusions):
  - {/path/to/unrelated.ts} -- {reason: owned by PRD-XX, unrelated, etc.}
```

**Fill instructions**: The numbered modify list comes from Report 01 (files identified for changes).
Read-only files come from Report 01 (Codebase Archaeologist — imports, type definitions, constants).
Explicit exclusions come from Report 04 (Risk & Edge Case Analyst guardrails) and Report 01 (adjacent files).

---

```markdown
### Prerequisites
- PRD-{N}: {what it changed, what field renames are in effect}
- PRD-{M}: {what it changed}
- None -- this PRD can be applied to a clean checkout.
```

**Fill instructions**: Check Report 03 for dependency chain. For each prerequisite PRD,
summarize the changes that affect the current PRD's target files. If no dependencies,
write "None -- this PRD can be applied to a clean checkout."

---

```markdown
### Execution Order
1. Read ALL target files before making any changes
2. Verify prerequisite state: grep for `{pattern}` in `{file}` -- must find `{expected}`
3. Apply changes in numbered order (Phase 1 -> Phase 2 -> ...)
4. Run build command after each phase
5. Run verification checklist after all phases
6. If build fails: read error message, check prerequisite chain, report
```

**Fill instructions**: Step 2 is critical -- the grep pattern must confirm that prerequisite
PRDs have been applied. Pull the pattern from the Field Rename Changelog or from a key
code change in the prerequisite PRD. Steps 3-6 are standard; customize only if the PRD
has a non-standard execution flow (e.g., database migration before code changes).

---

```markdown
### Field Rename Changelog
| Original Name | Current Name | Changed By |
|---------------|-------------|------------|
| {old_field}   | {new_field} | PRD-{N}    |
```

**Fill instructions**: Pull from Report 03 (Requirements Analyst) which should document
prior renames. Also check the codebase for any grep mismatches found by Report 01.
If no prior renames affect this PRD, write: "No prior renames affect this PRD."
This sub-section directly prevents the stale-field-reference failure pattern.

---

```markdown
### Fallback Strategy
- Line numbers don't match -> grep for content patterns (provided in each Change section)
- Field name missing -> check the Field Rename Changelog above
- Build fails after changes -> revert last change, report which change broke the build
- Function signature changed -> read current version, adapt the change to current structure
```

**Fill instructions**: The four fallback rules above are standard. Add project-specific
fallbacks if Report 03 or Report 05 identified unusual risks (e.g., "If the Notion API
returns a different schema, fall back to the cached version in data/cache/").

---

```markdown
### Checkpoint Protocol
- After each phase: write progress to `{checkpoint_path}/CHECKPOINT.md`
- Format: phase completed, files modified, build status, issues found
- If context budget >60%: write insurance checkpoint with full remaining plan and STOP
```

**Fill instructions**: Set the checkpoint path to `.prd-engine/{slug}/checkpoints/` or
the project's existing checkpoint directory pattern. Include this sub-section for any PRD
with more than 2 phases or estimated time over 30 minutes. For single-phase PRDs, write:
"Single-phase PRD -- no intermediate checkpoints required."

---

```markdown
### Exit Criteria
- [ ] Build passes with zero errors
- [ ] All verification checklist items in Section 12 confirmed
- [ ] No stale references to renamed fields (grep confirms)
- [ ] No changes outside the allowed file list in Scope above
- [ ] No "What NOT to Change" items were modified
```

**Fill instructions**: The five exit criteria above are standard and should always be
included. Add project-specific criteria if Report 03 or Report 05 identified additional
verification needs (e.g., "[ ] PDF renders without page-break splits across table rows").

---

## Usage Notes

- This template is read by the synthesizer agent during Phase 2 of the PRD Engine pipeline.
- The prd-template.md file (Section 3) already contains a copy of this structure inline.
  This standalone file serves as the authoritative reference with fill instructions.
- All 8 sub-sections (Environment, Scope, Prerequisites, Execution Order, Field Rename
  Changelog, Fallback Strategy, Checkpoint Protocol, Exit Criteria) are MANDATORY.
- An Agent Execution Instructions section missing any sub-section triggers a BLOCK verdict
  from the quality reviewer.
