# Debugging Anti-Patterns

## Anti-Pattern 1: Removing StrictMode to Fix Double Renders

### The Mistake

```tsx
// WRONG: Removing StrictMode to stop "duplicate" effects
createRoot(document.getElementById("root")!).render(
  // <StrictMode>  <-- removed to "fix" double-render
    <App />
  // </StrictMode>
);
```

### Why It Is Wrong

StrictMode double-invokes components and effects **on purpose** to expose bugs:
- Impure render functions that produce side effects
- Effects without proper cleanup
- State initializers with side effects

Removing StrictMode does NOT fix the bug — it hides it. The bug will manifest in production as:
- Memory leaks from uncleared subscriptions
- Stale data from missing cleanup
- Race conditions from uncancelled async operations

### The Correct Fix

ALWAYS keep StrictMode and fix the underlying issue:

```tsx
// CORRECT: Keep StrictMode, fix the effect
createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);

// Fix: Add cleanup to your effect
useEffect(() => {
  const controller = new AbortController();
  fetchData({ signal: controller.signal }).then(setData);
  return () => controller.abort(); // Cleanup cancels the fetch
}, []);
```

---

## Anti-Pattern 2: Using console.log Instead of DevTools

### The Mistake

```tsx
// WRONG: Littering components with console.log for debugging
function UserList({ users }: { users: User[] }): JSX.Element {
  console.log("UserList rendered", users);
  console.log("users length:", users.length);

  return (
    <ul>
      {users.map((user) => {
        console.log("rendering user:", user.id);
        return <li key={user.id}>{user.name}</li>;
      })}
    </ul>
  );
}
```

### Why It Is Wrong

- Console.log in render runs on EVERY render, flooding the console
- Does not show render timing or cause
- Must be manually added and removed
- Does not reveal the component tree or hook state
- StrictMode double-renders make output confusing

### The Correct Approach

Use React DevTools Components tab to inspect props, state, and hooks. Use the Profiler tab to understand render timing and causes:

1. **For state/props inspection**: Select the component in DevTools Components tab
2. **For render count/timing**: Use the Profiler tab with "why did this render" enabled
3. **For specific values**: Use breakpoints in the Sources tab instead of console.log
4. **For re-render tracking**: Use `why-did-you-render` library

If you MUST use console.log, ALWAYS remove it before committing. NEVER ship console.log statements to production.

---

## Anti-Pattern 3: Ignoring Key Warnings

### The Mistake

```tsx
// WRONG: Using array index as key for dynamic lists
function TodoList({ todos }: { todos: Todo[] }): JSX.Element {
  return (
    <ul>
      {todos.map((todo, index) => (
        <li key={index}> {/* NEVER use index for reorderable lists */}
          <input type="checkbox" checked={todo.done} />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

### Why It Is Wrong

When items are reordered, added, or removed from the middle of the list:
- React matches elements by key position, not by data identity
- Input state (checkbox, text input) stays attached to the wrong item
- DOM elements are unnecessarily destroyed and recreated
- Animations break because elements are treated as new

### The Correct Fix

ALWAYS use a stable, unique identifier from the data:

```tsx
// CORRECT: Stable unique key from data
function TodoList({ todos }: { todos: Todo[] }): JSX.Element {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}> {/* Stable ID from data */}
          <input type="checkbox" checked={todo.done} />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

Index keys are ONLY acceptable when ALL of these are true:
- The list is static (never reordered, filtered, or items added/removed)
- Items have no local state or uncontrolled inputs
- Items have no stable unique ID available

---

## Anti-Pattern 4: Suppressing TypeScript Errors During Debugging

### The Mistake

```tsx
// WRONG: Suppressing errors to "make it work"
function UserProfile({ userId }: { userId: string }): JSX.Element {
  const [user, setUser] = useState<User | null>(null);

  // @ts-ignore — TODO: fix later
  return <div>{user.name}</div>; // Runtime error: Cannot read properties of null
}
```

### Why It Is Wrong

TypeScript errors during debugging are SIGNALS, not obstacles:
- `user` is `null` initially — the type system is telling you to handle the null case
- `@ts-ignore` hides the bug that will crash at runtime
- "Fix later" comments become permanent technical debt

### The Correct Approach

ALWAYS handle the types properly — they guide you to the fix:

```tsx
// CORRECT: Handle the null state the type system warned about
function UserProfile({ userId }: { userId: string }): JSX.Element {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);

  if (user === null) {
    return <div>Loading...</div>;
  }

  return <div>{user.name}</div>; // TypeScript knows user is User here
}
```

---

## Anti-Pattern 5: Debugging Production Without Source Maps

### The Mistake

Trying to debug a production error like:
```
Uncaught Error: Minified React error #310; visit https://react.dev/errors/310
    at a.js:1:4521
    at b.js:1:891
```

Without source maps, you cannot:
- See original file names or line numbers
- Set meaningful breakpoints
- Understand the actual code path

### The Correct Approach

1. **ALWAYS generate source maps** for staging/QA environments:

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    sourcemap: true, // Generates .map files alongside bundles
  },
});
```

2. **For production**: Upload source maps to your error monitoring service (Sentry, Bugsnag) but do NOT serve them publicly:

```typescript
// vite.config.ts — hidden source maps for production
export default defineConfig({
  build: {
    sourcemap: "hidden", // Generates .map files but does not reference them in bundles
  },
});
```

3. **Look up minified error codes**: Visit the URL in the error message (e.g., `https://react.dev/errors/310`) to see the full error text with parameter values.

---

## Anti-Pattern 6: Debugging State by Mutating It

### The Mistake

```tsx
// WRONG: Mutating state to "test" a fix
function ItemList(): JSX.Element {
  const [items, setItems] = useState<string[]>(["a", "b", "c"]);

  const addItem = (): void => {
    items.push("d"); // NEVER mutate state directly
    setItems(items); // Same reference — React may skip the re-render
    console.log(items); // Shows ["a", "b", "c", "d"] but UI may not update
  };

  return (
    <div>
      {items.map((item, i) => <span key={i}>{item}</span>)}
      <button onClick={addItem}>Add</button>
    </div>
  );
}
```

### Why It Is Wrong

- React uses reference equality to detect state changes
- Mutating the existing array and calling `setState` with the same reference may not trigger a re-render
- Even when it does re-render, the "previous" state is corrupted because you mutated it
- This makes debugging harder because the state appears correct in console.log but the UI does not match

### The Correct Approach

ALWAYS create new references when updating state:

```tsx
// CORRECT: Immutable state update
const addItem = (): void => {
  setItems((prev) => [...prev, "d"]); // New array reference
};
```

---

## Anti-Pattern 7: Wrapping Everything in ErrorBoundary Without Granularity

### The Mistake

```tsx
// WRONG: Single error boundary at the root catches everything
function App(): JSX.Element {
  return (
    <ErrorBoundary fallback={<div>Something went wrong</div>}>
      <Header />
      <Sidebar />
      <MainContent />
      <Footer />
    </ErrorBoundary>
  );
}
// If MainContent throws, the ENTIRE app shows the fallback
```

### Why It Is Wrong

- A single error boundary replaces the entire UI with the fallback
- Users lose access to navigation, sidebar, and all other functionality
- You cannot tell which section caused the error without reading the error log
- Recovery requires a full page reload

### The Correct Approach

ALWAYS use granular error boundaries around independent UI sections:

```tsx
// CORRECT: Granular error boundaries preserve working UI
function App(): JSX.Element {
  return (
    <>
      <Header /> {/* Header stays visible even if content crashes */}
      <div className="layout">
        <ErrorBoundary fallback={<SidebarError />}>
          <Sidebar />
        </ErrorBoundary>
        <ErrorBoundary fallback={<ContentError />}>
          <MainContent />
        </ErrorBoundary>
      </div>
      <Footer />
    </>
  );
}
```

---

## Anti-Pattern 8: Not Checking for Duplicate React Installations

### The Mistake

Getting "Invalid hook call" errors and spending hours debugging hook logic, when the actual cause is two copies of React in the bundle.

### Why It Happens

- A dependency bundles its own copy of React
- `npm link` or monorepo setup creates duplicate React instances
- Mismatched `react` and `react-dom` versions

### The Correct Diagnostic

ALWAYS check for duplicate React installations first when you see "Invalid hook call":

```bash
# Check for multiple React installations
npm ls react
npm ls react-dom

# Expected: single version at root level
# Problem: multiple versions at different depths
```

If duplicates exist, resolve with:

```bash
# In the consuming package, force resolution to a single React
# package.json
{
  "overrides": {
    "react": "$react",
    "react-dom": "$react-dom"
  }
}
```

ALWAYS verify `react` and `react-dom` are the exact same version. Mismatched versions cause hooks to fail silently.
