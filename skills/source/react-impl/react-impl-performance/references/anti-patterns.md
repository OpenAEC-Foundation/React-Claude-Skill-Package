# react-impl-performance — Anti-Patterns

## Anti-Pattern 1: Premature Optimization

```tsx
// WRONG: Wrapping everything in memo/useMemo/useCallback "just in case"
const MemoizedHeader = memo(function Header() {
  return <h1>Welcome</h1>;
});

const title = useMemo(() => 'Welcome', []);
const handleClick = useCallback(() => alert('hi'), []);
```

**WHY this is wrong**: `memo`, `useMemo`, and `useCallback` add overhead (comparison cost, memory for cached values). For cheap components, the comparison cost exceeds the re-render cost. ALWAYS measure with Profiler first.

**CORRECT approach**: Write code without optimization. Profile with React DevTools. Optimize ONLY the components that show measurable slowness.

---

## Anti-Pattern 2: useMemo for Trivial Calculations

```tsx
// WRONG: String concatenation is trivial — useMemo adds overhead for no benefit
const fullName = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);

// CORRECT: Calculate inline during render
const fullName = `${firstName} ${lastName}`;
```

**Rule of thumb**: If the calculation takes less than 1ms on your slowest target device, do NOT memoize it.

---

## Anti-Pattern 3: useCallback Without memo on the Child

```tsx
// WRONG: useCallback is pointless — Child is NOT memoized
function Parent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return <Child onClick={handleClick} />;
}

// Child re-renders anyway because it is not wrapped in memo
function Child({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Click</button>;
}
```

**WHY this is wrong**: `useCallback` only prevents re-renders when paired with `React.memo` on the receiving component. Without `memo`, the child re-renders whenever the parent re-renders regardless of prop stability.

**CORRECT**: Either add `memo` to the child, or remove `useCallback` from the parent.

---

## Anti-Pattern 4: JSON.stringify in Custom arePropsEqual

```tsx
// WRONG: JSON.stringify is slower than just re-rendering
const MemoizedComponent = memo(MyComponent, (prev, next) => {
  return JSON.stringify(prev) === JSON.stringify(next);
});
```

**WHY this is wrong**: `JSON.stringify` traverses the entire prop tree, converting to strings, then comparing strings. This is almost always slower than letting React do a shallow comparison plus a re-render.

**CORRECT**: Use the default shallow comparison. If specific deep props need comparison, compare only those specific values.

---

## Anti-Pattern 5: Ignoring Function Props in Custom Comparisons

```tsx
// WRONG: Skipping function comparison causes stale closures
const MemoizedChild = memo(Child, (prev, next) => {
  return prev.data === next.data;
  // MISSING: prev.onSelect === next.onSelect
});
```

**WHY this is wrong**: If `onSelect` captures state via closure, the child will keep calling the old version with stale state. This leads to bugs where the handler uses outdated values.

**CORRECT**: Compare ALL props, or use the default comparison (which already does this).

---

## Anti-Pattern 6: Creating Objects/Arrays in JSX Props

```tsx
// WRONG: New array and object on every render — defeats memo
<MemoizedChart
  data={data.filter((d) => d.active)}     // new array every render
  style={{ marginTop: 10 }}               // new object every render
  colors={['red', 'blue', 'green']}       // new array every render
/>
```

**CORRECT**: Hoist constants and memoize computed values.

```tsx
const CHART_STYLE = { marginTop: 10 } as const;
const CHART_COLORS = ['red', 'blue', 'green'] as const;

function Dashboard({ data }: { data: DataPoint[] }) {
  const activeData = useMemo(
    () => data.filter((d) => d.active),
    [data]
  );

  return (
    <MemoizedChart
      data={activeData}
      style={CHART_STYLE}
      colors={CHART_COLORS}
    />
  );
}
```

---

## Anti-Pattern 7: Declaring React.lazy Inside a Component

```tsx
// WRONG: Creates a new lazy component on every render — causes remounting
function App() {
  const Dashboard = lazy(() => import('./Dashboard')); // recreated every render!

  return (
    <Suspense fallback={<Loading />}>
      <Dashboard />
    </Suspense>
  );
}
```

**WHY this is wrong**: Each render creates a new component type. React sees it as a different component and unmounts/remounts it, losing all state and triggering a new import.

**CORRECT**: ALWAYS declare `lazy` at module level.

```tsx
const Dashboard = lazy(() => import('./Dashboard'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Dashboard />
    </Suspense>
  );
}
```

---

## Anti-Pattern 8: Rendering Thousands of DOM Nodes

```tsx
// WRONG: 10,000 DOM nodes — browser becomes unresponsive
function UserList({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

**WHY this is wrong**: The browser must create, lay out, and paint 10,000+ DOM nodes. Scrolling becomes janky. Memory usage spikes. Initial render takes hundreds of milliseconds.

**CORRECT**: Use virtualization — render only the visible items plus a small overscan buffer. See `@tanstack/react-virtual` in the SKILL.md.

---

## Anti-Pattern 9: Effect Chains for Derived State

```tsx
// WRONG: Cascading effects cause multiple render cycles
const [items, setItems] = useState<Item[]>([]);
const [filteredItems, setFilteredItems] = useState<Item[]>([]);
const [count, setCount] = useState(0);

useEffect(() => {
  setFilteredItems(items.filter((i) => i.active));
}, [items]);

useEffect(() => {
  setCount(filteredItems.length);
}, [filteredItems]);
```

**WHY this is wrong**: Three renders: (1) items change, (2) filteredItems updates, (3) count updates. Each effect triggers a new render cycle.

**CORRECT**: Calculate derived values during render.

```tsx
const [items, setItems] = useState<Item[]>([]);
const filteredItems = useMemo(() => items.filter((i) => i.active), [items]);
const count = filteredItems.length;
// Single render with correct values
```

---

## Anti-Pattern 10: Misusing useMemo with Object Literal Syntax

```tsx
// WRONG: Returns undefined — block body without return
const options = useMemo(() => {
  matchMode: 'whole-word', text
}, [text]);

// CORRECT: Wrap object literal in parentheses
const options = useMemo(() => ({
  matchMode: 'whole-word' as const,
  text,
}), [text]);
```

**WHY this is wrong**: JavaScript interprets `{ matchMode: ... }` as a code block with a label, not an object literal. The function returns `undefined`.

---

## Anti-Pattern 11: Using flushSync for Performance

```tsx
// WRONG: flushSync breaks batching
import { flushSync } from 'react-dom';

function handleClick() {
  flushSync(() => setCount((c) => c + 1));
  flushSync(() => setFlag((f) => !f));
  // TWO renders instead of one
}
```

**WHY this is wrong**: `flushSync` forces synchronous DOM updates, breaking React's automatic batching. Each call causes a separate render cycle. ONLY use `flushSync` when you must read the DOM immediately after a state update (e.g., scrolling to a newly added element).

---

## Summary Table

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Memo everywhere | Comparison overhead > render cost | Measure first, then optimize |
| useMemo for trivial work | Overhead > calculation cost | Inline the calculation |
| useCallback without memo child | No effect, wasted overhead | Add memo or remove useCallback |
| JSON.stringify comparison | Slower than re-rendering | Use default shallow comparison |
| New objects in JSX props | Defeats memo | Hoist constants, useMemo for computed |
| lazy() inside component | Remounts every render | Declare at module level |
| Thousands of DOM nodes | Jank, high memory | Use @tanstack/react-virtual |
| Effect chains for derived state | Multiple render cycles | Calculate during render / useMemo |
| flushSync for batching | Breaks batching | Let React batch automatically |
