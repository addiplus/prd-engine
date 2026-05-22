# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-05-15

### Added

- Initial public release.
- Primary SKILL.md describing the 3-phase pipeline (research, synthesis, quality + adversarial review).
- Four intensity tiers: `quick` (4 agents), `standard` (13 agents), `deep` (15 agents), `exhaustive` (19 agents).
- Ten specialized agents shipped under `agents/prd/`:
  - `prd-engine-synthesizer` — composes PRD from research reports
  - `prd-reviewer` — scores draft on 5-dimension rubric (0-100)
  - `prd-blast-radius-analyst` — traces downstream consumer breakage
  - `prd-data-integrity-adversary` — corruption, race, parsing-overflow attacks
  - `prd-ux-regression-hunter` — layout, dark mode, discoverability gaps
  - `prd-code-pattern-enforcer` — BEFORE/AFTER block accuracy, style drift
  - `prd-scope-critic` — overengineering and cross-PRD conflicts
  - `adversarial-calibrator` — rates adversarial report quality
  - `adversarial-heatmap-generator` — agent x PRD severity matrix
  - `finding-deduplicator` — groups findings by semantic similarity
- Cookbook templates: PRD template, research protocols, quality rubric, agent execution template.
- Vague-prompt detection (Phase 0) and optional Socratic clarification (Phase 1) for `deep`+ tiers.
- Watchdog-driven timeouts with graduated response (warn at 50%, abandon at 100%).
- Score-to-action mapping: 80+ deliver, 60-79 revise, 40-59 rework, <40 escalate.
- Consensus severity escalation (2+ agents on same target → DANGER, 3+ → BLOCKING).
- Degraded-mode synthesis when optional research agents time out.
- Ships as both a Claude Code plugin (via `.claude-plugin/plugin.json`) and a drop-in skill.
