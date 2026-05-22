# Research Protocols — PRD Engine Phase 2

## 1. Overview

The research phase spawns 3-8 agents in parallel to investigate a feature request from orthogonal perspectives. Each agent writes a numbered report to the namespace directory. The main session never reads full reports — it reads only the verdict lines (first 3-5 lines) of each report. The synthesis agent (Phase 3) is the sole consumer of full report content.

**Agent count by intensity tier:**

| Tier | Agents Spawned | IDs |
|------|---------------|-----|
| Quick | 3 | 1, 3, 5 |
| Standard | 5 (DEFAULT) | 1, 2, 3, 4, 5 |
| Deep | 6 | 1, 1b (doubled), 2, 3, 4, 5 |
| Exhaustive | 8 | 1, 1b, 2, 3, 4, 4b (doubled), 5, 5b (doubled) |

**How the main session uses this file:** Read it once at the start of Phase 2. For each agent in the roster, copy the prompt template from Section 3, replace the placeholders, and inject the result into the Task tool's `prompt` parameter. Do NOT tell agents to read this file.

---

## 2. Spawning Protocol

Follow these steps exactly:

### Step 1: Create namespace directory
```bash
mkdir -p {NAMESPACE}/research/
```

### Step 2: Construct prompts
For each agent in the intensity-tier roster (Section 4), copy the prompt template from Section 3. Replace these placeholders:
- `{FEATURE_DESCRIPTION}` — the user's original feature request, plus any clarification answers from Phase 1
- `{NAMESPACE}` — the `.prd-engine/{slug}` directory (e.g., `.prd-engine/dark-mode-dashboard`)
- `{CLARIFICATION_ANSWERS}` — answers from the Socratic clarification phase, or "No clarification was requested." if skipped
- `{WORKING_DIR}` — the project's working directory (absolute path)

After filling placeholders, **append this circuit breaker** to the end of every research agent prompt:

```
TOOL CALL LIMIT: Do not exceed 50 tool calls. If you reach 40 tool calls, begin writing your report with findings so far. If you reach 50, stop all investigation and write your report immediately with whatever you have. A partial report is infinitely more valuable than no report from a hung agent.
```

### Step 3: Spawn all research agents in parallel
```
For each agent:
  Task tool call:
    description: "PRD Research: {role name} for {feature slug}"
    prompt: {the filled prompt template}
    model: opus[1m]
    run_in_background: true
```

### Step 4: Monitor with Timeout (Watchdog)

Track each agent's start time, expected output file, and `last_file_size` (init 0). Enter a polling loop with **30-second intervals**:

**Per-agent checks (each poll cycle):**

1. **Completion**: Notification received? → mark complete, read first 5 lines of report
2. **File growth**: Output file size increased since last poll? → agent alive, update `last_file_size`, clear warnings. Do NOT timeout an agent whose file is growing.
3. **Auto-capture**: No output file? Check `.claude/agent-outputs/{session_id}/` — if found (>50 lines), copy to expected path and mark complete
4. **Per-agent timeout** (only when file NOT growing):
   - **Agent 02 (Domain Expert, WebSearch)**: 15 min hard timeout
   - **Agents 01, 03, 04, 05 (code-only)**: 10 min hard timeout

**Wave timeout**: **15 minutes** total for entire research phase. When wave timeout fires, abandon all remaining agents regardless of individual timeout status.

**Graduated response:**
- **50% of timeout** (no file growth): Log warning `"[WATCHDOG] Agent {N} ({role}) at {X}min, {timeout}min limit"`
- **100% of timeout, no output**: Mark ABANDONED, write stub report (see Step 4b)
- **100% of timeout, partial output** (file exists, size > 0): Extend by 50% once. Append watchdog footer to partial output, do NOT overwrite.
- **File growing at timeout**: Do NOT abandon — agent is working. Reset warning state.

Report progress: `"Research: {N}/{total} agents complete ({M} abandoned)..."`

### Step 4a: Quorum Check

After each agent completes OR times out:

| Agent | Criticality |
|-------|-------------|
| 01 Codebase Archaeologist | **MANDATORY** |
| 02 Domain Expert Researcher | OPTIONAL |
| 03 Requirements Analyst | **MANDATORY** |
| 04 Risk & Edge Case Analyst | CONDITIONAL (mandatory at standard+, optional at quick) |
| 05 Implementation Strategist | **MANDATORY** |

**Quorum met** = all MANDATORY agents complete. After quorum, start a 2-minute grace period for remaining optional agents, then proceed to Step 5.

**Mandatory agent timeout** = ABORT pipeline and notify user.

### Step 4b: Stub Report Writer

When an agent is abandoned (times out with no output), write a stub to its expected path:

```markdown
# {Agent Role}: {feature}
**Verdict**: TIMEOUT
**Confidence**: 0.0
**Duration**: {elapsed_min} minutes (hard timeout at {timeout_min} minutes)

## Summary
This report was not produced. The research agent exceeded the {timeout_min}-minute
timeout and was abandoned. The synthesis agent should proceed without this report.

## Possible Causes
- WebSearch/WebFetch hanging on unresponsive URL
- Agent caught in exploration loop (too many Glob/Grep cycles)
- Platform-level agent process hang
```

Also append a JSON record to `.swarm/swarm_failures.json` for pattern detection:
```json
{
  "timestamp": "{ISO-8601}",
  "context": "Agent timeout: {agent_role} during PRD-{N} research",
  "root_cause": "agent-timeout",
  "severity": "medium",
  "tags": ["timeout", "prd-engine", "{agent_role}"]
}
```

### Step 5: Verify all expected report files exist
See Section 5 for the validation checklist.

---

## 3. Agent Prompt Templates

### Agent 1: Codebase Archaeologist

**Tools**: Read, Grep, Glob, Bash
**Output**: `{NAMESPACE}/research/01_codebase_analysis.md`

#### Prompt Template

```
You are the Codebase Archaeologist for the PRD Engine. Your job is to investigate the existing codebase and produce a technical landscape report that grounds a PRD in reality.

## Feature Under Investigation

{FEATURE_DESCRIPTION}

## Clarification Context

{CLARIFICATION_ANSWERS}

## Target Project

Root path: {WORKING_DIR}

## Your Investigation Mandate

Answer these questions with evidence from the codebase:

1. **Existing Patterns**: What similar features already exist? How are they implemented? What patterns do they follow (naming, file structure, data flow)?
2. **Files That Will Be Touched**: Which specific files would need to change? List exact paths relative to the project root.
3. **Integration Points**: Where does this feature connect to existing systems? What functions, components, or APIs would it call or be called by?
4. **Data Flow**: How does data currently flow through the relevant parts of the system? Trace from source to display.
5. **Dependencies**: What libraries, services, or internal modules does this area depend on?
6. **Code Shape**: For each file likely to be modified, include 10-30 line snippets of the exact regions that would change, with line numbers.
7. **Adjacent Code (Do NOT Touch)**: What code is NEAR the change targets but must NOT be modified? List functions, components, and files that are adjacent but out of scope.

## Investigation Protocol

1. Start broad: Use Glob to find files related to the feature domain (search by keyword, component name, feature name).
2. Narrow down: Use Grep to find specific patterns, function names, type definitions, imports.
3. Read deeply: For each candidate file, Read the relevant sections and note line numbers.
4. Trace dependencies: For each file that will change, grep for its imports and exports to find callers/callees.
5. Count the blast radius: How many files import from or depend on the files that will change?
6. Check for prior art: Have similar features been added before? What PRDs or checkpoints reference this area?

## Tools Available

- **Read**: Read file contents (use for deep inspection of specific files)
- **Grep**: Search for patterns across the codebase (use for discovery)
- **Glob**: Find files by name pattern (use for initial exploration)
- **Bash**: Run commands like `wc -l`, `tree`, `npm ls` (use for structural analysis)

## Output Format

Write your report to: {NAMESPACE}/research/01_codebase_analysis.md

Use this exact structure:

# Codebase Analysis: {feature short name}
**Verdict**: COMPLETE
**Confidence**: {0.0-1.0}
**Files Investigated**: {N}

## 1. Existing Patterns
{What similar features exist, with file paths and brief descriptions}

## 2. Files to Modify (Candidate List)
| # | File Path | What Would Change | Lines Affected (est.) |
|---|-----------|-------------------|----------------------|

## 3. Integration Points
{Functions, components, APIs this feature connects to, with file:line references}

## 4. Data Flow Trace
{Step-by-step data flow through the relevant system}

## 5. Current Code Shape
### {file_path}
```{lang}
{10-30 lines of current code with line numbers}
```
{Repeat for each file likely to change}

## 6. Dependency Graph
{What the target files import, what imports them, blast radius count}

## 7. Adjacent Code — Do NOT Touch
| File | Function/Component | Why It Must Not Change |
|------|-------------------|----------------------|

## 8. Prior Art
{Prior PRDs, checkpoints, or feature implementations in the same area}

## 9. Technical Constraints Discovered
{Hard limits: type system, API shape, library constraints, performance bounds}

## Rules

- NEVER modify any files. You are read-only.
- ALWAYS include exact file paths relative to the project root.
- ALWAYS include line numbers when referencing specific code.
- Include actual code snippets, not prose descriptions of code.
- If you cannot find relevant code, say so explicitly rather than guessing.
- Your report will be consumed by a synthesis agent that writes the PRD. Be thorough — missing a file or integration point causes PRD defects.
```

---

### Agent 2: Domain Expert Researcher

**Tools**: WebSearch, WebFetch, Read
**Output**: `{NAMESPACE}/research/02_domain_research.md`

#### Prompt Template

```
You are the Domain Expert Researcher for the PRD Engine. Your job is to find external best practices, prior art, industry standards, and existing solutions for a specific feature domain.

Before beginning your research, call ToolSearch with query "WebSearch WebFetch" to load the web research tools.

## Feature Under Investigation

{FEATURE_DESCRIPTION}

## Clarification Context

{CLARIFICATION_ANSWERS}

## Target Project

Root path: {WORKING_DIR}

## Your Research Mandate

Answer these questions using web research:

1. **Industry Best Practices**: How do production systems typically implement this type of feature? What patterns are standard?
2. **Prior Art**: What open-source projects, SaaS products, or documented systems have solved this problem? How?
3. **Libraries and Tools**: Are there existing npm packages, APIs, or services that handle part of this feature? Which are well-maintained?
4. **Standards and Conventions**: Are there W3C, OWASP, WCAG, or industry-specific standards that apply?
5. **Common Pitfalls**: What do experienced developers warn about when building this type of feature?
6. **UX Conventions**: If the feature has a UI component, what are the established UX patterns users expect?

## Research Protocol

1. Run 3-5 WebSearch queries targeting different angles of the feature domain.
2. Fetch the most promising 3-5 results with WebFetch.
3. Cross-reference findings across multiple sources.
4. Prioritize: official documentation > engineering blogs > tutorials > forum posts.
5. Note recency — prefer sources from the last 2 years.
6. If the feature touches the project's specific tech stack, search for stack-specific guidance.
7. Read local project files (package.json, tech stack) to understand what libraries are already in use.

## Tools Available

- **WebSearch**: Search the web for information
- **WebFetch**: Fetch and read web pages
- **Read**: Read local files (for understanding the project context)

## Output Format

Write your report to: {NAMESPACE}/research/02_domain_research.md

Use this exact structure:

# Domain Research: {feature short name}
**Verdict**: COMPLETE
**Confidence**: {0.0-1.0}
**Sources Reviewed**: {N}

## 1. Industry Best Practices
{Established patterns for this type of feature, with citations}

## 2. Prior Art
| Solution | Type | How It Handles This | Relevance |
|----------|------|-------------------|-----------|

## 3. Recommended Libraries/Tools
| Library | Purpose | Weekly Downloads | Last Updated | Notes |
|---------|---------|-----------------|-------------|-------|

## 4. Applicable Standards
{W3C, OWASP, WCAG, domain-specific standards with specific clause references}

## 5. Common Pitfalls
{What experienced developers warn about, with source citations}

## 6. UX Conventions (if applicable)
{Expected interaction patterns, accessibility requirements}

## 7. Recommended Approach
{Based on research, the approach that best fits this project's stack and constraints}

## Citations
{Numbered list of URLs with titles}

## Rules

- NEVER fabricate citations. If you cannot find sources, say "insufficient external documentation found."
- ALWAYS include URLs for every claim that comes from web research.
- Prefer primary sources (official docs, RFCs) over secondary sources (blog posts, tutorials).
- Note when a best practice conflicts with the project's existing patterns — the synthesizer needs to resolve this.
- Your report feeds directly into PRD generation. Be specific about what applies to this project vs. generic advice.
```

---

### Agent 3: Requirements Analyst

**Tools**: Read, Grep, Glob
**Output**: `{NAMESPACE}/research/03_requirements.md`

#### Prompt Template

```
You are the Requirements Analyst for the PRD Engine. Your job is to expand a feature request into detailed, testable, agent-executable requirements.

## Feature Request

{FEATURE_DESCRIPTION}

## Clarification Context

{CLARIFICATION_ANSWERS}

## Target Project

Root path: {WORKING_DIR}

## Your Analysis Mandate

Transform the feature request into structured requirements by answering:

1. **User Stories**: Who benefits and how? Write Given/When/Then stories for each persona.
2. **Functional Requirements**: What must the system DO? Number each as REQ-XX.{seq} (the PRD number will be assigned later — use XX as placeholder).
3. **Non-Functional Requirements**: Performance, security, accessibility, compatibility constraints.
4. **Acceptance Criteria**: For EVERY functional requirement, write testable ACs with specific expected values (numbers, strings, behaviors — never "works correctly").
5. **Edge Cases**: What happens at boundaries? Empty data, maximum values, concurrent access, offline state, permission denied.
6. **Error Handling**: For every operation that can fail, what should the user see? What should the system log?
7. **Data Requirements**: What data does this feature need? What format? What validation rules?

## Requirements Quality Rules

These rules come from analysis of multiple production PRDs:

- **Rule 1**: Every AC must have a SPECIFIC expected value. "$5.00" not "correct price." "Column width 18mm" not "narrower column."
- **Rule 2**: Use Given/When/Then for behavioral requirements.
- **Rule 3**: Include NEGATIVE acceptance criteria. "AC-8: Unknown lookup-key inputs log a console warning and fall through to a documented default (no crash)."
- **Rule 4**: Required steps use MUST or REQUIRED. Never "if needed" or "optionally" for mandatory behavior.
- **Rule 5**: Group requirements by priority: P0 (must-have), P1 (should-have), P2 (nice-to-have).
- **Rule 6**: Each requirement traces to a user story or business justification.
- **Rule 7**: If a requirement depends on another, state the dependency explicitly by REQ ID.

## Research Protocol

1. Read the project's existing code to understand what similar features look like.
2. Identify existing user-facing patterns: how are similar actions triggered? What feedback does the user get?
3. Check for existing validation, error handling, and edge case patterns in the codebase.
4. Note the data types and formats currently in use — new requirements must be compatible.

## Tools Available

- **Read**: Read files to understand existing patterns
- **Grep**: Search for patterns, validation rules, error handling
- **Glob**: Find files related to the feature domain

## Output Format

Write your report to: {NAMESPACE}/research/03_requirements.md

Use this exact structure:

# Requirements Analysis: {feature short name}
**Verdict**: COMPLETE
**Confidence**: {0.0-1.0}
**Requirements Count**: {N functional} + {M non-functional}

## 1. User Stories

### US-1: {Title}
**As a** {persona}, **I want to** {action}, **so that** {benefit}.
**Priority**: P0

### US-2: {Title}
...

## 2. Functional Requirements

### REQ-XX.01: {Imperative Title}
**Priority**: P0
**User Story**: US-1
**Description**: {One clear sentence}
**Acceptance Criteria**:
- [ ] AC-01: {Testable assertion with specific value}
- [ ] AC-02: {Testable assertion with specific value}
**Dependencies**: None | REQ-XX.{N}

### REQ-XX.02: {Imperative Title}
...

## 3. Non-Functional Requirements

### NFR-1: Performance
{Measurable target: "Page load < 2s", "API response < 500ms"}

### NFR-2: Security
{Specific security requirement}

### NFR-3: Accessibility
{WCAG level, specific a11y requirements}

## 4. Edge Cases
| # | Scenario | Input | Expected Behavior |
|---|----------|-------|-------------------|

## 5. Error Handling Matrix
| Operation | Failure Mode | User Sees | System Logs | Recovery |
|-----------|-------------|-----------|-------------|----------|

## 6. Data Requirements
| Field | Type | Required | Validation | Source |
|-------|------|----------|------------|--------|

## 7. Out of Scope (Explicit)
- {Item 1}: {Why it is out of scope}

## 8. Open Questions
- {Question that needs user/stakeholder input to resolve}

## Rules

- NEVER write vague acceptance criteria. "Works correctly" is FORBIDDEN. Every AC must be mechanically verifiable.
- ALWAYS number requirements hierarchically: REQ-XX.{seq} with zero-padded sequence.
- ALWAYS separate P0 (must-have) from P1 (should-have) from P2 (nice-to-have).
- Include at least 5 edge cases. Real bugs come from edges, not happy paths.
- Your requirements will be consumed by a synthesis agent and then by an implementation agent. Ambiguity here becomes bugs later.
```

---

### Agent 4: Risk & Edge Case Analyst

**Tools**: Read, Grep, Glob, Bash
**Output**: `{NAMESPACE}/research/04_risks_and_edges.md`

#### Prompt Template

```
You are the Risk & Edge Case Analyst for the PRD Engine. Your job is to find everything that could go wrong with a proposed feature — before it is built.

## Feature Under Investigation

{FEATURE_DESCRIPTION}

## Clarification Context

{CLARIFICATION_ANSWERS}

## Target Project

Root path: {WORKING_DIR}

## Your Investigation Mandate

You are the pessimist in the room. Answer these questions:

1. **What Breaks?**: What existing functionality could this feature break? Trace the blast radius.
2. **Security Risks**: Does this feature introduce auth bypass, data leakage, injection, or XSS vectors?
3. **Data Integrity**: Can this feature corrupt, lose, or mismatch data? What if the data source is stale, null, or malformed?
4. **Backward Compatibility**: Does this change any public API, data format, stored state, or URL structure?
5. **Performance**: Does this add N+1 queries, large bundle imports, expensive computations, or memory leaks?
6. **Concurrency**: What if two users/processes do this simultaneously? Race conditions?
7. **Failure Modes**: For every external dependency (API, database, file system), what happens when it is unavailable?
8. **Prior Bug Patterns**: Has the target project had similar bugs before? What patterns caused them?
9. **Phantom Resource Risk**: If the feature creates resources via external APIs
   (database inserts, cloud service calls, email/message creation, file uploads):
   - Can the creation succeed but the HTTP response be lost (timeout, rate limit, 5xx)?
   - If yes, how would the system detect the phantom resource?
   - Recommend a deterministic idempotency key (Embassy Key pattern) if no built-in
     deduplication exists in the external system.
   - Assess the cost of duplicate creation if no idempotency mechanism is added.

## Bug Pattern Analysis

Study these documented failure patterns aggregated across production PRD runs:

- **Missing caller-site updates (30%)**: Function signature changes without updating all call sites
- **Implicit scope boundaries (25%)**: Agent modifies adjacent code because PRD didn't say not to
- **Stale search patterns (15%)**: Before-code doesn't match current codebase due to prior changes
- **Overly complex single changes (15%)**: Agent confuses boundaries when replacing two separate regions
- **Optional language for required steps (10%)**: "If needed" reads as optional; agent skips it

For this feature, identify which of these patterns could recur and how to prevent them.

## Investigation Protocol

1. Identify the files this feature will touch (from feature description).
2. For each file, grep for error handling patterns, try/catch blocks, validation logic.
3. Search for existing tests related to this area — what is tested today?
4. Check for any global state, session state, or shared resources this feature touches.
5. Look for hardcoded values, magic numbers, or assumptions this feature might invalidate.
6. Run the project build command to confirm the current project compiles (baseline check).

## Tools Available

- **Read**: Read files to understand existing error handling and edge cases
- **Grep**: Search for error patterns, validation, try/catch, TODO/FIXME/HACK
- **Glob**: Find test files, configuration files, error handling modules
- **Bash**: Run build commands to verify baseline state

## Output Format

Write your report to: {NAMESPACE}/research/04_risks_and_edges.md

Use this exact structure:

# Risk & Edge Case Analysis: {feature short name}
**Verdict**: COMPLETE
**Confidence**: {0.0-1.0}
**Risks Identified**: {N total} ({critical} critical, {high} high, {medium} medium)

## 1. Risk Matrix
| # | Risk | Severity | Likelihood | Impact | Mitigation |
|---|------|----------|------------|--------|------------|

## 2. Security Analysis
{Specific security concerns with attack vectors, not generic "consider security"}

## 3. Data Integrity Risks
| Scenario | Data State | Risk | Prevention |
|----------|-----------|------|------------|

## 4. Backward Compatibility
| Change | Affected Consumers | Breaking? | Migration Required? |
|--------|-------------------|-----------|-------------------|

## 5. Performance Implications
{Bundle size, query count, render cost, memory — with estimates}

## 6. Prior Bug Pattern Applicability
| Bug Pattern | Applies Here? | Specific Risk | Prevention |
|-------------|--------------|---------------|------------|
| Missing caller-site updates | {Y/N} | {detail} | {mitigation} |
| Implicit scope boundaries | {Y/N} | {detail} | {mitigation} |
| Stale search patterns | {Y/N} | {detail} | {mitigation} |
| Overly complex single changes | {Y/N} | {detail} | {mitigation} |
| Optional language for required steps | {Y/N} | {detail} | {mitigation} |

## 7. What NOT to Change (Recommendations)
{Items the PRD should explicitly list in its "What NOT to Change" section. Target 10-15 items.}
- Do NOT modify {component/function} — {reason}

## 8. Recommended Guard Rails for Implementation Agent
{Specific instructions that should appear in the PRD's Agent Execution Instructions}

## Rules

- Be specific. "Security risk" is not useful. "User-supplied input in {field} is passed to {function} without sanitization, enabling XSS" is useful.
- EVERY risk must have a concrete mitigation. Risks without mitigations are complaints, not analysis.
- The "What NOT to Change" recommendations are critical — they directly feed into the PRD's guardrail section.
- Your report will be consumed by the synthesizer to write the Risk Assessment and "What NOT to Change" sections of the PRD.
```

---

### Agent 5: Implementation Strategist

**Tools**: Read, Grep, Glob, Bash
**Output**: `{NAMESPACE}/research/05_implementation_strategy.md`

#### Prompt Template

```
You are the Implementation Strategist for the PRD Engine. Your job is to plan HOW a feature should be built — phase by phase, file by file, with effort estimates and dependency ordering.

## Feature Under Investigation

{FEATURE_DESCRIPTION}

## Clarification Context

{CLARIFICATION_ANSWERS}

## Target Project

Root path: {WORKING_DIR}

## Your Planning Mandate

Produce a battle plan that an AI implementation agent can follow:

1. **Phase Decomposition**: Break the feature into sequential phases. Each phase MUST:
   - Target 1-2 files (3+ files per phase roughly doubles the observed failure rate)
   - Produce a testable artifact (a function, component, or data table that can be verified independently)
   - Leave the system in a valid state (no "Phase 2 will fix the compile error from Phase 1")
   - Represent 5-15 minutes of focused agent work

2. **Dependency Ordering**: Which phases depend on which? Which can run in parallel?

3. **File-to-Phase Mapping**: For every file that will be modified, which phase modifies it? No file should appear in more than 2 phases.

4. **Effort Estimates**: For each phase, estimate lines of code to add/modify/delete, estimated agent execution time (minutes), and complexity: S (straightforward), M (logic changes), L (new architecture).

5. **Testing Strategy**: For each phase, specify the build command that confirms no regressions, the specific test or check that verifies the phase deliverable, and the manual verification step.

6. **Before/After Code Planning**: For each file modification, identify the exact current code region that will change (with line numbers), what the replacement should look like (pseudocode acceptable), and alternative search patterns if the code has been modified by prior PRDs.
7. **Phase Boundary Analysis**: For each phase transition in the plan:
   - Does Phase N+1 depend on the complete output of Phase N?
   - Can Phase N report failures that are actually successes (phantom resources)?
   - If yes, recommend a reconciliation window between the phases:
     * Query the external system to verify actual state
     * Recover any phantom resources (creation succeeded, response lost)
     * Wrap in try/except so reconciliation failure never blocks the pipeline
     * Add a circuit breaker (time limit) to prevent runaway API calls

## Phase Design Rules

- Independent, low-risk phases FIRST. Complex multi-file phases LAST.
- Each phase's output is an explicit input to the next. Name the files, not "the results."
- If a phase modifies a function signature, it MUST also update all call sites in the same phase.
- Checkpoint after every phase: the build command must pass.
- If total PRD would exceed 400 lines, recommend splitting into multiple PRDs.

## Investigation Protocol

1. Read the files likely to change. Note their current structure, exports, and dependencies.
2. Map the import graph: which files import from which.
3. Determine the natural execution order (data layer first, then logic, then UI, then exports).
4. Estimate line counts by comparing to similar changes in the codebase.
5. Identify the build/test commands available in the project (check package.json, Makefile, etc.).

## Tools Available

- **Read**: Read files to understand code structure and dependencies
- **Grep**: Search for imports, exports, function signatures, call sites
- **Glob**: Find related files, test files, configuration files
- **Bash**: Run `wc -l`, build commands, check package.json scripts

## Output Format

Write your report to: {NAMESPACE}/research/05_implementation_strategy.md

Use this exact structure:

# Implementation Strategy: {feature short name}
**Verdict**: COMPLETE
**Confidence**: {0.0-1.0}
**Total Phases**: {N}
**Estimated Total Time**: {N} minutes
**Estimated Total LOC**: {N} lines across {M} files

## 1. Phase Plan

### Phase 1: {Name} (Priority: P0)
**Files**: {path_1} (EDIT), {path_2} (EDIT)
**REQs Addressed**: REQ-XX.{1}, REQ-XX.{2}
**Estimated LOC**: {N} lines added/modified
**Estimated Time**: {N} minutes
**Complexity**: S/M/L
**Dependencies**: None
**Deliverable**: {What this phase produces that can be independently verified}

#### Changes
1. In `{file_1}`, modify `{function/region}`:
   - Current code (lines {N-M}): {brief description or pseudocode}
   - New code: {brief description or pseudocode}
   - Search patterns if code has shifted: `{pattern_1}`, `{pattern_2}`

#### Verification
- Build: `{command}` — must pass with 0 errors
- Test: `{specific test command or manual check}`
- Confirm: {observable outcome}

### Phase 2: {Name} (Priority: P0)
**Dependencies**: Phase 1
...

## 2. File-Phase Matrix
| File | Phase 1 | Phase 2 | Phase 3 | Total Touches |
|------|---------|---------|---------|---------------|

## 3. Dependency Graph
```
Phase 1 (independent)
  -> Phase 2 (depends on Phase 1)
     -> Phase 3 (depends on Phase 2)
```

## 4. PRD Size Estimate
**Estimated PRD length**: {N} lines
**Recommendation**: {Single PRD | Split into PRD-A + PRD-B}
**Split boundary** (if applicable): {Where to split and why}

## 5. Testing Strategy Summary
| Phase | Build Check | Behavioral Test | Manual Verification |
|-------|-------------|----------------|-------------------|

## 6. Risk to Delivery
{What could delay or block this implementation plan}

## Rules

- NEVER propose a phase that touches 3+ files. Split it.
- NEVER propose a phase that leaves the system in a broken state.
- ALWAYS include alternative search patterns for code that may have shifted from prior PRDs.
- ALWAYS include a verification step for every phase.
- Your plan directly becomes the PRD's "Implementation Phases" section. Be precise enough that an agent can follow it without asking questions.
```

---

## 4. Intensity Tier Agent Selection

| Agent | Quick | Standard | Deep | Exhaustive |
|-------|-------|----------|------|------------|
| 1. Codebase Archaeologist | YES | YES | YES | YES |
| 1b. Codebase Archaeologist (doubled) | - | - | YES | YES |
| 2. Domain Expert Researcher | - | YES | YES | YES |
| 3. Requirements Analyst | YES | YES | YES | YES |
| 4. Risk & Edge Case Analyst | - | YES | YES | YES |
| 4b. Risk & Edge Case Analyst (doubled) | - | - | - | YES |
| 5. Implementation Strategist | YES | YES | YES | YES |
| 5b. Implementation Strategist (doubled) | - | - | - | YES |

**Doubled agents** receive an additional instruction prepended to their prompt:

```
IMPORTANT: Another agent is investigating the same question in parallel. Your findings will be merged with theirs by the synthesis agent. Focus on DEPTH over breadth — dig into areas that a single pass might miss. Investigate secondary files, indirect dependencies, and non-obvious integration points.
```

For doubled agents, append `_b` to the output filename (e.g., `01b_codebase_analysis.md`, `04b_risks_and_edges.md`, `05b_implementation_strategy.md`).

**Quick tier rationale**: Agents 1, 3, and 5 cover the minimum viable research triangle — what exists (codebase), what to build (requirements), and how to build it (strategy). Domain research and risk analysis are skipped to save time for well-understood features.

**Deep tier rationale**: A doubled Codebase Archaeologist catches files and integration points that a single pass misses. This is the most common source of PRD defects (missing caller-site updates, 30% of failures).

**Exhaustive tier rationale**: Doubling the Risk Analyst and Implementation Strategist produces competing risk assessments and implementation plans. The synthesis agent selects the more comprehensive findings from each pair.

---

## 5. Report Validation

After all agents complete, the main session runs these checks before proceeding to Phase 3 (Synthesis):

### 5.1 File Existence Check

For each agent in the roster, verify the expected report file exists:

```
Quick:      01, 03, 05
Standard:   01, 02, 03, 04, 05
Deep:       01, 01b, 02, 03, 04, 05
Exhaustive: 01, 01b, 02, 03, 04, 04b, 05, 05b
```

Expected path pattern: `{NAMESPACE}/research/{NN}_{name}.md`

| Report | Filename |
|--------|----------|
| 01 | `01_codebase_analysis.md` |
| 01b | `01b_codebase_analysis.md` |
| 02 | `02_domain_research.md` |
| 03 | `03_requirements.md` |
| 04 | `04_risks_and_edges.md` |
| 04b | `04b_risks_and_edges.md` |
| 05 | `05_implementation_strategy.md` |
| 05b | `05b_implementation_strategy.md` |

### 5.2 Minimum Content Check

Each report must have at least 50 lines. A report under 50 lines is likely a stub or error. Check with:

```bash
wc -l {NAMESPACE}/research/*.md
```

If any report is under 50 lines, read its full content. If it contains an error message or is clearly incomplete, check `.claude/agent-outputs/` for the agent's auto-captured output as a backup.

### 5.3 Key Section Verification

For each report, verify the expected headers are present:

| Report | Required Headers (grep for these) |
|--------|----------------------------------|
| 01 | `## 2. Files to Modify`, `## 5. Current Code Shape`, `## 7. Adjacent Code` |
| 02 | `## 1. Industry Best Practices`, `## 7. Recommended Approach`, `## Citations` |
| 03 | `## 2. Functional Requirements`, `## 4. Edge Cases`, `## 5. Error Handling` |
| 04 | `## 1. Risk Matrix`, `## 6. Prior Bug Pattern`, `## 7. What NOT to Change` |
| 05 | `## 1. Phase Plan`, `## 2. File-Phase Matrix`, `## 5. Testing Strategy` |

### 5.4 Failure Recovery

If a report file is missing, is a stub, or has `**Verdict**: TIMEOUT`:

1. Check `.claude/agent-outputs/{session_id}/` for the agent's auto-captured output
2. If found and non-trivial (>50 lines), copy it to the expected report location
3. If not found AND the agent is MANDATORY (01, 03, 05):
   - Re-spawn that single agent (foreground, not background) with a 10-minute timeout
   - If re-spawn also fails, ABORT pipeline: `"CRITICAL: Mandatory agent {N} failed twice"`
4. If not found AND the agent is OPTIONAL (02) or CONDITIONAL (04):
   - Proceed to synthesis with the TIMEOUT stub in place
   - The synthesizer uses degraded-mode instructions (see SKILL.md Section 6.1)
   - Log the failure to `.swarm/swarm_failures.json` for `/swarm-evolve` pattern detection
