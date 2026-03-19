# React Claude Skill Package — Skill Index

24 deterministic skills for React 18.x and 19.x development with Claude.

---

## Core Skills (3)

| Skill | Description | Lines |
|-------|-------------|-------|
| [react-core-architecture](skills/source/react-core/react-core-architecture/SKILL.md) | Virtual DOM, fiber reconciler, rendering phases, component lifecycle | 266 |
| [react-core-state](skills/source/react-core/react-core-state/SKILL.md) | State management architecture: useState, useReducer, Context, external stores | 246 |
| [react-core-concurrent](skills/source/react-core/react-core-concurrent/SKILL.md) | Suspense, transitions, deferred values, code splitting, streaming SSR | 281 |

## Syntax Skills (8)

| Skill | Description | Lines |
|-------|-------------|-------|
| [react-syntax-hooks-basic](skills/source/react-syntax/react-syntax-hooks-basic/SKILL.md) | 7 essential hooks: useState, useEffect, useContext, useRef, useMemo, useCallback, useReducer | 393 |
| [react-syntax-hooks-advanced](skills/source/react-syntax/react-syntax-hooks-advanced/SKILL.md) | useId, useTransition, useDeferredValue, useSyncExternalStore + React 19 hooks (use, useActionState, useOptimistic, useFormStatus) | 386 |
| [react-syntax-jsx](skills/source/react-syntax/react-syntax-jsx/SKILL.md) | JSX expressions, conditional rendering, lists/keys, fragments, TypeScript generics | 351 |
| [react-syntax-components](skills/source/react-syntax/react-syntax-components/SKILL.md) | Function components, props, memo, forwardRef, lazy, portals, composition patterns | 366 |
| [react-syntax-events](skills/source/react-syntax/react-syntax-events/SKILL.md) | Synthetic events, TypeScript event types, handler patterns, delegation | 331 |
| [react-syntax-context](skills/source/react-syntax/react-syntax-context/SKILL.md) | createContext, useContext, Provider pattern, performance optimization | 344 |
| [react-syntax-refs](skills/source/react-syntax/react-syntax-refs/SKILL.md) | useRef, forwardRef, useImperativeHandle, callback refs, React 19 ref-as-prop | 344 |
| [react-syntax-forms](skills/source/react-syntax/react-syntax-forms/SKILL.md) | Controlled/uncontrolled inputs, React 19 form actions, useFormStatus, useActionState | 458 |

## Implementation Skills (7)

| Skill | Description | Lines |
|-------|-------------|-------|
| [react-impl-project-setup](skills/source/react-impl/react-impl-project-setup/SKILL.md) | Vite + React + TypeScript setup, project structure, ESLint, env vars | 378 |
| [react-impl-testing](skills/source/react-impl/react-impl-testing/SKILL.md) | React Testing Library + Vitest, queries, user-event, renderHook | 347 |
| [react-impl-performance](skills/source/react-impl/react-impl-performance/SKILL.md) | React.memo, Profiler, React Compiler, code splitting, virtualization | 358 |
| [react-impl-styling](skills/source/react-impl/react-impl-styling/SKILL.md) | CSS Modules, Tailwind CSS, inline styles, className patterns, dark mode | 472 |
| [react-impl-server-components](skills/source/react-impl/react-impl-server-components/SKILL.md) | Server/Client Components, directives, Server Actions, serialization rules | 345 |
| [react-impl-routing](skills/source/react-impl/react-impl-routing/SKILL.md) | React Router v6+, createBrowserRouter, loaders, actions, lazy routes | 500 |
| [react-impl-data-fetching](skills/source/react-impl/react-impl-data-fetching/SKILL.md) | TanStack Query, useQuery, useMutation, Suspense, caching, pagination | 436 |

## Error Skills (4)

| Skill | Description | Lines |
|-------|-------------|-------|
| [react-errors-boundaries](skills/source/react-errors/react-errors-boundaries/SKILL.md) | Error boundaries, getDerivedStateFromError, recovery, react-error-boundary | 435 |
| [react-errors-hooks](skills/source/react-errors/react-errors-hooks/SKILL.md) | Rules of Hooks violations, stale closures, infinite loops, cleanup mistakes | 375 |
| [react-errors-hydration](skills/source/react-errors/react-errors-hydration/SKILL.md) | SSR hydration mismatches, common causes, debugging, React 19 improvements | 333 |
| [react-errors-debugging](skills/source/react-errors/react-errors-debugging/SKILL.md) | React DevTools, Strict Mode, console warnings, error messages, profiling | 310 |

## Agent Skills (2)

| Skill | Description | Lines |
|-------|-------------|-------|
| [react-agents-review](skills/source/react-agents/react-agents-review/SKILL.md) | Code review checklist: hooks, TypeScript, performance, accessibility, security | 303 |
| [react-agents-project-scaffolder](skills/source/react-agents/react-agents-project-scaffolder/SKILL.md) | Project generation: Vite config, structure, routing, state, testing, styling | 414 |

---

## Statistics

| Metric | Value |
|--------|-------|
| Total skills | 24 |
| Total SKILL.md lines | 8,592 |
| Average lines per skill | 358 |
| Reference files | 72+ |
| Anti-patterns documented | 200+ |
| React versions covered | 18.x, 19.x |
