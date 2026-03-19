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
