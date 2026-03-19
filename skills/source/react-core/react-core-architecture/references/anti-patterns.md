# react-core-architecture -- Anti-Patterns

## Render Phase Violations

### NEVER: Mutate State During Render

```tsx
// BAD: Mutating state during render causes infinite loops
function Counter({ items }: { items: string[] }): React.ReactElement {
  const [count, setCount] = useState(0);
  setCount(items.length); // NEVER -- triggers re-render during render

  return <p>{count}</p>;
}

// GOOD: Derive values directly or use useEffect for side effects
function Counter({ items }: { items: string[] }): React.ReactElement {
  const count = items.length; // Derive from props -- no state needed

  return <p>{count}</p>;
}
```

### NEVER: Mutate Props or External Variables in Render

```tsx
// BAD: Mutating an array during render is impure
let globalList: string[] = [];

function BadComponent({ item }: { item: string }): React.ReactElement {
  globalList.push(item); // NEVER -- side effect in render phase
  return <span>{item}</span>;
}

// GOOD: Use state or effects for mutations
function GoodComponent({ item }: { item: string }): React.ReactElement {
  useEffect(() => {
    globalList.push(item);
    return () => {
      globalList = globalList.filter(i => i !== item);
    };
  }, [item]);

  return <span>{item}</span>;
}
```

### NEVER: Perform Side Effects in Render

```tsx
// BAD: Fetch data during render
function UserProfile({ userId }: { userId: string }): React.ReactElement {
  // NEVER -- network request in render phase
  const response = fetch(`/api/users/${userId}`);
  // ...
}

// GOOD: Use useEffect for side effects
function UserProfile({ userId }: { userId: string }): React.ReactElement {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    let cancelled = false;
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then((data: User) => {
        if (!cancelled) setUser(data);
      });
    return () => { cancelled = true; };
  }, [userId]);

  return user ? <h1>{user.name}</h1> : <p>Loading...</p>;
}
```

---

## Component Design Violations

### NEVER: Use Inheritance for Component Reuse

```tsx
// BAD: Class inheritance for shared behavior
class BaseComponent extends React.Component {
  getFormattedDate() {
    return new Date().toLocaleDateString();
  }
}

class DateDisplay extends BaseComponent {
  render() {
    return <span>{this.getFormattedDate()}</span>;
  }
}

// GOOD: Use composition with hooks or children
function usFormattedDate(): string {
  return new Date().toLocaleDateString();
}

function DateDisplay(): React.ReactElement {
  const date = usFormattedDate();
  return <span>{date}</span>;
}
```

### NEVER: Create Components Inside Other Components

```tsx
// BAD: Component defined inside render -- recreated every render
function Parent(): React.ReactElement {
  // NEVER -- this creates a new component type on every render
  // causing unmount/remount of the entire subtree and losing state
  function Child(): React.ReactElement {
    const [count, setCount] = useState(0); // State resets every parent render!
    return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
  }

  return <Child />;
}

// GOOD: Define components at module scope
function Child(): React.ReactElement {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

function Parent(): React.ReactElement {
  return <Child />;
}
```

---

## Reconciliation Anti-Patterns

### NEVER: Use Array Index as Key for Dynamic Lists

```tsx
interface Item {
  id: string;
  text: string;
}

// BAD: Index keys cause bugs when items are reordered, inserted, or deleted
function BadList({ items }: { items: Item[] }): React.ReactElement {
  return (
    <ul>
      {items.map((item, index) => (
        // NEVER for dynamic lists -- causes incorrect state association
        <li key={index}>
          <input defaultValue={item.text} />
        </li>
      ))}
    </ul>
  );
}
// When an item is removed from the middle, inputs shift to wrong items

// GOOD: Use stable unique identifiers
function GoodList({ items }: { items: Item[] }): React.ReactElement {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>
          <input defaultValue={item.text} />
        </li>
      ))}
    </ul>
  );
}
```

### NEVER: Generate Random Keys

```tsx
// BAD: Random keys force remount on every render
function BadList({ items }: { items: string[] }): React.ReactElement {
  return (
    <ul>
      {items.map(item => (
        // NEVER -- Math.random() produces new key each render = full remount
        <li key={Math.random()}>{item}</li>
      ))}
    </ul>
  );
}

// GOOD: Use stable identifiers from data
function GoodList({ items }: { items: Array<{ id: string; text: string }> }): React.ReactElement {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.text}</li>
      ))}
    </ul>
  );
}
```

---

## Entry Point Anti-Patterns

### NEVER: Use createRoot for Server-Rendered HTML

```tsx
// BAD: Discards all server-rendered markup, causes flash of empty content
import { createRoot } from 'react-dom/client';

const root = createRoot(document.getElementById('root')!);
root.render(<App />); // Server HTML is thrown away

// GOOD: Use hydrateRoot to attach to existing server markup
import { hydrateRoot } from 'react-dom/client';

hydrateRoot(document.getElementById('root')!, <App />);
```

### NEVER: Create Multiple Roots for the Same DOM Node

```tsx
// BAD: Creating multiple roots for the same container
const container = document.getElementById('root')!;
const root1 = createRoot(container);
const root2 = createRoot(container); // NEVER -- undefined behavior

root1.render(<App />);
root2.render(<App />); // Overwrites root1

// GOOD: Reuse the same root instance
const root = createRoot(container);
root.render(<App />);
// Later, to update:
root.render(<UpdatedApp />); // Same root, new content
```

---

## State Management Anti-Patterns

### NEVER: Pass Data Upward by Mutating Parent References

```tsx
// BAD: Child mutates parent's data directly
function Parent(): React.ReactElement {
  const data = { count: 0 };

  return <Child data={data} />;
}

function Child({ data }: { data: { count: number } }): React.ReactElement {
  return (
    // NEVER -- mutating props breaks unidirectional data flow
    <button onClick={() => { data.count += 1; }}>
      {data.count}
    </button>
  );
}

// GOOD: Use callbacks to communicate upward
function Parent(): React.ReactElement {
  const [count, setCount] = useState(0);

  return <Child count={count} onIncrement={() => setCount(c => c + 1)} />;
}

function Child({ count, onIncrement }: {
  count: number;
  onIncrement: () => void;
}): React.ReactElement {
  return <button onClick={onIncrement}>{count}</button>;
}
```

### NEVER: Mutate State Objects Directly

```tsx
interface UserState {
  name: string;
  age: number;
}

// BAD: Mutating state object in place
function BadForm(): React.ReactElement {
  const [user, setUser] = useState<UserState>({ name: '', age: 0 });

  const handleNameChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
    user.name = e.target.value; // NEVER -- React does not detect this mutation
    setUser(user);              // Same reference, no re-render triggered
  };

  return <input value={user.name} onChange={handleNameChange} />;
}

// GOOD: Create new object with spread
function GoodForm(): React.ReactElement {
  const [user, setUser] = useState<UserState>({ name: '', age: 0 });

  const handleNameChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
    setUser(prev => ({ ...prev, name: e.target.value }));
  };

  return <input value={user.name} onChange={handleNameChange} />;
}
```

---

## StrictMode Anti-Patterns

### NEVER: Rely on Render Count or Timing

```tsx
// BAD: Assumes component renders exactly once on mount
let renderCount = 0;

function BadComponent(): React.ReactElement {
  renderCount += 1; // In StrictMode: increments twice per mount

  useEffect(() => {
    // NEVER assume this runs once -- StrictMode runs setup->cleanup->setup
    analytics.trackPageView();
  }, []);

  return <div>Renders: {renderCount}</div>;
}

// GOOD: Use refs for render counting, guard effects properly
function GoodComponent(): React.ReactElement {
  const renderCountRef = useRef(0);
  renderCountRef.current += 1;

  useEffect(() => {
    // Effect with proper cleanup works correctly even with double-invocation
    const controller = new AbortController();
    analytics.trackPageView({ signal: controller.signal });
    return () => controller.abort();
  }, []);

  return <div>Renders: {renderCountRef.current}</div>;
}
```

---

## React 19 Migration Anti-Patterns

### NEVER: Use forwardRef in React 19 (Deprecated)

```tsx
// BAD in React 19: forwardRef is unnecessary
import { forwardRef } from 'react';

const MyInput = forwardRef<HTMLInputElement, { label: string }>(
  ({ label }, ref) => (
    <label>
      {label}
      <input ref={ref} />
    </label>
  )
);

// GOOD in React 19: ref is a regular prop
function MyInput({ label, ref }: {
  label: string;
  ref?: React.Ref<HTMLInputElement>;
}): React.ReactElement {
  return (
    <label>
      {label}
      <input ref={ref} />
    </label>
  );
}
```

### NEVER: Use Context.Provider in React 19

```tsx
import { createContext } from 'react';

const MyContext = createContext('default');

// BAD in React 19: Provider is deprecated
function BadApp(): React.ReactElement {
  return (
    <MyContext.Provider value="hello">
      <Child />
    </MyContext.Provider>
  );
}

// GOOD in React 19: Use Context directly
function GoodApp(): React.ReactElement {
  return (
    <MyContext value="hello">
      <Child />
    </MyContext>
  );
}
```
