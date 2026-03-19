# Vooronderzoek React — Skill Package Research Index

## Date: 2026-03-19
## Status: Complete

---

## Research Summary

The vooronderzoek for the React Skill Package was conducted using WebFetch against official React documentation (react.dev) and ecosystem sources. Research was split into 3 parallel fragments covering the complete React API surface.

### Research Fragments

| Fragment | File | Sections | Sources Fetched |
|----------|------|----------|-----------------|
| Part 1: Core Hooks & API | [react-hooks-api-research.md](fragments/react-hooks-api-research.md) | §1-§12 | 20+ react.dev pages |
| Part 2: React 19 Features | [react-19-features-research.md](fragments/react-19-features-research.md) | §1-§9 | 10+ react.dev pages |
| Part 3: DOM, Testing, Patterns | [react-dom-testing-patterns-research.md](fragments/react-dom-testing-patterns-research.md) | §1-§8 | 18 react.dev pages |

---

## Key Findings

### React API Surface (React 18 + 19)

**Hooks (18 total)**:
- State: useState, useReducer
- Effects: useEffect, useLayoutEffect, useInsertionEffect
- Context: useContext
- Refs: useRef, useImperativeHandle
- Performance: useMemo, useCallback
- Concurrent: useTransition, useDeferredValue
- Utility: useId, useSyncExternalStore, useDebugValue
- React 19 only: use(), useActionState, useOptimistic, useFormStatus

**Components**: Fragment, Suspense, StrictMode, Profiler, Activity (experimental)

**APIs**: createContext, forwardRef (deprecated in 19), lazy, memo, startTransition, cache, act

### React 19 Major Changes
1. **React Compiler** — Automatic memoization replaces manual useMemo/useCallback/memo
2. **Server Components** — Default component type in RSC frameworks, async/await support
3. **Form Actions** — `<form action={fn}>`, useActionState, useFormStatus, useOptimistic
4. **ref as prop** — No more forwardRef needed, ref cleanup functions
5. **use() hook** — Read promises and context, can be called conditionally
6. **Metadata hoisting** — `<title>`, `<meta>`, `<link>` hoist to `<head>` automatically
7. **Stylesheet precedence** — `<link precedence="...">` for CSS ordering

### Skill Coverage Mapping

| Research Area | Skills Covered |
|---------------|---------------|
| Core hooks (§1-§5) | react-syntax-hooks-basic, react-syntax-hooks-advanced |
| Concurrent features (§6) | react-core-concurrent |
| React 19 hooks (§8) | react-syntax-hooks-advanced, react-syntax-forms |
| Components & APIs (§9-§10) | react-syntax-components, react-core-architecture |
| Rules of Hooks (§11) | react-syntax-hooks-basic, react-errors-hooks |
| DOM APIs | react-core-architecture |
| Forms | react-syntax-forms |
| Error boundaries | react-errors-boundaries |
| JSX patterns | react-syntax-jsx |
| Performance | react-impl-performance |
| Testing | react-impl-testing |
| Anti-patterns | react-errors-hooks, react-errors-debugging |
| Server Components | react-impl-server-components |
| React Compiler | react-impl-performance |
| Data fetching | react-impl-data-fetching |

---

## Sources Verified

All research was verified against official sources listed in SOURCES.md:
- https://react.dev/reference/react (primary API reference)
- https://react.dev/learn (official tutorials)
- https://react.dev/blog/2024/12/05/react-19 (React 19 release)
- https://react.dev/learn/react-compiler (Compiler docs)
- https://react.dev/reference/rsc/server-components (RSC reference)
- https://testing-library.com/docs/react-testing-library/intro (Testing)
