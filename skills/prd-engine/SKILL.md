---
name: prd-engine
description: "Generate comprehensive, agent-executable PRDs using parallel Opus research agents. Use when user says 'PRD', 'write a PRD', 'spec out', 'requirements document', 'feature plan', 'create a spec'. NOT for reviewing existing PRDs or general product questions."
triggers: ["write a PRD", "create a PRD", "PRD for", "spec out", "requirements for", "feature plan", "new PRD", "draft a PRD", "I need a PRD", "plan this feature", "requirements document", "product spec", "write requirements"]
---

# PRD Engine

Generate agent-executable Product Requirements Documents using parallel research agents, structured synthesis, and quality scoring. Produces PRDs with before/after code blocks, verification tables, acceptance criteria, and explicit scope boundaries.

For quick internal feature specs without research, use the `product-manager` agent directly instead.

---

## 1. Quick Start

**Step 1: Describe the feature.**
```
You: "Write a PRD for adding dark mode to the dashboard"
You: "PRD for the payment integration API --deep"
You: "Quick PRD for fixing the header alignment bug"
```

**Step 2: Review the plan.** The engine shows agents, intensity, and estimated time before spawning anything.

**Step 3: Say "go".** Or adjust: `more agents`, `fewer agents`, `add clarification`, `cancel`.

**Step 4: Wait.** Research agents run in parallel (background). Synthesis and quality gate run sequentially. Progress updates appear as agents complete.

**Step 5: Review.** Final PRD is delivered to `.prd-engine/{slug}/PRD.md` with a summary of quality score, requirement count, files touched, and next steps.

---

## 2. How It Works

The engine runs a 3-phase pipeline with 4-19 agents depending on intensity:

```
Phase 1: RESEARCH (parallel, background)
  3-8 Opus agents investigate the feature from orthogonal perspectives:
  codebase analysis, domain research, requirements, risks/edge cases,
  implementation strategy. Higher tiers double key agents for depth.
  Each writes a numbered report to .prd-engine/{slug}/research/

Phase 2: SYNTHESIS (sequential, foreground)
  Synthesizer agent reads ALL research reports + the PRD template.
  Writes a complete PRD draft with before/after code, verification
  tables, and agent execution instructions.
  Validator agent checks draft for completeness and consistency.

Phase 3: QUALITY GATE + ADVERSARIAL REVIEW (parallel, background)
  Reviewer agent scores the draft on a 5-dimension rubric (0-100).
  Contrarian agent (deep+) challenges assumptions and flags risks.
  5 adversarial expert agents (standard+) attack the PRD from
  orthogonal angles: blast radius, data integrity, UX regression,
  code patterns, and scope/complexity. All run in parallel.
  Score + adversarial findings determine: deliver, revise, or escalate.
```

**Quality gate guarantee**: The quality gate phase (reviewer agent) MUST execute for standard, deep, and exhaustive tiers regardless of edge case handling (vague prompt detection, scope warning generation). Edge case pre-processing is a Phase 0 step that precedes -- never replaces -- the standard pipeline.

---

## 3. Phase 0: Vague Prompt Detection

Before entering the main pipeline, the engine checks for vague or under-specified prompts. A prompt is considered vague when ANY of these conditions are met:

- No specific system, component, or feature is named
- No measurable criteria or concrete goal is stated
- Prompt is under 15 words

**Blocking behavior**: When vagueness is detected AND `--no-questions` is NOT set AND tier is NOT `quick`:

1. **STOP.** Present 3-5 targeted clarification questions to the user.
2. **WAIT** for the user's response before creating the workspace directory or spawning any agents.
3. If the user says "go" or "just write it", proceed with best-effort assumptions and tag each one with `[ASSUMPTION]` in the feature description passed to research agents.
4. Only create `.prd-engine/{slug}/` AFTER clarification is resolved.

When `--no-questions` IS set or tier IS `quick`, skip vague prompt detection entirely and proceed to Phase 1.

---

## 4. Phase 1: Socratic Clarification

For `deep` and `exhaustive` intensity (or when `--clarify` is passed), the engine asks 3-5 targeted questions BEFORE spawning research agents:

```
Before I spawn the research council, let me confirm a few things:

1. Who is the primary user of this feature? (end user, admin, internal ops?)
2. Are there any hard constraints? (budget, timeline, tech stack lock-in)
3. Does this integrate with existing systems? (APIs, databases, services)
4. What does success look like? (metric, behavior, user outcome)
5. Anything explicitly out of scope?

Answer as many as you like, or say "go" to let the research agents figure it out.
```

This phase is **skipped** for `quick` and `standard` intensity to keep latency low. Users can also skip it with `--no-questions` or by saying "just write it."

Answers are appended to the feature description that every research agent receives.

---

## 5. Phase 2: Parallel Research

After confirming the plan, the main session creates the workspace and spawns research agents.

### 5.1 Create Workspace

```bash
mkdir -p .prd-engine/{slug}/research/
```

Write `config.json` to the workspace directory. The schema is standardized:

```json
{
  "prd_number": 35,
  "slug": "feature-name",
  "intensity": "standard",
  "status": "DRAFT",
  "feature": "Description of the feature",
  "agents": 8,
  "created": "2026-03-19"
}
```

**Mandatory fields**: `prd_number`, `slug`, `intensity`, `status`, `feature`, `agents`, `created`. Use `intensity` exclusively (do NOT use a separate `tier` field -- they are the same concept). The `status` field tracks pipeline progress: `DRAFT` (initial), `RESEARCH` (Phase 2), `SYNTHESIS` (Phase 3), `REVIEW` (quality gate), `DELIVERED` (complete), or `FAILED`.

### 5.2 Read Research Protocols

Read `cookbook/research-protocols.md` (one-time). Extract each agent's prompt template. Fill placeholders: `{FEATURE_DESCRIPTION}`, `{NAMESPACE}`, `{CLARIFICATION_ANSWERS}`, `{WORKING_DIR}`.

### 5.3 Spawn Research Agents

Spawn ALL research agents in parallel using the Agent tool. Every agent runs in background with `model: opus`.

```
For each agent in the roster, call the Agent tool:

  Agent tool call:
    prompt: "{self-contained prompt extracted from research-protocols.md
              with all placeholders filled}"
    model: opus
    run_in_background: true

  Agent 1 (Codebase Archaeologist)   -> .prd-engine/{slug}/research/01_codebase_analysis.md
  Agent 2 (Domain Expert Researcher) -> .prd-engine/{slug}/research/02_domain_research.md        [standard+]
  Agent 3 (Requirements Analyst)     -> .prd-engine/{slug}/research/03_requirements.md
  Agent 4 (Risk & Edge Case Analyst) -> .prd-engine/{slug}/research/04_risks_and_edges.md        [standard+]
  Agent 5 (Implementation Strategist)-> .prd-engine/{slug}/research/05_implementation_strategy.md
```

For Deep and Exhaustive tiers, doubled agents write to `{NN}b_` filenames (e.g., `01b_codebase_analysis.md`). See `cookbook/research-protocols.md` Section 4 for the full tier table.

### 5.4 Monitor with Timeout (Watchdog)

Track each agent's start time, expected output file path, and `last_file_size` (init 0). Enter a polling loop with **30-second intervals**:

**Per-agent checks (each poll cycle):**

1. **Completion check**: Has the agent sent a completion notification? If yes → mark complete, read first 5 lines of report.
2. **File growth check**: Does the output file exist? Compare its size to `last_file_size`. If size increased → agent is alive, update `last_file_size`, clear any warning state. Do NOT timeout an agent whose file is actively growing.
3. **Auto-capture fallback**: If no output file exists, check `.claude/agent-outputs/{session_id}/` for auto-captured output. If found (>50 lines), copy to expected path and mark complete.
4. **Per-agent timeout** (fires only when file is NOT growing):
   - **WebSearch agents** (Agent 02 Domain Expert): **15 min** hard timeout
   - **Code-only agents** (01, 03, 04, 05): **10 min** hard timeout
   - **Synthesis/quality agents**: **20 min** hard timeout

**Wave timeout** (hard cap for entire research phase): **15 minutes**. When wave timeout fires, abandon ALL remaining pending agents regardless of individual timeout status.

**Graduated response:**
- **At 50% of timeout** (no file growth): Log warning — `"[WATCHDOG] Agent {N} ({role}) at {X}min of {timeout}min limit — no output growth"`
- **At 100% of timeout with no output file**: Mark ABANDONED, write stub report (see 5.5)
- **At 100% of timeout WITH partial output** (file exists, size > 0): Grant one 50% extension. Do NOT overwrite partial output — the stub report appends a `## WATCHDOG: Agent Abandoned` footer instead.
- **File actively growing at timeout**: Do NOT abandon. The agent is working. Reset warning state and continue polling.

As each agent completes (normally or abandoned), read only the **first 5 lines** (verdict + confidence + key findings) of its report. Do NOT read full reports — that is the synthesizer's job.

Report progress after each event: `"Research: {completed}/{total} agents complete ({abandoned} abandoned)"`

After ALL agents complete or are abandoned: `"Research phase complete: {N} reports collected, {M} abandoned by watchdog."`

### 5.5 Quorum Check

After each agent completes OR times out, check quorum:

| Agent | Role | Criticality |
|-------|------|-------------|
| 01 Codebase Archaeologist | Existing patterns, file inventory | **MANDATORY** |
| 02 Domain Expert Researcher | External best practices | OPTIONAL |
| 03 Requirements Analyst | Structured requirements | **MANDATORY** |
| 04 Risk & Edge Case Analyst | Risk matrix, guardrails | CONDITIONAL (mandatory at standard+, optional at quick) |
| 05 Implementation Strategist | Phase plan, file matrix | **MANDATORY** |

**Quorum met** = all MANDATORY agents complete (regardless of optional agent status).

**After quorum**: Start a **2-minute grace period** for remaining optional agents, then proceed to Phase 3.

**Mandatory agent timeout**: ABORT pipeline and notify user: `"CRITICAL: Mandatory agent {N} ({role}) timed out after {X}min. Pipeline cannot proceed. Check .claude/agent-outputs/ for partial work."`

**Stub report for abandoned agents**: When an agent is abandoned, write a stub to its expected output path:

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

**Log to `.swarm/swarm_failures.json`** (append): Record the timeout for pattern detection:
```json
{
  "timestamp": "{ISO-8601}",
  "context": "Agent timeout: {agent_role} during PRD-{N} research",
  "root_cause": "agent-timeout",
  "severity": "medium",
  "tags": ["timeout", "prd-engine", "{agent_role}"],
  "swarm_metadata": {
    "namespace": "{slug}",
    "agent_count": "{N}",
    "timed_out_agent": "{role}",
    "elapsed_minutes": "{M}",
    "timeout_minutes": "{T}"
  }
}
```

---

## 6. Phase 3: Synthesis + Quality Gate

### 6.1 Spawn Synthesizer (foreground, blocking)

**Degraded-mode prefix**: If any research agents timed out (stub reports with `**Verdict**: TIMEOUT`), prepend the following to the synthesizer prompt before the standard instructions:

```
DEGRADED MODE: The following research reports are UNAVAILABLE due to agent timeout:
- {report_name}: {agent_role}

Compensate by:
- Deriving domain insights from codebase analysis (01) findings
- Marking affected PRD sections with [DEGRADED: {missing_report}]
- Reducing quality gate threshold by 5 points per missing optional report

Per-agent degradation impacts:
- 02 (Domain Expert) missing: No external best practices or prior art. Note "Domain research unavailable — recommend manual review before implementation."
- 04 (Risk Analyst) missing: Reduced risk coverage. Generate basic risk section from codebase (01) and requirements (03) findings. Quality gate score capped at 75.
- Doubled agents (01b, 04b, 05b) missing: No impact — primary agent report is sufficient.
```

**Standard synthesizer prompt**:

```
Agent tool call:
  prompt: |
    You are a PRD Synthesizer. Compose a production-grade PRD from research reports.

    {DEGRADED_MODE_PREFIX if applicable, empty string otherwise}

    FEATURE: {feature description + clarification answers}
    PRD NUMBER: {XX}
    DATE: {today}

    Read these files:
    1. cookbook/prd-template.md  (use Glob "**/prd-engine/cookbook/prd-template.md" if direct read fails — works for both drop-in and plugin install paths)
    2. All research reports in .prd-engine/{slug}/research/ (Glob for *.md):
       - 01_codebase_analysis.md (Codebase Archaeologist)
       - 02_domain_research.md (Domain Expert Researcher) [if exists]
       - 03_requirements.md (Requirements Analyst)
       - 04_risks_and_edges.md (Risk & Edge Case Analyst) [if exists]
       - 05_implementation_strategy.md (Implementation Strategist)
       - Plus any doubled reports (01b_, 04b_, 05b_) [if exists]

    For any report with **Verdict**: TIMEOUT, ignore its content and compensate
    using findings from other reports. Mark affected PRD sections accordingly.

    Fill the PRD template using findings from ALL available research reports.
    Resolve contradictions between reports (e.g., file count discrepancies).
    Write output to: .prd-engine/{slug}/PRD_DRAFT.md

    CRITICAL RULES:
    - Every requirement MUST have before/after code blocks
    - Every acceptance criterion MUST have a specific expected value
    - "What NOT to Change" section MUST have >= 10 items
    - Total PRD length: budget = 200 + (REQ_count * 60), capped at 600; warn at 500, force-split at 600
    - Include Agent Execution Instructions from the template
    - All reference data inline -- no external file references
    - If the feature creates resources via external APIs where the response can be lost, the PRD MUST include Section 8a (Idempotency Strategy)
    - If the feature changes how existing artifacts are identified or detected, the PRD MUST include Section 8b (Backward Compatibility Strategy)
    - If the feature has 2+ phases where Phase N+1 depends on state created in Phase N, the Implementation Phases MUST include a reconciliation step between the dependent phases
  model: opus
  run_in_background: false
```

### 6.2 Spawn Validator (foreground, blocking)

Reads PRD draft + research reports. Checks for: missing before/after code blocks, vague acceptance criteria, file count mismatches, untraceable requirements. Writes to `.prd-engine/{slug}/research/09_validation.md`.

If the validator finds issues, append its feedback to the synthesis prompt and re-run the synthesizer (max 1 revision cycle).

### 6.3 Spawn Quality Gate Agents (standard+ intensity)

**Reviewer** (standard+): Spawn in background.
```
Agent tool call:
  agent: prd-reviewer
  prompt: |
    Score the PRD draft on 5 dimensions (0-100 total) using your
    built-in scoring rubric.
    Read: .prd-engine/{slug}/PRD_DRAFT.md
    Also read research reports in .prd-engine/{slug}/research/ for cross-reference.
    Write score + top 3 suggestions to: .prd-engine/{slug}/research/10a_quality_review.md
  model: opus
  run_in_background: true
```

**Contrarian** (deep+ only): Spawn in background alongside reviewer.
```
Agent tool call:
  prompt: |
    You are a PRD Contrarian. Challenge assumptions, flag over-engineering,
    identify unstated assumptions. Do NOT use a rubric -- freeform critique.
    Read: .prd-engine/{slug}/PRD_DRAFT.md
    Write concerns with severity to: .prd-engine/{slug}/research/10b_contrarian.md
  model: opus
  run_in_background: true
```

### 6.4 Adversarial Expert Review (standard+ intensity)

Spawn all 5 adversarial agents in parallel alongside the quality gate agents. Each reads the PRD draft and the codebase, then writes a structured adversarial report.

```
Agent tool calls (all parallel, all background, all model: opus):

  Agent: prd-blast-radius-analyst
  prompt: |
    Read the PRD at .prd-engine/{slug}/PRD_DRAFT.md
    Target codebase: {working_dir}
    Also check for related PRDs: Glob .prd-engine/*/PRD.md
    Write your blast radius report to: .prd-engine/{slug}/research/expert_1_blast_radius.md

  Agent: prd-data-integrity-adversary
  prompt: |
    Read the PRD at .prd-engine/{slug}/PRD_DRAFT.md
    Target codebase: {working_dir}
    Read source files referenced in the PRD. Read any data/fixture/mock files.
    Write your data integrity report to: .prd-engine/{slug}/research/expert_2_data_integrity.md

  Agent: prd-ux-regression-hunter
  prompt: |
    Read the PRD at .prd-engine/{slug}/PRD_DRAFT.md
    Target codebase: {working_dir}
    Read every component file the PRD modifies AND their downstream consumers.
    Write your UX regression report to: .prd-engine/{slug}/research/expert_3_ux_regression.md

  Agent: prd-code-pattern-enforcer
  prompt: |
    Read the PRD at .prd-engine/{slug}/PRD_DRAFT.md
    Target codebase: {working_dir}
    Read every source file referenced in BEFORE/AFTER blocks.
    Write your code pattern report to: .prd-engine/{slug}/research/expert_4_code_patterns.md

  Agent: prd-scope-critic
  prompt: |
    Read the PRD at .prd-engine/{slug}/PRD_DRAFT.md
    Target codebase: {working_dir}
    Also read related PRDs if any: Glob .prd-engine/*/PRD.md
    Write your scope review to: .prd-engine/{slug}/research/expert_5_scope_critic.md
```

**Monitor quality gate + adversarial agents with the same watchdog pattern from Section 5.4:**
- **Quality gate agents** (reviewer, contrarian): **15 min** hard timeout per agent
- **Adversarial expert agents** (5 agents): **12 min** hard timeout per agent
- **Quality phase wave timeout**: **25 min** total for all quality + adversarial agents
- Poll every 30 seconds. Track file growth. Apply graduated response (warn at 50%, abandon at 100%).
- Adversarial agents are OPTIONAL — if any time out, proceed with remaining reports. Minimum **3/5** adversarial reports is sufficient for meaningful coverage.
- If the **reviewer** (10a) times out, skip the score-to-action step and deliver the PRD as-is with a note: `"Quality score unavailable — reviewer agent timed out. Manual review recommended."`

Wait for agents to complete or be abandoned. Read the **first 10 lines** of each completed report (verdict + top findings). Collect all DANGER/VULNERABLE/GAP/STALE/BLOCKING findings.

### 6.5 Incorporate Adversarial Findings

After all adversarial agents complete, check for critical findings:

1. **DANGER** (blast radius), **VULNERABLE** (data integrity), **GAP** (UX), **STALE/BUG** (code patterns), **CONFLICT/OVERENGINEERED** (scope) — these are critical.
2. If ANY critical findings exist, append a `### Adversarial Review Findings` section to the PRD draft listing each critical finding with its source agent, severity, and one-line summary.
3. For STALE BEFORE/AFTER blocks (code pattern enforcer), apply corrections directly to the PRD draft before delivery.
4. For DANGER items with fix code provided, include the fix in the PRD's verification checklist.

The adversarial review does NOT block delivery. It enriches the PRD with known risks. The quality gate score still determines the deliver/revise/rework action.

### 6.5.1 Consensus Severity Escalation

After collecting all adversarial reports, the main session cross-references findings before incorporating them:

1. Parse each adversarial report for findings with severity DANGER, VULNERABLE, GAP, STALE, BUG, CONFLICT, or BLOCKING.
2. For each finding, extract the target: `{file_path}:{function_or_parameter}`.
3. Group findings by target. If 2+ agents flagged the same target:
   - Tag the finding `[CONSENSUS: N/5 agents]` in the Adversarial Review Findings table.
   - Escalate to DANGER if not already at that severity.
4. If a finding has 3+ agent consensus, it is **BLOCKING**: the synthesizer MUST add or modify a REQ to address the gap. It cannot be listed as ACCEPTED or DEFERRED in the findings table.
5. Non-consensus findings remain informational (listed in the findings table as-is).

**Example**: when 4/5 adversarial agents independently flag the same data-threading gap across phases, consensus escalation prevents that gap from being listed as separate findings and treated as already-handled — it becomes BLOCKING and must be addressed in the PRD body.

### 6.7 Incorporate Contrarian Findings (deep+ only)

After the contrarian agent completes (if spawned), read `10b_contrarian.md`. Scan for any findings with severity HIGH or CRITICAL. If found, append a `### Known Issues (Contrarian Review)` sub-section to Section 13 (Risk Assessment) in `PRD_DRAFT.md` before proceeding to the score-to-action step. List each HIGH/CRITICAL finding with its severity and a one-line summary.

### 6.8 Score-to-Action Mapping

Read the reviewer's score from the first 5 lines of `10a_quality_review.md`:

| Score | Action |
|-------|--------|
| **80-100** | DELIVER as-is. Copy `PRD_DRAFT.md` to `PRD.md`. |
| **60-79** | REVISE. Read reviewer's top 3 suggestions, apply surgical edits to draft, write `PRD.md`. |
| **40-59** | REWORK. Notify user: "PRD scored {X}/100. Top issues: {list}. Re-synthesize or deliver as-is?" |
| **0-39** | ESCALATE. "PRD has fundamental gaps. Review research at `.prd-engine/{slug}/research/`." |

For `quick` intensity (no quality gate), copy `PRD_DRAFT.md` directly to `PRD.md`.

---

## 7. Intensity Tiers

| Tier | Research | Synthesis | Quality + Adversarial | Total | Time | Use Case |
|------|----------|-----------|----------------------|-------|------|----------|
| **quick** | 3 agents (1,3,5) | 1 synthesizer | None | **4** | 5-8 min | Small bug fix, internal tool, well-understood domain |
| **standard** | 5 agents (1-5) | 1 synth + 1 validator | 1 reviewer + 5 adversarial experts | **13** | 12-18 min | Most features **(DEFAULT)** |
| **deep** | 6 agents (1,1b,2,3,4,5) | 1 synth + 1 validator | 1 reviewer + 1 contrarian + 5 adversarial | **15** | 20-28 min | Complex features, cross-cutting, external-facing |
| **exhaustive** | 8 agents (1,1b,2,3,4,4b,5,5b) | 2 synths + 1 scorer + 1 validator | 1 reviewer + 1 contrarian + 5 adversarial | **19** | 30-45 min | Mission-critical, zero-ambiguity |

Natural language detection:

| User Says | Tier |
|-----------|------|
| "quick PRD", "simple PRD", "rough PRD", "fast" | quick |
| *(no signal)* | standard |
| "thorough PRD", "detailed PRD", "comprehensive", "deep" | deep |
| "exhaustive PRD", "maximum detail", "production PRD", "zero ambiguity" | exhaustive |

At **exhaustive** tier, two independent synthesizers produce competing PRD drafts. A scorer agent selects the better one (evolutionary selection pattern).

---

## 8. Configuration

| Flag | Values | Default |
|------|--------|---------|
| `--intensity` / `-i` | quick, standard, deep, exhaustive | standard |
| `--number` / `-n` | Integer (e.g., 30) | Auto-increment from highest existing PRD-*.md |
| `--output` / `-o` | Directory path | `.prd-engine/{slug}/` |
| `--clarify` | Flag | Enabled for deep+, disabled for quick/standard |
| `--no-questions` | Flag | Skip Socratic clarification |
| `--target` | Project path | Repository root |
| `--skip-quality` | Flag | Skip quality gate (not recommended) |

**Auto-increment logic**: Glob for `**/PRD-*.md` and `**/PRD_*.md`, extract highest number, add 1.

**Slug generation**: Lowercase, hyphenated, max 40 chars. Example: "adding dark mode to the dashboard" becomes `dark-mode-dashboard`.

---

## 9. Output Location

```
.prd-engine/{slug}/
  config.json                       # Run configuration
  research/
    01_codebase_analysis.md         # Research agent reports (01-05, plus b variants)
    ...
    08_prd_draft.md                 # Synthesis output (copy)
    09_validation.md                # Validator report
    10a_quality_review.md           # Reviewer score
    10b_contrarian.md               # Contrarian concerns (deep+)
    expert_1_blast_radius.md        # Adversarial: downstream consumer breakage
    expert_2_data_integrity.md      # Adversarial: corruption, races, phantom state
    expert_3_ux_regression.md       # Adversarial: layout, dark mode, discoverability
    expert_4_code_patterns.md       # Adversarial: BEFORE/AFTER accuracy, style drift
    expert_5_scope_critic.md        # Adversarial: overengineering, cross-PRD conflicts
  PRD_DRAFT.md                      # First draft
  PRD.md                            # Final delivered PRD
```

If `--output` is specified, the final `PRD.md` is also copied to that location.

---

## 10. Progressive Disclosure (Cookbook Files)

The main session loads ONLY this SKILL.md. Everything else is loaded by agents or on demand.

| File | Loaded By | When | Lines |
|------|-----------|------|-------|
| `cookbook/prd-template.md` | Synthesizer agent | Phase 2 (synthesis) | ~420 |
| `cookbook/research-protocols.md` | Main session (one-time read) | Phase 1 setup | ~735 |
| `cookbook/quality-rubric.md` | Reviewer agent (if available) | Phase 3 | ~90 |
| `cookbook/agent-execution-template.md` | Synthesizer agent (if available) | Phase 2 | ~80 |
| `agents/prd/prd-blast-radius-analyst.md` | Agent tool | Phase 3 (adversarial) | ~340 |
| `agents/prd/prd-data-integrity-adversary.md` | Agent tool | Phase 3 (adversarial) | ~250 |
| `agents/prd/prd-ux-regression-hunter.md` | Agent tool | Phase 3 (adversarial) | ~280 |
| `agents/prd/prd-code-pattern-enforcer.md` | Agent tool | Phase 3 (adversarial) | ~260 |
| `agents/prd/prd-scope-critic.md` | Agent tool | Phase 3 (adversarial) | ~300 |

The main session reads `research-protocols.md` once, extracts per-agent prompts, and injects them into each Agent tool call. Agents receive self-contained prompts with zero file-read overhead.

**Exception**: The synthesizer reads `prd-template.md` directly (too large to inject into the Agent prompt without bloating main session context).

**Graceful fallback**: If `cookbook/quality-rubric.md` or `cookbook/agent-execution-template.md` do not exist, the reviewer uses its built-in rubric (embedded in `prd-reviewer.md`) and the synthesizer uses the Agent Execution Instructions template from Section 3 of `prd-template.md`. These cookbook files are optional enhancements, not hard dependencies.

---

## 11. Quick Reference Card

**Invoke**: "Write a PRD for {feature}" or "PRD for {feature} --deep"

**Default**: standard intensity, 13 agents (5 research + synth + validator + reviewer + 5 adversarial experts), 12-18 minutes

**Pipeline**: Vague Prompt Check (Phase 0) -> Clarify (optional) -> Research (parallel) -> Synthesize -> Quality Gate + Adversarial Review (parallel) -> Deliver

**Output**: `.prd-engine/{slug}/PRD.md`

**Research reports**: `.prd-engine/{slug}/research/01-05_*.md`

**Quality score bands**: 80+ deliver | 60-79 revise | 40-59 rework | <40 escalate

**After delivery**:
- Review: `Read .prd-engine/{slug}/PRD.md`
- Implement: "Build PRD-{XX}" or `/swarm build .prd-engine/{slug}/PRD.md`
- Iterate: "Revise PRD-{XX}: add mobile support"

**Key rules enforced**:
- Every requirement has before/after code blocks
- Every acceptance criterion has a specific expected value
- "What NOT to Change" section has 10+ items
- PRD is self-contained (no external references)
- File count in metadata matches files in body
- Target length 200 + (REQ_count × 60), capped at 600 (warn at 500, force-split at 600)

**config.json mandatory fields**: `prd_number`, `slug`, `intensity`, `status`, `feature`, `agents`, `created`

**Testing**: An integrity-test suite (342 assertions across template schema, scoring rubric, agent protocols, and cross-file contracts) is maintained by the author; contact via GitHub if you want the test harness for adaptation.
