# react-core-architecture -- Working Examples

## Basic Application Entry Point

### React 18+ with createRoot

```tsx
// main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

const rootElement = document.getElementById('root');
if (!rootElement) throw new Error('Root element not found');

const root = createRoot(rootElement);
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

### Multiple Roots (Partial React Integration)

```tsx
// Mount React widgets into an existing non-React page
import { createRoot } from 'react-dom/client';
import { Navigation } from './Navigation';
import { Comments } from './Comments';

const navRoot = createRoot(document.getElementById('navigation')!);
navRoot.render(<Navigation />);

const commentsRoot = createRoot(document.getElementById('comments')!);
commentsRoot.render(<Comments />);
```

---

## React Element Creation

### JSX vs createElement

```tsx
// These are equivalent:

// JSX syntax
const element = <h1 className="greeting">Hello, world!</h1>;

// createElement syntax
import { createElement } from 'react';
const element = createElement('h1', { className: 'greeting' }, 'Hello, world!');

// Both produce this object:
// {
//   type: 'h1',
//   props: { className: 'greeting', children: 'Hello, world!' },
//   key: null,
//   ref: null
// }
```

### Component Elements vs Host Elements

```tsx
import { createElement } from 'react';

interface GreetingProps {
  name: string;
}

function Greeting({ name }: GreetingProps): React.ReactElement {
  // Returns a HOST element (string type = DOM node)
  return <h1>Hello, {name}</h1>;
}

// A COMPONENT element (function type = React component)
const componentElement = <Greeting name="Taylor" />;
// Equivalent: createElement(Greeting, { name: 'Taylor' })

// A HOST element (string type = DOM node)
const hostElement = <div id="container" />;
// Equivalent: createElement('div', { id: 'container' })
```

---

## Rendering Phases in Action

### Pure Render Phase

```tsx
interface CounterProps {
  initialCount: number;
}

// CORRECT: Pure component -- same props always produce the same output
function Counter({ initialCount }: CounterProps): React.ReactElement {
  const [count, setCount] = React.useState(initialCount);

  // This function body is the "render phase" -- it MUST be pure
  const doubled = count * 2; // Pure calculation -- OK

  return (
    <div>
      <p>Count: {count} (doubled: {doubled})</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}
```

### Side Effects in the Correct Phase

```tsx
import { useState, useEffect } from 'react';

interface User {
  id: string;
  name: string;
}

function UserProfile({ userId }: { userId: string }): React.ReactElement {
  const [user, setUser] = useState<User | null>(null);

  // Side effect in useEffect -- runs AFTER the commit phase
  useEffect(() => {
    let cancelled = false;

    async function fetchUser(): Promise<void> {
      const response = await fetch(`/api/users/${userId}`);
      const data: User = await response.json();
      if (!cancelled) {
        setUser(data);
      }
    }

    fetchUser();

    // Cleanup runs on unmount or before re-running
    return () => {
      cancelled = true;
    };
  }, [userId]);

  if (!user) return <p>Loading...</p>;
  return <h1>{user.name}</h1>;
}
```

---

## Component Lifecycle Demonstration

### Mount, Update, Unmount

```tsx
import { useState, useEffect, useRef } from 'react';

function LifecycleDemo(): React.ReactElement {
  const [count, setCount] = useState(0);
  const renderCountRef = useRef(0);

  // Increments on every render (mount + updates)
  renderCountRef.current += 1;

  // Runs on mount only (empty dependency array)
  useEffect(() => {
    console.log('MOUNTED');

    return () => {
      console.log('UNMOUNTED');
    };
  }, []);

  // Runs on mount and every time count changes
  useEffect(() => {
    console.log(`Count updated to: ${count}`);

    return () => {
      console.log(`Cleaning up for count: ${count}`);
    };
  }, [count]);

  return (
    <div>
      <p>Count: {count}</p>
      <p>Render count: {renderCountRef.current}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}
```

### Lifecycle sequence for the above component:
```
// Mount:
//   1. Component function called (render phase)
//   2. DOM inserted (commit phase)
//   3. "MOUNTED" logged
//   4. "Count updated to: 0" logged

// Click "Increment":
//   1. Component function called with count=1 (render phase)
//   2. DOM patched (commit phase)
//   3. "Cleaning up for count: 0" logged (previous effect cleanup)
//   4. "Count updated to: 1" logged (new effect runs)

// Unmount:
//   1. "Cleaning up for count: 1" logged
//   2. "UNMOUNTED" logged
//   3. DOM removed
```

---

## Composition Patterns

### Children Composition (Preferred)

```tsx
interface CardProps {
  title: string;
  children: React.ReactNode;
}

function Card({ title, children }: CardProps): React.ReactElement {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="card-body">{children}</div>
    </div>
  );
}

function App(): React.ReactElement {
  return (
    <Card title="User Profile">
      <p>Name: Taylor</p>
      <p>Role: Developer</p>
    </Card>
  );
}
```

### Render Props Pattern

```tsx
interface DataFetcherProps<T> {
  url: string;
  render: (data: T | null, loading: boolean) => React.ReactNode;
}

function DataFetcher<T>({ url, render }: DataFetcherProps<T>): React.ReactElement {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);

    fetch(url)
      .then(res => res.json())
      .then((result: T) => {
        if (!cancelled) {
          setData(result);
          setLoading(false);
        }
      });

    return () => { cancelled = true; };
  }, [url]);

  return <>{render(data, loading)}</>;
}

// Usage
function UserList(): React.ReactElement {
  return (
    <DataFetcher<User[]>
      url="/api/users"
      render={(users, loading) => {
        if (loading) return <p>Loading...</p>;
        if (!users) return <p>No data</p>;
        return (
          <ul>
            {users.map(u => <li key={u.id}>{u.name}</li>)}
          </ul>
        );
      }}
    />
  );
}
```

### Custom Hook Extraction (Best Practice)

```tsx
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

function useFetch<T>(url: string): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);

    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then((result: T) => {
        if (!cancelled) {
          setData(result);
          setLoading(false);
        }
      })
      .catch((err: Error) => {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      });

    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// Usage -- clean and composable
function UserList(): React.ReactElement {
  const { data: users, loading, error } = useFetch<User[]>('/api/users');

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;
  if (!users) return <p>No data</p>;

  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}
```

---

## Reconciliation and Keys

### Stable Keys for Lists

```tsx
interface TodoItem {
  id: string;
  text: string;
  completed: boolean;
}

function TodoList({ items }: { items: TodoItem[] }): React.ReactElement {
  return (
    <ul>
      {items.map(item => (
        // ALWAYS use a stable, unique identifier as key
        <li key={item.id}>
          <span style={{ textDecoration: item.completed ? 'line-through' : 'none' }}>
            {item.text}
          </span>
        </li>
      ))}
    </ul>
  );
}
```

### Key for Resetting Component State

```tsx
function App(): React.ReactElement {
  const [userId, setUserId] = useState('1');

  return (
    <div>
      <button onClick={() => setUserId('1')}>User 1</button>
      <button onClick={() => setUserId('2')}>User 2</button>
      {/* Changing key forces React to unmount and remount the component,
          resetting all internal state */}
      <UserProfile key={userId} userId={userId} />
    </div>
  );
}
```

---

## React 19: ref as Prop

```tsx
// React 18: requires forwardRef
import { forwardRef } from 'react';

const InputV18 = forwardRef<HTMLInputElement, { placeholder: string }>(
  ({ placeholder }, ref) => {
    return <input ref={ref} placeholder={placeholder} />;
  }
);

// React 19: ref is a regular prop
function InputV19({ placeholder, ref }: {
  placeholder: string;
  ref?: React.Ref<HTMLInputElement>;
}): React.ReactElement {
  return <input ref={ref} placeholder={placeholder} />;
}
```

## React 19: Context as Provider

```tsx
import { createContext, useContext } from 'react';

const ThemeContext = createContext<'light' | 'dark'>('light');

// React 18
function AppV18(): React.ReactElement {
  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}

// React 19: Context component IS the provider
function AppV19(): React.ReactElement {
  return (
    <ThemeContext value="dark">
      <Page />
    </ThemeContext>
  );
}
```
