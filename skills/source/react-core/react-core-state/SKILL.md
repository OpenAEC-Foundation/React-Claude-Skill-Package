---
name: react-core-state
description: >
  Use when designing state architecture, choosing between state management
  approaches, or solving state-related problems. Prevents the common mistake of
  mutating state directly or choosing the wrong state tool for the complexity
  level. Covers useState vs useReducer, state lifting, Context API, external
  stores (Zustand/Jotai), immutability rules, server vs client state.
  Keywords: useState, useReducer, Zustand, Context, state lifting, immutability.
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-core-state

## Quick Reference

### State Tool Selection

| Scenario | Tool | Why |
|----------|------|-----|
| Single primitive or simple object | `useState` | Minimal boilerplate, direct updates |
| Complex object with multiple sub-values | `useReducer` | Centralized transitions, predictable logic |
| State needed by distant descendants | Context + `useState`/`useReducer` | Avoids prop drilling |
| High-frequency updates read by many components | External store (Zustand/Jotai) | Bypasses Context re-render cascade |
| Server data (fetched, cached, synced) | TanStack Query / SWR | Handles caching, revalidation, deduplication |
| Optimistic UI during async mutation | `useOptimistic` (React 19) | Instant feedback, automatic rollback |
| Form submission state | `useActionState` (React 19) | Tracks pending state, works with Server Actions |
| Subscribing to non-React store | `useSyncExternalStore` | Tear-free reads, SSR-safe |

### State Placement Decision

| Question | If YES | If NO |
|----------|--------|-------|
| Only this component needs it? | Local `useState` | Keep reading |
| Parent and siblings need it? | Lift to closest common ancestor | Keep reading |
| Many distant components need it? | Context or external store | Keep reading |
| Updates are frequent (>60fps)? | External store with selectors | Context is fine |
| Data comes from the server? | TanStack Query / server state lib | Client state |

---

## Critical Warnings

**NEVER** mutate state directly. `state.items.push(item)` does NOT trigger a re-render. ALWAYS create a new reference: `setState(prev => [...prev.items, item])`.

**NEVER** store derived values in state. If `fullName` can be computed from `firstName` and `lastName`, compute it during render. Storing it creates sync bugs.

**NEVER** call `useState` or `useReducer` inside loops, conditions, or nested functions. React relies on call order to track state identity.

**NEVER** read state immediately after calling `setState` and expect the new value. State updates apply on the NEXT render.

**NEVER** use Context for high-frequency updates (mouse position, animations). Every consumer re-renders when the context value changes. Use an external store with selectors instead.

**NEVER** create a new context value object on every render without `useMemo`. This defeats `React.memo` on all consumers.

**ALWAYS** use the updater form `setState(prev => prev + 1)` when the next state depends on the previous state. Direct `setState(count + 1)` causes stale closures in batched updates.

**ALWAYS** use an initializer function for expensive initial state: `useState(() => computeExpensive())`, NOT `useState(computeExpensive())`.

---

## Decision Tree

```
Need to manage data in a React component?
|
+-- Is it server data (API responses, DB records)?
|   YES --> Use TanStack Query or SWR (server state library)
|   NO  |
|       v
+-- Is it a single value or simple object?
|   YES --> useState
|   NO  |
|       v
+-- Does it have complex transitions (multiple fields change together)?
|   YES --> useReducer
|   NO  --> useState with object spread
|
After choosing the hook:
|
+-- Does only THIS component need it?
|   YES --> Keep it local
|   NO  |
|       v
+-- Do a parent and a few siblings need it?
|   YES --> Lift state to closest common ancestor
|   NO  |
|       v
+-- Do many distant components need it?
    |
    +-- Are updates infrequent (theme, locale, auth)?
    |   YES --> Context API
    |   NO  |
    |       v
    +-- Are updates frequent or need fine-grained subscriptions?
        YES --> External store (Zustand, Jotai)
```

---

## State Patterns

### useState -- Simple Local State

```typescript
const [count, setCount] = useState<number>(0);
const [user, setUser] = useState<User | null>(null);

// Updater form -- ALWAYS use when depending on previous state
setCount(prev => prev + 1);

// Lazy initializer -- ALWAYS use for expensive computations
const [data, setData] = useState<ExpensiveData>(() => parseExpensiveData(raw));
```

### useReducer -- Complex State Logic

Use `useReducer` when: multiple state fields change in response to a single event, or state transitions follow business rules.

```typescript
type State = { items: Item[]; status: "idle" | "loading" | "error" };
type Action =
  | { type: "FETCH_START" }
  | { type: "FETCH_SUCCESS"; items: Item[] }
  | { type: "FETCH_ERROR" };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "FETCH_START":
      return { ...state, status: "loading" };
    case "FETCH_SUCCESS":
      return { items: action.items, status: "idle" };
    case "FETCH_ERROR":
      return { ...state, status: "error" };
  }
}

const [state, dispatch] = useReducer(reducer, { items: [], status: "idle" });
```

### Context API -- Cross-Cutting State

See [references/patterns.md](references/patterns.md) for full Context patterns with `useMemo` optimization.

```typescript
// 1. Create context with typed default
const ThemeContext = createContext<Theme>("light");

// 2. Provider with memoized value
function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>("light");
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return <ThemeContext value={value}>{children}</ThemeContext>;
}

// 3. Consumer hook
function useTheme() {
  return useContext(ThemeContext);
}
```

### External Stores -- High-Performance Global State

Use when Context re-renders too many components. See [references/patterns.md](references/patterns.md) for Zustand/Jotai examples.

```typescript
// Zustand -- minimal API, selector-based subscriptions
import { create } from "zustand";

const useStore = create<StoreState>((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
}));

// Component only re-renders when count changes
function Counter() {
  const count = useStore((s) => s.count);
  return <span>{count}</span>;
}
```

### useSyncExternalStore -- Non-React Store Subscription

```typescript
function useOnlineStatus(): boolean {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}

function subscribe(callback: () => void) {
  window.addEventListener("online", callback);
  window.addEventListener("offline", callback);
  return () => {
    window.removeEventListener("online", callback);
    window.removeEventListener("offline", callback);
  };
}

function getSnapshot(): boolean { return navigator.onLine; }
function getServerSnapshot(): boolean { return true; }
```

---

## Anti-Patterns

See [references/anti-patterns.md](references/anti-patterns.md) for complete anti-pattern catalog with fixes.

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Direct mutation | No re-render | Create new reference |
| Storing derived state | Sync bugs | Compute during render |
| Props in state (mirror) | Stale data | Use props directly or key reset |
| Context for frequent updates | Re-render cascade | External store with selectors |
| Missing updater form | Stale closures | `setState(prev => ...)` |
| Overly broad context | Unrelated re-renders | Split into focused contexts |

---

## Version Notes (React 18 vs 19)

### React 18
- `useState`, `useReducer`, `useContext`, `useSyncExternalStore` available
- Automatic batching in event handlers, timeouts, promises (new in 18)
- `startTransition` for non-urgent updates

### React 19 Additions
- **`useActionState`**: Replaces `useFormState`. Returns `[state, action, isPending]`. Supports async reducers with side effects. Works with Server Actions and `<form action={...}>`.
- **`useOptimistic`**: Immediate UI updates during async actions. Automatically rolls back on failure. Must be called inside a Transition or Action.
- **Context as provider**: Use `<Context value={...}>` directly instead of `<Context.Provider value={...}>` (Provider syntax still works but is deprecated).
- **Server Components**: Server Components have NO state. They render once on the server. ALWAYS place stateful logic in Client Components (`"use client"`).

---

## Reference Links

- [references/examples.md](references/examples.md) -- State management code examples with TypeScript
- [references/patterns.md](references/patterns.md) -- State architecture patterns (Context, external stores, server state)
- [references/anti-patterns.md](references/anti-patterns.md) -- State management mistakes and fixes

### Official Sources

- https://react.dev/learn/managing-state
- https://react.dev/reference/react/useState
- https://react.dev/reference/react/useReducer
- https://react.dev/reference/react/useContext
- https://react.dev/reference/react/useSyncExternalStore
- https://react.dev/reference/react/useActionState
- https://react.dev/reference/react/useOptimistic
