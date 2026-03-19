# React Claude Skill Package

<div align="center">

**A comprehensive Claude skill package for React 18 & 19 development**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![React](https://img.shields.io/badge/React-18%20%7C%2019-61DAFB?logo=react&logoColor=white)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-TSX-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![OpenAEC Foundation](https://img.shields.io/badge/OpenAEC-Foundation-blue)](https://github.com/OpenAEC-Foundation)

</div>

---

## What is this?

This repository contains a curated set of **Claude skills** that teach [Claude](https://claude.ai) how to write, review, debug, and architect React applications with expert-level precision.

Each skill is a structured knowledge file that Claude loads contextually — giving it deep understanding of React hooks, component patterns, Server Components, concurrent features, and the React 19 Compiler.

## Technology Coverage

| Area | Topics | Version |
|------|--------|---------|
| **Hooks API** | useState, useEffect, useRef, useContext, useMemo, useCallback, useReducer, custom hooks | React 18 + 19 |
| **React 19 Features** | React Compiler, use() hook, Actions, useFormStatus, useOptimistic, Server Components | React 19 |
| **Components** | Composition, props, children, render props, HOCs, compound components | React 18 + 19 |
| **Performance** | Memoization, Suspense, transitions, deferred values, code splitting | React 18 + 19 |
| **Testing** | React Testing Library, Vitest, component testing, hook testing | React 18 + 19 |
| **Error Handling** | Error boundaries, debugging, hydration errors, anti-patterns | React 18 + 19 |
| **Architecture** | Rendering model, state management, Server/Client Components | React 18 + 19 |

## Skill Categories

| Category | Prefix | Purpose |
|----------|--------|---------|
| **Syntax** | `react-syntax-*` | Hooks API, component patterns, JSX, event handling |
| **Implementation** | `react-impl-*` | Testing, performance, SSR, build workflows |
| **Errors** | `react-errors-*` | Error boundaries, debugging, anti-patterns |
| **Core** | `react-core-*` | Architecture, rendering model, state management |
| **Agents** | `react-agents-*` | Validation, code generation, scaffolding |

## Current Progress

| Phase | Status | Description |
|-------|--------|-------------|
| 1. Infrastructure | 50% | Repository setup, core files, protocols |
| 2. Vooronderzoek | -- | Broad React 18/19 research |
| 3. Deep Dives | -- | Per-topic targeted research |
| 4. Architecture | -- | Skill boundaries, dependencies |
| 5. Creation | -- | Agent-based skill writing |
| 6. Validation | -- | Quality checks, accuracy review |
| 7. Publication | -- | Final polish, GitHub release |

## Repository Structure

```
├── CLAUDE.md              # Project protocols and instructions
├── ROADMAP.md             # Status and progress tracking
├── REQUIREMENTS.md        # Quality guarantees
├── DECISIONS.md           # Architectural decisions
├── SOURCES.md             # Official reference URLs
├── WAY_OF_WORK.md         # 7-phase methodology
├── LESSONS.md             # Lessons learned
├── CHANGELOG.md           # Version history
├── docs/
│   ├── masterplan/        # Execution plan
│   └── research/          # Research documents
└── skills/
    └── source/
        ├── react-syntax/  # Hooks, JSX, component API
        ├── react-impl/    # Testing, performance, SSR
        ├── react-errors/  # Error handling, debugging
        ├── react-core/    # Architecture, rendering
        └── react-agents/  # Validation, code generation
```

## Methodology

This package follows the **7-phase research-first methodology** proven in:
- [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package)
- [Tauri 2 Skill Package](https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package)
- [Blender-Bonsai Skill Package](https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package)

Research comes before creation — every skill is verified against official React documentation at [react.dev](https://react.dev).

## How to Use

1. Clone this repository
2. Add the skills directory to your Claude project or Claude Code configuration
3. Claude will automatically load relevant skills based on your React development queries

## License

[MIT](LICENSE) — OpenAEC Foundation, 2026
