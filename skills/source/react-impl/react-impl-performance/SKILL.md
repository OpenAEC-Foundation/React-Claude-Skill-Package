---
name: react-impl-performance
description: >
  Use when optimizing render performance, profiling components, reducing bundle
  size, or implementing code splitting. Prevents the common mistake of premature
  optimization or missing obvious re-render bottlenecks. Covers React.memo,
  useMemo, useCallback, React Compiler (React 19), Profiler, React.lazy, bundle
  analysis, virtualization, image optimization.
  Keywords: React.memo, useMemo, useCallback, code splitting, Profiler, lazy.
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-impl-performance

## Quick Reference

### Performance Optimization Tools

| Tool | Purpose | React Version | When to Use |
|------|---------|---------------|-------------|
| `React.memo` | Skip re-renders when props unchanged | 18 + 19 | Expensive component, same props frequently |
| `useMemo` | Cache expensive computation results | 18 + 19 | Calculation >1ms on target hardware |
| `useCallback` | Stable function reference for children | 18 + 19 | Function prop to `memo`-wrapped child |
| React Compiler | Automatic memoization at build time | 19+ | Replaces manual memo/useMemo/useCallback |
| `<Profiler>` | Measure render durations | 18 + 19 | Identify slow components |
| React DevTools | Visual profiling (flamegraph, ranked) | 18 + 19 | Interactive performance investigation |
| `React.lazy` | Code splitting per route/component | 18 + 19 | Reduce initial bundle size |
| `@tanstack/virtual` | Virtualize long lists | 18 + 19 | Lists with 1000+ items |

### Critical Warnings

**NEVER** optimize before measuring. ALWAYS use the Profiler component or React DevTools to identify the actual bottleneck first. Premature optimization adds complexity without measurable benefit.

**NEVER** wrap every component in `React.memo`. The shallow comparison itself has a cost. ONLY use `memo` when a component re-renders frequently with the same props AND rendering is noticeably slow.

**NEVER** use `JSON.stringify` in a custom `arePropsEqual` function for `React.memo`. This is slower than just re-rendering the component.

**NEVER** rely on `useMemo` as a semantic guarantee. React MAY discard cached values (on suspend, during development). Use `useRef` if you need a persistent reference.

---

## Decision Tree: When to Optimize

```
Component renders slowly?
├── NO → STOP. Do not optimize.
└── YES → Measure with Profiler/DevTools
    ├── Re-renders with same props?
    │   ├── Props are primitives → Use React.memo
    │   ├── Props include objects → useMemo the object, then React.memo
    │   └── Props include functions → useCallback the function, then React.memo
    ├── Expensive computation during render?
    │   └── Use useMemo with dependency array
    ├── Large bundle size / slow initial load?
    │   ├── Route-based → React.lazy + Suspense
    │   └── Feature-based → Dynamic import + React.lazy
    ├── Long list (1000+ items)?
    │   └── Use @tanstack/virtual
    └── Using React 19?
        └── Enable React Compiler → removes need for manual memo/useMemo/useCallback
```

---

## React.memo

Wraps a component to skip re-rendering when props have not changed (shallow `Object.is` comparison per prop).

```tsx
import { memo } from 'react';

interface ExpensiveListProps {
  items: readonly string[];
  onSelect: (item: string) => void;
}

const ExpensiveList = memo<ExpensiveListProps>(function ExpensiveList({
  items,
  onSelect,
}) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item} onClick={() => onSelect(item)}>{item}</li>
      ))}
    </ul>
  );
});
```

**ALWAYS** ensure props passed to a `memo` component are referentially stable. Passing a new object or function literal on every render defeats `memo` entirely.

`memo` does NOT prevent re-renders caused by:
- Internal state changes (`useState`, `useReducer`)
- Context value changes (`useContext`)

---

## useMemo

Caches the result of an expensive calculation between re-renders.

```tsx
import { useMemo } from 'react';

function FilteredList({ items, query }: { items: Item[]; query: string }) {
  const filtered = useMemo(
    () => items.filter((item) => item.name.includes(query)),
    [items, query]
  );

  return <ul>{filtered.map((item) => <li key={item.id}>{item.name}</li>)}</ul>;
}
```

**ALWAYS** include every reactive value used inside the calculation in the dependency array. **NEVER** omit the dependency array — this recalculates every render, defeating the purpose.

---

## useCallback

Returns a stable function reference between re-renders. Equivalent to `useMemo(() => fn, deps)`.

```tsx
import { useCallback } from 'react';

function Parent({ items }: { items: Item[] }) {
  const handleSelect = useCallback((id: string) => {
    console.log('Selected:', id);
  }, []);

  return <MemoizedChild items={items} onSelect={handleSelect} />;
}
```

**ALWAYS** use updater functions to remove state from the dependency array:

```tsx
// WRONG: todos changes every update, useCallback is useless
const handleAdd = useCallback((text: string) => {
  setTodos([...todos, { id: nextId++, text }]);
}, [todos]);

// CORRECT: updater removes todos dependency
const handleAdd = useCallback((text: string) => {
  setTodos((prev) => [...prev, { id: nextId++, text }]);
}, []);
```

---

## React Compiler (React 19+)

The React Compiler automatically applies memoization at build time, replacing manual `memo`, `useMemo`, and `useCallback`. When enabled, you do NOT need to write these manually.

### Setup (Vite)

```bash
npm install -D babel-plugin-react-compiler@latest
```

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: ['babel-plugin-react-compiler'],
      },
    }),
  ],
});
```

The babel plugin MUST run **first** in the pipeline. See [references/patterns.md](references/patterns.md) for Next.js and Webpack setup.

### Opt-Out Directive

```tsx
function ProblematicComponent() {
  "use no memo";
  // Compiler skips this component
  return <div>Not compiled</div>;
}
```

### Verification

In React DevTools, compiled components show a "Memo" badge with a sparkle icon.

**ALWAYS** install `eslint-plugin-react-hooks@latest` — it identifies Rules of React violations that prevent the compiler from optimizing a component.

---

## Profiler Component

Measures render performance programmatically. ALWAYS use this to identify bottlenecks before optimizing.

```tsx
import { Profiler } from 'react';

function onRender(
  id: string,
  phase: 'mount' | 'update' | 'nested-update',
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number,
): void {
  console.log(`${id} [${phase}]: ${actualDuration.toFixed(2)}ms (base: ${baseDuration.toFixed(2)}ms)`);
}

function App() {
  return (
    <Profiler id="Dashboard" onRender={onRender}>
      <Dashboard />
    </Profiler>
  );
}
```

| Callback Parameter | Meaning |
|-------------------|---------|
| `actualDuration` | Time spent rendering this commit (memoization benefit visible here) |
| `baseDuration` | Time to render without any memoization (worst case) |
| `phase` | `'mount'` = first render, `'update'` = re-render |

Compare `actualDuration` vs `baseDuration` to measure memoization effectiveness.

**Caveats**: Disabled in production builds by default. Use a profiling build for production measurement.

---

## React DevTools Profiler

Use the browser extension Profiler tab for interactive investigation:

1. **Flamegraph** — visual tree of render durations. Gray = did not render (memoized)
2. **Ranked chart** — components sorted by render time (longest first)
3. **"Why did this render?"** — enable in Profiler settings to see the exact cause

**ALWAYS** check "Why did this render?" before adding memoization. Common causes:
- Parent re-rendered (fix with `memo`)
- Props changed (check referential equality)
- Context changed (split contexts or use selectors)
- Hook state changed (expected behavior)

---

## Code Splitting with React.lazy

Split code by route or feature to reduce initial bundle size.

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

**ALWAYS** use route-based splitting as the primary strategy — it provides the biggest impact with the least effort.

**NEVER** place `lazy()` calls inside a component. ALWAYS declare them at module level to prevent recreating the lazy component on every render.

---

## Bundle Analysis

Use `rollup-plugin-visualizer` (Vite) to identify large chunks:

```bash
npm install -D rollup-plugin-visualizer
```

```ts
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true, filename: 'stats.html' }),
  ],
});
```

Run `npm run build` and inspect `stats.html` to find oversized dependencies.

---

## Virtualization for Long Lists

NEVER render 1000+ DOM nodes. Use `@tanstack/react-virtual` to render only visible items.

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function VirtualList({ items }: { items: string[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 35,
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {items[virtualRow.index]}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Reference Links

- [references/examples.md](references/examples.md) — Complete optimization examples with before/after code
- [references/patterns.md](references/patterns.md) — Performance patterns: React Compiler setup, context optimization, lazy loading strategies
- [references/anti-patterns.md](references/anti-patterns.md) — Common performance mistakes and premature optimization traps

### Official Sources

- https://react.dev/reference/react/memo
- https://react.dev/reference/react/useMemo
- https://react.dev/reference/react/useCallback
- https://react.dev/reference/react/Profiler
- https://react.dev/reference/react/lazy
- https://react.dev/learn/react-compiler
