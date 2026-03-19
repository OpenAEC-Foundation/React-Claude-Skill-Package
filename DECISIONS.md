# Architectural Decisions — React Claude Skill Package

Numbered decisions with rationale. Immutable once recorded.

---

## D-001: English-Only Content
- **Date**: 2026-03-19
- **Decision**: All skill content, code comments, and documentation within skills MUST be in English.
- **Rationale**: Claude operates in English. Skills must be unambiguous for the AI model. Project management files (ROADMAP, etc.) may use Dutch per global CLAUDE.md instructions, but skill content is English-only.
- **Impact**: All skills, references, and code examples.

## D-002: MIT License
- **Date**: 2026-03-19
- **Decision**: The project uses the MIT License.
- **Rationale**: Consistent with other OpenAEC Foundation skill packages. Maximum permissiveness for community adoption.
- **Impact**: LICENSE file, all source files.

## D-003: 7-Phase Research-First Methodology
- **Date**: 2026-03-19
- **Decision**: Follow the proven 7-phase methodology: Infrastructure → Vooronderzoek → Deep Dives → Architecture → Creation → Validation → Publication.
- **Rationale**: Proven effective in ERPNext, Blender-Bonsai, and Tauri skill packages. Research-first ensures skills are accurate and comprehensive before creation begins.
- **Impact**: All project phases, ROADMAP.md structure.

## D-004: React 18 + React 19 Dual Version Coverage
- **Date**: 2026-03-19
- **Decision**: All skills MUST support React 18.x. React 19 features MUST be clearly marked with version badges. Where APIs differ, show BOTH versions.
- **Rationale**: React 18 is the current stable widely-adopted version. React 19 introduces significant new features (React Compiler, Server Components improvements, new hooks). Skills must serve developers on both versions.
- **Impact**: All skills must note version-specific behavior. React 19-only features (use(), React Compiler, Actions) need clear labeling.

## D-005: SKILL.md Under 500 Lines
- **Date**: 2026-03-19
- **Decision**: Every SKILL.md entry point MUST be under 500 lines. Extended content goes in references/ subdirectory.
- **Rationale**: Claude's context window is valuable. Concise skills load faster and leave more room for user code. Heavy examples and reference tables belong in references/ files that are loaded on demand.
- **Impact**: Skill structure, references/ directory usage.

## D-006: TypeScript/TSX as Primary Code Language
- **Date**: 2026-03-19
- **Decision**: All code examples MUST use TypeScript/TSX with proper type annotations. Plain JavaScript examples are only used when contrasting typed vs untyped approaches.
- **Rationale**: TypeScript is the industry standard for React development. Type annotations serve as documentation and help Claude generate type-safe code. React's own types (@types/react) are mature and comprehensive.
- **Impact**: All code examples in skills and references.

## D-007: WebFetch for Documentation Verification
- **Date**: 2026-03-19
- **Decision**: Use WebFetch tool to verify documentation against react.dev during research phases.
- **Rationale**: React documentation on react.dev is actively maintained and may change. WebFetch ensures we reference the latest API signatures and patterns, not stale cached knowledge.
- **Impact**: Research protocol, source verification workflow.

## D-008: Split Hooks into Basic and Advanced
- **Date**: 2026-03-19
- **Decision**: Split hooks into `syntax-hooks-basic` (7 core hooks) and `syntax-hooks-advanced` (specialized + React 19 hooks).
- **Rationale**: React has 18+ hooks. One skill would exceed 500 lines. Basic covers useState, useEffect, useContext, useRef, useMemo, useCallback, useReducer. Advanced covers useId, useTransition, useDeferredValue, useSyncExternalStore, plus React 19 hooks (use, useActionState, useOptimistic).
- **Impact**: Skill inventory, batch plan.

## D-009: Merge Forms into Single Skill
- **Date**: 2026-03-19
- **Decision**: Merge controlled inputs AND React 19 form actions into a single `syntax-forms` skill.
- **Rationale**: Forms are a single developer workflow. Splitting controlled inputs from form actions would force loading two skills for one task.
- **Impact**: Skill inventory.

## D-010: Add Data Fetching Skill
- **Date**: 2026-03-19
- **Decision**: Add `impl-data-fetching` as a dedicated skill.
- **Rationale**: Data fetching (TanStack Query, use() hook, Suspense) is a critical React workflow that doesn't fit cleanly in any other skill.
- **Impact**: Skill inventory, batch 6.

## D-011: Add Styling Skill
- **Date**: 2026-03-19
- **Decision**: Add `impl-styling` as a dedicated skill.
- **Rationale**: CSS modules, Tailwind, and CSS-in-JS are common React questions that need dedicated coverage.
- **Impact**: Skill inventory, batch 5.

## D-012: Rename SSR to Server Components
- **Date**: 2026-03-19
- **Decision**: Rename `impl-ssr` to `impl-server-components`.
- **Rationale**: Server Components are the React 19 paradigm. SSR is just one aspect. The skill covers Server/Client Components, directives, and Server Actions.
- **Impact**: Skill naming.

## D-013: Batch Ordering — Core First
- **Date**: 2026-03-19
- **Decision**: Reorder batches: core first, then syntax, then impl, errors last before agents.
- **Rationale**: Ensures dependency chain is respected. Core skills provide architecture context for syntax skills.
- **Impact**: Batch execution plan.

## D-014: Remove Standalone Portals Skill
- **Date**: 2026-03-19
- **Decision**: Remove standalone "portals" skill. Portal usage covered in react-syntax-components and react-core-architecture.
- **Rationale**: Too thin for its own skill.
- **Impact**: Skill count (26 → 25 after this removal).

## D-015: Merge State Management into Core State
- **Date**: 2026-03-19
- **Decision**: Merge state management into `core-state` covering useState + useReducer + Context + external stores (Zustand/Jotai).
- **Rationale**: State is a fundamental concept. Covering it holistically in one core skill prevents fragmentation. Context API gets its own syntax skill for the Provider pattern details.
- **Impact**: Skill inventory.

## D-016: Research Fragments as Vooronderzoek Strategy
- **Date**: 2026-03-19
- **Decision**: Use research fragments (separate files per topic area) instead of a single monolithic vooronderzoek document. The vooronderzoek-react.md serves as index pointing to 3 detailed fragment files (3,620+ lines total).
- **Rationale**: React's API surface is too broad for a single research document. Splitting into hooks-and-state, react19-features, and dom-testing-patterns fragments enables parallel research agents and keeps individual files manageable.
- **Impact**: Research structure, Phase 2 deliverables.

## D-017: Inline WebFetch as Topic Research Strategy
- **Date**: 2026-03-19
- **Decision**: Use inline WebFetch during skill creation (Phase 5) instead of separate Phase 4 topic-research documents.
- **Rationale**: The deep vooronderzoek fragments (3,620+ lines) provide sufficient foundation. Per-skill research would duplicate content already captured in fragments. Agents verify specific API details via WebFetch during skill writing.
- **Impact**: Phase 4 deliverables, topic-research directory intentionally empty.
