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
