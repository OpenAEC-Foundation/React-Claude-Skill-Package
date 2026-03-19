# Advanced Hooks — API Reference Table

## React 18+ Hooks

### useId

```typescript
function useId(): string;
```

| Aspect | Detail |
|--------|--------|
| Parameters | None |
| Returns | `string` — unique ID stable across re-renders and SSR hydration |
| Re-render trigger | Never (ID is stable) |
| SSR behavior | Generates matching IDs on server and client |

---

### useTransition

```typescript
function useTransition(): [
  isPending: boolean,
  startTransition: (action: () => void | Promise<void>) => void
];
```

| Aspect | Detail |
|--------|--------|
| Parameters | None |
| Returns | `[isPending, startTransition]` |
| `isPending` | `true` while transition is in progress |
| `startTransition` | Stable identity; callback executes immediately |
| Re-render trigger | When `isPending` changes |

---

### useDeferredValue

```typescript
// React 18
function useDeferredValue<T>(value: T): T;

// React 19
function useDeferredValue<T>(value: T, initialValue?: T): T;
```

| Aspect | Detail |
|--------|--------|
| `value` | Any type — the value to defer |
| `initialValue` | (React 19 only) Value used on initial render |
| Returns | `T` — deferred copy that lags behind during urgent updates |
| Comparison | `Object.is` to detect changes |
| Re-render trigger | Two-phase: immediate with old value, then background with new value |

---

### useSyncExternalStore

```typescript
function useSyncExternalStore<T>(
  subscribe: (callback: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T
): T;
```

| Parameter | Type | Requirement |
|-----------|------|-------------|
| `subscribe` | `(callback: () => void) => () => void` | MUST be stable (outside component or `useCallback`) |
| `getSnapshot` | `() => T` | MUST return immutable/cached data |
| `getServerSnapshot` | `() => T` (optional) | MUST match between server and client hydration |

| Aspect | Detail |
|--------|--------|
| Returns | `T` — current store snapshot |
| Re-render trigger | When `getSnapshot()` returns different value (via `Object.is`) |

---

### useInsertionEffect

```typescript
function useInsertionEffect(
  setup: () => (void | (() => void)),
  dependencies?: ReadonlyArray<unknown>
): void;
```

| Aspect | Detail |
|--------|--------|
| Timing | Before DOM mutations (before `useLayoutEffect`) |
| `setState` | NOT allowed inside |
| Refs | NOT attached yet |
| SSR | Does NOT run |
| Intended users | CSS-in-JS library authors ONLY |

---

### useDebugValue

```typescript
function useDebugValue<T>(value: T, format?: (value: T) => any): void;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `value` | `T` | Value shown in React DevTools |
| `format` | `(value: T) => any` (optional) | Lazy formatter; only called when inspected |

| Aspect | Detail |
|--------|--------|
| Returns | `void` |
| Purpose | Label custom hooks in DevTools |
| Performance | Formatter defers computation until inspection |

---

## React 19 Hooks

### use

```typescript
function use<T>(resource: Promise<T>): T;
function use<T>(context: React.Context<T>): T;
```

| Resource Type | Behavior |
|---------------|----------|
| `Promise<T>` | Suspends component until resolved; integrates with `<Suspense>` |
| `React.Context<T>` | Returns context value; CAN be called conditionally |

| Aspect | Detail |
|--------|--------|
| Conditionals | ALLOWED (unique among hooks) |
| Loops | ALLOWED |
| `try`/`catch` | NOT allowed — use Error Boundary |
| Promise errors | Caught by nearest Error Boundary |

---

### useActionState

```typescript
function useActionState<State>(
  action: (state: State, payload: unknown) => State | Promise<State>,
  initialState: State,
  permalink?: string
): [state: State, dispatch: (payload: unknown) => void, isPending: boolean];
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `action` | `(state: S, payload: P) => S \| Promise<S>` | Reducer-like; can be async with side effects |
| `initialState` | `S` | Initial state; MUST be serializable for Server Functions |
| `permalink` | `string` (optional) | URL for progressive enhancement before JS loads |

| Return | Type | Description |
|--------|------|-------------|
| `state` | `S` | Current state |
| `dispatch` | `(payload: P) => void` | Triggers action; stable identity |
| `isPending` | `boolean` | `true` while action executes |

---

### useOptimistic

```typescript
function useOptimistic<State, Action>(
  passthrough: State,
  reducer?: (currentState: State, action: Action) => State
): [State, (action: Action) => void];
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `passthrough` | `State` | Real state; returned when no Action is pending |
| `reducer` | `(current: S, action: A) => S` (optional) | Pure function to compute optimistic state |

| Return | Type | Description |
|--------|------|-------------|
| `optimisticState` | `State` | Optimistic value during action; real value otherwise |
| `setOptimistic` | `(action: Action) => void` | MUST be called inside `startTransition` or Action |

---

### useFormStatus

```typescript
// Import from 'react-dom', NOT 'react'
function useFormStatus(): {
  pending: boolean;
  data: FormData | null;
  method: 'get' | 'post';
  action: ((...args: any[]) => any) | null;
};
```

| Property | Type | Description |
|----------|------|-------------|
| `pending` | `boolean` | `true` when parent `<form>` is submitting |
| `data` | `FormData \| null` | Form data being submitted; `null` when idle |
| `method` | `'get' \| 'post'` | HTTP method of the parent form |
| `action` | `function \| null` | Reference to form's `action` prop function |

| Aspect | Detail |
|--------|--------|
| Parameters | None |
| Requirement | Component MUST be rendered inside a `<form>` |
| Import | `import { useFormStatus } from 'react-dom'` |

---

## Effect Execution Order

| Hook | Fires | Blocks Paint | Use Case |
|------|-------|-------------|----------|
| `useInsertionEffect` | Before DOM mutations | Yes | CSS-in-JS style injection |
| `useLayoutEffect` | After DOM mutations, before paint | Yes | DOM measurement |
| `useEffect` | After paint | No | Side effects, subscriptions |
