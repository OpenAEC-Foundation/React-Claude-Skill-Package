# Lessons Learned

Numbered discoveries and patterns found during development of the React Claude Skill Package.

---

## L-001: Repository initialization from proven template
- **Date**: 2026-03-19
- **Phase**: 1 — Infrastructure
- **Lesson**: The 7-phase research-first methodology, proven in ERPNext and Tauri packages, provides a reliable scaffold for new technology skill packages. Adapting the template to React required mapping React-specific concepts (hooks, components, Server Components, concurrent features) into the established skill category structure.
- **Impact**: Faster bootstrap with consistent quality expectations across all OpenAEC skill packages.

## L-002: Parallel agent execution maximizes throughput
- **Date**: 2026-03-19
- **Phase**: 5 — Skill Creation
- **Lesson**: Running 3 research agents + 3 skill creation agents in parallel, and overlapping batches (launching Batch N+1 while Batch N is still running), dramatically reduces wall-clock time. All 24 skills were created in 8 batches using this overlapping strategy. Key enabler: each agent prompt contains complete scope and context, eliminating inter-agent dependencies at the file level.
- **Impact**: 24 skills created in a single session. The Tauri package (27 skills) used a similar approach. This confirms the pattern works across technology domains.

## L-003: Single-commit anti-pattern
- **Date**: 2026-03-19
- **Phase**: Cross-cutting (Audit)
- **Lesson**: Combining all phases (2 through 7) into a single git commit breaks traceability and makes it impossible to verify methodology compliance per phase. The audit found that while ROADMAP.md claimed 7 separate phases, git history showed only 2 substantive commits. Future packages MUST commit after each phase or batch, following the "Phase X.Y: [action] [subject]" convention.
- **Impact**: All future skill packages. Commit discipline is not just about version control — it's evidence of methodology compliance.

## L-004: YAML description format matters for skill activation
- **Date**: 2026-03-19
- **Phase**: 5 — Skill Creation (Audit)
- **Lesson**: All 24 skills used passive descriptions ("Guides...", "Documents...") instead of action-oriented triggers ("Use when..."). The description field is Claude's primary signal for loading a skill. A description starting with "Use when [trigger]. Prevents [anti-pattern]." using YAML folded block scalar (`>`) is the standard that maximizes trigger accuracy and prevents YAML parsing issues.
- **Impact**: All skill descriptions. The folded block scalar format also avoids YAML quoting problems that caused validation failures in the ERPNext package.
