# React Hooks & Component API Research — Part 1

> **Scope**: Core Hooks, Component APIs, React APIs, Rules of Hooks
> **Sources**: All data sourced from https://react.dev/reference/react/
> **Date**: 2026-03-19
> **Covers**: React 18.x and React 19.x

---

## Table of Contents

- [§1 State Hooks](#1-state-hooks)
- [§2 Effect Hooks](#2-effect-hooks)
- [§3 Context Hooks](#3-context-hooks)
- [§4 Ref Hooks](#4-ref-hooks)
- [§5 Performance Hooks](#5-performance-hooks)
- [§6 Transition & Deferred Hooks](#6-transition--deferred-hooks)
- [§7 Utility Hooks](#7-utility-hooks)
- [§8 React 19 New Hooks](#8-react-19-new-hooks)
- [§9 Built-in Components](#9-built-in-components)
- [§10 React APIs](#10-react-apis)
- [§11 Rules of Hooks](#11-rules-of-hooks)
- [§12 React 18 vs 19 Summary](#12-react-18-vs-19-summary)

---

## §1 State Hooks

### §1.1 useState

**Source**: https://react.dev/reference/react/useState

**Signature**:
```typescript
function useState<S>(initialState: S | (() => S)): [S, Dispatch<SetStateAction<S>>];

type SetStateAction<S> = S | ((prevState: S) => S);
type Dispatch<A> = (value: A) => void;
```

**Parameters**:
- `initialState: S | (() => S)` — Initial value or lazy initializer function (called once on mount). Must be pure, take no arguments.

**Returns**: `[currentState, setState]`
- `currentState: S` — Current state value.
- `setState: Dispatch<SetStateAction<S>>` — Setter function. Accepts direct value or updater function `(prev) => next`. Returns `undefined`.

**Key Behavior**:
- State updates are batched; screen updates AFTER all event handlers complete.
- Comparison uses `Object.is()` — identical values skip re-render.
- `setState` has stable identity across renders (safe to omit from dependency arrays).
- Calling `setState` does NOT update the value in the currently executing code.

**Common Anti-patterns**:

| Anti-pattern | Problem | Fix |
|---|---|---|
| `useState(createTodos())` | Initializer runs every render | `useState(createTodos)` — pass function reference |
| `obj.x = 2; setObj(obj)` | Mutation, same reference | `setObj({...obj, x: 2})` — new object |
| `todos.push(item); setTodos(todos)` | Array mutation | `setTodos([...todos, item])` |
| `onClick={handleClick()}` | Calls during render | `onClick={handleClick}` — pass reference |
| `useState(myFunction)` | Function treated as initializer | `useState(() => myFunction)` |

**React 18 vs 19**: No API changes. Behavior consistent across versions.

---

### §1.2 useReducer

**Source**: https://react.dev/reference/react/useReducer

**Signature**:
```typescript
function useReducer<R extends Reducer<any, any>>(
  reducer: R,
  initialArg: ReducerState<R>,
  init?: (arg: ReducerState<R>) => ReducerState<R>
): [ReducerState<R>, Dispatch<ReducerAction<R>>];

type Reducer<S, A> = (prevState: S, action: A) => S;
```

**Parameters**:
- `reducer: (state: S, action: A) => S` — Pure function returning next state. MUST NOT mutate state.
- `initialArg: S` — Value from which initial state is calculated.
- `init?: (initialArg: S) => S` — Optional lazy initializer. Pass the function itself (not its result).

**Returns**: `[state, dispatch]`
- `state: S` — Current state value.
- `dispatch: (action: A) => void` — Triggers reducer. Stable identity. Returns `undefined`.

**Key Behavior**:
- `dispatch` queues an update for the next render; reading state immediately after returns the old value.
- React skips re-render if new state === current state (via `Object.is`).
- Batched with other state updates.
- Convention: actions are objects with a `type` property.

**Common Anti-patterns**:

| Anti-pattern | Problem | Fix |
|---|---|---|
| `state.age++; return state` | Mutates state | `return {...state, age: state.age + 1}` |
| `useReducer(reducer, createInitial())` | Recreates every render | `useReducer(reducer, arg, createInitial)` |

**React 18 vs 19**: No API changes.

---

## §2 Effect Hooks

### §2.1 useEffect

**Source**: https://react.dev/reference/react/useEffect

**Signature**:
```typescript
function useEffect(
  setup: () => (void | (() => void)),
  dependencies?: ReadonlyArray<unknown>
): void;
```

**Parameters**:
- `setup` — Effect function. May return a cleanup function `() => void`.
- `dependencies?` — Array of reactive values compared via `Object.is`.

**Dependency Array Behavior**:

| Pattern | Runs when | Use case |
|---|---|---|
| `[dep1, dep2]` | Any dependency changes | Most common — sync to specific values |
| `[]` | Only on mount (cleanup on unmount) | One-time setup |
| Omitted | Every commit | Rarely needed — usually a mistake |

**Execution Order**:
1. Component renders and commits to DOM.
2. Browser paints.
3. `useEffect` setup runs.
4. On dependency change: cleanup (old values) → setup (new values).
5. On unmount: final cleanup.

**Key Behavior**:
- Runs AFTER browser paint (non-blocking).
- Does NOT run during SSR.
- StrictMode: runs setup → cleanup → setup in development.

**Common Anti-patterns**:

| Anti-pattern | Problem | Fix |
|---|---|---|
| Missing dependency | Stale closure | Include all reactive values in deps array |
| Object in deps `[options]` | New reference every render | Move object creation inside effect, or destructure primitives |
| No race condition guard | Stale async response | Use `let ignore = false` pattern with cleanup |
| Fetching in effect without cleanup | Memory leak / stale data | Return cleanup that sets ignore flag |

**Race Condition Pattern**:
```typescript
useEffect(() => {
  let ignore = false;
  fetchData(id).then(result => {
    if (!ignore) setData(result);
  });
  return () => { ignore = true; };
}, [id]);
```

**React 18 vs 19**: No API changes. StrictMode double-effect is React 18+ behavior.

---

### §2.2 useLayoutEffect

**Source**: https://react.dev/reference/react/useLayoutEffect

**Signature**:
```typescript
function useLayoutEffect(
  setup: () => (void | (() => void)),
  dependencies?: ReadonlyArray<unknown>
): void;
```

**Execution Timing** (compared to useEffect):

| Hook | Fires | Blocks paint? |
|---|---|---|
| `useInsertionEffect` | Before DOM mutations | Yes |
| `useLayoutEffect` | After DOM mutations, before paint | Yes |
| `useEffect` | After paint | No |

**Primary Use Case**: Measuring DOM layout before the browser paints (tooltips, popovers).

**Pattern**: Render → Commit → Measure (useLayoutEffect) → Re-render → Paint.

**Caveats**:
- Blocks browser repainting — use sparingly.
- Does NOT run during SSR (throws warning on server).
- State updates inside trigger all remaining Effects immediately.

**React 18 vs 19**: No API changes.

---

### §2.3 useInsertionEffect

**Source**: https://react.dev/reference/react/useInsertionEffect

**Signature**:
```typescript
function useInsertionEffect(
  setup: () => (void | (() => void)),
  dependencies?: ReadonlyArray<unknown>
): void;
```

**Fires**: Before React makes DOM changes. Runs before `useLayoutEffect`.

**Use Case**: CSS-in-JS libraries injecting `<style>` tags at runtime.

**Limitations**:
- Cannot call `setState` inside.
- Refs are NOT attached yet.
- Client-only (does not run during SSR).
- DOM may or may not be updated — do not rely on specific timing.

**NEVER use this hook unless building a CSS-in-JS library.**

**React 18 vs 19**: No API changes.

---

## §3 Context Hooks

### §3.1 useContext

**Source**: https://react.dev/reference/react/useContext

**Signature**:
```typescript
function useContext<T>(context: React.Context<T>): T;
```

**Parameters**:
- `context` — The context object created by `createContext`. Does not hold information itself.

**Returns**: The context value from the closest matching `Provider` above in the tree. Falls back to `defaultValue` from `createContext` if no provider found.

**Key Behavior**:
- Searches upward through the component tree.
- Provider in the SAME component does NOT affect `useContext` in that component.
- React automatically re-renders ALL consumers when provider value changes.
- Comparison uses `Object.is`.
- `memo` does NOT prevent receiving fresh context values.

**Optimization Pattern**:
```typescript
const contextValue = useMemo(() => ({
  currentUser,
  login
}), [currentUser, login]);

return <AuthContext value={contextValue}>{children}</AuthContext>;
```

**Common Anti-patterns**:

| Anti-pattern | Problem | Fix |
|---|---|---|
| `<Ctx>` without `value` prop | Passes `undefined` | ALWAYS pass `value` prop |
| Creating new object in provider every render | All consumers re-render | Use `useMemo` for context value |
| Duplicate module builds | Context mismatch (different objects) | Ensure single module instance |

**React 18 vs 19**: In React 19, `<Context value={...}>` replaces `<Context.Provider value={...}>`. See §10.1 createContext.

---

## §4 Ref Hooks

### §4.1 useRef

**Source**: https://react.dev/reference/react/useRef

**Signature**:
```typescript
function useRef<T>(initialValue: T): { current: T };
function useRef<T>(initialValue: T | null): { current: T | null };
```

**Parameters**:
- `initialValue: T` — Initial value for `.current`. Ignored after initial render.

**Returns**: Object with single `current` property. Same object identity across renders.

**Read/Write Rules**:
- ALLOWED: In event handlers, in effects, during initialization (null check).
- NOT ALLOWED: Reading or writing `.current` during render (breaks purity).

**Key Behavior**:
- Changing `.current` does NOT trigger re-render.
- React is not aware of ref mutations.
- Same object reference across renders (unlike objects created in render body).
- StrictMode: ref object created twice in development (one discarded).

**Common Use Cases**: Storing timeout/interval IDs, DOM node references, mutable values not needed for rendering.

**React 18 vs 19**: No API changes.

---

### §4.2 useImperativeHandle

**Source**: https://react.dev/reference/react/useImperativeHandle

**Signature**:
```typescript
function useImperativeHandle<T, R extends T>(
  ref: React.Ref<T>,
  createHandle: () => R,
  dependencies?: ReadonlyArray<unknown>
): void;
```

**Parameters**:
- `ref` — The ref received as a prop (React 19) or from `forwardRef` (React 18).
- `createHandle: () => R` — Returns the handle object to expose. Re-executes when deps change.
- `dependencies?` — Reactive values referenced in `createHandle`.

**Pattern**:
```typescript
// React 19
function MyInput({ ref, ...props }: { ref: React.Ref<MyInputHandle> }) {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => ({
    focus() { inputRef.current?.focus(); },
    scrollIntoView() { inputRef.current?.scrollIntoView(); },
  }), []);
  return <input {...props} ref={inputRef} />;
}

// React 18
const MyInput = forwardRef(function MyInput(props, ref) {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => ({
    focus() { inputRef.current?.focus(); },
  }), []);
  return <input {...props} ref={inputRef} />;
});
```

**React 18 vs 19**: In React 19, `ref` is available directly as a prop — `forwardRef` is no longer needed.

---

## §5 Performance Hooks

### §5.1 useMemo

**Source**: https://react.dev/reference/react/useMemo

**Signature**:
```typescript
function useMemo<T>(factory: () => T, deps: ReadonlyArray<unknown>): T;
```

**Parameters**:
- `factory: () => T` — Pure function, no arguments. Called on mount and when deps change.
- `deps` — Array of reactive values. Compared via `Object.is`.

**Returns**: Cached result of `factory()`. Returns cached value when deps unchanged.

**When to Use**:
1. Calculation is noticeably slow AND dependencies rarely change.
2. Value is passed to a `memo()`-wrapped component.
3. Value is used as a dependency of another hook.

**Caveats**:
- React MAY discard cache (file edits in dev, component suspension). Do NOT rely on it as a semantic guarantee.
- StrictMode: factory called twice in development.
- React Compiler (React 19) may make manual `useMemo` unnecessary.

**React 18 vs 19**: React Compiler in 19 automatically memoizes values, reducing need for manual `useMemo`.

---

### §5.2 useCallback

**Source**: https://react.dev/reference/react/useCallback

**Signature**:
```typescript
function useCallback<T extends (...args: any[]) => any>(
  callback: T,
  deps: ReadonlyArray<unknown>
): T;
```

**Parameters**:
- `callback: T` — The function to cache. React returns (not calls) it.
- `deps` — Reactive values. Compared via `Object.is`.

**Returns**: Cached function reference. Returns new function only when deps change.

**Relationship to useMemo**:
```typescript
// These are equivalent:
useCallback(fn, deps)
useMemo(() => fn, deps)
```

**When to Use**:
1. Passing to a `memo()`-wrapped child component.
2. Function is a dependency of another hook (e.g., `useEffect`).

**Updater Pattern** (remove state dependency):
```typescript
// Instead of:
const handleAdd = useCallback((text: string) => {
  setTodos([...todos, createTodo(text)]);
}, [todos]); // todos changes every update

// Use updater:
const handleAdd = useCallback((text: string) => {
  setTodos(prev => [...prev, createTodo(text)]);
}, []); // Empty deps — stable reference
```

**React 18 vs 19**: React Compiler in 19 automatically memoizes functions, reducing need for manual `useCallback`.

---

## §6 Transition & Deferred Hooks

### §6.1 useTransition

**Source**: https://react.dev/reference/react/useTransition

**Signature**:
```typescript
function useTransition(): [
  isPending: boolean,
  startTransition: (action: () => void | Promise<void>) => void
];
```

**Parameters**: None.

**Returns**:
- `isPending: boolean` — `true` while a transition is in progress.
- `startTransition` — Marks state updates as non-blocking transitions.

**Key Behavior**:
- `startTransition` callback executes immediately (not deferred).
- State updates inside are interruptible by higher-priority updates (e.g., typing).
- Prevents Suspense from showing fallback for already-visible content.
- `startTransition` has stable identity.

**Caveats**:
- CANNOT update controlled text inputs in transitions.
- State updates after `await` require nested `startTransition` (JavaScript async context limitation).
- Only wraps state you own (have the setter for). For prop values, use `useDeferredValue`.
- `setTimeout` callbacks lose transition context — wrap `startTransition` inside `setTimeout`, not the other way around.

**React 18 vs 19**:
- React 19 adds higher-level abstractions built on transitions: `useActionState`, `<form>` actions, Server Functions.
- `useTransition` remains the lower-level primitive in both versions.

---

### §6.2 useDeferredValue

**Source**: https://react.dev/reference/react/useDeferredValue

**Signature**:
```typescript
// React 18
function useDeferredValue<T>(value: T): T;

// React 19
function useDeferredValue<T>(value: T, initialValue?: T): T;
```

**Parameters**:
- `value: T` — The value to defer. Any type.
- `initialValue?: T` — (React 19 only) Value to use during initial render.

**Returns**: Deferred value that lags behind during updates. Returns `initialValue` on first render (if provided).

**How Deferral Works**:
1. Immediate re-render: React uses old deferred value.
2. Background re-render: React attempts update with new value (interruptible).
3. If background render completes, deferred value updates on screen.

**Key Behavior**:
- No fixed delay — adaptive to device performance.
- Background re-render is interruptible.
- When inside `useTransition`, always returns new value immediately.
- Uses `Object.is` to detect changes.
- Values should be primitives or objects created outside rendering.

**React 18 vs 19**: React 19 adds optional `initialValue` parameter for controlling first-render value.

---

## §7 Utility Hooks

### §7.1 useId

**Source**: https://react.dev/reference/react/useId

**Signature**:
```typescript
function useId(): string;
```

**Returns**: Unique ID string. Stable across re-renders for a mounted component.

**Use Cases**:
- Accessibility attributes (`aria-describedby`, `htmlFor`).
- Generating IDs for multiple related elements (use as prefix: `id + '-firstName'`).
- SSR hydration — ensures server/client IDs match.

**Caveats**:
- NEVER use for list keys (keys should come from data).
- NEVER use for cache keys.
- Cannot be used in async Server Components.
- Multiple React apps: use `identifierPrefix` option on root.

**React 18 vs 19**: Introduced in React 18. No changes in 19.

---

### §7.2 useSyncExternalStore

**Source**: https://react.dev/reference/react/useSyncExternalStore

**Signature**:
```typescript
function useSyncExternalStore<T>(
  subscribe: (callback: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T
): T;
```

**Parameters**:
- `subscribe` — Subscribes to store changes. Returns unsubscribe function. MUST be stable (declare outside component or use `useCallback`).
- `getSnapshot: () => T` — Returns immutable snapshot. Repeated calls must return same value (via `Object.is`) if store unchanged. New object every call = infinite re-render loop.
- `getServerSnapshot?: () => T` — For SSR. Must match between server and client.

**Returns**: Current store snapshot.

**When to Use**: Third-party state libraries, browser APIs (e.g., `navigator.onLine`), non-React state.

**Caveats**:
- `getSnapshot` MUST return immutable data or cached/memoized results.
- Different `subscribe` function reference causes resubscription every render.
- Not recommended to suspend based on store values.

**React 18 vs 19**: Introduced in React 18. No changes in 19.

---

### §7.3 useDebugValue

**Source**: https://react.dev/reference/react/useDebugValue

**Signature**:
```typescript
function useDebugValue<T>(value: T, format?: (value: T) => any): void;
```

**Parameters**:
- `value: T` — Value displayed in React DevTools.
- `format?` — Lazy formatter. Only called when component is inspected.

**Use Case**: Adding labels to custom hooks in DevTools. Only useful for shared library hooks.

**React 18 vs 19**: No changes.

---

## §8 React 19 New Hooks

### §8.1 use()

**Source**: https://react.dev/reference/react/use

**Signature**:
```typescript
function use<T>(resource: Promise<T>): T;
function use<T>(context: React.Context<T>): T;
```

**Parameters**:
- `resource` — A `Promise` or `Context` object.

**Returns**: Resolved value of the Promise, or context value.

**Key Differences from useContext**:

| Feature | `useContext` | `use` |
|---|---|---|
| Conditionals | NOT allowed | ALLOWED in `if` statements |
| Loops | NOT allowed | ALLOWED in `for` loops |
| Promise support | No | Yes (suspends component) |

**How it Works with Promises**:
- Component suspends while Promise is pending (integrates with `<Suspense>`).
- Resolved value is returned.
- Rejected Promises caught by nearest Error Boundary.
- CANNOT be used inside `try`/`catch`.

**Caveats**:
- Must be called inside a Component or Hook function.
- Prefer `async`/`await` in Server Components over `use`.
- Promises created in Client Components are recreated every render — create in Server Components and pass down.
- Error: "Suspense Exception: This is not a real error!" = calling `use` outside component or in `try`/`catch`.

**React 18 vs 19**: React 19 only. Not available in React 18.

---

### §8.2 useActionState

**Source**: https://react.dev/reference/react/useActionState

> Previously named `useFormState` in earlier React 19 canaries.

**Signature**:
```typescript
function useActionState<State>(
  action: (state: State, payload: unknown) => State | Promise<State>,
  initialState: State,
  permalink?: string
): [state: State, dispatch: (payload: unknown) => void, isPending: boolean];
```

**Parameters**:
- `action` — Reducer-like function. Receives previous state + action payload. Can be async with side effects.
- `initialState: State` — Initial state. Must be serializable when using Server Functions.
- `permalink?: string` — Unique URL for progressive enhancement with Server Functions.

**Returns**: `[state, dispatch, isPending]`
- `state: State` — Current state (initially `initialState`).
- `dispatch` — Triggers the action. Can be passed to `<form action={dispatch}>`.
- `isPending: boolean` — Whether any dispatched actions are pending.

**Key Behavior**:
- When passed to `<form action>`, React wraps submission in a Transition.
- Multiple dispatches queue sequentially (each receives previous return value).
- If action throws, all queued actions are cancelled and Error Boundary triggered.
- `dispatch` has stable identity.

**Caveats**:
- `dispatch` MUST be called from within `startTransition` or an Action prop.
- State updates after `await` require nested `startTransition`.
- Handles request ordering automatically (unlike raw `useTransition`).

**React 18 vs 19**: React 19 only.

---

### §8.3 useOptimistic

**Source**: https://react.dev/reference/react/useOptimistic

**Signature**:
```typescript
function useOptimistic<State, Action>(
  passthrough: State,
  reducer?: (currentState: State, action: Action) => State
): [State, (action: Action) => void];
```

**Parameters**:
- `passthrough: State` — The real state value. Returned when no Action is pending.
- `reducer?` — Pure function computing optimistic state from current state + action.

**Returns**: `[optimisticState, setOptimistic]`
- `optimisticState` — Equals `passthrough` when no Action pending; reducer result during Action.
- `setOptimistic` — Dispatches optimistic update.

**Key Behavior**:
- Optimistic state is temporary — only renders while an Action is in progress.
- When Action completes, optimistic and real state converge.
- MUST be called inside `startTransition` or an Action prop.
- If called outside a Transition, briefly renders then immediately reverts.

**When to Use Reducer**:
- Lists: reducer ensures new items are added to latest base state.
- Multiple related values: ensures consistency.
- Multiple action types: handles different operations cleanly.

**React 18 vs 19**: React 19 only.

---

### §8.4 useFormStatus

**Source**: https://react.dev/reference/react-dom/hooks/useFormStatus

> Note: This hook is in `react-dom`, not `react`.

**Signature**:
```typescript
function useFormStatus(): {
  pending: boolean;
  data: FormData | null;
  method: 'get' | 'post';
  action: ((...args: any[]) => any) | null;
};
```

**Parameters**: None.

**Returns**: Status object for the parent `<form>`:
- `pending` — `true` if form is submitting.
- `data` — `FormData` of current submission, or `null`.
- `method` — HTTP method (`'get'` or `'post'`).
- `action` — Reference to the form's `action` function, or `null`.

**Critical Requirement**: MUST be called from a component rendered INSIDE a `<form>`. Does NOT work in the same component that renders the `<form>`.

```typescript
// WRONG — always returns pending: false
function Form() {
  const { pending } = useFormStatus();
  return <form action={submit}><button disabled={pending}>Submit</button></form>;
}

// CORRECT — separate child component
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Submit</button>;
}
function Form() {
  return <form action={submit}><SubmitButton /></form>;
}
```

**React 18 vs 19**: React 19 only (part of React DOM).

---

## §9 Built-in Components

### §9.1 Fragment

**Source**: https://react.dev/reference/react/Fragment

**Props**:
- `key?: string | number` — Only with explicit `<Fragment>` syntax.
- `ref?` — (Canary) Accepts ref object or callback. NOT available with `<>` shorthand.

**Shorthand**: `<>...</>` — Cannot pass `key` or `ref`.

**When key is needed**: Rendering lists of fragments:
```typescript
posts.map(post => (
  <Fragment key={post.id}>
    <PostTitle title={post.title} />
    <PostBody body={post.body} />
  </Fragment>
));
```

**Caveat**: Switching between `<><Child /></>` and `[<Child />]` or `<Child />` does NOT reset state (one level deep only).

---

### §9.2 Suspense

**Source**: https://react.dev/reference/react/Suspense

**Props**:
- `children: ReactNode` — The UI to render.
- `fallback: ReactNode` — Placeholder shown while children are loading.

**Compatible Data Sources**:
- `lazy()` for code splitting.
- `use()` for cached Promises.
- Frameworks: Relay, Next.js Suspense-enabled fetching.
- Does NOT detect data fetched in Effects or event handlers.

**Key Behaviors**:
- All children under one Suspense boundary are treated as one unit.
- Nested `<Suspense>` enables progressive reveal.
- `startTransition` prevents fallback for already-visible content.
- State is NOT preserved for renders that suspended before first mount.
- Layout Effects are cleaned up when content is hidden by Suspense, re-fired when shown.

**SSR Integration**: Enables streaming server rendering and selective hydration.

**React 18 vs 19**: Core Suspense available since React 18. React 19 adds `use()` integration.

---

### §9.3 StrictMode

**Source**: https://react.dev/reference/react/StrictMode

**Props**: None.

**Development-only Checks**:
1. **Double rendering** — Calls component functions twice to detect impure rendering.
2. **Double effects** — Runs setup → cleanup → setup for every Effect.
3. **Double ref callbacks** — Runs setup → cleanup → setup for callback refs.
4. **Deprecated API warnings** — Warns about `UNSAFE_componentWillMount`, etc.

**Caveats**:
- No opt-out for components inside `<StrictMode>`.
- All checks are development-only — no production impact.
- Console logs during second render appear dimmed in React DevTools.

**React 18 vs 19**: Consistent across versions.

---

### §9.4 Profiler

**Source**: https://react.dev/reference/react/Profiler

**Props**:
- `id: string` — Identifies the measured section.
- `onRender: ProfilerOnRenderCallback` — Called on every commit.

**onRender Callback Signature**:
```typescript
type ProfilerOnRenderCallback = (
  id: string,
  phase: 'mount' | 'update' | 'nested-update',
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number
) => void;
```

- `actualDuration` — Time spent rendering (shows memoization effectiveness).
- `baseDuration` — Worst-case time without optimizations.

**Caveat**: Disabled in production builds by default. Requires special profiling build for production measurement.

---

### §9.5 Activity (Experimental)

**Source**: https://react.dev/reference/react/Activity

**Status**: EXPERIMENTAL — React 19+ experimental release only.

**Props**:
- `mode: 'visible' | 'hidden'` — Controls visibility.
- `children: ReactNode` — The UI to show/hide.

**State Preservation**: Preserves ALL state (hooks, DOM form values, scroll position, focus) when hidden. Uses `display: none`.

**Effect Behavior**:
- Cleanup functions run when `mode` changes to `'hidden'`.
- Effects re-created when `mode` changes to `'visible'`.
- Hidden children still re-render at lower priority.

**Differs from Unmounting**:

| Aspect | Hidden Activity | Unmount |
|---|---|---|
| State | Preserved | Lost |
| DOM | Preserved (`display: none`) | Destroyed |
| Effects | Cleaned up | Cleaned up |
| Re-renders | Continue (low priority) | Stop |
| Memory | Higher | Lower |

**Caveats**:
- Text-only components render nothing when hidden (no DOM element to hide).
- DOM side effects persist (video/audio continue playing) — add cleanup.
- Not recommended for production without careful testing.

---

## §10 React APIs

### §10.1 createContext

**Source**: https://react.dev/reference/react/createContext

**Signature**:
```typescript
function createContext<T>(defaultValue: T): React.Context<T>;
```

**Returns**: Context object usable as Provider and with `useContext`.

**React 18 vs 19**:
```typescript
// React 19 — render context directly as provider
<ThemeContext value={theme}><Page /></ThemeContext>

// React 18 — must use .Provider
<ThemeContext.Provider value={theme}><Page /></ThemeContext.Provider>
```

`Context.Consumer` is legacy — ALWAYS use `useContext()` instead.

---

### §10.2 forwardRef

**Source**: https://react.dev/reference/react/forwardRef

**Status**: DEPRECATED in React 19. Pass `ref` as a prop directly.

**Signature**:
```typescript
function forwardRef<T, P = {}>(
  render: (props: P, ref: React.Ref<T>) => React.ReactNode
): React.ForwardRefExoticComponent<React.PropsWithoutRef<P> & React.RefAttributes<T>>;
```

**React 18 vs 19**:
```typescript
// React 18 — forwardRef required
const MyInput = forwardRef<HTMLInputElement, Props>((props, ref) => {
  return <input ref={ref} {...props} />;
});

// React 19 — ref is a regular prop
function MyInput({ ref, ...props }: { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
}
```

---

### §10.3 lazy

**Source**: https://react.dev/reference/react/lazy

**Signature**:
```typescript
function lazy<T extends React.ComponentType<any>>(
  load: () => Promise<{ default: T }>
): React.LazyExoticComponent<T>;
```

**Key Rules**:
- ALWAYS declare at module top level (not inside components).
- Must be used with `<Suspense>`.
- `load` is called only once; result is cached.
- Module must have a `.default` export.

---

### §10.4 memo

**Source**: https://react.dev/reference/react/memo

**Signature**:
```typescript
function memo<P extends object>(
  Component: React.ComponentType<P>,
  arePropsEqual?: (prevProps: P, nextProps: P) => boolean
): React.MemoExoticComponent<React.ComponentType<P>>;
```

**Default comparison**: Shallow equality via `Object.is` on each prop.

**Custom comparison**: `arePropsEqual(prev, next)` — return `true` to skip re-render.

**Caveats**:
- Only prevents re-renders from PARENT changes.
- Component still re-renders on own state or context changes.
- Ineffective with constantly-changing props (need `useMemo`/`useCallback` in parent).
- React Compiler (React 19) makes `memo` largely unnecessary.

---

### §10.5 startTransition

**Source**: https://react.dev/reference/react/startTransition

**Signature**:
```typescript
function startTransition(action: () => void): void;
```

**Differs from useTransition**:

| Feature | `startTransition` | `useTransition` |
|---|---|---|
| Usage | Anywhere (outside components OK) | Hook (components only) |
| `isPending` | No | Yes |

**React 18 vs 19**: Consistent across versions.

---

### §10.6 cache

**Source**: https://react.dev/reference/react/cache

**Signature**:
```typescript
function cache<T extends (...args: any[]) => any>(fn: T): T;
```

**Scope**: Server Components ONLY.

**Key Behavior**:
- Caches results based on arguments (compared via `Object.is`).
- Cache invalidated per server request.
- Each `cache()` call creates a separate cache — share via module exports.
- Non-primitive arguments compared by reference.
- Calling outside components does not use cache.

---

### §10.7 act

**Source**: https://react.dev/reference/react/act

**Signature**:
```typescript
function act(actFn: () => Promise<void>): Promise<void>;
```

**Use Case**: Testing. Wraps renders and interactions to ensure updates are flushed.

**Requirements**:
- ALWAYS use `await act(async () => {})` — sync version deprecated.
- Must set `global.IS_REACT_ACT_ENVIRONMENT = true` in test setup.
- Container must be added to `document` for DOM events.

---

## §11 Rules of Hooks

**Source**: https://react.dev/reference/rules/rules-of-hooks

### Rule 1: Only Call Hooks at the Top Level

NEVER call hooks inside:
- Loops (`for`, `while`)
- Conditions (`if`, ternary)
- Nested functions
- `try`/`catch`/`finally` blocks
- After conditional `return` statements
- Event handlers
- Class components
- Functions passed to `useMemo`, `useReducer`, or `useEffect`

ALWAYS call hooks:
- At the top level of function components
- At the top level of custom hooks

**Exception**: `use()` (React 19) CAN be called inside conditionals and loops.

**Reasoning**: React relies on consistent call order to maintain state correctly.

### Rule 2: Only Call Hooks from React Functions

NEVER call hooks from regular JavaScript functions.

ALWAYS call hooks from:
- React function components
- Custom hooks (functions starting with `use`)

### Enforcement

Use `eslint-plugin-react-hooks` to automatically catch violations.

---

## §12 React 18 vs 19 Summary

### New in React 19

| API | Category | Description |
|---|---|---|
| `use()` | Hook | Read Promises and Context with conditional support |
| `useActionState` | Hook | Manage state from async actions (replaces `useFormState`) |
| `useOptimistic` | Hook | Optimistic UI updates during async actions |
| `useFormStatus` | Hook (react-dom) | Form submission status for child components |
| `Activity` | Component | Experimental — hide/show UI preserving state |
| `cache` | API | Memoize functions in Server Components |

### Changed in React 19

| API | Change |
|---|---|
| `createContext` | Render `<Context value>` directly (no `.Provider` needed) |
| `forwardRef` | Deprecated — `ref` is now a regular prop |
| `useDeferredValue` | Added optional `initialValue` parameter |
| `useImperativeHandle` | `ref` available as prop (no `forwardRef` needed) |
| `memo` / `useMemo` / `useCallback` | React Compiler auto-memoizes (manual use becomes optional) |

### Unchanged in React 19

All other hooks and APIs (`useState`, `useEffect`, `useReducer`, `useContext`, `useRef`, `useLayoutEffect`, `useInsertionEffect`, `useId`, `useSyncExternalStore`, `useDebugValue`, `useTransition`, `startTransition`, `lazy`, `act`, `Fragment`, `Suspense`, `StrictMode`, `Profiler`) maintain the same API surface.

### React Compiler Impact

The React Compiler (available with React 19) automatically:
- Memoizes component renders (replaces `memo`)
- Memoizes computed values (replaces `useMemo`)
- Memoizes function references (replaces `useCallback`)

Manual memoization hooks become optional but remain supported.
