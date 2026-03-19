# Hook Usage Examples

> All examples use TypeScript with strict typing. Verified against https://react.dev/reference/react/

---

## useState Examples

### Basic State with Types

```typescript
import { useState } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
}

function UserProfile() {
  const [name, setName] = useState<string>('');
  const [age, setAge] = useState<number>(0);
  const [isActive, setIsActive] = useState<boolean>(false);
  const [user, setUser] = useState<User | null>(null);
  const [tags, setTags] = useState<string[]>([]);

  return (
    <div>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <button onClick={() => setAge(prev => prev + 1)}>Age: {age}</button>
      <button onClick={() => setIsActive(prev => !prev)}>
        {isActive ? 'Active' : 'Inactive'}
      </button>
    </div>
  );
}
```

### Lazy Initialization

```typescript
function ExpensiveInitialState() {
  // CORRECT — function reference, called once on mount
  const [data, setData] = useState<Map<string, number>>(() => {
    const map = new Map<string, number>();
    for (let i = 0; i < 10000; i++) {
      map.set(`key-${i}`, i);
    }
    return map;
  });

  return <div>Items: {data.size}</div>;
}
```

### Object State — Immutable Updates

```typescript
interface FormState {
  firstName: string;
  lastName: string;
  email: string;
}

function RegistrationForm() {
  const [form, setForm] = useState<FormState>({
    firstName: '',
    lastName: '',
    email: '',
  });

  // CORRECT — spread operator creates new object
  const updateField = (field: keyof FormState, value: string) => {
    setForm(prev => ({ ...prev, [field]: value }));
  };

  return (
    <form>
      <input
        value={form.firstName}
        onChange={(e) => updateField('firstName', e.target.value)}
      />
      <input
        value={form.lastName}
        onChange={(e) => updateField('lastName', e.target.value)}
      />
      <input
        value={form.email}
        onChange={(e) => updateField('email', e.target.value)}
      />
    </form>
  );
}
```

### Array State — Immutable Updates

```typescript
interface Todo {
  id: number;
  text: string;
  done: boolean;
}

function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [nextId, setNextId] = useState<number>(1);

  const addTodo = (text: string) => {
    setTodos(prev => [...prev, { id: nextId, text, done: false }]);
    setNextId(prev => prev + 1);
  };

  const toggleTodo = (id: number) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      )
    );
  };

  const removeTodo = (id: number) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  };

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <span
            style={{ textDecoration: todo.done ? 'line-through' : 'none' }}
            onClick={() => toggleTodo(todo.id)}
          >
            {todo.text}
          </span>
          <button onClick={() => removeTodo(todo.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

### Storing a Function in State

```typescript
// CORRECT — wrap in arrow function so React does not call it as initializer
const [formatter, setFormatter] = useState<(v: number) => string>(
  () => (v: number) => v.toFixed(2)
);

// Update: also wrap in arrow function
setFormatter(() => (v: number) => v.toLocaleString());
```

---

## useEffect Examples

### Subscription with Cleanup

```typescript
import { useState, useEffect } from 'react';

function WindowSize() {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });

  useEffect(() => {
    const handleResize = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []); // Empty deps — subscribe once on mount

  return <div>{size.width} x {size.height}</div>;
}
```

### Data Fetching with Race Condition Guard

```typescript
interface Article {
  id: string;
  title: string;
  body: string;
}

function ArticleView({ articleId }: { articleId: string }) {
  const [article, setArticle] = useState<Article | null>(null);
  const [loading, setLoading] = useState<boolean>(true);

  useEffect(() => {
    let ignore = false;
    setLoading(true);

    async function loadArticle() {
      const response = await fetch(`/api/articles/${articleId}`);
      const data: Article = await response.json();
      if (!ignore) {
        setArticle(data);
        setLoading(false);
      }
    }

    loadArticle();
    return () => { ignore = true; };
  }, [articleId]);

  if (loading) return <div>Loading...</div>;
  if (!article) return <div>Not found</div>;
  return <article><h1>{article.title}</h1><p>{article.body}</p></article>;
}
```

### Timer with Cleanup

```typescript
function Stopwatch() {
  const [seconds, setSeconds] = useState<number>(0);
  const [isRunning, setIsRunning] = useState<boolean>(false);

  useEffect(() => {
    if (!isRunning) return;

    const intervalId = setInterval(() => {
      setSeconds(prev => prev + 1);
    }, 1000);

    return () => clearInterval(intervalId);
  }, [isRunning]);

  return (
    <div>
      <p>{seconds}s</p>
      <button onClick={() => setIsRunning(prev => !prev)}>
        {isRunning ? 'Stop' : 'Start'}
      </button>
      <button onClick={() => { setIsRunning(false); setSeconds(0); }}>Reset</button>
    </div>
  );
}
```

### Document Title Sync

```typescript
function PageTitle({ title }: { title: string }) {
  useEffect(() => {
    const previousTitle = document.title;
    document.title = title;
    return () => { document.title = previousTitle; };
  }, [title]);

  return null;
}
```

---

## useContext Examples

### Creating and Consuming Context

```typescript
import { createContext, useContext, useState, useMemo, type ReactNode } from 'react';

interface AuthContextType {
  user: { name: string; role: string } | null;
  login: (name: string, role: string) => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

// Custom hook with null check — ALWAYS use this pattern
function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (context === null) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}

function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<{ name: string; role: string } | null>(null);

  // Memoize to prevent unnecessary consumer re-renders
  const value = useMemo<AuthContextType>(() => ({
    user,
    login: (name: string, role: string) => setUser({ name, role }),
    logout: () => setUser(null),
  }), [user]);

  // React 19: <AuthContext value={value}>
  // React 18: <AuthContext.Provider value={value}>
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

function UserGreeting() {
  const { user, logout } = useAuth();
  if (!user) return <p>Please log in</p>;
  return (
    <div>
      <p>Hello, {user.name} ({user.role})</p>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

---

## useRef Examples

### DOM Reference

```typescript
import { useRef } from 'react';

function AutoFocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleClick = () => {
    inputRef.current?.focus();
    inputRef.current?.select();
  };

  return (
    <div>
      <input ref={inputRef} type="text" placeholder="Click button to focus" />
      <button onClick={handleClick}>Focus Input</button>
    </div>
  );
}
```

### Mutable Value (Previous State)

```typescript
import { useState, useRef, useEffect } from 'react';

function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);

  useEffect(() => {
    ref.current = value;
  });

  return ref.current;
}

function Counter() {
  const [count, setCount] = useState<number>(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}, Previous: {prevCount ?? 'N/A'}</p>
      <button onClick={() => setCount(prev => prev + 1)}>Increment</button>
    </div>
  );
}
```

### Storing Interval ID

```typescript
function Ticker() {
  const [count, setCount] = useState<number>(0);
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);

  const start = () => {
    if (intervalRef.current !== null) return; // Prevent duplicates
    intervalRef.current = setInterval(() => {
      setCount(prev => prev + 1);
    }, 1000);
  };

  const stop = () => {
    if (intervalRef.current !== null) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  };

  useEffect(() => {
    return () => stop(); // Cleanup on unmount
  }, []);

  return (
    <div>
      <p>{count}</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

---

## useMemo Examples

### Expensive Computation

```typescript
import { useState, useMemo } from 'react';

interface Product {
  id: number;
  name: string;
  category: string;
  price: number;
}

function ProductList({ products }: { products: Product[] }) {
  const [filter, setFilter] = useState<string>('');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');

  const filteredAndSorted = useMemo<Product[]>(() => {
    const filtered = products.filter(p =>
      p.name.toLowerCase().includes(filter.toLowerCase())
    );
    return filtered.sort((a, b) => {
      if (sortBy === 'name') return a.name.localeCompare(b.name);
      return a.price - b.price;
    });
  }, [products, filter, sortBy]);

  return (
    <div>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <select value={sortBy} onChange={(e) => setSortBy(e.target.value as 'name' | 'price')}>
        <option value="name">Name</option>
        <option value="price">Price</option>
      </select>
      <ul>
        {filteredAndSorted.map(p => (
          <li key={p.id}>{p.name} — ${p.price}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Stable Object for Context Provider

```typescript
function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const value = useMemo(() => ({
    theme,
    toggle: () => setTheme(prev => prev === 'light' ? 'dark' : 'light'),
  }), [theme]);

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}
```

---

## useCallback Examples

### Stable Callback for Memo-wrapped Child

```typescript
import { useState, useCallback, memo } from 'react';

interface ButtonProps {
  onClick: () => void;
  label: string;
}

const ExpensiveButton = memo(function ExpensiveButton({ onClick, label }: ButtonProps) {
  console.log(`Rendering: ${label}`);
  return <button onClick={onClick}>{label}</button>;
});

function Parent() {
  const [count, setCount] = useState<number>(0);
  const [name, setName] = useState<string>('');

  // Stable reference — ExpensiveButton skips re-render when name changes
  const increment = useCallback(() => {
    setCount(prev => prev + 1);
  }, []);

  return (
    <div>
      <p>Count: {count}</p>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <ExpensiveButton onClick={increment} label="Increment" />
    </div>
  );
}
```

### Callback as Effect Dependency

```typescript
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState<string[]>([]);

  const fetchResults = useCallback(async () => {
    const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
    const data: string[] = await response.json();
    return data;
  }, [query]);

  useEffect(() => {
    let ignore = false;
    fetchResults().then(data => {
      if (!ignore) setResults(data);
    });
    return () => { ignore = true; };
  }, [fetchResults]);

  return <ul>{results.map((r, i) => <li key={i}>{r}</li>)}</ul>;
}
```

---

## useReducer Examples

### Typed Reducer with Discriminated Union Actions

```typescript
import { useReducer } from 'react';

interface TodoState {
  todos: Array<{ id: number; text: string; done: boolean }>;
  nextId: number;
}

type TodoAction =
  | { type: 'added'; text: string }
  | { type: 'toggled'; id: number }
  | { type: 'deleted'; id: number };

function todoReducer(state: TodoState, action: TodoAction): TodoState {
  switch (action.type) {
    case 'added':
      return {
        ...state,
        todos: [...state.todos, { id: state.nextId, text: action.text, done: false }],
        nextId: state.nextId + 1,
      };
    case 'toggled':
      return {
        ...state,
        todos: state.todos.map(t =>
          t.id === action.id ? { ...t, done: !t.done } : t
        ),
      };
    case 'deleted':
      return {
        ...state,
        todos: state.todos.filter(t => t.id !== action.id),
      };
  }
}

const initialState: TodoState = { todos: [], nextId: 1 };

function TodoApp() {
  const [state, dispatch] = useReducer(todoReducer, initialState);

  return (
    <div>
      <button onClick={() => dispatch({ type: 'added', text: 'New task' })}>
        Add Todo
      </button>
      <ul>
        {state.todos.map(todo => (
          <li key={todo.id}>
            <span
              onClick={() => dispatch({ type: 'toggled', id: todo.id })}
              style={{ textDecoration: todo.done ? 'line-through' : 'none' }}
            >
              {todo.text}
            </span>
            <button onClick={() => dispatch({ type: 'deleted', id: todo.id })}>
              Delete
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Lazy Initialization

```typescript
interface AppState {
  settings: Record<string, string>;
}

function createInitialState(defaultSettings: Record<string, string>): AppState {
  // Expensive: reads from localStorage, merges with defaults
  const stored = JSON.parse(localStorage.getItem('settings') ?? '{}') as Record<string, string>;
  return { settings: { ...defaultSettings, ...stored } };
}

type AppAction = { type: 'update'; key: string; value: string };

function settingsReducer(state: AppState, action: AppAction): AppState {
  switch (action.type) {
    case 'update':
      return { settings: { ...state.settings, [action.key]: action.value } };
  }
}

function Settings() {
  // Third argument: lazy init function, called once with second argument
  const [state, dispatch] = useReducer(
    settingsReducer,
    { theme: 'light', lang: 'en' },
    createInitialState
  );

  return <div>{JSON.stringify(state.settings)}</div>;
}
```

### useReducer + useContext for State Management

```typescript
import { createContext, useContext, useReducer, type ReactNode, type Dispatch } from 'react';

// Define state and actions
interface CountState { count: number }
type CountAction = { type: 'increment' } | { type: 'decrement' } | { type: 'reset' };

function countReducer(state: CountState, action: CountAction): CountState {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    case 'reset':     return { count: 0 };
  }
}

// Two contexts: one for state, one for dispatch (dispatch is stable, avoids unnecessary re-renders)
const CountStateContext = createContext<CountState | null>(null);
const CountDispatchContext = createContext<Dispatch<CountAction> | null>(null);

function CountProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(countReducer, { count: 0 });
  return (
    <CountStateContext.Provider value={state}>
      <CountDispatchContext.Provider value={dispatch}>
        {children}
      </CountDispatchContext.Provider>
    </CountStateContext.Provider>
  );
}

function useCountState(): CountState {
  const ctx = useContext(CountStateContext);
  if (ctx === null) throw new Error('useCountState must be within CountProvider');
  return ctx;
}

function useCountDispatch(): Dispatch<CountAction> {
  const ctx = useContext(CountDispatchContext);
  if (ctx === null) throw new Error('useCountDispatch must be within CountProvider');
  return ctx;
}
```
