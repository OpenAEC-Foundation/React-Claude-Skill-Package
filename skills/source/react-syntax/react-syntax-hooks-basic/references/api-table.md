# Hook Signatures and Return Types

> Complete TypeScript signatures for the 7 essential React hooks.
> Source: https://react.dev/reference/react/

---

## useState

```typescript
function useState<S>(initialState: S | (() => S)): [S, Dispatch<SetStateAction<S>>];

type SetStateAction<S> = S | ((prevState: S) => S);
type Dispatch<A> = (value: A) => void;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `initialState` | `S \| (() => S)` | Initial value or lazy initializer (called once on mount) |

| Return | Type | Description |
|--------|------|-------------|
| `[0]` state | `S` | Current state value |
| `[1]` setState | `Dispatch<SetStateAction<S>>` | Setter — accepts value or updater function. Stable identity. |

### setState Behavior

| Call pattern | Effect |
|-------------|--------|
| `setState(newValue)` | Replaces state with `newValue` |
| `setState(prev => next)` | Computes next state from previous |
| `setState(sameValue)` | Skips re-render (`Object.is` comparison) |

---

## useEffect

```typescript
function useEffect(
  setup: () => (void | (() => void)),
  dependencies?: ReadonlyArray<unknown>
): void;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `setup` | `() => (void \| (() => void))` | Effect function. May return cleanup function. |
| `dependencies` | `ReadonlyArray<unknown> \| undefined` | Reactive values compared via `Object.is` |

| Return | Type | Description |
|--------|------|-------------|
| — | `void` | No return value |

### Execution Timing

| Event | What happens |
|-------|-------------|
| Mount | `setup()` runs after browser paint |
| Dependency change | `cleanup()` with old values → `setup()` with new values |
| Unmount | `cleanup()` runs |
| StrictMode (dev) | `setup()` → `cleanup()` → `setup()` on mount |

---

## useContext

```typescript
function useContext<T>(context: React.Context<T>): T;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `context` | `React.Context<T>` | Context object from `createContext` |

| Return | Type | Description |
|--------|------|-------------|
| value | `T` | Value from closest `Provider` above, or `defaultValue` |

### createContext (companion API)

```typescript
function createContext<T>(defaultValue: T): React.Context<T>;
```

### Provider Syntax

| Version | Syntax |
|---------|--------|
| React 18 | `<MyContext.Provider value={val}>` |
| React 19 | `<MyContext value={val}>` |

---

## useRef

```typescript
function useRef<T>(initialValue: T): MutableRefObject<T>;
function useRef<T>(initialValue: T | null): RefObject<T>;
function useRef<T = undefined>(): MutableRefObject<T | undefined>;

interface MutableRefObject<T> { current: T; }
interface RefObject<T> { readonly current: T | null; }
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `initialValue` | `T` | Initial `.current` value (ignored after first render) |

| Return | Type | Description |
|--------|------|-------------|
| ref | `{ current: T }` | Mutable object with stable identity across renders |

### TypeScript Overload Rules

| Call | Return type | `.current` |
|------|------------|------------|
| `useRef<HTMLDivElement>(null)` | `RefObject<HTMLDivElement>` | `readonly`, `T \| null` |
| `useRef<number>(0)` | `MutableRefObject<number>` | Mutable, `T` |
| `useRef<number \| null>(null)` | `MutableRefObject<number \| null>` | Mutable, `T \| null` |

---

## useMemo

```typescript
function useMemo<T>(factory: () => T, deps: ReadonlyArray<unknown>): T;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `factory` | `() => T` | Pure function, no arguments. Called on mount and when deps change. |
| `deps` | `ReadonlyArray<unknown>` | Reactive values compared via `Object.is` |

| Return | Type | Description |
|--------|------|-------------|
| value | `T` | Cached result of `factory()` |

### Cache Guarantee

React MAY discard cached values (during development edits, Suspense, etc.). NEVER rely on `useMemo` for semantic correctness — it is a performance optimization only.

---

## useCallback

```typescript
function useCallback<T extends (...args: any[]) => any>(
  callback: T,
  deps: ReadonlyArray<unknown>
): T;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `callback` | `T` | Function to cache (React returns it, does NOT call it) |
| `deps` | `ReadonlyArray<unknown>` | Reactive values compared via `Object.is` |

| Return | Type | Description |
|--------|------|-------------|
| fn | `T` | Cached function reference. New function only when deps change. |

### Equivalence

```typescript
useCallback(fn, deps) === useMemo(() => fn, deps)
```

---

## useReducer

```typescript
function useReducer<S, A>(
  reducer: (state: S, action: A) => S,
  initialArg: S,
  init?: (arg: S) => S
): [S, Dispatch<A>];

type Dispatch<A> = (action: A) => void;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `reducer` | `(state: S, action: A) => S` | Pure function. MUST return new state, NEVER mutate. |
| `initialArg` | `S` | Value from which initial state is calculated |
| `init` | `((arg: S) => S) \| undefined` | Lazy initializer. Pass function reference, NOT result. |

| Return | Type | Description |
|--------|------|-------------|
| `[0]` state | `S` | Current state value |
| `[1]` dispatch | `Dispatch<A>` | Trigger reducer. Stable identity. Returns `void`. |

### dispatch Behavior

| Call | Effect |
|------|--------|
| `dispatch(action)` | Queues re-render with `reducer(currentState, action)` |
| `dispatch(action)` when result is same | Skips re-render (`Object.is` comparison) |

---

## Stable Identities Summary

These values are guaranteed stable across renders and safe to omit from dependency arrays:

| Value | Source |
|-------|--------|
| `setState` | `useState` |
| `dispatch` | `useReducer` |
| `ref` object | `useRef` |

ALWAYS include all other reactive values in dependency arrays.
