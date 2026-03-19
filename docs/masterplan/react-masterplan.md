# React Skill Package — Definitive Masterplan

## Status

Phase 3 complete. Finalized skill inventory with research-informed scope.
Date: 2026-03-19

---

## Decisions Made During Refinement

| # | Decision | Rationale |
|---|----------|-----------|
| D-01 | **Split** hooks into `syntax-hooks-basic` and `syntax-hooks-advanced` | React has 18+ hooks. One skill would exceed 500 lines. Basic covers the 7 essential hooks (useState, useEffect, useContext, useRef, useMemo, useCallback, useReducer). Advanced covers useId, useTransition, useDeferredValue, useSyncExternalStore, plus React 19 hooks (use, useActionState, useOptimistic). |
| D-02 | **Merged** forms into `syntax-forms` covering both controlled inputs AND React 19 form actions | Forms are a single developer workflow. Splitting controlled inputs from form actions would force loading two skills for one task. |
| D-03 | **Added** `impl-data-fetching` | Data fetching (TanStack Query, use() hook, Suspense) is a critical React workflow that doesn't fit cleanly in any other skill. |
| D-04 | **Added** `impl-styling` | CSS modules, Tailwind, and CSS-in-JS are common React questions that need dedicated coverage. |
| D-05 | **Renamed** `impl-ssr` to `impl-server-components` | Server Components are the React 19 paradigm. SSR is just one aspect. The skill covers Server/Client Components, directives, and Server Actions. |
| D-06 | **Reordered** batches: core first, then syntax, then impl, errors last before agents | Ensures dependency chain is respected. Core skills provide architecture context for syntax skills. |
| D-07 | **Removed** standalone "portals" skill | Too thin for its own skill. Portal usage covered in react-syntax-components and react-core-architecture. |
| D-08 | **Merged** state management into `core-state` covering useState + useReducer + Context + external stores | State is a fundamental concept. Covering it holistically in one core skill prevents fragmentation. Context API gets its own syntax skill for the Provider pattern details. |

**Result**: 26 initial topics → **24 definitive skills** (1 split, 2 additions, 2 merges, 1 removal).

---

## Definitive Skill Inventory (24 skills)

### react-core/ (3 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `react-core-architecture` | React rendering model; virtual DOM; fiber architecture; reconciliation; component lifecycle (mount/update/unmount); React element tree; key concepts | `React.createElement`, `ReactDOM.createRoot`, fiber, reconciler | M | None |
| `react-core-state` | State management overview; useState vs useReducer decision; lifting state; Context for cross-cutting state; external stores (Zustand/Jotai patterns); state immutability rules | `useState`, `useReducer`, `createContext`, `useSyncExternalStore` | L | None |
| `react-core-concurrent` | Concurrent rendering; Suspense for loading states; transitions (startTransition, useTransition); deferred values; streaming SSR; React 19 improvements | `Suspense`, `startTransition`, `useTransition`, `useDeferredValue`, `lazy` | M | core-architecture |

### react-syntax/ (8 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `react-syntax-hooks-basic` | Core 7 hooks: useState, useEffect, useContext, useRef, useMemo, useCallback, useReducer; Rules of Hooks; cleanup patterns; dependency arrays | `useState`, `useEffect`, `useContext`, `useRef`, `useMemo`, `useCallback`, `useReducer` | L | None |
| `react-syntax-hooks-advanced` | useId, useTransition, useDeferredValue, useSyncExternalStore, useInsertionEffect, useDebugValue; React 19: use(), useActionState, useOptimistic, useFormStatus | `useId`, `useTransition`, `useDeferredValue`, `use()`, `useActionState`, `useOptimistic` | L | syntax-hooks-basic |
| `react-syntax-jsx` | JSX expressions; conditional rendering patterns; rendering lists with keys; fragments; TypeScript generics for components; embedding expressions; JSX spread attributes | JSX, `<Fragment>`, key prop, `React.FC<P>`, `React.PropsWithChildren` | M | None |
| `react-syntax-components` | Function components; props typing; children patterns; composition; React.memo; forwardRef; lazy; Portals; compound components; render props; HOC pattern | `React.memo`, `forwardRef`, `lazy`, `createPortal`, component composition | L | syntax-jsx |
| `react-syntax-events` | Event handling; synthetic event system; TypeScript event types; event delegation; custom events; form events; keyboard/mouse events; event handler typing | `React.MouseEvent`, `React.ChangeEvent`, `React.FormEvent`, `React.KeyboardEvent` | M | syntax-jsx |
| `react-syntax-context` | createContext with TypeScript generics; useContext; Provider pattern; default values; multiple contexts; context performance; avoiding re-renders | `createContext`, `useContext`, `React.Provider`, context selector patterns | M | syntax-hooks-basic |
| `react-syntax-refs` | useRef for DOM access and mutable values; forwardRef with TypeScript; useImperativeHandle; callback refs; ref cleanup (React 19); ref as prop (React 19) | `useRef`, `forwardRef`, `useImperativeHandle`, callback refs | M | syntax-hooks-basic |
| `react-syntax-forms` | Controlled vs uncontrolled inputs; form submission patterns; React 19 form actions; useFormStatus; useActionState; optimistic updates with useOptimistic; file inputs | `<form action>`, `useFormStatus`, `useActionState`, `useOptimistic`, controlled inputs | L | syntax-hooks-basic, syntax-events |

### react-impl/ (7 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `react-impl-testing` | React Testing Library patterns; render/screen/fireEvent/waitFor; Vitest setup; testing hooks; testing async; accessibility testing; snapshot testing | `render`, `screen`, `fireEvent`, `waitFor`, `renderHook`, `act` | L | syntax-hooks-basic |
| `react-impl-performance` | React.memo; useMemo/useCallback optimization; React Compiler (React 19); Profiler; code splitting; bundle analysis; virtualization; image optimization | `React.memo`, `Profiler`, React Compiler, `React.lazy`, dynamic import | L | syntax-hooks-basic |
| `react-impl-server-components` | Server vs Client Components; 'use server'/'use client' directives; Server Actions; serialization rules; when to use each; framework integration (Next.js patterns) | `'use server'`, `'use client'`, Server Actions, RSC payload | L | core-architecture |
| `react-impl-routing` | React Router v6+ patterns; createBrowserRouter; route configuration; nested routes; loaders/actions; lazy routes; protected routes; URL parameters | `createBrowserRouter`, `RouterProvider`, `useNavigate`, `useParams`, `useSearchParams`, `Outlet` | M | syntax-components |
| `react-impl-styling` | CSS Modules; Tailwind CSS with React; CSS-in-JS patterns; inline styles; className patterns; conditional classes; global vs scoped styles | CSS Modules, `clsx`/`cn`, Tailwind, styled-components patterns | M | syntax-jsx |
| `react-impl-project-setup` | Vite + React + TypeScript setup; project structure; tsconfig.json; ESLint/Prettier; path aliases; environment variables; monorepo patterns | `npm create vite@latest`, `tsconfig.json`, `vite.config.ts` | M | None |
| `react-impl-data-fetching` | TanStack Query patterns; useQuery/useMutation; Suspense for data; React 19 use() hook; error handling; caching strategies; optimistic updates | `useQuery`, `useMutation`, `QueryClient`, `use()`, `Suspense` | L | syntax-hooks-basic, core-concurrent |

### react-errors/ (4 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `react-errors-boundaries` | Error boundary class component; getDerivedStateFromError; componentDidCatch; error recovery; fallback UI; error boundary placement; nested boundaries | `getDerivedStateFromError`, `componentDidCatch`, error boundary pattern | M | core-architecture |
| `react-errors-hooks` | Rules of Hooks violations; conditional hook calls; hooks in loops; missing dependencies; stale closures; infinite re-render loops; cleanup mistakes | Rules of Hooks, eslint-plugin-react-hooks, dependency array | M | syntax-hooks-basic |
| `react-errors-hydration` | SSR hydration mismatch causes; suppressHydrationWarning; debugging hydration errors; common causes (Date, Math.random, browser APIs); React 19 improvements | `hydrateRoot`, `suppressHydrationWarning`, hydration errors | M | impl-server-components |
| `react-errors-debugging` | React DevTools; Strict Mode double-rendering; console warnings; performance profiling; component stack traces; common error messages and solutions | `StrictMode`, React DevTools, Profiler, error messages | M | core-architecture |

### react-agents/ (2 skills)

| Name | Scope | Key APIs | Complexity | Dependencies |
|------|-------|----------|------------|-------------|
| `react-agents-review` | Validation checklist for generated React code; Rules of Hooks compliance; TypeScript type checking; performance anti-patterns; accessibility; Server Component boundaries | All validation rules from all skills | M | ALL syntax + impl skills |
| `react-agents-project-scaffolder` | Generate complete React project; Vite config; TypeScript setup; component structure; routing; state management; testing setup; recommended folder structure | All scaffolding patterns | L | ALL core + syntax skills |

---

## Batch Execution Plan (DEFINITIVE)

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-architecture`, `core-state`, `core-concurrent` | 3 | None | Foundation skills, no dependencies |
| 2 | `syntax-hooks-basic`, `syntax-jsx`, `impl-project-setup` | 3 | None / Batch 1 | Primary syntax + project setup (independent) |
| 3 | `syntax-hooks-advanced`, `syntax-components`, `syntax-events` | 3 | Batch 2 | Advanced hooks + component patterns |
| 4 | `syntax-context`, `syntax-refs`, `syntax-forms` | 3 | Batch 2 | Remaining syntax skills |
| 5 | `impl-testing`, `impl-performance`, `impl-styling` | 3 | Batch 2 | Implementation patterns |
| 6 | `impl-server-components`, `impl-routing`, `impl-data-fetching` | 3 | Batch 1-2 | Advanced implementation |
| 7 | `errors-boundaries`, `errors-hooks`, `errors-debugging` | 3 | Batch 1-2 | Error skills |
| 8 | `errors-hydration`, `agents-review`, `agents-project-scaffolder` | 3 | ALL above | Final error + agents |

**Total**: 24 skills across 8 batches.

---

## Per-Skill Agent Prompts

### Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package
RESEARCH_DIR = C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\docs\research\fragments
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md
```

---

### Batch 1

#### Prompt: react-core-architecture

```
## Task: Create the react-core-architecture skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-core\react-core-architecture\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (working code examples)
3. references/api-table.md (React core type reference)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: react-core-architecture
description: "Guides React application architecture including virtual DOM, fiber reconciler, component tree model, rendering phases (render + commit), element creation, component lifecycle (mount/update/unmount), and React element vs component distinction. Activates when creating React apps, understanding React rendering model, or reasoning about React's component architecture and update cycle."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- React rendering model: virtual DOM, fiber architecture, reconciliation algorithm
- Rendering phases: render phase (pure, interruptible) vs commit phase (DOM mutations)
- React element tree: createElement, JSX compilation, element vs component
- Component lifecycle: mount → render → commit → update → unmount
- Key React concepts: unidirectional data flow, declarative UI, composition over inheritance
- createRoot API and rendering entry point
- StrictMode behavior (double-rendering in development)
- React 18 concurrent rendering vs React 19 improvements

### Research Sections to Read
From research fragments:
- react-hooks-api-research.md (for component model context)
- react-dom-testing-patterns-research.md §1 (DOM APIs)
- react-19-features-research.md (React 19 architecture changes)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples must be TypeScript/TSX
- Include Critical Warnings section with NEVER rules
- Note React 18 vs 19 differences where applicable
```

#### Prompt: react-core-state

```
## Task: Create the react-core-state skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-core\react-core-state\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (state management patterns with code)
3. references/patterns.md (state architecture patterns)
4. references/anti-patterns.md (state management mistakes)

### YAML Frontmatter
---
name: react-core-state
description: "Guides React state management architecture including useState vs useReducer selection, state lifting patterns, Context API for cross-cutting concerns, external store integration (Zustand/Jotai), state immutability rules, and server state vs client state distinction. Activates when designing state architecture, choosing state management approach, or solving state-related problems in React applications."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- State management decision tree: local vs lifted vs context vs external store
- useState for simple local state; useReducer for complex state logic
- Lifting state up: identify the closest common ancestor
- Context API overview: when to use, performance implications
- External stores: Zustand (recommended lightweight), Jotai (atomic), useSyncExternalStore
- Server state vs client state (TanStack Query for server state)
- State immutability: NEVER mutate state directly, ALWAYS create new references
- Derived state: compute from existing state, NEVER store derived values
- React 19: state management with Server Components and Actions

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include decision tree as primary guide
- All code examples in TypeScript
```

#### Prompt: react-core-concurrent

```
## Task: Create the react-core-concurrent skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-core\react-core-concurrent\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (concurrent features code examples)
3. references/patterns.md (Suspense and transition patterns)
4. references/anti-patterns.md (concurrent feature mistakes)

### YAML Frontmatter
---
name: react-core-concurrent
description: "Guides React concurrent rendering features including Suspense for loading states, startTransition and useTransition for non-blocking updates, useDeferredValue for expensive computations, lazy loading with React.lazy, and streaming SSR. Activates when implementing loading states, optimizing perceived performance, handling slow renders, or using Suspense boundaries."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Concurrent rendering concept: interruptible rendering, priority-based updates
- Suspense: fallback UI, nested Suspense boundaries, Suspense for lazy loading
- startTransition: marking updates as non-urgent, keeping UI responsive
- useTransition: isPending state for transition feedback
- useDeferredValue: deferring expensive re-renders
- React.lazy + Suspense for code splitting
- Streaming SSR with Suspense (React 18+)
- React 19: Suspense improvements, use() hook for promises/context
- When to use which: transitions vs deferred values vs Suspense

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include decision tree: which concurrent feature for which use case
- All code examples in TypeScript/TSX
```

---

### Batch 2

#### Prompt: react-syntax-hooks-basic

```
## Task: Create the react-syntax-hooks-basic skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-syntax\react-syntax-hooks-basic\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (hook usage examples)
3. references/api-table.md (hook signatures and return types)
4. references/anti-patterns.md (hook mistakes)

### YAML Frontmatter
---
name: react-syntax-hooks-basic
description: "Guides the 7 essential React hooks: useState for local state, useEffect for side effects, useContext for context consumption, useRef for DOM refs and mutable values, useMemo for expensive computations, useCallback for stable function references, and useReducer for complex state. Covers Rules of Hooks, dependency arrays, and cleanup patterns. Activates when using any core React hook, writing side effects, managing component state, or optimizing re-renders."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- useState: signature, updater function, lazy initialization, TypeScript typing
- useEffect: dependency array, cleanup function, when it runs, async patterns
- useContext: consuming context, TypeScript typing
- useRef: DOM references, mutable values (.current), TypeScript typing
- useMemo: memoizing values, dependency array, when to use (and when NOT to)
- useCallback: memoizing functions, dependency array, when to use
- useReducer: reducer pattern, dispatch, TypeScript typing, init function
- Rules of Hooks: top-level only, React functions only, no conditional calls
- Dependency array rules: referential equality, objects/arrays as deps

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- MUST include Rules of Hooks as Critical Warnings
- All code examples in TypeScript/TSX with proper type annotations
- Include cleanup pattern for useEffect
```

#### Prompt: react-syntax-jsx

```
## Task: Create the react-syntax-jsx skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-syntax\react-syntax-jsx\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (JSX patterns and code examples)
3. references/anti-patterns.md (JSX mistakes)

### YAML Frontmatter
---
name: react-syntax-jsx
description: "Guides JSX and TSX syntax including expressions in JSX, conditional rendering patterns, rendering lists with keys, fragments, TypeScript generics for components, JSX spread attributes, and embedding JavaScript expressions. Activates when writing JSX/TSX, rendering lists, conditional rendering, or typing React components with TypeScript generics."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- JSX compilation: what JSX compiles to (createElement), new JSX transform
- Expressions in JSX: {expression}, limitations (statements not allowed)
- Conditional rendering: ternary, && short-circuit, early return, IIFE pattern
- Lists and keys: map(), key prop, NEVER use index as key (exceptions), key rules
- Fragments: <></>, <Fragment key={k}> for keyed fragments
- TypeScript generics: React.FC<P>, PropsWithChildren, generic components
- JSX spread attributes: {...props}, rest/spread pattern
- Self-closing tags, boolean attributes, className (not class)
- String literals vs expressions: when to use quotes vs braces
- Embedding components: PascalCase for custom components

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples in TSX
- Include the key rules as Critical Warnings
```

#### Prompt: react-impl-project-setup

```
## Task: Create the react-impl-project-setup skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-impl\react-impl-project-setup\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (project setup workflows)
3. references/patterns.md (project structure patterns)
4. references/anti-patterns.md (setup mistakes)

### YAML Frontmatter
---
name: react-impl-project-setup
description: "Guides React project setup with Vite including TypeScript configuration, project structure conventions, ESLint and Prettier setup, path aliases, environment variables, and recommended folder structure. Activates when creating a new React project, configuring Vite, setting up TypeScript, or establishing project conventions."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript and Vite."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Vite + React + TypeScript: npm create vite@latest, template selection
- Project structure: src/, components/, hooks/, utils/, types/, pages/
- tsconfig.json: strict mode, path aliases, JSX config
- vite.config.ts: plugins, resolve aliases, dev server, build options
- ESLint: @eslint/js, typescript-eslint, eslint-plugin-react-hooks, eslint-plugin-react-refresh
- Environment variables: VITE_ prefix, import.meta.env, .env files
- Package.json scripts: dev, build, preview, lint, type-check
- NEVER use Create React App (deprecated, unmaintained)
- React 19: installation considerations (npm install react@19 react-dom@19)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- Include step-by-step project creation workflow
- Include recommended folder structure diagram
- All config examples must be complete and functional
```

---

### Batch 3

#### Prompt: react-syntax-hooks-advanced

```
## Task: Create the react-syntax-hooks-advanced skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-syntax\react-syntax-hooks-advanced\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (advanced hook patterns)
3. references/api-table.md (hook signatures with TypeScript types)
4. references/anti-patterns.md (advanced hook mistakes)

### YAML Frontmatter
---
name: react-syntax-hooks-advanced
description: "Guides advanced React hooks: useId for accessible IDs, useTransition for non-blocking state updates, useDeferredValue for deferred rendering, useSyncExternalStore for external store subscriptions, useInsertionEffect for CSS-in-JS, useDebugValue for DevTools. React 19 hooks: use() for promises and context, useActionState for form actions, useOptimistic for optimistic UI, useFormStatus for pending form state. Activates when using advanced hooks, integrating external stores, or working with React 19 form features."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- useId: generating unique IDs, accessibility, SSR-safe
- useTransition: isPending + startTransition, non-blocking updates
- useDeferredValue: deferring expensive values, comparison with useTransition
- useSyncExternalStore: subscribing to external stores, snapshot pattern, server snapshot
- useInsertionEffect: CSS-in-JS library use only, runs before DOM mutations
- useDebugValue: custom hook debugging in DevTools
- React 19 — use(): reading promises (Suspense integration), reading context
- React 19 — useActionState: form action state management, pending state
- React 19 — useOptimistic: optimistic UI updates during async actions
- React 19 — useFormStatus: reading parent form's pending/data/method/action

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- React 19 hooks MUST be clearly marked with version badge
- All code examples in TypeScript/TSX
- Include decision tree: which advanced hook for which problem
```

#### Prompt: react-syntax-components

```
## Task: Create the react-syntax-components skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-syntax\react-syntax-components\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (component patterns)
3. references/patterns.md (composition patterns: compound, render props, HOC)
4. references/anti-patterns.md (component mistakes)

### YAML Frontmatter
---
name: react-syntax-components
description: "Guides React component patterns including function component typing, props with TypeScript interfaces, children patterns, React.memo for performance, forwardRef for ref forwarding, React.lazy for code splitting, createPortal for rendering outside the DOM tree, compound components, render props, and higher-order components. Activates when creating components, typing props, forwarding refs, or implementing component composition patterns."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Function components: typing with TypeScript, return types
- Props: interface/type definition, optional props, default values, destructuring
- Children: React.ReactNode, PropsWithChildren, children as function (render props)
- React.memo: when to use, custom comparison function, TypeScript typing
- forwardRef: TypeScript generics, combining with useImperativeHandle
- React.lazy: dynamic import, Suspense fallback, route-based splitting
- createPortal: modal/tooltip/overlay patterns, event bubbling through portals
- Compound components: Context-based compound pattern
- Render props: function-as-children, render prop pattern
- HOC pattern: typing HOCs, when to use (rare in modern React)
- React 19: ref as prop (no more forwardRef needed), cleanup function from ref callback

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include decision tree: which component pattern for which use case
- All code examples in TypeScript/TSX
- Note React 19 changes to forwardRef
```

#### Prompt: react-syntax-events

```
## Task: Create the react-syntax-events skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-syntax\react-syntax-events\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (event handling examples)
3. references/api-table.md (all synthetic event types with TypeScript)
4. references/anti-patterns.md (event handling mistakes)

### YAML Frontmatter
---
name: react-syntax-events
description: "Guides React event handling including synthetic event system, TypeScript event type annotations, event handler patterns, form events, keyboard and mouse events, event delegation model, stopPropagation and preventDefault, and custom event patterns. Activates when handling user interactions, typing event handlers, working with form inputs, or implementing keyboard/mouse interactions."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Synthetic event system: React wraps native events, event pooling removed in React 17+
- TypeScript event types: React.MouseEvent, React.ChangeEvent, React.FormEvent, React.KeyboardEvent, React.FocusEvent, React.DragEvent, React.TouchEvent, React.ClipboardEvent
- Event handler typing: React.EventHandler, inline handlers, extracted handlers
- Form events: onChange, onSubmit, onInput, controlled input patterns
- Mouse events: onClick, onDoubleClick, onMouseEnter/Leave, onContextMenu
- Keyboard events: onKeyDown, onKeyUp, key property, modifier keys
- Focus events: onFocus, onBlur, relatedTarget
- stopPropagation vs preventDefault: when to use each
- Event delegation: React attaches to root, not individual elements
- Capture phase: onClickCapture and similar

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- MUST include TypeScript event type table
- All code examples in TypeScript/TSX
- Include common event handler typing patterns
```

---

### Batch 4

#### Prompt: react-syntax-context

```
## Task: Create the react-syntax-context skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-syntax\react-syntax-context\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (context patterns and code)
3. references/anti-patterns.md (context mistakes)

### YAML Frontmatter
---
name: react-syntax-context
description: "Guides React Context API including createContext with TypeScript generics, useContext for consuming context, Provider pattern, default values, multiple contexts, context performance optimization, and avoiding unnecessary re-renders. Activates when sharing state across components without prop drilling, implementing theme/auth/locale providers, or optimizing context performance."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- createContext: TypeScript generics, default value (null pattern with assertion)
- useContext: consuming context, throwing on missing provider
- Provider pattern: value prop, nested providers
- Custom provider components: combining state + context
- Multiple contexts: splitting by concern, composing providers
- Performance: value object stability, useMemo for context value, splitting read/write contexts
- Context + useReducer: dispatch-based context pattern
- React 19: use(Context) as alternative to useContext (can read in conditionals)
- When NOT to use Context: frequently changing values, use external store instead

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include TypeScript patterns for type-safe context
- Include performance optimization section
```

#### Prompt: react-syntax-refs

```
## Task: Create the react-syntax-refs skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-syntax\react-syntax-refs\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (ref patterns and code)
3. references/anti-patterns.md (ref mistakes)

### YAML Frontmatter
---
name: react-syntax-refs
description: "Guides React ref patterns including useRef for DOM access and mutable values, forwardRef with TypeScript generics, useImperativeHandle for exposing component APIs, callback refs, ref cleanup in React 19, and ref as prop in React 19. Activates when accessing DOM elements, forwarding refs to child components, implementing imperative APIs, or managing mutable values across renders."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- useRef: DOM element access, mutable value storage, TypeScript typing (HTMLElement | null)
- useRef for mutable values: .current does NOT trigger re-render, timer IDs, previous values
- forwardRef: TypeScript generics, wrapping components, combining with useImperativeHandle
- useImperativeHandle: exposing limited API, TypeScript interface for handle
- Callback refs: function refs, measuring DOM, conditional refs
- React 19: ref as regular prop (no forwardRef needed), cleanup function from ref callback
- When to use refs vs state: refs for values that don't affect rendering

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Note React 19 changes prominently (ref as prop, cleanup)
- All code examples in TypeScript/TSX
```

#### Prompt: react-syntax-forms

```
## Task: Create the react-syntax-forms skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-syntax\react-syntax-forms\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (form patterns and code)
3. references/patterns.md (form architecture patterns)
4. references/anti-patterns.md (form mistakes)

### YAML Frontmatter
---
name: react-syntax-forms
description: "Guides React form handling including controlled vs uncontrolled inputs, form submission patterns, React 19 form actions with <form action>, useFormStatus for pending state, useActionState for action results, useOptimistic for optimistic UI, file inputs, and validation patterns. Activates when building forms, handling form submission, implementing form validation, or using React 19 form actions."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Controlled inputs: value + onChange, TypeScript typing, all input types
- Uncontrolled inputs: defaultValue, useRef, when to use
- Form submission: onSubmit, preventDefault, FormData
- React 19 form actions: <form action={fn}>, async action functions
- useFormStatus: pending state, data, method, action — MUST be in child component
- useActionState: previous state + form data, pending flag, permalink
- useOptimistic: optimistic updates during async actions
- File inputs: always uncontrolled, FileList handling
- Select and textarea: controlled patterns
- Multi-field forms: single state object vs individual states
- Validation: HTML5 validation, custom validation, error display

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- React 19 form features MUST be clearly marked
- Include decision tree: controlled vs uncontrolled vs form actions
- All code examples in TypeScript/TSX
```

---

### Batch 5

#### Prompt: react-impl-testing

```
## Task: Create the react-impl-testing skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-impl\react-impl-testing\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (test examples for all patterns)
3. references/api-table.md (RTL API reference)
4. references/anti-patterns.md (testing mistakes)

### YAML Frontmatter
---
name: react-impl-testing
description: "Guides React testing with React Testing Library and Vitest including render/screen/fireEvent/waitFor patterns, testing hooks with renderHook, testing async operations, accessibility testing, user-event for realistic interactions, act() for state updates, and snapshot testing. Activates when writing component tests, testing hooks, setting up test infrastructure, or debugging test failures."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript, React Testing Library, and Vitest."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Vitest setup: vitest.config.ts, jsdom environment, setup files
- render: rendering components, wrapper option for providers
- screen: queries (getBy, queryBy, findBy), priority order (role > label > text > testId)
- fireEvent vs @testing-library/user-event: prefer user-event for realistic interaction
- waitFor: testing async operations, timeout configuration
- renderHook: testing custom hooks, result.current
- act(): when needed, wrapping state updates
- Testing patterns: user interaction, form submission, API mocking, error states
- Accessibility testing: role-based queries enforce accessible markup
- Snapshot testing: when to use (sparingly), toMatchSnapshot
- NEVER use Enzyme (deprecated, incompatible with React 18+)
- NEVER test implementation details (internal state, instance methods)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete Vitest setup example
- Include query priority table
- All code examples in TypeScript/TSX
```

#### Prompt: react-impl-performance

```
## Task: Create the react-impl-performance skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-impl\react-impl-performance\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (optimization examples)
3. references/patterns.md (performance patterns)
4. references/anti-patterns.md (performance mistakes and premature optimization)

### YAML Frontmatter
---
name: react-impl-performance
description: "Guides React performance optimization including React.memo for component memoization, useMemo and useCallback for value/function stability, React Compiler automatic memoization (React 19), Profiler for measuring renders, code splitting with React.lazy, bundle analysis, virtualization for long lists, and image optimization patterns. Activates when optimizing render performance, profiling components, reducing bundle size, or implementing code splitting."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- React.memo: when to use, custom comparison, TypeScript typing
- useMemo: memoizing expensive computations, dependency array
- useCallback: stable function references for child components
- React Compiler (React 19): automatic memoization, what it replaces, setup
- Profiler component: measuring render times, onRender callback
- React DevTools Profiler: flamegraph, ranked, why-did-render
- Code splitting: React.lazy + Suspense, route-based splitting, component-based
- Bundle analysis: vite-plugin-visualizer, analyzing chunk sizes
- Virtualization: react-window / @tanstack/virtual for long lists
- Image optimization: lazy loading, srcSet, next/image patterns
- NEVER optimize prematurely: measure first, then optimize

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include decision tree: when to optimize
- React Compiler section must note it's React 19 only
- All code examples in TypeScript/TSX
```

#### Prompt: react-impl-styling

```
## Task: Create the react-impl-styling skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-impl\react-impl-styling\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (styling approach examples)
3. references/anti-patterns.md (styling mistakes)

### YAML Frontmatter
---
name: react-impl-styling
description: "Guides React styling approaches including CSS Modules for scoped styles, Tailwind CSS integration, CSS-in-JS patterns, inline styles with TypeScript, className patterns with clsx/cn, conditional classes, global vs scoped styles, and CSS custom properties. Activates when styling React components, choosing a CSS approach, implementing responsive design, or managing CSS architecture."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- CSS Modules: .module.css, TypeScript declarations, composition
- Tailwind CSS: className patterns, cn/clsx utility, conditional classes, responsive
- Inline styles: React.CSSProperties type, when to use (dynamic values only)
- className patterns: string concatenation, clsx library, cn utility
- Global styles: index.css, CSS custom properties (variables)
- CSS-in-JS overview: styled-components/emotion patterns (brief, not primary recommendation)
- Responsive design: media queries in CSS Modules, Tailwind breakpoints
- Dark mode: CSS custom properties, prefers-color-scheme, Tailwind dark mode
- Decision tree: CSS Modules (default) vs Tailwind (utility-first) vs CSS-in-JS (component libraries)
- React 19: stylesheet precedence with <link precedence="...">

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- CSS Modules as primary recommendation, Tailwind as alternative
- All code examples in TypeScript/TSX
```

---

### Batch 6

#### Prompt: react-impl-server-components

```
## Task: Create the react-impl-server-components skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-impl\react-impl-server-components\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (Server Component patterns)
3. references/patterns.md (Server/Client boundary patterns)
4. references/anti-patterns.md (RSC mistakes)

### YAML Frontmatter
---
name: react-impl-server-components
description: "Guides React Server Components including Server vs Client Component rules, 'use server' and 'use client' directives, Server Actions for mutations, serialization rules across the boundary, component composition across server/client, and framework integration with Next.js. Activates when building with Server Components, deciding server vs client boundaries, implementing Server Actions, or migrating to the Server Component model."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x (partial) or 19.x with a supporting framework (Next.js 13+)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Server Components: default in RSC-supporting frameworks, run on server only
- Client Components: 'use client' directive, required for interactivity/hooks/browser APIs
- Server Actions: 'use server' directive, async functions for mutations, form integration
- Serialization rules: what can cross the server/client boundary (serializable props only)
- Composition patterns: Server Component wrapping Client Component, passing Server Components as children
- When to use Server vs Client: decision tree based on data, interactivity, browser APIs
- Framework integration: Next.js App Router as primary example
- Streaming: Suspense boundaries with Server Components
- Data fetching: async Server Components, direct database/API access
- React 19: stable Server Components support, Server Functions

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include clear decision tree for Server vs Client
- Include serialization rules table
- Note that RSC requires a supporting framework
```

#### Prompt: react-impl-routing

```
## Task: Create the react-impl-routing skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-impl\react-impl-routing\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (routing patterns)
3. references/api-table.md (React Router hooks and components)
4. references/anti-patterns.md (routing mistakes)

### YAML Frontmatter
---
name: react-impl-routing
description: "Guides React Router v6+ patterns including createBrowserRouter, route configuration, nested routes with Outlet, data loaders and actions, lazy routes for code splitting, protected routes, URL parameters, search parameters, and navigation. Activates when implementing client-side routing, setting up route configuration, handling navigation, or building protected routes."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with React Router 6.4+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- createBrowserRouter: route configuration object, RouterProvider
- Route components: element, errorElement, loader, action, lazy
- Nested routes: Outlet component, index routes, layout routes
- Navigation: useNavigate, Link, NavLink (active styling)
- URL parameters: useParams, dynamic segments (:param)
- Search parameters: useSearchParams, reading/updating query strings
- Data loaders: loader function, useLoaderData, defer for streaming
- Route actions: action function, useActionData, form submissions
- Lazy routes: lazy() for route-level code splitting
- Protected routes: auth wrapper pattern, redirect on unauthorized
- Error handling: errorElement, useRouteError
- NEVER use react-router-dom < 6.4 patterns (BrowserRouter with <Routes>)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Use createBrowserRouter as the primary pattern (data router API)
- All code examples in TypeScript/TSX
```

#### Prompt: react-impl-data-fetching

```
## Task: Create the react-impl-data-fetching skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-impl\react-impl-data-fetching\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (data fetching patterns)
3. references/patterns.md (caching and error handling patterns)
4. references/anti-patterns.md (data fetching mistakes)

### YAML Frontmatter
---
name: react-impl-data-fetching
description: "Guides React data fetching patterns including TanStack Query for server state management, useQuery and useMutation patterns, Suspense for loading states, React 19 use() hook for promises, error boundaries for fetch errors, caching strategies, optimistic updates, and pagination patterns. Activates when fetching data from APIs, managing server state, implementing loading/error states, or choosing a data fetching strategy."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- TanStack Query: QueryClientProvider setup, useQuery, useMutation, queryKey conventions
- useQuery: queryKey, queryFn, enabled, staleTime, gcTime, refetch
- useMutation: mutationFn, onSuccess/onError/onSettled, invalidateQueries
- Suspense integration: useSuspenseQuery, Suspense boundaries for loading
- React 19 use() hook: reading promises in render, caching requirements
- Error handling: error boundaries, useQuery error state, retry configuration
- Caching: staleTime, gcTime, prefetching, optimistic updates
- Pagination: useInfiniteQuery, cursor-based pagination
- NEVER fetch in useEffect without proper cleanup (race conditions)
- NEVER manage server state with useState (stale data, no caching)
- Decision tree: TanStack Query (recommended) vs use() + Suspense vs useEffect (avoid)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- TanStack Query as primary recommendation
- Include complete setup example
- All code examples in TypeScript/TSX
```

---

### Batch 7

#### Prompt: react-errors-boundaries

```
## Task: Create the react-errors-boundaries skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-errors\react-errors-boundaries\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (error boundary patterns)
3. references/anti-patterns.md (error boundary mistakes)

### YAML Frontmatter
---
name: react-errors-boundaries
description: "Guides React error boundary implementation including getDerivedStateFromError for fallback UI, componentDidCatch for error logging, error boundary placement strategy, error recovery patterns, nested boundaries, and error boundary limitations. Activates when implementing error handling, creating fallback UI, catching render errors, or designing error recovery in React applications."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Error boundary class component (still REQUIRED — no hook equivalent)
- getDerivedStateFromError: return state update for fallback UI
- componentDidCatch: error logging, error info with component stack
- Placement strategy: page-level, feature-level, component-level
- Error recovery: resetErrorBoundary, key-based reset
- react-error-boundary library: ErrorBoundary component, useErrorBoundary hook
- Nested boundaries: inner catches first, outer as fallback
- What error boundaries DO NOT catch: event handlers, async code, SSR, boundary itself
- TypeScript typing for error boundary components
- Testing error boundaries with React Testing Library

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete reusable ErrorBoundary class component
- Include react-error-boundary library as recommended alternative
- List what boundaries do NOT catch as Critical Warnings
```

#### Prompt: react-errors-hooks

```
## Task: Create the react-errors-hooks skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-errors\react-errors-hooks\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (hook error examples with fixes)
3. references/anti-patterns.md (all common hook mistakes)

### YAML Frontmatter
---
name: react-errors-hooks
description: "Diagnoses and resolves common React hook errors including Rules of Hooks violations, conditional hook calls, missing useEffect dependencies, stale closures, infinite re-render loops, cleanup mistakes, and eslint-plugin-react-hooks warnings. Activates when encountering hook-related errors, infinite loops, stale state issues, or eslint hook warnings."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Rules of Hooks violations: conditional calls, hooks in loops, hooks in regular functions
- eslint-plugin-react-hooks: exhaustive-deps rule, understanding warnings
- Missing dependencies: stale closures, object/array dependencies
- Infinite re-render loops: setState in render, useEffect without deps, object deps
- Stale closures: timer callbacks, event listeners, async operations
- Cleanup mistakes: missing cleanup, cleanup timing, async cleanup
- useEffect fires after paint (not synchronously)
- Common error messages and their solutions
- Diagnostic table: Symptom → Cause → Fix format

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Format as diagnostic table where possible
- Include code examples showing WRONG and CORRECT patterns
- All code examples in TypeScript/TSX
```

#### Prompt: react-errors-debugging

```
## Task: Create the react-errors-debugging skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-errors\react-errors-debugging\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (debugging workflow examples)
3. references/anti-patterns.md (debugging mistakes)

### YAML Frontmatter
---
name: react-errors-debugging
description: "Guides React debugging including React DevTools usage, Strict Mode double-rendering behavior, console warning interpretation, performance profiling with Profiler, component stack traces, common error messages and solutions, and development vs production error differences. Activates when debugging React applications, interpreting console warnings, profiling performance, or understanding Strict Mode behavior."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- React DevTools: Components tab, Profiler tab, highlight updates
- Strict Mode: why components render twice in dev, effects run twice, finding impure components
- Console warnings: "Each child should have a unique key", "Cannot update during render", "Maximum update depth exceeded"
- Component stack traces: reading error stacks, identifying error source
- Performance profiling: DevTools Profiler, flame chart, ranked chart
- Development vs production: error messages stripped in prod, sourcemaps
- Common error messages table: message → cause → fix
- React 18/19: improved error messages, better stack traces in React 19
- Debugging tools: React DevTools, browser DevTools, why-did-you-render

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include common error messages diagnostic table
- Include Strict Mode explanation as prominent section
```

---

### Batch 8

#### Prompt: react-errors-hydration

```
## Task: Create the react-errors-hydration skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-errors\react-errors-hydration\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (hydration error examples with fixes)
3. references/anti-patterns.md (hydration mistakes)

### YAML Frontmatter
---
name: react-errors-hydration
description: "Diagnoses and resolves React hydration mismatch errors including server/client HTML differences, common causes (Date, Math.random, browser APIs, conditional rendering), suppressHydrationWarning usage, debugging hydration errors, and React 19 hydration improvements. Activates when encountering hydration warnings, debugging SSR/SSG mismatches, or implementing server-rendered React applications."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- What hydration is: attaching event handlers to server-rendered HTML
- Hydration mismatch causes: Date/time, Math.random, browser-only APIs (window, localStorage), conditional rendering based on client state, different data on server vs client
- Debugging: React 19 improved hydration error messages with diffs
- Fixes: useEffect for client-only code, dynamic import with ssr: false, isClient pattern
- suppressHydrationWarning: when to use (timestamps, third-party widgets)
- hydrateRoot API: usage, options
- Common hydration errors table: message → cause → fix
- React 19 improvements: better error messages, diff reporting, recoverable errors
- Extension/browser plugin interference causing hydration errors

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Format as diagnostic table
- Include common causes ranked by frequency
```

#### Prompt: react-agents-review

```
## Task: Create the react-agents-review skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-agents\react-agents-review\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (review scenarios: good code, bad code, fixes)
3. references/anti-patterns.md (all anti-patterns consolidated)

### YAML Frontmatter
---
name: react-agents-review
description: "Validates generated React code for correctness by checking Rules of Hooks compliance, TypeScript type safety, performance anti-patterns, accessibility, Server Component boundaries, proper state management, event handling cleanup, and testing patterns. Activates when reviewing React code, validating a React project, checking for common mistakes, or performing code review."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Rules of Hooks: no conditional calls, top-level only, correct dependencies
- TypeScript: proper typing for props, events, refs, state, context
- Performance: unnecessary re-renders, missing memoization where needed, premature optimization
- State management: derived state stored (should compute), state duplication, prop drilling
- Event handling: missing cleanup, stale closures, incorrect typing
- Forms: uncontrolled when should be controlled, missing validation
- Component patterns: prop drilling, God components, missing composition
- Server Components: client code in server components, non-serializable props
- Accessibility: missing aria labels, non-semantic HTML, keyboard navigation
- Testing: implementation detail testing, missing error case tests
- Security: dangerouslySetInnerHTML, XSS vectors, user input handling

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Structure as a checklist grouped by area
- Each check: what to verify, expected correct state, common failure mode
```

#### Prompt: react-agents-project-scaffolder

```
## Task: Create the react-agents-project-scaffolder skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\React-Claude-Skill-Package\skills\source\react-agents\react-agents-project-scaffolder\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/examples.md (complete scaffold output examples)
3. references/patterns.md (project structure patterns)

### YAML Frontmatter
---
name: react-agents-project-scaffolder
description: "Generates complete React project structure including Vite configuration, TypeScript setup, component architecture, routing setup, state management, testing infrastructure, styling approach, and recommended folder structure. Activates when scaffolding a new React project, adding React to an existing project, or generating a feature-complete React application structure."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript and Vite."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Project structure generation: complete file/folder listing
- Vite config: react plugin, path aliases, env vars, build options
- TypeScript config: strict mode, paths, include/exclude
- Component scaffolding: feature-based structure, shared components, layouts
- Routing setup: React Router v6 with createBrowserRouter
- State management: local state default, Zustand for complex apps, TanStack Query for server state
- Testing setup: Vitest config, RTL setup file, example test
- Styling: CSS Modules (default), Tailwind (alternative), global styles
- ESLint + Prettier: configuration files
- Package.json: all dependencies and scripts
- Decision tree: what to include based on project size/requirements
- .gitignore, .env.example, README template

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete file listing with content templates
- Include dependency decision tree
- Generated code MUST follow all patterns from syntax skills
```

---

## Appendix: Skill Directory Structure

```
skills/source/
├── react-core/
│   ├── react-core-architecture/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── examples.md
│   │       ├── api-table.md
│   │       └── anti-patterns.md
│   ├── react-core-state/
│   │   ├── SKILL.md
│   │   └── references/
│   └── react-core-concurrent/
│       ├── SKILL.md
│       └── references/
├── react-syntax/
│   ├── react-syntax-hooks-basic/
│   ├── react-syntax-hooks-advanced/
│   ├── react-syntax-jsx/
│   ├── react-syntax-components/
│   ├── react-syntax-events/
│   ├── react-syntax-context/
│   ├── react-syntax-refs/
│   └── react-syntax-forms/
├── react-impl/
│   ├── react-impl-testing/
│   ├── react-impl-performance/
│   ├── react-impl-server-components/
│   ├── react-impl-routing/
│   ├── react-impl-styling/
│   ├── react-impl-project-setup/
│   └── react-impl-data-fetching/
├── react-errors/
│   ├── react-errors-boundaries/
│   ├── react-errors-hooks/
│   ├── react-errors-hydration/
│   └── react-errors-debugging/
└── react-agents/
    ├── react-agents-review/
    └── react-agents-project-scaffolder/
```
