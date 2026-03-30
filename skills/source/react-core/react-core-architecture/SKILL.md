---
name: react-core-architecture
description: >
  Use when reasoning about React's rendering model, understanding the component
  tree, or debugging unexpected update behavior. Prevents the common mistake of
  confusing React elements with components or misunderstanding render vs commit
  phases. Covers virtual DOM, fiber reconciler, rendering phases, element
  creation, component lifecycle (mount/update/unmount).
  Keywords: virtual DOM, fiber, reconciler, render phase, commit phase, lifecycle, how React works, rendering, virtual DOM, component tree, why does it re-render..
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-core-architecture

## Quick Reference

### Architecture Layers

| Layer | Role | Key Concept |
|-------|------|-------------|
| React Elements | Lightweight descriptions of UI | Immutable objects created by JSX/createElement |
| Components | Functions that return elements | Pure functions of props and state |
| Fiber Tree | Internal work-in-progress tree | Enables incremental rendering and prioritization |
| Reconciler | Diffing algorithm | Compares previous and next element trees |
| Renderer | Platform-specific output | `react-dom` for web, `react-native` for mobile |

### Core Principles

| Principle | Rule |
|-----------|------|
| Unidirectional Data Flow | Data ALWAYS flows from parent to child via props |
| Declarative UI | ALWAYS describe what the UI should look like, NEVER imperatively mutate the DOM |
| Composition over Inheritance | ALWAYS compose components, NEVER use class inheritance for component reuse |
| Pure Rendering | The render phase MUST be a pure function of props and state |
| Immutable Updates | NEVER mutate state or props directly; ALWAYS create new references |

### React Element vs Component

| Concept | What It Is | Example |
|---------|-----------|---------|
| React Element | Immutable plain object describing a DOM node or component | `{ type: 'div', props: { children: 'Hello' } }` |
| Component | Function that accepts props and returns React elements | `function Greeting({ name }: Props) { return <h1>{name}</h1>; }` |
| Fiber | Internal mutable work unit tracking a component instance | Not directly accessible; managed by React internals |

---

## Critical Warnings

**NEVER** mutate state or props during rendering -- rendering MUST be a pure calculation. Mutations cause inconsistent UI and break concurrent features.

**NEVER** rely on render timing or count -- React MAY call your component multiple times, skip renders, or pause and resume rendering. StrictMode double-invokes components in development.

**NEVER** perform side effects in the render phase (network requests, subscriptions, DOM mutations) -- ALWAYS use `useEffect` or event handlers for side effects.

**NEVER** use inheritance to share behavior between components -- ALWAYS use composition (children, render props, or custom hooks).

**NEVER** call `root.render()` where `hydrateRoot()` is needed -- for server-rendered HTML, ALWAYS use `hydrateRoot` to preserve server markup and attach event handlers.

**NEVER** assume synchronous DOM updates after `root.render()` -- rendering is asynchronous. Use `flushSync()` ONLY when synchronous behavior is explicitly required.

---

## Rendering Model

### JSX Compilation

JSX is syntactic sugar for `React.createElement()` calls:

```tsx
// JSX (what you write)
<Greeting name="Taylor" />

// Compiled output (what React sees)
createElement(Greeting, { name: 'Taylor' })
```

The returned React element is an immutable object:

```tsx
{
  type: Greeting,       // Component function or string tag
  props: { name: 'Taylor' },
  key: null,
  ref: null
}
```

**ALWAYS** use capital letters for component names in JSX -- lowercase names resolve to HTML tags, not components.

### Three-Phase Rendering Cycle

React updates the screen in three sequential steps:

| Phase | What Happens | Interruptible? |
|-------|-------------|----------------|
| 1. Trigger | Initial `root.render()` call or a state update via `setState` | N/A |
| 2. Render | React calls component functions and diffs the element tree | Yes (concurrent mode) |
| 3. Commit | React applies minimal DOM mutations to match the new tree | No (synchronous) |

After the commit phase, the **browser paints** the updated screen.

### Render Phase (Pure)

- React calls your component function to produce a new element tree
- Compares the new tree with the previous tree (reconciliation)
- In concurrent mode (React 18+), this phase is **interruptible** -- React can pause, resume, or discard work
- MUST be pure: no side effects, no DOM mutations, no subscriptions

### Commit Phase (Synchronous)

- React applies the **minimal set** of DOM changes identified during reconciliation
- Runs `useLayoutEffect` cleanup and setup synchronously
- The browser paints the screen
- Runs `useEffect` cleanup and setup asynchronously after paint

### Reconciliation Algorithm

React's diffing strategy uses two key heuristics:

1. **Different element types** produce different trees -- React tears down the old subtree and builds a new one
2. **Keys** identify which children remain stable across re-renders -- ALWAYS provide stable keys for list items

```tsx
// React preserves <input> because the element type and position match
<div>
  <input value={text} />  {/* Same position, same type = preserved */}
</div>
```

---

## Fiber Architecture (React 16+)

The Fiber reconciler replaced the legacy stack reconciler to enable:

| Capability | Description |
|-----------|-------------|
| Incremental rendering | Split rendering work into chunks across multiple frames |
| Priority scheduling | Urgent updates (user input) preempt lower-priority work |
| Pause and resume | Interrupt in-progress work without losing progress |
| Concurrent rendering | Prepare multiple UI versions simultaneously (React 18+) |

Each fiber node represents a component instance and contains:

- `type` -- the component function or host element tag
- `stateNode` -- the DOM node (for host elements) or component instance
- `child`, `sibling`, `return` -- tree navigation pointers
- `memoizedState` -- the linked list of hooks for this component
- `pendingProps`, `memoizedProps` -- current and previous props
- `lanes` -- priority bits for scheduling (React 18+)

**NEVER** access fiber internals directly -- they are private implementation details that change between React versions.

---

## Component Lifecycle

### Function Component Lifecycle

```
Mount:      Component called -> Elements created -> DOM inserted -> Effects run
Update:     State/props change -> Component re-called -> Reconciliation -> DOM patched -> Effects re-run
Unmount:    Effect cleanups run -> DOM removed
```

| Phase | What Runs | When |
|-------|----------|------|
| Mount | Component function, then `useEffect` callbacks | First render, after DOM insertion |
| Update | Component function, then `useEffect` cleanups + callbacks (if deps changed) | On state or props change |
| Unmount | `useEffect` cleanup functions | When component is removed from tree |

### Entry Point: createRoot

```tsx
import { createRoot } from 'react-dom/client';
import { StrictMode } from 'react';
import App from './App';

const root = createRoot(document.getElementById('root')!, {
  onCaughtError: (error, errorInfo) => {
    console.error('Caught:', error, errorInfo.componentStack);
  },
  onUncaughtError: (error, errorInfo) => {
    console.error('Uncaught:', error, errorInfo.componentStack);
  },
});

root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

**ALWAYS** wrap the root in `<StrictMode>` during development to detect impure renders, missing effect cleanups, and deprecated APIs.

---

## Component Tree Model

### Render Tree

The render tree represents the component hierarchy for a single render pass:

- **Nodes** are React components (not HTML elements)
- **Root node** is the top-level component passed to `root.render()`
- **Top-level components** near the root affect performance of all descendants
- **Leaf components** at the bottom are frequently re-rendered

The tree changes dynamically with conditional rendering -- different state produces different subtrees.

### Unidirectional Data Flow

```
State (parent) --> Props (child) --> Props (grandchild)
     ^                                      |
     |                                      |
     +-------- Callbacks (events) <---------+
```

Data flows DOWN through props. Communication UP happens through callback functions passed as props. NEVER pass data upward by mutating parent state from a child without using a callback.

---

## StrictMode Behavior (Development Only)

| Check | Method | Purpose |
|-------|--------|---------|
| Impure rendering | Double-invokes component functions | Catches render-phase mutations |
| Missing effect cleanup | Runs setup -> cleanup -> setup cycle | Catches missing cleanup functions |
| Missing ref cleanup | Double ref callback cycle | Catches ref-related memory leaks |
| Deprecated APIs | Static warnings | Flags legacy lifecycle methods |

StrictMode checks run ONLY in development. They have zero impact on production builds.

---

## React 18 vs React 19

| Feature | React 18 | React 19 |
|---------|----------|----------|
| Concurrent rendering | Introduced via `createRoot` | Stable, improved scheduling |
| Server Components | Experimental | Stable |
| `ref` forwarding | Requires `forwardRef()` | `ref` is a regular prop |
| Context Provider | `<MyContext.Provider value={}>` | `<MyContext value={}>` |
| Form handling | Manual state management | Built-in Actions pattern |
| `use()` API | Not available | Reads Promises and Context in render |
| `useActionState` | Not available | Manages async action state |
| `useOptimistic` | Not available | Optimistic UI updates |
| Metadata tags | Require `react-helmet` or similar | Native `<title>`, `<meta>`, `<link>` hoisting |
| Error callbacks | `onRecoverableError` only | `onCaughtError`, `onUncaughtError`, `onRecoverableError` |

---

## Reference Links

- [references/examples.md](references/examples.md) -- Working code examples for rendering, lifecycle, and tree structure
- [references/api-table.md](references/api-table.md) -- React core type reference and API signatures
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do, with explanations

### Official Sources

- https://react.dev/learn/render-and-commit
- https://react.dev/learn/understanding-your-ui-as-a-tree
- https://react.dev/reference/react/createElement
- https://react.dev/reference/react-dom/client/createRoot
- https://react.dev/reference/react/StrictMode
- https://react.dev/blog/2024/04/25/react-19
