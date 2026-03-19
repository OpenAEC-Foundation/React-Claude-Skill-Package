# Hook Anti-Patterns — Complete Catalog

> Every common hook mistake organized by category.
> Each entry: what is wrong, WHY it fails, and what to do instead.

---

## Category 1: Rules of Hooks Violations

### AP-1.1: Hook Inside Condition

**What**: Calling `useState`, `useEffect`, or any hook inside an `if` block, ternary, or `&&` expression.

**Why it fails**: React tracks hooks by call order. Conditional hooks change the call order between renders, corrupting state for all subsequent hooks.

**Fix**: Move ALL hooks above conditional logic. Put conditional behavior inside the hook's callback or after hooks are declared.

---

### AP-1.2: Hook After Early Return

**What**: Placing hooks after a `return` statement that may execute before them.

**Why it fails**: Same as AP-1.1 — when the early return executes, hooks below it are skipped, changing call order.

**Fix**: Declare ALL hooks at the top of the component, before any `return` statements.

---

### AP-1.3: Hook Inside Loop

**What**: Calling hooks inside `for`, `while`, or `do...while` loops.

**Why it fails**: Loop iterations may vary between renders, changing the number and order of hook calls.

**Fix**: Use a single `useState` with an object or array to manage dynamic collections.

---

### AP-1.4: Hook in Regular Function

**What**: Calling hooks from a function that is not a component (capitalized name) or custom hook (`use` prefix).

**Why it fails**: React cannot associate hook state with a component if the function is not recognized as a React function.

**Fix**: Rename to a custom hook with `use` prefix, or move hook logic into the component.

---

### AP-1.5: Hook in Event Handler

**What**: Declaring `useState` or `useEffect` inside an `onClick`, `onChange`, or other event handler function.

**Why it fails**: Event handlers are not React render functions. Hooks must run during rendering, not during event handling.

**Fix**: Declare hooks at the component top level. Use the setter from `useState` inside event handlers.

---

### AP-1.6: Hook in Class Component

**What**: Calling hooks inside a React class component's methods.

**Why it fails**: Hooks are fundamentally incompatible with class components. They rely on function component fiber internals.

**Fix**: Convert to a function component, or extract hook logic into a wrapper function component.

---

### AP-1.7: Hook in try/catch/finally

**What**: Wrapping hook calls in try/catch blocks.

**Why it fails**: If the try block throws before all hooks run, subsequent hooks are skipped, changing call order.

**Fix**: Move hooks outside try/catch. Handle errors within effect callbacks or with Error Boundaries.

---

## Category 2: useState Anti-Patterns

### AP-2.1: Expensive Initializer Called Every Render

**What**: `useState(createExpensiveObject())` — calling the function with parentheses.

**Why it fails**: The initializer expression runs every render, even though React only uses the result on mount.

**Fix**: `useState(createExpensiveObject)` — pass the function reference without calling it.

---

### AP-2.2: Direct State Mutation

**What**: Modifying state objects or arrays directly, then calling setState with the same reference.

**Why it fails**: `Object.is()` comparison sees the same reference and skips re-render. Even if React did re-render, the previous render's state is already corrupted.

**Fix**: ALWAYS create new objects/arrays: `setState({...obj, key: newValue})` or `setState([...arr, newItem])`.

---

### AP-2.3: Reading State Immediately After setState

**What**: Expecting `count` to reflect the new value right after `setCount(count + 1)`.

**Why it fails**: State updates are scheduled for the next render. The current variable holds the snapshot from this render.

**Fix**: Use functional updater `setCount(prev => prev + 1)` for chained updates, or derive from the value you just set.

---

### AP-2.4: Storing Function in useState

**What**: `useState(myFunction)` when you want to store a function as state.

**Why it fails**: React treats any function passed to `useState` as a lazy initializer and calls it.

**Fix**: `useState(() => myFunction)` — wrap in an arrow function to prevent invocation.

---

### AP-2.5: Duplicating Props in State

**What**: Copying a prop into state and using an effect to sync them.

**Why it fails**: Creates two sources of truth. State lags behind the prop by one render cycle.

**Fix**: Use the prop directly. If you need transformation, compute it during render or use `useMemo`.

---

## Category 3: useEffect Anti-Patterns

### AP-3.1: Missing Dependency Array

**What**: `useEffect(() => { ... })` without the second argument.

**Why it fails**: Effect runs after every single render. If it calls setState, it triggers another render, creating an infinite loop.

**Fix**: ALWAYS provide a dependency array. Use `[]` for mount-only, or `[dep1, dep2]` for specific triggers.

---

### AP-3.2: Object or Array in Dependencies

**What**: `useEffect(() => { ... }, [someObject])` where `someObject` is created during render.

**Why it fails**: JavaScript objects have referential identity. A new `{}` is never `===` to the previous `{}`, so the effect runs every render.

**Fix**: Destructure to primitive values in the dependency array, move object creation inside the effect, or memoize with `useMemo`.

---

### AP-3.3: Function in Dependencies Without useCallback

**What**: `useEffect(() => { fn() }, [fn])` where `fn` is defined inline in the component.

**Why it fails**: Functions are recreated every render. New reference every time = effect runs every render.

**Fix**: Move the function inside the effect, wrap with `useCallback`, or extract to module scope if it has no reactive dependencies.

---

### AP-3.4: Missing Cleanup for Subscriptions

**What**: Setting up event listeners, WebSocket connections, or intervals without returning a cleanup function.

**Why it fails**: On re-render with changed deps, a new subscription is created without removing the old one. On unmount, the subscription persists, causing memory leaks and state updates on unmounted components.

**Fix**: ALWAYS return a cleanup function that reverses the setup: `removeEventListener`, `.close()`, `clearInterval`.

---

### AP-3.5: Async Race Condition Without Ignore Flag

**What**: Fetching data in useEffect without guarding against stale responses.

**Why it fails**: User navigates away, dependency changes, or component unmounts while fetch is pending. The response arrives and calls setState on stale or unmounted state.

**Fix**: Use `let ignore = false` pattern, `AbortController`, or a data-fetching library that handles cancellation.

---

### AP-3.6: Using useEffect for Derived/Computed State

**What**: `useEffect(() => { setFullName(first + ' ' + last); }, [first, last])`.

**Why it fails**: Creates an unnecessary extra render cycle. Component renders with stale derived value, then effect runs, then re-renders with correct value.

**Fix**: Compute during render: `const fullName = first + ' ' + last`. Use `useMemo` if the computation is expensive.

---

### AP-3.7: Using useEffect to Reset State on Prop Change

**What**: `useEffect(() => { setComment(''); }, [userId])`.

**Why it fails**: Renders stale state for one frame before the effect clears it. Visible flicker on fast transitions.

**Fix**: Use `key` prop: `<CommentBox key={userId} />`. React creates a fresh component instance with fresh state.

---

### AP-3.8: Using useEffect for Event-Driven Logic

**What**: Watching state in an effect to trigger a notification or side effect that should only happen on user action.

**Why it fails**: Effect fires whenever the watched value changes for ANY reason (mount, re-render, parent update), not just user action. Notifications appear on page reload.

**Fix**: Put the logic in the event handler that triggered the state change.

---

### AP-3.9: Effect Chains (Cascading Effects)

**What**: Multiple `useEffect` calls where each one sets state that triggers the next.

**Why it fails**: Each state update causes a re-render. Chain of N effects = N unnecessary intermediate renders. Difficult to trace data flow.

**Fix**: Consolidate all related state updates into a single event handler or single effect. Use `useReducer` for complex state transitions.

---

### AP-3.10: Subscribing to External Store via useEffect

**What**: Using `useEffect` with `addEventListener` to subscribe to browser APIs or external stores.

**Why it fails**: Susceptible to tearing (reading inconsistent values during concurrent rendering). Requires manual subscription management.

**Fix**: Use `useSyncExternalStore` — designed specifically for external store subscriptions with concurrent rendering safety.

---

## Category 4: Stale Closure Anti-Patterns

### AP-4.1: Timer with Stale State

**What**: `setInterval(() => setCount(count + 1), 1000)` inside a `useEffect` with `[]` deps.

**Why it fails**: The closure captures `count` at mount time (value 0). Every tick sets count to `0 + 1 = 1`.

**Fix**: Use functional updater: `setCount(prev => prev + 1)`. The updater always receives the current value.

---

### AP-4.2: Event Listener with Stale Closure

**What**: Adding a DOM event listener in a `useEffect` with `[]` that reads component state.

**Why it fails**: The listener closure captures state at the time of registration. State changes are invisible to the listener.

**Fix**: Use `useRef` to hold the latest value, updated on every render. Read `ref.current` in the listener.

---

### AP-4.3: Debounced/Throttled Function with Stale Closure

**What**: Creating a debounced function in `useMemo` or `useCallback` that reads state via closure.

**Why it fails**: The memoized function captures the state from its creation render. When it finally executes after the debounce delay, the state may have changed.

**Fix**: Pass the current value as an argument to the debounced function instead of reading it from closure.

---

## Category 5: Performance Hook Anti-Patterns

### AP-5.1: Premature useMemo/useCallback

**What**: Wrapping every value and function in `useMemo`/`useCallback` "for performance."

**Why it fails**: Memoization has overhead (comparison + cache storage). For cheap computations, memoization costs more than recalculation.

**Fix**: Profile first with React DevTools. Only memoize when: computation is measurably slow, value is passed to `memo()` child, or value is a dependency of another hook.

---

### AP-5.2: useMemo Without Return (Object Literal)

**What**: `useMemo(() => { key: 'value' }, [dep])` — block body without explicit return.

**Why it fails**: JavaScript interprets `{ key: 'value' }` as a block with a labeled statement, not an object literal. Returns `undefined`.

**Fix**: Wrap in parentheses: `useMemo(() => ({ key: 'value' }), [dep])`.

---

### AP-5.3: memo() with Unstable Props from Parent

**What**: Wrapping a child in `memo()` while the parent passes new objects/functions as props every render.

**Why it fails**: `memo()` compares props with `Object.is()`. New object/function reference = always re-renders, making `memo()` useless.

**Fix**: In the parent: use `useMemo` for object props, `useCallback` for function props. Or pass primitives instead of objects.

---

### AP-5.4: Custom arePropsEqual Skipping Function Props

**What**: Writing a custom comparison for `memo()` that ignores function props to avoid re-renders.

**Why it fails**: When the parent re-renders with a new closure, the child keeps the old function. The old function captures stale state, causing bugs.

**Fix**: NEVER ignore function props in comparisons. Use `useCallback` in the parent to stabilize function references instead.

---

## Category 6: useRef Anti-Patterns

### AP-6.1: Reading/Writing ref.current During Render

**What**: `return <div>{myRef.current}</div>` or `myRef.current = computedValue` in the render body.

**Why it fails**: Refs are not tracked by React. Reading during render makes the component impure (result depends on mutation timing). React may read the component output at unexpected times during concurrent rendering.

**Fix**: Read/write refs ONLY in event handlers, effects, or during initialization (null check pattern).

---

### AP-6.2: Using Ref for Values That Should Trigger Re-render

**What**: Storing a value in a ref and expecting the UI to update when it changes.

**Why it fails**: Ref mutations do not trigger re-renders. The screen shows stale data.

**Fix**: Use `useState` for any value that affects rendered output.

---

## Category 7: useContext Anti-Patterns

### AP-7.1: Provider Without value Prop

**What**: `<MyContext>` or `<MyContext.Provider>` without passing a `value` prop.

**Why it fails**: Consumers receive `undefined` instead of the default from `createContext`.

**Fix**: ALWAYS pass `value` to providers: `<MyContext value={actualValue}>`.

---

### AP-7.2: New Object in Provider Value Every Render

**What**: `<ThemeContext value={{ theme, setTheme }}>` creating a new object inline.

**Why it fails**: New object reference on every parent render forces ALL context consumers to re-render, regardless of whether `theme` or `setTheme` changed.

**Fix**: Memoize the value: `const value = useMemo(() => ({ theme, setTheme }), [theme, setTheme])`.

---

## Category 8: React 19 Specific Anti-Patterns

### AP-8.1: use() in try/catch

**What**: Calling `use(promise)` inside a `try`/`catch` block.

**Why it fails**: Throws "Suspense Exception: This is not a real error!" — React uses this exception internally for Suspense flow control.

**Fix**: Call `use()` at the component top level. Use Error Boundaries for error handling.

---

### AP-8.2: Creating Promise Inside Client Component for use()

**What**: Creating a new Promise inside a component and passing it to `use()`.

**Why it fails**: A new Promise is created every render, causing the component to suspend infinitely.

**Fix**: Create the Promise in a Server Component and pass it down, or cache/memoize it outside the component.

---

### AP-8.3: useFormStatus in Form-Rendering Component

**What**: Calling `useFormStatus()` in the same component that renders the `<form>`.

**Why it fails**: `useFormStatus` reads the status of a parent `<form>`. If there is no parent form, it returns `{ pending: false }`.

**Fix**: Extract the component using `useFormStatus` into a child component rendered inside the `<form>`.

---

### AP-8.4: useOptimistic Outside Transition

**What**: Calling `setOptimistic()` outside of `startTransition` or a form action.

**Why it fails**: The optimistic update briefly renders, then immediately reverts because there is no pending Action to maintain it.

**Fix**: ALWAYS call `setOptimistic` inside `startTransition` or from within a form action handler.
