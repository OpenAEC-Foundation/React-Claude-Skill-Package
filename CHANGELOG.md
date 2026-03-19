# Changelog

All notable changes to the React Claude Skill Package will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Fixed
- Migrated all 24 YAML descriptions from quoted strings to folded block scalar (`>`) format
- Rewrote all descriptions to "Use when..." trigger pattern for better skill activation
- Synchronized 8 masterplan refinement decisions to DECISIONS.md (D-008 through D-015)
- Added research strategy decisions D-016 and D-017
- Trimmed react-impl-routing SKILL.md from 500 to 493 lines
- Fixed hedging language ("ALWAYS consider using" → "ALWAYS use") in react-errors-boundaries
- Updated ROADMAP.md metrics to reflect actual decision count (17)
- Added audit lessons L-003 and L-004 to LESSONS.md

## [1.0.0] - 2026-03-19

### Added
- **24 production-ready skills** covering React 18.x and 19.x
- **Core skills (3)**: architecture, state management, concurrent features
- **Syntax skills (8)**: hooks-basic, hooks-advanced, JSX, components, events, context, refs, forms
- **Implementation skills (7)**: project-setup, testing, performance, styling, server-components, routing, data-fetching
- **Error skills (4)**: boundaries, hooks errors, hydration, debugging
- **Agent skills (2)**: code review checklist, project scaffolder
- **72+ reference files** with extended examples, API tables, and patterns
- **200+ documented anti-patterns** across all skills
- **3 research fragments** from vooronderzoek against react.dev
- **Definitive masterplan** with 24 skills across 8 execution batches
- INDEX.md skill catalog with links and statistics
- README.md with complete skill tables and usage instructions

### Infrastructure
- Repository initialized with 7-phase research-first methodology
- Core files: CLAUDE.md, ROADMAP.md, REQUIREMENTS.md, DECISIONS.md, SOURCES.md, WAY_OF_WORK.md, LESSONS.md
- 15 architectural decisions (7 in DECISIONS.md + 8 in masterplan)
- MIT License, .gitignore
- Research verified against official React documentation (react.dev)
