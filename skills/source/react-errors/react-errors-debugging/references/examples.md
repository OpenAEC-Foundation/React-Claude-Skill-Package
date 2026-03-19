# Debugging Workflow Examples

## Example 1: Diagnosing an Infinite Re-render Loop

### Symptom
Console shows: "Maximum update depth exceeded. This can happen when a component calls setState inside useEffect, but useEffect either doesn't have a dependency array, or one of the dependencies changes on every render."

### Step-by-Step Debugging Workflow

1. **Read the error stack trace** — identify which component is looping:

```
Warning: Maximum update depth exceeded.
    at SearchResults (SearchResults.tsx:12)
    at Dashboard (Dashboard.tsx:5)
    at App (App.tsx:3)
```

2. **Open the component** and find all `useEffect` + `useState` pairs:

```tsx
// BUG: This causes infinite re-renders
function SearchResults({ query }: { query: string }): JSX.Element {
  const [results, setResults] = useState<Item[]>([]);
  const [filters, setFilters] = useState({ sort: "name", order: "asc" });

  // This creates a new object every render, so the effect re-runs every time
  const options = { query, ...filters };

  useEffect(() => {
    fetchResults(options).then(setResults);
  }, [options]); // options is a NEW object every render!

  return <div>{results.map((r) => <ResultCard key={r.id} item={r} />)}</div>;
}
```

3. **Identify the problem**: `options` is a new object reference every render, so the `useEffect` dependency is never stable.

4. **Fix with useMemo or inline dependencies**:

```tsx
// FIXED: Stable dependencies
function SearchResults({ query }: { query: string }): JSX.Element {
  const [results, setResults] = useState<Item[]>([]);
  const [filters, setFilters] = useState({ sort: "name", order: "asc" });

  useEffect(() => {
    fetchResults({ query, ...filters }).then(setResults);
  }, [query, filters.sort, filters.order]); // primitive deps are stable

  return <div>{results.map((r) => <ResultCard key={r.id} item={r} />)}</div>;
}
```

---

## Example 2: Debugging a Hydration Mismatch

### Symptom
Console shows: "Hydration failed because the initial UI does not match what was rendered on the server."

### Step-by-Step Debugging Workflow

1. **Check for browser-only values in render**:

```tsx
// BUG: window.innerWidth differs between server (undefined) and client
function Layout({ children }: { children: React.ReactNode }): JSX.Element {
  const isMobile = window.innerWidth < 768; // NEVER access window during SSR render
  return <div className={isMobile ? "mobile" : "desktop"}>{children}</div>;
}
```

2. **Fix with useEffect for client-only values**:

```tsx
// FIXED: Server and client render the same initial HTML
function Layout({ children }: { children: React.ReactNode }): JSX.Element {
  const [isMobile, setIsMobile] = useState<boolean>(false);

  useEffect(() => {
    setIsMobile(window.innerWidth < 768);
    const handleResize = (): void => setIsMobile(window.innerWidth < 768);
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return <div className={isMobile ? "mobile" : "desktop"}>{children}</div>;
}
```

3. **React 19 improvement**: The error message now shows an exact diff of what mismatched:

```
Uncaught Error: Hydration failed because the initial UI does not match.

- Server: <div class="desktop">
+ Client: <div class="mobile">
```

### Common Hydration Mismatch Sources

| Source | Why It Mismatches | Fix |
|--------|------------------|-----|
| `Date.now()` / `new Date()` | Different timestamp on server vs client | Pass timestamp as prop from server |
| `Math.random()` | Different value each call | Use seeded random or generate server-side |
| `window` / `document` access | Undefined on server | Guard with `useEffect` or `typeof window` |
| Browser extensions | Inject extra DOM nodes | React 19 handles `<script>`/`<style>` in head/body |
| Conditional rendering on auth | Server has no auth state | Use `useEffect` for auth-dependent UI |

---

## Example 3: Performance Profiling a Slow List

### Symptom
A list of 200 items feels sluggish when filtering. User types in a search box and there is visible lag.

### Step-by-Step Profiling Workflow

1. **Open React DevTools Profiler** -> click gear icon -> enable "Record why each component rendered while profiling"

2. **Record**: Click the blue record button, type a character in the search box, click stop

3. **Analyze the flamegraph**:
   - Look for the widest bars (longest render time)
   - Grey bars did NOT render (good -- they were memoized or unchanged)
   - Yellow/orange bars took significant time

4. **Check "Why did this render?"** for the slow component:
   - "Props changed: items" — the parent is passing a new array reference
   - "Hook 3 changed" — a hook value updated

5. **Identify the problem**:

```tsx
// BUG: items is a new array every render, causing ALL ListItems to re-render
function SearchableList({ allItems }: { allItems: Item[] }): JSX.Element {
  const [query, setQuery] = useState<string>("");

  // This creates a new array every render
  const filtered = allItems.filter((item) =>
    item.name.toLowerCase().includes(query.toLowerCase())
  );

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {filtered.map((item) => (
        <ListItem key={item.id} item={item} />
      ))}
    </div>
  );
}

function ListItem({ item }: { item: Item }): JSX.Element {
  // Expensive render (formatting, calculations)
  return <div>{formatItem(item)}</div>;
}
```

6. **Fix with useMemo and React.memo**:

```tsx
import { memo, useMemo, useState } from "react";

function SearchableList({ allItems }: { allItems: Item[] }): JSX.Element {
  const [query, setQuery] = useState<string>("");

  const filtered = useMemo(
    () =>
      allItems.filter((item) =>
        item.name.toLowerCase().includes(query.toLowerCase())
      ),
    [allItems, query]
  );

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {filtered.map((item) => (
        <MemoizedListItem key={item.id} item={item} />
      ))}
    </div>
  );
}

const MemoizedListItem = memo(function ListItem({ item }: { item: Item }): JSX.Element {
  return <div>{formatItem(item)}</div>;
});
```

7. **Re-profile**: Record the same interaction and compare. The flamegraph should show grey (skipped) bars for unchanged items.

---

## Example 4: Debugging a Missing Cleanup in useEffect

### Symptom
Strict Mode causes a subscription to fire twice, or data appears duplicated after navigating away and back.

### Step-by-Step Debugging Workflow

1. **Strict Mode reveals the issue** by mounting, unmounting, and remounting the component. Console shows:

```
Connected to room: general
Connected to room: general   // <-- duplicate! No cleanup happened
```

2. **Find the effect without cleanup**:

```tsx
// BUG: No cleanup — Strict Mode shows duplicate connections
function ChatRoom({ roomId }: { roomId: string }): JSX.Element {
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    const socket = connectToRoom(roomId);
    socket.on("message", (msg: Message) => {
      setMessages((prev) => [...prev, msg]);
    });
    console.log(`Connected to room: ${roomId}`);
    // MISSING: return cleanup function
  }, [roomId]);

  return <MessageList messages={messages} />;
}
```

3. **Add cleanup**:

```tsx
// FIXED: Cleanup disconnects on unmount and before re-running
function ChatRoom({ roomId }: { roomId: string }): JSX.Element {
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    const socket = connectToRoom(roomId);
    socket.on("message", (msg: Message) => {
      setMessages((prev) => [...prev, msg]);
    });
    console.log(`Connected to room: ${roomId}`);
    return () => {
      socket.disconnect();
      console.log(`Disconnected from room: ${roomId}`);
    };
  }, [roomId]);

  return <MessageList messages={messages} />;
}
```

4. **Verify with Strict Mode**: Console now shows the correct mount/unmount/remount pattern:

```
Connected to room: general
Disconnected from room: general    // cleanup ran
Connected to room: general         // remount — this is normal in StrictMode
```

---

## Example 5: Tracking Down "Cannot Update While Rendering"

### Symptom
Console shows: "Cannot update a component (`Parent`) while rendering a different component (`Child`)."

### Debugging Workflow

1. **Identify the components** from the warning — `Parent` state is being set inside `Child` render:

```tsx
// BUG: Child sets Parent state during its own render
function Parent(): JSX.Element {
  const [count, setCount] = useState<number>(0);
  return <Child count={count} onUpdate={setCount} />;
}

function Child({ count, onUpdate }: { count: number; onUpdate: (n: number) => void }): JSX.Element {
  if (count > 10) {
    onUpdate(0); // NEVER call parent setState during render!
  }
  return <div>{count}</div>;
}
```

2. **Fix by moving to useEffect**:

```tsx
// FIXED: State update happens in an effect, not during render
function Child({ count, onUpdate }: { count: number; onUpdate: (n: number) => void }): JSX.Element {
  useEffect(() => {
    if (count > 10) {
      onUpdate(0);
    }
  }, [count, onUpdate]);

  return <div>{count}</div>;
}
```

---

## Example 6: Using why-did-you-render to Find Unnecessary Renders

### Setup

```tsx
// src/wdyr.ts — import this FIRST in your entry point
import React from "react";

if (process.env.NODE_ENV === "development") {
  const whyDidYouRender = await import("@welldone-software/why-did-you-render");
  whyDidYouRender.default(React, {
    trackAllPureComponents: true,
    logOnDifferentValues: true,
  });
}
```

```tsx
// src/main.tsx
import "./wdyr"; // MUST be first import
import { createRoot } from "react-dom/client";
import { App } from "./App";

createRoot(document.getElementById("root")!).render(<App />);
```

### Tag Components to Track

```tsx
function ExpensiveTable({ data }: { data: Row[] }): JSX.Element {
  return (
    <table>
      {data.map((row) => <TableRow key={row.id} row={row} />)}
    </table>
  );
}

// Enable tracking on this specific component
ExpensiveTable.whyDidYouRender = true;
```

### Interpreting Output

The console will show messages like:

```
ExpensiveTable
  Re-rendered because of props changes:
    data: {prev: Array(50), next: Array(50)}
    Values are equal by value but not by reference.
```

This tells you: the `data` array has the same contents but a new reference — memoize the array with `useMemo` in the parent.
