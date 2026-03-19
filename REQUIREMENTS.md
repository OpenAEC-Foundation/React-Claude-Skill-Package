# Requirements — React Claude Skill Package

## Purpose
Define what skills must achieve, quality guarantees, and per-area requirements for the React skill package.

---

## Global Skill Requirements

### Format
- YAML frontmatter with `name` and `description` fields
- `description` MUST contain trigger words that help Claude match the skill to user queries
- SKILL.md is the entry point — ALWAYS under 500 lines
- Heavy content (long examples, reference tables) goes in `references/` subdirectory
- English-only content (no Dutch or other languages in skill files)

### Language & Style
- Deterministic language: "ALWAYS use X when Y" / "NEVER do X because Y"
- No hedging: avoid "you might consider", "it could be useful", "perhaps"
- Action-oriented: every section tells Claude what to DO
- Code examples in TypeScript/TSX (never plain JavaScript unless showing a contrast)
- All code examples MUST include proper TypeScript types

### Structure (in order)
1. **Quick Reference** — Most common patterns, copy-paste ready
2. **Decision Trees** — "When X, use Y" branching logic
3. **Patterns** — Detailed patterns with context and rationale
4. **Anti-Patterns** — What NOT to do and why
5. **Version Notes** — React 18 vs 19 differences (where applicable)
6. **Reference Links** — Links to references/ files and official docs

### Version Coverage
- All skills MUST work with React 18.x
- React 19 features MUST be clearly marked with version badges
- Where APIs differ between React 18 and 19, show BOTH versions
- Deprecated patterns MUST be flagged with migration guidance

---

## Per-Area Requirements

### Syntax Skills (react-syntax-*)
Skills covering React API syntax, hooks, JSX patterns, event handling.

- **Hooks**: MUST follow Rules of Hooks (top-level only, React function components/custom hooks only)
- **Custom Hooks**: MUST show the `use` prefix convention and TypeScript return types
- **JSX/TSX**: MUST show proper TypeScript generic patterns for components
- **Events**: MUST use React's synthetic event types (React.MouseEvent, React.ChangeEvent, etc.)
- **Context**: MUST show createContext with proper TypeScript generics
- **Refs**: MUST cover useRef, forwardRef, and useImperativeHandle with types
- **React 19 hooks**: MUST cover use(), useFormStatus, useFormState, useOptimistic where applicable

### Implementation Skills (react-impl-*)
Skills covering development workflows, testing, performance, SSR.

- **Testing**: MUST use React Testing Library patterns (not Enzyme)
- **Performance**: MUST cover React.memo, useMemo, useCallback with profiling guidance
- **SSR/RSC**: MUST distinguish between Server Components and Client Components
- **Build**: MUST reference Vite as primary build tool (not Create React App)
- **Routing**: MUST cover React Router v6+ patterns
- **Forms**: MUST show controlled vs uncontrolled patterns with React 19 Actions

### Error Skills (react-errors-*)
Skills covering error handling, debugging, common mistakes.

- **Error Boundaries**: MUST show class component pattern (still required in React 18/19)
- **Debugging**: MUST reference React DevTools and Strict Mode behavior
- **Anti-patterns**: MUST explain WHY each pattern is wrong, not just what to avoid
- **Hydration errors**: MUST cover SSR/SSG hydration mismatch debugging
- **Rules of Hooks violations**: MUST show common violations and fixes

### Core Skills (react-core-*)
Skills covering architecture, rendering model, state management.

- **Rendering**: MUST explain reconciliation, virtual DOM, fiber architecture
- **State**: MUST cover useState, useReducer, external state (Zustand/Jotai patterns)
- **Concurrent features**: MUST cover Suspense, transitions, deferred values
- **React Compiler**: MUST explain automatic memoization (React 19)
- **Component composition**: MUST show compound components, render props, HOC patterns

### Agent Skills (react-agents-*)
Skills providing intelligent orchestration and code generation.

- **Validation agent**: MUST check generated React code against Rules of Hooks and TypeScript types
- **Code generation**: MUST produce TypeScript/TSX output with proper types
- **Component scaffolding**: MUST follow project conventions (file structure, naming)

---

## Quality Gates

### Gate 1: Format Validation
- [ ] YAML frontmatter present and valid
- [ ] SKILL.md under 500 lines
- [ ] English-only content
- [ ] File naming follows `react-{category}-{topic}` convention

### Gate 2: Content Validation
- [ ] Deterministic language throughout
- [ ] TypeScript types in all code examples
- [ ] React 18/19 version coverage where applicable
- [ ] Anti-patterns section present
- [ ] All code examples tested/verified against official docs

### Gate 3: Cross-Reference Validation
- [ ] All references/ files exist and are linked
- [ ] No broken internal links
- [ ] Sources traceable to SOURCES.md URLs

### Gate 4: Technical Accuracy
- [ ] Hooks follow Rules of Hooks
- [ ] Component patterns match official React documentation
- [ ] No deprecated patterns without migration notes
- [ ] Server Component / Client Component boundaries correct
