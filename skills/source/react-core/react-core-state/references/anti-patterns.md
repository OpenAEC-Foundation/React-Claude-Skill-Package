# react-core-state -- Anti-Patterns

## AP-1: Direct State Mutation

**Problem**: Mutating state objects or arrays in place does NOT trigger a re-render. React uses `Object.is` comparison and sees the same reference.

```typescript
// WRONG -- mutates existing array, no re-render
function addItem(item: Item) {
  items.push(item);
  setItems(items); // Same reference -- React skips update
}

// WRONG -- mutates existing object
function updateUser() {
  user.name = "New Name";
  setUser(user); // Same reference -- React skips update
}
```

**Fix**: ALWAYS create new references.

```typescript
// CORRECT -- new array
function addItem(item: Item) {
  setItems(prev => [...prev, item]);
}

// CORRECT -- new object
function updateUser() {
  setUser(prev => ({ ...prev, name: "New Name" }));
}

// For deeply nested state, use Immer:
import { produce } from "immer";
setUser(produce(draft => {
  draft.address.city = "Amsterdam";
}));
```

---

## AP-2: Storing Derived State

**Problem**: Storing values that can be computed from other state creates synchronization bugs. The derived value can become stale.

```typescript
// WRONG -- derived state stored separately
const [items, setItems] = useState<Item[]>([]);
const [filteredItems, setFilteredItems] = useState<Item[]>([]);
const [totalPrice, setTotalPrice] = useState<number>(0);

useEffect(() => {
  setFilteredItems(items.filter(i => i.active));
}, [items]);

useEffect(() => {
  setTotalPrice(filteredItems.reduce((sum, i) => sum + i.price, 0));
}, [filteredItems]);
// Bug: two extra renders, possible intermediate inconsistent state
```

**Fix**: Compute during render. Use `useMemo` if the computation is expensive.

```typescript
// CORRECT -- derive during render
const [items, setItems] = useState<Item[]>([]);

const filteredItems = useMemo(
  () => items.filter(i => i.active),
  [items]
);

const totalPrice = useMemo(
  () => filteredItems.reduce((sum, i) => sum + i.price, 0),
  [filteredItems]
);
// No extra renders, always consistent
```

---

## AP-3: Props Mirrored in State

**Problem**: Copying props into state causes the state to become stale when props change.

```typescript
// WRONG -- props copied into state
function UserCard({ user }: { user: User }) {
  const [name, setName] = useState(user.name);
  // name is now stale if parent updates user.name
  return <p>{name}</p>;
}
```

**Fix A**: Use props directly (no state needed for display).

```typescript
// CORRECT -- use prop directly
function UserCard({ user }: { user: User }) {
  return <p>{user.name}</p>;
}
```

**Fix B**: If the component needs to "reset" on prop change, use the `key` prop.

```typescript
// CORRECT -- parent resets via key
<EditableUserCard key={user.id} initialName={user.name} />

function EditableUserCard({ initialName }: { initialName: string }) {
  const [name, setName] = useState(initialName);
  // Fresh state when key changes -- no stale data
  return <input value={name} onChange={e => setName(e.target.value)} />;
}
```

---

## AP-4: Context for High-Frequency Updates

**Problem**: Every component that calls `useContext(SomeContext)` re-renders whenever the context value changes. For frequent updates (mouse position, scroll, animations), this causes performance degradation.

```typescript
// WRONG -- mouse position in context updates ALL consumers
const MouseContext = createContext<{ x: number; y: number }>({ x: 0, y: 0 });

function App() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    // Every mousemove re-renders EVERY consumer
    <MouseContext value={pos}>
      <div onMouseMove={e => setPos({ x: e.clientX, y: e.clientY })}>
        <HundredComponents /> {/* All re-render on every mouse move */}
      </div>
    </MouseContext>
  );
}
```

**Fix**: Use an external store with selectors.

```typescript
// CORRECT -- Zustand with selector, only Cursor re-renders
const useMouseStore = create<{ x: number; y: number; setPos: (x: number, y: number) => void }>(
  (set) => ({
    x: 0,
    y: 0,
    setPos: (x, y) => set({ x, y }),
  })
);

function Cursor() {
  const x = useMouseStore(s => s.x);
  const y = useMouseStore(s => s.y);
  return <div style={{ left: x, top: y }} className="cursor" />;
}

// Other components do NOT re-render
function Sidebar() {
  return <nav>...</nav>; // Unaffected by mouse movement
}
```

---

## AP-5: Missing Updater Function (Stale Closure)

**Problem**: Using the direct value from the closure instead of the updater form causes stale state when multiple updates are batched.

```typescript
// WRONG -- stale closure in rapid clicks or timeouts
function Counter() {
  const [count, setCount] = useState(0);

  const handleTripleIncrement = () => {
    setCount(count + 1); // Uses stale count from closure
    setCount(count + 1); // Same stale count
    setCount(count + 1); // Result: count + 1, not count + 3
  };
}
```

**Fix**: ALWAYS use updater form when depending on previous state.

```typescript
// CORRECT -- updater receives latest pending state
function Counter() {
  const [count, setCount] = useState(0);

  const handleTripleIncrement = () => {
    setCount(prev => prev + 1); // prev = 0 -> 1
    setCount(prev => prev + 1); // prev = 1 -> 2
    setCount(prev => prev + 1); // prev = 2 -> 3
  };
}
```

---

## AP-6: Overly Broad Context (God Context)

**Problem**: Putting all application state in a single context forces every consumer to re-render on any state change.

```typescript
// WRONG -- everything in one context
const AppContext = createContext<{
  user: User;
  theme: Theme;
  notifications: Notification[];
  cart: CartItem[];
  locale: string;
} | null>(null);

// Changing the theme re-renders the cart, notification list, etc.
```

**Fix**: Split into focused, domain-specific contexts.

```typescript
// CORRECT -- separate contexts by domain
const UserContext = createContext<UserContextType | null>(null);
const ThemeContext = createContext<ThemeContextType | null>(null);
const NotificationContext = createContext<NotificationContextType | null>(null);
const CartContext = createContext<CartContextType | null>(null);

// Theme changes only re-render ThemeContext consumers
```

---

## AP-7: New Context Value Object on Every Render

**Problem**: Creating a new object literal in the provider re-renders all consumers on every parent render, even if values have not changed.

```typescript
// WRONG -- new object on every render
function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  return (
    // { user, setUser } is a new object every render
    <AuthContext value={{ user, setUser }}>
      {children}
    </AuthContext>
  );
}
```

**Fix**: ALWAYS wrap context values in `useMemo`.

```typescript
// CORRECT -- stable reference when user hasn't changed
function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const value = useMemo(() => ({ user, setUser }), [user]);

  return <AuthContext value={value}>{children}</AuthContext>;
}
```

---

## AP-8: useEffect for State Synchronization

**Problem**: Using `useEffect` to sync state from props or other state causes extra renders and is harder to reason about.

```typescript
// WRONG -- useEffect to transform state
const [items, setItems] = useState<Item[]>([]);
const [sortedItems, setSortedItems] = useState<Item[]>([]);

useEffect(() => {
  setSortedItems([...items].sort((a, b) => a.name.localeCompare(b.name)));
}, [items]);
// Renders twice: once with stale sortedItems, once with sorted
```

**Fix**: Compute synchronously during render.

```typescript
// CORRECT -- computed during same render
const [items, setItems] = useState<Item[]>([]);
const sortedItems = useMemo(
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);
// Single render with correct data
```

---

## AP-9: State for Ref-Appropriate Values

**Problem**: Using `useState` for values that do not affect rendering (timers, DOM references, previous values) causes unnecessary re-renders.

```typescript
// WRONG -- state triggers re-render on every interval tick
const [intervalId, setIntervalId] = useState<number | null>(null);
```

**Fix**: Use `useRef` for values that do not affect the visual output.

```typescript
// CORRECT -- ref does not trigger re-render
const intervalRef = useRef<number | null>(null);

useEffect(() => {
  intervalRef.current = window.setInterval(() => {
    // ...
  }, 1000);
  return () => {
    if (intervalRef.current) clearInterval(intervalRef.current);
  };
}, []);
```

---

## AP-10: Initializing State with Expensive Function Call

**Problem**: Passing a function call (not a function reference) to `useState` runs the computation on every render.

```typescript
// WRONG -- parseData() runs on EVERY render, result discarded after first
const [data, setData] = useState(parseExpensiveData(rawInput));
```

**Fix**: Pass an initializer function (no parentheses).

```typescript
// CORRECT -- parseData only runs once on mount
const [data, setData] = useState(() => parseExpensiveData(rawInput));
```
