# Component Anti-Patterns

> Reference file for `react-syntax-components`. Common mistakes with components and how to fix them.

---

## AP-01: Defining Components Inside Components

**Severity**: Critical -- causes state reset on every parent render.

```tsx
// WRONG -- Inner component is recreated every render, destroying state
function Parent(): React.ReactElement {
  function Child(): React.ReactElement {
    const [count, setCount] = useState(0); // resets to 0 every time Parent renders
    return <button onClick={() => setCount(count + 1)}>{count}</button>;
  }
  return <Child />;
}

// CORRECT -- define at module scope
function Child(): React.ReactElement {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

function Parent(): React.ReactElement {
  return <Child />;
}
```

**Why**: React uses the component function's identity to determine if a component is the same between renders. A new function reference = new component = full remount.

---

## AP-02: Using `React.FC` / `React.FunctionComponent`

**Severity**: Low -- works but adds no value and has historical baggage.

```tsx
// AVOID -- React.FC adds implicit children (React 17), obscures return type
const Greeting: React.FC<GreetingProps> = ({ name }) => {
  return <h1>Hello, {name}</h1>;
};

// PREFERRED -- explicit, clear, no hidden behavior
function Greeting({ name }: GreetingProps): React.ReactElement {
  return <h1>Hello, {name}</h1>;
}
```

**Why**: `React.FC` historically included implicit `children` in the props type (fixed in React 18 types), does not support generics natively, and hides the return type. Plain function declarations are clearer.

---

## AP-03: Spreading Props Without Filtering

**Severity**: Medium -- passes invalid attributes to DOM elements.

```tsx
// WRONG -- custom props leak to DOM, causing React warnings
interface CardProps {
  readonly variant: 'primary' | 'secondary';
  readonly elevated: boolean;
}

function Card({ ...props }: CardProps): React.ReactElement {
  return <div {...props}>Content</div>; // variant and elevated go to <div>
}

// CORRECT -- destructure known props, spread the rest
function Card({ variant, elevated, ...rest }: CardProps & React.HTMLAttributes<HTMLDivElement>): React.ReactElement {
  return (
    <div
      className={`card card-${variant} ${elevated ? 'elevated' : ''}`}
      {...rest}
    >
      Content
    </div>
  );
}
```

---

## AP-04: Memo Without Stable Props

**Severity**: Medium -- `React.memo` does nothing if parent recreates props every render.

```tsx
// WRONG -- new function reference every render, memo is useless
function Parent(): React.ReactElement {
  const [items, setItems] = useState<string[]>([]);

  return (
    <MemoizedList
      items={items}
      onItemClick={(item) => console.log(item)} // new function every render
    />
  );
}

// CORRECT -- stabilize callback with useCallback
function Parent(): React.ReactElement {
  const [items, setItems] = useState<string[]>([]);

  const handleItemClick = useCallback((item: string) => {
    console.log(item);
  }, []);

  return (
    <MemoizedList
      items={items}
      onItemClick={handleItemClick}
    />
  );
}
```

---

## AP-05: Conditional Hook Calls

**Severity**: Critical -- violates Rules of Hooks, causes runtime crashes.

```tsx
// WRONG -- hook called conditionally
function UserProfile({ userId }: { userId: string | null }): React.ReactElement {
  if (!userId) return <p>No user selected</p>;

  const [user, setUser] = useState<User | null>(null); // called conditionally!

  // ...
}

// CORRECT -- hooks at top level, conditional rendering below
function UserProfile({ userId }: { userId: string | null }): React.ReactElement {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    if (!userId) return;
    fetchUser(userId).then(setUser);
  }, [userId]);

  if (!userId) return <p>No user selected</p>;

  return <div>{user?.name}</div>;
}
```

---

## AP-06: Using Index as Key for Dynamic Lists

**Severity**: High -- causes state corruption when items are reordered, inserted, or removed.

```tsx
// WRONG -- index key breaks state association when list changes
{items.map((item, index) => (
  <ItemEditor key={index} item={item} />  // state gets mismatched after reorder
))}

// CORRECT -- use stable, unique identifier from data
{items.map((item) => (
  <ItemEditor key={item.id} item={item} />
))}
```

**When index key is acceptable**: ONLY for truly static lists that NEVER change order, are NEVER filtered, and items have no internal state.

---

## AP-07: forwardRef in React 19

**Severity**: Low -- works but deprecated and unnecessary.

```tsx
// UNNECESSARY in React 19
const Input = React.forwardRef<HTMLInputElement, InputProps>((props, ref) => {
  return <input ref={ref} {...props} />;
});

// PREFERRED in React 19 -- ref as regular prop
function Input({ ref, ...props }: InputProps & { ref?: React.Ref<HTMLInputElement> }): React.ReactElement {
  return <input ref={ref} {...props} />;
}
```

**Why**: React 19 passes `ref` as a regular prop. `forwardRef` adds unnecessary wrapping. Keep `forwardRef` ONLY if you must support React 18.

---

## AP-08: Lazy Inside Component Body

**Severity**: Critical -- destroys component state on every render.

```tsx
// WRONG -- lazy component recreated every render
function App(): React.ReactElement {
  const Page = React.lazy(() => import('./Page')); // new lazy wrapper every render!
  return (
    <Suspense fallback={<Loading />}>
      <Page />
    </Suspense>
  );
}

// CORRECT -- declare at module scope
const Page = React.lazy(() => import('./Page'));

function App(): React.ReactElement {
  return (
    <Suspense fallback={<Loading />}>
      <Page />
    </Suspense>
  );
}
```

---

## AP-09: Missing Suspense for Lazy Components

**Severity**: Critical -- causes unrecoverable error.

```tsx
// WRONG -- lazy component without Suspense boundary throws error
const AdminPanel = React.lazy(() => import('./AdminPanel'));

function App(): React.ReactElement {
  return <AdminPanel />; // ERROR: A component suspended while responding to synchronous input
}

// CORRECT -- wrap in Suspense
function App(): React.ReactElement {
  return (
    <React.Suspense fallback={<div>Loading...</div>}>
      <AdminPanel />
    </React.Suspense>
  );
}
```

---

## AP-10: Portal Event Bubbling Surprise

**Severity**: Medium -- events from portals bubble through React tree, not DOM tree.

```tsx
// SURPRISE -- clicking inside the modal triggers Parent's onClick
function Parent(): React.ReactElement {
  return (
    <div onClick={() => console.log('Parent clicked!')}>
      <Modal isOpen={true} onClose={() => {}}>
        <button>Click me</button>  {/* triggers Parent onClick! */}
      </Modal>
    </div>
  );
}

// FIX -- stop propagation in the portal content
function Modal({ isOpen, onClose, children }: ModalProps): React.ReactElement | null {
  if (!isOpen) return null;

  return createPortal(
    <div onClick={(e) => e.stopPropagation()}>
      {children}
    </div>,
    document.body
  );
}
```

**Why**: React's event system follows the React component tree, not the DOM tree. Portal children are logically still inside the parent React tree.

---

## AP-11: Mutating Props

**Severity**: Critical -- violates React's unidirectional data flow.

```tsx
// WRONG -- mutating props object
function UserCard(props: UserCardProps): React.ReactElement {
  props.name = props.name.toUpperCase(); // NEVER mutate props
  return <h2>{props.name}</h2>;
}

// CORRECT -- derive values without mutation
function UserCard({ name }: UserCardProps): React.ReactElement {
  const displayName = name.toUpperCase();
  return <h2>{displayName}</h2>;
}
```

---

## AP-12: HOC Applied Inside Render

**Severity**: Critical -- creates a new component type every render.

```tsx
// WRONG -- new component type every render
function App(): React.ReactElement {
  const EnhancedList = withSorting(List); // recreated every render!
  return <EnhancedList items={items} />;
}

// CORRECT -- apply HOC at module scope
const EnhancedList = withSorting(List);

function App(): React.ReactElement {
  return <EnhancedList items={items} />;
}
```

---

## Summary Table

| ID | Anti-Pattern | Severity | Fix |
|----|-------------|----------|-----|
| AP-01 | Component inside component | Critical | Define at module scope |
| AP-02 | Using React.FC | Low | Plain function declaration |
| AP-03 | Unfiltered prop spreading | Medium | Destructure known props first |
| AP-04 | Memo without stable props | Medium | useCallback/useMemo in parent |
| AP-05 | Conditional hook calls | Critical | Hooks at top level always |
| AP-06 | Index as key (dynamic lists) | High | Use stable data ID |
| AP-07 | forwardRef in React 19 | Low | ref as regular prop |
| AP-08 | lazy inside component | Critical | Declare at module scope |
| AP-09 | lazy without Suspense | Critical | Wrap in Suspense boundary |
| AP-10 | Portal event bubbling | Medium | stopPropagation in portal |
| AP-11 | Mutating props | Critical | Derive new values |
| AP-12 | HOC inside render | Critical | Apply at module scope |
