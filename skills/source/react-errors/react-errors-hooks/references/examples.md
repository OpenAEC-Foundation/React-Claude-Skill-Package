# Hook Error Examples — Wrong and Correct Patterns

> Complete code examples for every hook error category.
> All examples use TypeScript/TSX and are verified against react.dev documentation.

---

## 1. Rules of Hooks Violations

### 1.1 Conditional Hook Call

```typescript
// WRONG: Hook called inside condition
function UserProfile({ userId }: { userId: string | null }) {
  if (userId) {
    const [user, setUser] = useState<User | null>(null); // ERROR: conditional
    useEffect(() => {
      fetchUser(userId).then(setUser);
    }, [userId]);
    return <div>{user?.name}</div>;
  }
  return <p>Select a user</p>;
}

// CORRECT: All hooks at top level, condition below
function UserProfile({ userId }: { userId: string | null }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    if (!userId) return;
    fetchUser(userId).then(setUser);
  }, [userId]);

  if (!userId) return <p>Select a user</p>;
  return <div>{user?.name}</div>;
}
```

### 1.2 Hook After Early Return

```typescript
// WRONG: Hook declared after early return
function Dashboard({ isAdmin }: { isAdmin: boolean }) {
  if (!isAdmin) return <p>Access denied</p>;

  const [stats, setStats] = useState<Stats | null>(null); // ERROR
  useEffect(() => {
    loadStats().then(setStats);
  }, []);

  return <StatsPanel stats={stats} />;
}

// CORRECT: Hooks first, then early returns
function Dashboard({ isAdmin }: { isAdmin: boolean }) {
  const [stats, setStats] = useState<Stats | null>(null);

  useEffect(() => {
    if (!isAdmin) return;
    loadStats().then(setStats);
  }, [isAdmin]);

  if (!isAdmin) return <p>Access denied</p>;
  return <StatsPanel stats={stats} />;
}
```

### 1.3 Hook Inside Loop

```typescript
// WRONG: Hook called inside a loop
function FieldList({ fields }: { fields: string[] }) {
  const values: string[] = [];
  for (const field of fields) {
    const [value, setValue] = useState(""); // ERROR: hook in loop
    values.push(value);
  }
  return <div>{values.join(", ")}</div>;
}

// CORRECT: Single state object for all fields
function FieldList({ fields }: { fields: string[] }) {
  const [values, setValues] = useState<Record<string, string>>(() =>
    Object.fromEntries(fields.map((f) => [f, ""]))
  );

  function handleChange(field: string, value: string) {
    setValues((prev) => ({ ...prev, [field]: value }));
  }

  return (
    <div>
      {fields.map((field) => (
        <input
          key={field}
          value={values[field] ?? ""}
          onChange={(e) => handleChange(field, e.target.value)}
        />
      ))}
    </div>
  );
}
```

### 1.4 Hook in Regular Function

```typescript
// WRONG: Hook in non-component, non-hook function
function createCounter() {
  const [count, setCount] = useState(0); // ERROR: not a React function
  return { count, increment: () => setCount((c) => c + 1) };
}

// CORRECT: Custom hook (prefix with "use")
function useCounter(initial: number = 0) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount((c) => c + 1), []);
  return { count, increment };
}
```

### 1.5 Hook in Event Handler

```typescript
// WRONG: Hook called inside event handler
function SearchForm() {
  function handleSubmit() {
    const [results, setResults] = useState<string[]>([]); // ERROR
    fetchResults().then(setResults);
  }
  return <button onClick={handleSubmit}>Search</button>;
}

// CORRECT: Hook at component top level
function SearchForm() {
  const [results, setResults] = useState<string[]>([]);

  function handleSubmit() {
    fetchResults().then(setResults);
  }

  return (
    <div>
      <button onClick={handleSubmit}>Search</button>
      <ResultList items={results} />
    </div>
  );
}
```

### 1.6 Hook in try/catch

```typescript
// WRONG: Hook inside try/catch block
function DataViewer({ id }: { id: string }) {
  try {
    const [data, setData] = useState<Data | null>(null); // ERROR
    return <div>{data?.name}</div>;
  } catch {
    return <p>Error</p>;
  }
}

// CORRECT: Hooks outside try/catch, error boundary for errors
function DataViewer({ id }: { id: string }) {
  const [data, setData] = useState<Data | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetchData(id)
      .then(setData)
      .catch((err) => setError(err.message));
  }, [id]);

  if (error) return <p>Error: {error}</p>;
  return <div>{data?.name}</div>;
}
```

---

## 2. Infinite Re-render Loops

### 2.1 Calling Handler During Render

```typescript
// WRONG: onClick calls function immediately (note the parentheses)
function Button() {
  const [count, setCount] = useState(0);
  return <button onClick={setCount(count + 1)}>Click</button>; // Runs during render!
}

// CORRECT: Pass function reference
function Button() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Click</button>;
}
```

### 2.2 useEffect Creates New Object in Dependencies

```typescript
// WRONG: filter object recreated every render
function TodoList({ todos }: { todos: Todo[] }) {
  const [filter, setFilter] = useState("all");
  const filterConfig = { type: filter, active: true }; // New object each render

  useEffect(() => {
    applyFilter(filterConfig);
  }, [filterConfig]); // Always "changes" = infinite loop

  return <div>...</div>;
}

// CORRECT: Destructure to primitives in dependency array
function TodoList({ todos }: { todos: Todo[] }) {
  const [filter, setFilter] = useState("all");

  useEffect(() => {
    const filterConfig = { type: filter, active: true };
    applyFilter(filterConfig);
  }, [filter]); // Primitive = stable comparison

  return <div>...</div>;
}
```

### 2.3 useSyncExternalStore getSnapshot Returns New Object

```typescript
// WRONG: New object every call = infinite loop
const size = useSyncExternalStore(subscribe, () => ({
  width: window.innerWidth,
  height: window.innerHeight,
}));

// CORRECT: Cache the snapshot, return same reference when unchanged
let cachedSize = { width: 0, height: 0 };
function getSnapshot() {
  const next = { width: window.innerWidth, height: window.innerHeight };
  if (next.width === cachedSize.width && next.height === cachedSize.height) {
    return cachedSize;
  }
  cachedSize = next;
  return cachedSize;
}
const size = useSyncExternalStore(subscribe, getSnapshot);
```

---

## 3. Stale Closures

### 3.1 Debounced Callback with Stale State

```typescript
// WRONG: Debounced function captures stale searchTerm
function Search() {
  const [searchTerm, setSearchTerm] = useState("");
  const [results, setResults] = useState<string[]>([]);

  const debouncedSearch = useMemo(
    () =>
      debounce(() => {
        fetchResults(searchTerm).then(setResults); // Stale searchTerm
      }, 300),
    [] // searchTerm not in deps
  );

  return <input onChange={(e) => { setSearchTerm(e.target.value); debouncedSearch(); }} />;
}

// CORRECT: Pass value as argument, not via closure
function Search() {
  const [searchTerm, setSearchTerm] = useState("");
  const [results, setResults] = useState<string[]>([]);

  const debouncedSearch = useMemo(
    () =>
      debounce((term: string) => {
        fetchResults(term).then(setResults);
      }, 300),
    []
  );

  return (
    <input
      onChange={(e) => {
        setSearchTerm(e.target.value);
        debouncedSearch(e.target.value); // Pass current value directly
      }}
    />
  );
}
```

### 3.2 Ref to Hold Latest Value for Callbacks

```typescript
// Pattern: useRef to always access latest state in long-lived callbacks
function useLatest<T>(value: T): React.MutableRefObject<T> {
  const ref = useRef(value);
  ref.current = value; // Updated synchronously every render
  return ref;
}

function ChatRoom({ roomId, onMessage }: { roomId: string; onMessage: (msg: string) => void }) {
  const onMessageRef = useLatest(onMessage);

  useEffect(() => {
    const conn = createConnection(roomId);
    conn.on("message", (msg) => {
      onMessageRef.current(msg); // Always calls latest onMessage
    });
    conn.connect();
    return () => conn.disconnect();
  }, [roomId]); // onMessage NOT in deps, accessed via ref
}
```

---

## 4. Cleanup Mistakes

### 4.1 WebSocket Without Cleanup

```typescript
// WRONG: Connection stays open after unmount or roomId change
function Chat({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<string[]>([]);

  useEffect(() => {
    const ws = new WebSocket(`wss://chat.example.com/${roomId}`);
    ws.onmessage = (event) => {
      setMessages((prev) => [...prev, event.data]);
    };
    // No cleanup!
  }, [roomId]);
}

// CORRECT: Close connection on cleanup
function Chat({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<string[]>([]);

  useEffect(() => {
    const ws = new WebSocket(`wss://chat.example.com/${roomId}`);
    ws.onmessage = (event) => {
      setMessages((prev) => [...prev, event.data]);
    };
    return () => ws.close();
  }, [roomId]);
}
```

### 4.2 AbortController for Fetch Cleanup

```typescript
// CORRECT: Cancel in-flight requests on cleanup
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((res) => res.json())
      .then((data) => setUser(data))
      .catch((err) => {
        if (err.name !== "AbortError") {
          console.error("Fetch failed:", err);
        }
      });

    return () => controller.abort();
  }, [userId]);

  return user ? <div>{user.name}</div> : <p>Loading...</p>;
}
```

### 4.3 IntersectionObserver Without Cleanup

```typescript
// WRONG: Observer never disconnected
function LazyImage({ src }: { src: string }) {
  const imgRef = useRef<HTMLImageElement>(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) setIsVisible(true);
    });
    if (imgRef.current) observer.observe(imgRef.current);
    // No cleanup!
  }, []);
}

// CORRECT: Disconnect observer on cleanup
function LazyImage({ src }: { src: string }) {
  const imgRef = useRef<HTMLImageElement>(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const node = imgRef.current;
    if (!node) return;

    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) setIsVisible(true);
    });
    observer.observe(node);
    return () => observer.disconnect();
  }, []);
}
```

---

## 5. eslint-plugin-react-hooks Fixes

### 5.1 Function Dependency Creates Infinite Loop

```typescript
// WRONG: createOptions recreated every render, effect runs infinitely
function ChatRoom({ roomId }: { roomId: string }) {
  function createOptions() {
    return { serverUrl: "https://localhost:1234", roomId };
  }

  useEffect(() => {
    const conn = createConnection(createOptions());
    conn.connect();
    return () => conn.disconnect();
  }, [createOptions]); // New function reference every render
}

// CORRECT: Move function inside effect
function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const options = { serverUrl: "https://localhost:1234", roomId };
    const conn = createConnection(options);
    conn.connect();
    return () => conn.disconnect();
  }, [roomId]); // Only primitive dependency
}
```

### 5.2 Props Object in Dependencies

```typescript
// WRONG: Entire props object as dependency
function DataGrid({ config }: { config: GridConfig }) {
  useEffect(() => {
    initGrid(config);
  }, [config]); // Parent re-render = new config object = effect re-runs

  return <div id="grid" />;
}

// CORRECT: Destructure to specific properties
function DataGrid({ config }: { config: GridConfig }) {
  const { columns, pageSize, sortBy } = config;

  useEffect(() => {
    initGrid({ columns, pageSize, sortBy });
  }, [columns, pageSize, sortBy]); // Primitives are stable
}
```

---

## 6. "You Might Not Need an Effect" Fixes

### 6.1 Derived State — Calculate During Render

```typescript
// WRONG: Extra render cycle for computed value
function FilteredList({ items, query }: { items: Item[]; query: string }) {
  const [filtered, setFiltered] = useState<Item[]>([]);

  useEffect(() => {
    setFiltered(items.filter((item) => item.name.includes(query)));
  }, [items, query]);

  return <ul>{filtered.map((item) => <li key={item.id}>{item.name}</li>)}</ul>;
}

// CORRECT: Compute during render (or useMemo if expensive)
function FilteredList({ items, query }: { items: Item[]; query: string }) {
  const filtered = useMemo(
    () => items.filter((item) => item.name.includes(query)),
    [items, query]
  );

  return <ul>{filtered.map((item) => <li key={item.id}>{item.name}</li>)}</ul>;
}
```

### 6.2 Reset State — Use Key Prop

```typescript
// WRONG: Effect to reset comment on profile change
function ProfilePage({ userId }: { userId: string }) {
  const [comment, setComment] = useState("");

  useEffect(() => {
    setComment(""); // Flickers: renders old comment first
  }, [userId]);

  return <textarea value={comment} onChange={(e) => setComment(e.target.value)} />;
}

// CORRECT: Key forces fresh component instance
function ProfilePage({ userId }: { userId: string }) {
  return <CommentBox key={userId} />;
}

function CommentBox() {
  const [comment, setComment] = useState(""); // Fresh state per key
  return <textarea value={comment} onChange={(e) => setComment(e.target.value)} />;
}
```

### 6.3 Notify Parent — Do It in the Event Handler

```typescript
// WRONG: Effect to notify parent of state change
function Toggle({ onChange }: { onChange: (isOn: boolean) => void }) {
  const [isOn, setIsOn] = useState(false);

  useEffect(() => {
    onChange(isOn); // Fires on mount too, not just user action
  }, [isOn, onChange]);

  return <button onClick={() => setIsOn(!isOn)}>{isOn ? "ON" : "OFF"}</button>;
}

// CORRECT: Notify in the same event handler
function Toggle({ onChange }: { onChange: (isOn: boolean) => void }) {
  const [isOn, setIsOn] = useState(false);

  function handleClick() {
    const nextIsOn = !isOn;
    setIsOn(nextIsOn);
    onChange(nextIsOn); // Batched with setState in same event
  }

  return <button onClick={handleClick}>{isOn ? "ON" : "OFF"}</button>;
}
```

---

## 7. React 19 Specific Hook Errors

### 7.1 use() Outside Component or in try/catch

```typescript
// WRONG: "Suspense Exception: This is not a real error!"
function DataView({ dataPromise }: { dataPromise: Promise<Data> }) {
  try {
    const data = use(dataPromise); // ERROR: use() inside try/catch
    return <div>{data.name}</div>;
  } catch {
    return <p>Error</p>;
  }
}

// CORRECT: use() at component top level, Error Boundary for errors
function DataView({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise); // Suspends until resolved
  return <div>{data.name}</div>;
}

// Wrap in Suspense + ErrorBoundary
<ErrorBoundary fallback={<p>Error</p>}>
  <Suspense fallback={<p>Loading...</p>}>
    <DataView dataPromise={dataPromise} />
  </Suspense>
</ErrorBoundary>
```

### 7.2 useFormStatus in Wrong Component

```typescript
// WRONG: useFormStatus in the same component as <form>
function ContactForm() {
  const { pending } = useFormStatus(); // Always returns pending: false
  return (
    <form action={submitContact}>
      <input name="email" />
      <button disabled={pending}>Send</button>
    </form>
  );
}

// CORRECT: Extract button into child component
function SubmitButton() {
  const { pending } = useFormStatus(); // Reads parent <form> status
  return <button disabled={pending}>{pending ? "Sending..." : "Send"}</button>;
}

function ContactForm() {
  return (
    <form action={submitContact}>
      <input name="email" />
      <SubmitButton />
    </form>
  );
}
```
