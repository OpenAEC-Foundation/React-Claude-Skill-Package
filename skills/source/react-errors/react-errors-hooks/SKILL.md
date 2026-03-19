---
name: react-errors-hooks
description: "Diagnoses and resolves common React hook errors including Rules of Hooks violations, conditional hook calls, missing useEffect dependencies, stale closures, infinite re-render loops, cleanup mistakes, and eslint-plugin-react-hooks warnings. Activates when encountering hook-related errors, infinite loops, stale state issues, or eslint hook warnings."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-errors-hooks

## Quick Reference — Error Diagnostic Table

| Symptom | Cause | Fix |
|---------|-------|-----|
| `React has detected a change in the order of Hooks` | Conditional hook call or early return before hooks | Move ALL hooks above any conditional logic or returns |
| `Invalid hook call` | Hook called in regular function, class, or nested scope | ONLY call hooks from function components or custom hooks |
| `Too many re-renders` | `setState` called during render body | Move state update into event handler or useEffect |
| Component re-renders infinitely with useEffect | Missing dependency array or new object/array in deps | Add `[]` deps, destructure objects to primitives, or use `useMemo` |
| State shows old value in setTimeout/setInterval | Stale closure captures initial state value | Use functional updater `setState(prev => ...)` or `useRef` |
| State shows old value in event listener | Event listener closed over stale state | Use `useRef` to hold latest value, or re-register listener |
| `Warning: Can't perform a React state update on unmounted component` | Async callback updates state after unmount | Return cleanup function from useEffect with ignore flag |
| `React Hook useEffect has a missing dependency` | eslint-plugin-react-hooks exhaustive-deps violation | Add the dependency, move code into effect, or restructure |
| `React Hook "useX" is called conditionally` | Hook inside if/else, loop, or after early return | Restructure to call hook unconditionally at top level |
| `React Hook "useX" cannot be called in a class component` | Hook used inside a class component | Convert to function component or extract hook logic |
| Effect fires twice in development | StrictMode double-invocation (expected behavior) | Ensure cleanup function properly reverses setup |
| `useFormStatus` always returns `pending: false` | Hook called in same component as `<form>` | Move `useFormStatus` into a child component rendered inside the form |

---

## Rules of Hooks Violations

### Rule 1: ALWAYS Call Hooks at the Top Level

NEVER call hooks inside:
- Conditions (`if`, ternary, `&&`)
- Loops (`for`, `while`, `do...while`)
- Nested functions or callbacks
- `try`/`catch`/`finally` blocks
- After conditional `return` statements
- Event handler functions
- Functions passed to `useMemo`, `useReducer`, or `useEffect`

**Exception**: `use()` (React 19 only) CAN be called inside conditionals and loops.

```typescript
// WRONG: Hook after conditional return
function Profile({ userId }: { userId: string | null }) {
  if (!userId) return <p>No user selected</p>;
  const [user, setUser] = useState<User | null>(null); // ERROR
  // ...
}

// CORRECT: Hook before any conditional logic
function Profile({ userId }: { userId: string | null }) {
  const [user, setUser] = useState<User | null>(null);
  if (!userId) return <p>No user selected</p>;
  // ...
}
```

### Rule 2: ALWAYS Call Hooks from React Functions

NEVER call hooks from regular JavaScript functions.

```typescript
// WRONG: Hook in a utility function
function getWindowSize() {
  const [size, setSize] = useState({ width: 0, height: 0 }); // ERROR
  return size;
}

// CORRECT: Prefix with "use" to make it a custom hook
function useWindowSize() {
  const [size, setSize] = useState({ width: 0, height: 0 });
  // ... effect to track window size
  return size;
}
```

---

## Infinite Re-render Loops

### Pattern 1: setState in Render Body

```typescript
// WRONG: Causes "Too many re-renders"
function Counter() {
  const [count, setCount] = useState(0);
  setCount(count + 1); // Triggers re-render during render
  return <p>{count}</p>;
}

// CORRECT: Update in event handler
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

### Pattern 2: useEffect Without Dependency Array

```typescript
// WRONG: Runs every render, setState triggers re-render, infinite loop
function DataLoader() {
  const [data, setData] = useState<string[]>([]);
  useEffect(() => {
    fetchData().then(setData); // No deps = runs every render
  });
  return <List items={data} />;
}

// CORRECT: Specify dependency array
function DataLoader() {
  const [data, setData] = useState<string[]>([]);
  useEffect(() => {
    fetchData().then(setData);
  }, []); // Runs once on mount
  return <List items={data} />;
}
```

### Pattern 3: New Object/Array Reference in Dependencies

```typescript
// WRONG: New object every render = deps always "change"
function ChatRoom({ roomId }: { roomId: string }) {
  const options = { serverUrl: "https://localhost:1234", roomId };
  useEffect(() => {
    const conn = createConnection(options);
    conn.connect();
    return () => conn.disconnect();
  }, [options]); // New reference every render = infinite loop
}

// CORRECT: Move object creation inside effect
function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const options = { serverUrl: "https://localhost:1234", roomId };
    const conn = createConnection(options);
    conn.connect();
    return () => conn.disconnect();
  }, [roomId]); // Primitive dependency = stable
}
```

---

## Stale Closures

### Timer Callbacks Reading Old State

```typescript
// WRONG: interval callback captures initial count (always 0)
function Timer() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // Always sets to 1
    }, 1000);
    return () => clearInterval(id);
  }, []); // count not in deps, closure captures 0
}

// CORRECT: Use functional updater
function Timer() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => {
      setCount(prev => prev + 1); // Reads latest state
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

### Event Listeners with Stale Refs

```typescript
// WRONG: Handler captures stale state
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0);
  const [message, setMessage] = useState("");

  useEffect(() => {
    function handleScroll() {
      setScrollY(window.scrollY);
      setMessage(`Scrolled ${message.length} chars`); // Stale message
    }
    window.addEventListener("scroll", handleScroll);
    return () => window.removeEventListener("scroll", handleScroll);
  }, []); // message not in deps
}

// CORRECT: Use ref for latest value
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0);
  const [message, setMessage] = useState("");
  const messageRef = useRef(message);
  messageRef.current = message;

  useEffect(() => {
    function handleScroll() {
      setScrollY(window.scrollY);
      setMessage(`Scrolled ${messageRef.current.length} chars`);
    }
    window.addEventListener("scroll", handleScroll);
    return () => window.removeEventListener("scroll", handleScroll);
  }, []);
}
```

---

## useEffect Cleanup Mistakes

### Missing Cleanup — Subscription Leak

```typescript
// WRONG: No cleanup, listener accumulates on every re-render
useEffect(() => {
  window.addEventListener("resize", handleResize);
}, [handleResize]);

// CORRECT: Return cleanup function
useEffect(() => {
  window.addEventListener("resize", handleResize);
  return () => window.removeEventListener("resize", handleResize);
}, [handleResize]);
```

### Missing Cleanup — Async Race Condition

```typescript
// WRONG: Stale responses overwrite fresh data
useEffect(() => {
  fetchUser(userId).then(user => setUser(user));
}, [userId]);

// CORRECT: Ignore flag prevents stale updates
useEffect(() => {
  let ignore = false;
  fetchUser(userId).then(user => {
    if (!ignore) setUser(user);
  });
  return () => { ignore = true; };
}, [userId]);
```

### Missing Cleanup — Timer Leak

```typescript
// WRONG: Interval never cleared on unmount
useEffect(() => {
  setInterval(() => setCount(c => c + 1), 1000);
}, []);

// CORRECT: Store ID and clear on cleanup
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

---

## useEffect Timing

| Hook | Fires | Blocks Paint? | Use Case |
|------|-------|---------------|----------|
| `useInsertionEffect` | Before DOM mutations | Yes | CSS-in-JS libraries ONLY |
| `useLayoutEffect` | After DOM mutations, before paint | Yes | DOM measurement (tooltips, popovers) |
| `useEffect` | After paint | No | Data fetching, subscriptions, most side effects |

NEVER use `useLayoutEffect` unless you need to measure DOM before the browser paints. It blocks rendering and hurts performance.

NEVER use `useInsertionEffect` unless you are building a CSS-in-JS library.

---

## eslint-plugin-react-hooks

### exhaustive-deps — When to Fix vs Suppress

**ALWAYS fix** by default. Suppression is ONLY acceptable in these rare cases:

| Scenario | Action |
|----------|--------|
| Missing primitive dependency | Add it to the array |
| Missing object/function dependency | Move creation inside effect, or use `useMemo`/`useCallback` |
| Intentionally run only on mount | Use `[]` and verify no reactive values are read |
| External stable reference (e.g., dispatch, ref) | Safe to omit — setState, dispatch, and refs are stable |
| Genuinely need to read latest value without re-triggering | Use `useRef` to hold the value |

```typescript
// WRONG: Suppressing a real dependency
useEffect(() => {
  fetchData(query); // query IS reactive
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, []); // Bug: runs with stale query

// CORRECT: Include the dependency
useEffect(() => {
  fetchData(query);
}, [query]);
```

---

## "You Might Not Need an Effect"

NEVER use `useEffect` for these scenarios:

| Scenario | Wrong Approach | Correct Approach |
|----------|---------------|-----------------|
| Data transformation | `useEffect` + `setState` | Calculate during render |
| Expensive computation | `useEffect` + `setState` | `useMemo` |
| Reset state on prop change | `useEffect` + `setState` | `key` prop on component |
| Event-driven logic | `useEffect` watching state | Put logic in event handler |
| Notify parent of changes | `useEffect` + parent callback | Call parent callback in event handler |
| External store subscription | `useEffect` + `addEventListener` | `useSyncExternalStore` |
| App initialization | `useEffect` with `[]` | Module-level code or guarded top-level call |

---

## StrictMode Double-Invocation

In development, React StrictMode runs setup + cleanup + setup for every Effect. This is NOT a bug.

**If your component breaks under StrictMode**, your cleanup function does not properly reverse the setup:

```typescript
// WRONG: Breaks under StrictMode (duplicate connections)
useEffect(() => {
  const conn = createConnection(roomId);
  conn.connect(); // Called twice, two connections open
}, [roomId]);

// CORRECT: Cleanup reverses setup
useEffect(() => {
  const conn = createConnection(roomId);
  conn.connect();
  return () => conn.disconnect(); // Second setup works clean
}, [roomId]);
```

---

## Common useState Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| `useState(expensiveFn())` | Function runs every render | `useState(expensiveFn)` — pass reference |
| `obj.x = 5; setObj(obj)` | Same reference, no re-render | `setObj({...obj, x: 5})` — new object |
| `arr.push(item); setArr(arr)` | Mutation, same reference | `setArr([...arr, item])` |
| `useState(() => myFn)` when storing a function | React calls it as initializer | `useState(() => () => myFn)` — wrap in arrow |
| Reading state right after setState | State updates are async | Use functional updater or useEffect |

---

## Reference Links

- [references/examples.md](references/examples.md) -- Hook error examples with complete wrong/correct code pairs
- [references/anti-patterns.md](references/anti-patterns.md) -- Comprehensive catalog of hook mistakes organized by category

### Official Sources

- https://react.dev/reference/rules/rules-of-hooks
- https://react.dev/reference/react/useEffect
- https://react.dev/reference/react/useState
- https://react.dev/learn/you-might-not-need-an-effect
- https://react.dev/learn/synchronizing-with-effects
- https://react.dev/learn/removing-effect-dependencies
