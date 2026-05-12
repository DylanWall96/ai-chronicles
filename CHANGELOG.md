# Changelog

## [Unreleased]

### 2026-05-12 — Chronicle 001 v3

Added:
- Mental model: "match the model to the task" — tiered model usage by task complexity
- Scaling: model-tier-per-subagent guidance
- Quick Reference row for model-tier matching

### 2026-05-09 — Chronicle 001 v2

Added:
- Mental model: "don't fix a context problem by switching models"
- Hooks section as the deterministic enforcement layer
- Cross-Session State section covering the persistent-Markdown-directory pattern
- Skills security trust hierarchy (mirrors plugins)
- Quick Reference entries for hooks, skills-as-third-party-code, persistence, and context-over-model

Changed:
- Sharpened phrasing throughout, removed appeals to authority
- Promoted thoughts/ pattern from a workflow bullet to a full section

### Initial commit
- Initial chronicle: Effective Claude Code Setup
- Repo structure: README as index, chronicles/ for individual entries
- CC-BY-4.0 license

## Sources informing the current draft

- Anthropic Engineering — context engineering, harness design, agent skills, multi-agent systems
- Claude Code documentation
- HumanLayer blog — CLAUDE.md authoring, advanced context engineering
- GitHub — AGENTS.md large-repo analysis
- Independent research on long-context degradation
