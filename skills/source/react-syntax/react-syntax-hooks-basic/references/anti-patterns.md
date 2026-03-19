# Hook Anti-Patterns

> What NOT to do when using React hooks, with explanations and correct alternatives.
> Source: https://react.dev/reference/react/

---

## Rules of Hooks Violations

### Conditional Hook Calls

```typescript
// WRONG — hooks called conditionally
function UserProfile({ userId }: { userId: string | null }) {
  if (!userId) {
    return <p>No user selected</p>;
  }
  // VIOLATION: hook call order changes when userId is null
  const [user, setUser] = useState<User | null>(null);
  useEffect(() => { fetchUser(userId).then(setUser); }, [userId]);
  return <div>{user?.name}</div>;
}

// CORRECT — hooks ALWAYS at top level
function UserProfile({ userId }: { userId: string | null }) {
  const [user, setUser] = useState<User | null>(null);
  useEffect(() => {
    if (!userId) return;
    let ignore = false;
    fetchUser(userId).then(u => { if (!ignore) setUser(u); });
    return () => { ignore = true; };
  }, [userId]);

  if (!userId) return <p>No user selected</p>;
  return <div>{user?.name}</div>;
}
```

### Hooks Inside Loops

```typescript
// WRONG — hook called in loop
function MultiCounter({ count }: { count: number }) {
  const values: number[] = [];
  for (let i = 0; i < count; i++) {
    const [val] = useState(0); // VIOLATION
    values.push(val);
  }
  return <div>{values.join(', ')}</div>;
}

// CORRECT — extract to child component
function Counter() {
  const [val, setVal] = useState<number>(0);
  return <button onClick={() => setVal(v => v + 1)}>{val}</button>;
}

function MultiCounter({ count }: { count: number }) {
  return (
    <div>
      {Array.from({ length: count }, (_, i) => <Counter key={i} />)}
    </div>
  );
}
```

### Hooks in Event Handlers

```typescript
// WRONG — hook called inside event handler
function SearchBox() {
  const handleSearch = () => {
    const [query, setQuery] = useState(''); // VIOLATION
  };
  return <button onClick={handleSearch}>Search</button>;
}

// CORRECT — hook at top level
function SearchBox() {
  const [query, setQuery] = useState<string>('');
  const handleSearch = () => {
    // use query here
    console.log('Searching:', query);
  };
  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <button onClick={handleSearch}>Search</button>
    </div>
  );
}
```

---

## useState Anti-Patterns

### Mutating State Directly

```typescript
// WRONG — mutating existing object
const [user, setUser] = useState<{ name: string; age: number }>({ name: 'Alice', age: 30 });

const updateAge = () => {
  user.age = 31;       // Mutation — React does not detect this
  setUser(user);       // Same reference — React skips re-render
};

// CORRECT — create new object
const updateAge = () => {
  setUser(prev => ({ ...prev, age: 31 }));
};
```

### Mutating Array State

```typescript
// WRONG — push mutates the existing array
const [items, setItems] = useState<string[]>([]);

const addItem = (item: string) => {
  items.push(item);    // Mutation
  setItems(items);     // Same reference — no re-render
};

// CORRECT — spread into new array
const addItem = (item: string) => {
  setItems(prev => [...prev, item]);
};
```

### Calling Initializer Function

```typescript
// WRONG — createRows() called on EVERY render
const [rows, setRows] = useState(createRows());

// CORRECT — pass function reference, called once on mount
const [rows, setRows] = useState(createRows);
// or: useState(() => createRows())
```

### Reading State Right After Setting

```typescript
// WRONG — expects immediate update
function Counter() {
  const [count, setCount] = useState<number>(0);

  const handleClick = () => {
    setCount(count + 1);
    console.log(count); // Still 0! State updates are batched.
  };
}

// CORRECT — use updater for sequential updates, or accept the batched behavior
const handleClick = () => {
  setCount(prev => {
    const next = prev + 1;
    console.log('Next:', next);
    return next;
  });
};
```

---

## useEffect Anti-Patterns

### Async Callback

```typescript
// WRONG — async function returns Promise, not cleanup function
useEffect(async () => {
  const data = await fetchData();
  setData(data);
}, []);

// CORRECT — define async function inside effect
useEffect(() => {
  let ignore = false;
  async function load() {
    const data = await fetchData();
    if (!ignore) setData(data);
  }
  load();
  return () => { ignore = true; };
}, []);
```

### Missing Cleanup

```typescript
// WRONG — event listener leaks on unmount and re-render
useEffect(() => {
  window.addEventListener('scroll', handleScroll);
}, []);

// CORRECT — cleanup removes listener
useEffect(() => {
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

### Object in Dependency Array

```typescript
// WRONG — new object reference every render, effect runs every render
function Chart({ data }: { data: number[] }) {
  const options = { animate: true, color: 'blue' };

  useEffect(() => {
    drawChart(data, options);
  }, [data, options]); // options is a new object every render!
}

// CORRECT — move object inside effect or memoize
function Chart({ data }: { data: number[] }) {
  useEffect(() => {
    const options = { animate: true, color: 'blue' };
    drawChart(data, options);
  }, [data]);
}

// OR: extract primitives as dependencies
function Chart({ data, animate, color }: { data: number[]; animate: boolean; color: string }) {
  useEffect(() => {
    drawChart(data, { animate, color });
  }, [data, animate, color]);
}
```

### Missing Race Condition Guard

```typescript
// WRONG — stale response overwrites fresh data
useEffect(() => {
  fetchUser(userId).then(setUser);
}, [userId]);
// If userId changes rapidly: response for old userId may arrive AFTER new one

// CORRECT — ignore stale responses
useEffect(() => {
  let ignore = false;
  fetchUser(userId).then(user => {
    if (!ignore) setUser(user);
  });
  return () => { ignore = true; };
}, [userId]);
```

### Omitting Dependencies to Prevent Infinite Loops

```typescript
// WRONG — suppressing the linter instead of fixing the issue
useEffect(() => {
  fetchData(options); // options changes every render
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, []); // Missing dependency: options

// CORRECT — stabilize the dependency
const stableOptions = useMemo(() => options, [options.key1, options.key2]);
useEffect(() => {
  fetchData(stableOptions);
}, [stableOptions]);
```

---

## useContext Anti-Patterns

### Missing Provider Check

```typescript
// WRONG — returns undefined silently if no provider
function useTheme() {
  return useContext(ThemeContext); // Could be undefined
}

// CORRECT — explicit null check with helpful error
function useTheme(): Theme {
  const ctx = useContext(ThemeContext);
  if (ctx === null) {
    throw new Error('useTheme must be used within a ThemeProvider');
  }
  return ctx;
}
```

### Creating New Object Every Render in Provider

```typescript
// WRONG — all consumers re-render on every parent render
function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
  // { theme, setTheme } is a NEW object every render
}

// CORRECT — memoize the context value
function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}
```

---

## useRef Anti-Patterns

### Reading/Writing ref.current During Render

```typescript
// WRONG — impure render, breaks concurrent features
function Counter() {
  const countRef = useRef<number>(0);
  countRef.current += 1; // Writing during render — VIOLATION
  return <p>Renders: {countRef.current}</p>; // Reading during render — VIOLATION
}

// CORRECT — read/write in effects or event handlers
function Counter() {
  const countRef = useRef<number>(0);

  useEffect(() => {
    countRef.current += 1;
  });

  return <p>Check console for render count</p>;
}
```

### Using ref When State is Needed

```typescript
// WRONG — changing ref does not trigger re-render
function NameDisplay() {
  const nameRef = useRef<string>('');

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    nameRef.current = e.target.value; // UI does NOT update
  };

  return (
    <div>
      <input onChange={handleChange} />
      <p>Name: {nameRef.current}</p> {/* Always shows initial value */}
    </div>
  );
}

// CORRECT — use state for values that affect rendering
function NameDisplay() {
  const [name, setName] = useState<string>('');

  return (
    <div>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <p>Name: {name}</p>
    </div>
  );
}
```

---

## useMemo / useCallback Anti-Patterns

### Premature Memoization

```typescript
// WRONG — memoizing a trivial calculation adds overhead
const doubled = useMemo(() => count * 2, [count]);

// CORRECT — just compute it
const doubled = count * 2;
```

### Missing useMemo When Passing Objects to memo() Children

```typescript
// WRONG — memo() on child is useless because style is new object every render
const ExpensiveChild = memo(function ExpensiveChild({ style }: { style: React.CSSProperties }) {
  return <div style={style}>Expensive</div>;
});

function Parent() {
  return <ExpensiveChild style={{ color: 'red', fontSize: 16 }} />;
  // style is a NEW object every render — memo() never skips
}

// CORRECT — memoize the object
function Parent() {
  const style = useMemo<React.CSSProperties>(() => ({ color: 'red', fontSize: 16 }), []);
  return <ExpensiveChild style={style} />;
}
```

### useCallback Without memo() Child

```typescript
// WRONG — useCallback alone does nothing useful
function Parent() {
  const [count, setCount] = useState<number>(0);

  // This memoization has no effect if Child is NOT wrapped in memo()
  const handleClick = useCallback(() => {
    setCount(prev => prev + 1);
  }, []);

  return <Child onClick={handleClick} />;
}

// CORRECT — useCallback + memo() together
const Child = memo(function Child({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Click</button>;
});
```

---

## useReducer Anti-Patterns

### Mutating State in Reducer

```typescript
// WRONG — mutates state object
function reducer(state: { items: string[] }, action: { type: 'add'; item: string }) {
  state.items.push(action.item); // Mutation!
  return state; // Same reference — React may skip re-render
}

// CORRECT — return new state
function reducer(state: { items: string[] }, action: { type: 'add'; item: string }) {
  return { items: [...state.items, action.item] };
}
```

### Calling Lazy Initializer Instead of Passing It

```typescript
// WRONG — createInitialState() runs every render
const [state, dispatch] = useReducer(reducer, createInitialState());

// CORRECT — pass as third argument, called once with second argument
const [state, dispatch] = useReducer(reducer, defaultArg, createInitialState);
```

---

## Dependency Array Anti-Patterns Summary

| Anti-pattern | Why it fails | Fix |
|-------------|-------------|-----|
| Object literal in deps `[{ id }]` | New reference every render | Use primitive: `[id]` |
| Array literal in deps `[[a, b]]` | New reference every render | Use elements: `[a, b]` |
| Function in deps without `useCallback` | New reference every render | Wrap in `useCallback` |
| Omitting deps to "fix" loops | Stale closures, bugs | Stabilize the values instead |
| `// eslint-disable` on deps rule | Hides real bugs | Fix the dependency, do not suppress |
| Empty `[]` when deps exist | Effect uses stale values | Include all reactive values |
