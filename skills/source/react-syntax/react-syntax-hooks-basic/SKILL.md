---
name: react-syntax-hooks-basic
description: >
  Use when using any core React hook, writing side effects, managing component
  state, or optimizing re-renders. Prevents the common mistake of missing
  dependency array entries or forgetting useEffect cleanup. Covers useState,
  useEffect, useContext, useRef, useMemo, useCallback, useReducer, Rules of
  Hooks, dependency arrays, cleanup patterns.
  Keywords: useState, useEffect, useRef, useMemo, useCallback, useReducer, hooks.
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-syntax-hooks-basic

## Quick Reference

### The 7 Essential Hooks

| Hook | Purpose | Returns |
|------|---------|---------|
| `useState<S>` | Local component state | `[state, setState]` |
| `useEffect` | Side effects after render | `void` |
| `useContext<T>` | Consume context value | `T` |
| `useRef<T>` | Mutable ref / DOM access | `{ current: T }` |
| `useMemo<T>` | Memoize expensive values | `T` |
| `useCallback<T>` | Memoize function references | `T` |
| `useReducer<S, A>` | Complex state with actions | `[state, dispatch]` |

### Import

```typescript
import { useState, useEffect, useContext, useRef, useMemo, useCallback, useReducer } from 'react';
```

### Critical Warnings — Rules of Hooks

**NEVER** call hooks inside loops, conditions, nested functions, `try`/`catch`/`finally` blocks, or after conditional `return` statements. React relies on consistent call order to maintain state correctly.

**NEVER** call hooks from regular JavaScript functions. ALWAYS call hooks from React function components or custom hooks (functions starting with `use`).

**ALWAYS** call hooks at the top level of your function component or custom hook — before any early returns.

**ALWAYS** use `eslint-plugin-react-hooks` to catch violations automatically.

```typescript
// WRONG — conditional hook call
function Profile({ userId }: { userId: string | null }) {
  if (!userId) return null;
  const [user, setUser] = useState<User | null>(null); // VIOLATION
}

// CORRECT — hooks before early return
function Profile({ userId }: { userId: string | null }) {
  const [user, setUser] = useState<User | null>(null);
  if (!userId) return null;
  // ... rest of component
}
```

---

## useState — Local State

### Signature

```typescript
const [state, setState] = useState<S>(initialState: S | (() => S));
```

### Decision Tree

- Need state that triggers re-render on change? → `useState`
- Initial value is expensive to compute? → Pass a **function**: `useState(() => computeExpensive())`
- State depends on previous state? → Use **updater function**: `setState(prev => prev + 1)`
- State is an object or array? → ALWAYS create a new reference when updating

### Key Rules

- `setState` is **stable** across renders — safe to omit from dependency arrays.
- State updates are **batched** — the screen updates after all event handlers complete.
- `Object.is()` comparison — identical values skip re-render.
- Calling `setState` does NOT update the variable in the currently executing code.

### TypeScript Typing

```typescript
const [count, setCount] = useState<number>(0);
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<string[]>([]);
```

**React 18 vs 19**: No API changes. Behavior consistent across versions.

---

## useEffect — Side Effects

### Signature

```typescript
useEffect(setup: () => (void | (() => void)), dependencies?: unknown[]): void;
```

### Dependency Array Behavior

| Pattern | Runs when | Use case |
|---------|-----------|----------|
| `[dep1, dep2]` | Any dependency changes | Most common — sync to specific values |
| `[]` | Only on mount (cleanup on unmount) | One-time setup (subscriptions, timers) |
| Omitted | Every commit | Rarely needed — usually a mistake |

### Decision Tree

- Subscribing to external system? → `useEffect` with cleanup
- Fetching data? → `useEffect` with ignore flag for race conditions
- Need effect before paint? → Use `useLayoutEffect` instead
- Transforming data for render? → Do it during render, NOT in an effect

### Key Rules

**NEVER** pass an `async` function directly as the effect callback. Effects must return `void` or a cleanup function — `async` returns a `Promise`.

```typescript
// WRONG
useEffect(async () => {
  const data = await fetchData();
}, []);

// CORRECT
useEffect(() => {
  let ignore = false;
  async function fetchData() {
    const result = await api.getData(id);
    if (!ignore) setData(result);
  }
  fetchData();
  return () => { ignore = true; };
}, [id]);
```

**ALWAYS** return a cleanup function when your effect creates subscriptions, timers, or event listeners.

```typescript
useEffect(() => {
  const handler = (e: KeyboardEvent) => { /* ... */ };
  window.addEventListener('keydown', handler);
  return () => window.removeEventListener('keydown', handler);
}, []);
```

- Runs AFTER browser paint (non-blocking).
- Does NOT run during SSR.
- StrictMode (dev): runs setup → cleanup → setup to catch missing cleanups.

**React 18 vs 19**: No API changes.

---

## useContext — Context Consumption

### Signature

```typescript
const value = useContext<T>(context: React.Context<T>): T;
```

### Key Rules

- Returns the value from the **closest** matching `<Provider>` above in the tree.
- Falls back to `defaultValue` from `createContext` if no provider found.
- A provider in the **same** component does NOT affect `useContext` in that component.
- React re-renders ALL consumers when the provider value changes (`Object.is` comparison).
- `memo()` does NOT prevent receiving fresh context values.

### TypeScript Typing

```typescript
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | null>(null);

function useTheme(): ThemeContextType {
  const ctx = useContext(ThemeContext);
  if (ctx === null) throw new Error('useTheme must be used within ThemeProvider');
  return ctx;
}
```

**React 18 vs 19**: In React 19, use `<Context value={...}>` directly. In React 18, use `<Context.Provider value={...}>`.

---

## useRef — Refs and Mutable Values

### Signature

```typescript
const ref = useRef<T>(initialValue: T): { current: T };
```

### Decision Tree

- Need a DOM element reference? → `useRef<HTMLElement>(null)`
- Need a mutable value that does NOT trigger re-render? → `useRef`
- Need to store a timeout/interval ID? → `useRef`
- Need a value that triggers re-render on change? → Use `useState` instead

### Key Rules

- Changing `.current` does **NOT** trigger a re-render.
- **NEVER** read or write `.current` during render (breaks component purity). Use it in event handlers and effects only.
- Same object identity across renders.

### TypeScript Typing

```typescript
const inputRef = useRef<HTMLInputElement>(null);
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
const renderCount = useRef<number>(0);
```

**ALWAYS** type DOM refs as `HTMLElement | null` with initial value `null`:

```typescript
const divRef = useRef<HTMLDivElement>(null);
```

**React 18 vs 19**: No API changes.

---

## useMemo — Memoize Values

### Signature

```typescript
const memoized = useMemo<T>(factory: () => T, deps: unknown[]): T;
```

### Decision Tree

- Calculation is noticeably slow AND dependencies rarely change? → `useMemo`
- Value is passed to a `memo()`-wrapped child? → `useMemo`
- Value is used as a dependency of another hook? → `useMemo`
- Calculation is trivial? → Do NOT use `useMemo` — the overhead outweighs the benefit

### Key Rules

- Factory must be a **pure** function with no arguments.
- Dependencies compared via `Object.is`.
- React MAY discard the cache — do NOT rely on it as a semantic guarantee.
- React Compiler (React 19) auto-memoizes, reducing need for manual `useMemo`.

**React 18 vs 19**: React Compiler in 19 makes manual `useMemo` optional in many cases.

---

## useCallback — Memoize Functions

### Signature

```typescript
const memoized = useCallback<T extends (...args: any[]) => any>(fn: T, deps: unknown[]): T;
```

### Decision Tree

- Passing a function to a `memo()`-wrapped child? → `useCallback`
- Function is a dependency of `useEffect` or another hook? → `useCallback`
- Function is only used in event handlers, not passed down? → Do NOT use `useCallback`

### Equivalence with useMemo

```typescript
useCallback(fn, deps)  ===  useMemo(() => fn, deps)
```

### Updater Pattern — Remove State Dependencies

```typescript
// WRONG — deps change every update
const handleAdd = useCallback((text: string) => {
  setTodos([...todos, createTodo(text)]);
}, [todos]);

// CORRECT — empty deps, stable reference
const handleAdd = useCallback((text: string) => {
  setTodos(prev => [...prev, createTodo(text)]);
}, []);
```

**React 18 vs 19**: React Compiler in 19 makes manual `useCallback` optional in many cases.

---

## useReducer — Complex State

### Signature

```typescript
const [state, dispatch] = useReducer<S, A>(
  reducer: (state: S, action: A) => S,
  initialArg: S,
  init?: (arg: S) => S
): [S, Dispatch<A>];
```

### Decision Tree

- State has multiple sub-values updated together? → `useReducer`
- State transitions depend on complex logic? → `useReducer`
- Need to pass state update logic down without prop drilling? → `useReducer` + context
- Simple boolean or single value? → `useState` is simpler

### Key Rules

- Reducer MUST be **pure** — NEVER mutate state, ALWAYS return a new object.
- `dispatch` is **stable** — safe to omit from dependency arrays.
- Convention: actions are objects with a `type` property.
- Lazy init: pass `init` function as third argument to avoid recreating initial state.

### TypeScript Typing

```typescript
type State = { count: number; step: number };
type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setStep'; payload: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + state.step };
    case 'decrement': return { ...state, count: state.count - state.step };
    case 'setStep':   return { ...state, step: action.payload };
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });
```

**React 18 vs 19**: No API changes.

---

## Dependency Array Rules

### Referential Equality

Dependencies are compared with `Object.is`. For primitives this is value comparison. For objects, arrays, and functions it is **reference** comparison.

| Dependency type | Stable across renders? | Solution if unstable |
|----------------|----------------------|---------------------|
| Primitive (string, number, boolean) | Yes (same value = same) | No action needed |
| Object/array created in render | No (new reference each render) | `useMemo` or move outside |
| Function created in render | No (new reference each render) | `useCallback` or move outside |
| State setter (`setState`) | Yes (stable identity) | Safe to omit |
| `dispatch` from `useReducer` | Yes (stable identity) | Safe to omit |
| Ref object from `useRef` | Yes (stable identity) | Safe to omit |

### Common Mistakes

**NEVER** pass an object or array literal as a dependency — it creates a new reference every render, causing the effect to run on every render:

```typescript
// WRONG — runs every render
useEffect(() => { /* ... */ }, [{ id: userId }]);

// CORRECT — use the primitive value
useEffect(() => { /* ... */ }, [userId]);
```

**NEVER** omit dependencies to "fix" infinite loops — fix the root cause (unstable references) instead.

---

## Reference Links

- [references/examples.md](references/examples.md) — Working code examples for all 7 hooks
- [references/api-table.md](references/api-table.md) — Complete hook signatures and return types
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do, with explanations

### Official Sources

- https://react.dev/reference/react/useState
- https://react.dev/reference/react/useEffect
- https://react.dev/reference/react/useContext
- https://react.dev/reference/react/useRef
- https://react.dev/reference/react/useMemo
- https://react.dev/reference/react/useCallback
- https://react.dev/reference/react/useReducer
- https://react.dev/reference/rules/rules-of-hooks
