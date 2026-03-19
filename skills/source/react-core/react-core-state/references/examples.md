# react-core-state -- Code Examples

## useState Examples

### Simple Counter with TypeScript

```typescript
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState<number>(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(prev => prev + 1)}>Increment</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

### Object State with Immutable Updates

```typescript
interface FormData {
  name: string;
  email: string;
  age: number;
}

function ProfileForm() {
  const [form, setForm] = useState<FormData>({
    name: "",
    email: "",
    age: 0,
  });

  // ALWAYS spread the previous state when updating one field
  const updateField = <K extends keyof FormData>(
    field: K,
    value: FormData[K]
  ) => {
    setForm(prev => ({ ...prev, [field]: value }));
  };

  return (
    <form>
      <input
        value={form.name}
        onChange={e => updateField("name", e.target.value)}
      />
      <input
        value={form.email}
        onChange={e => updateField("email", e.target.value)}
      />
      <input
        type="number"
        value={form.age}
        onChange={e => updateField("age", Number(e.target.value))}
      />
    </form>
  );
}
```

### Array State -- Add, Remove, Update

```typescript
interface Todo {
  id: number;
  text: string;
  done: boolean;
}

function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [nextId, setNextId] = useState<number>(1);

  // Add -- ALWAYS create new array
  const addTodo = (text: string) => {
    setTodos(prev => [...prev, { id: nextId, text, done: false }]);
    setNextId(prev => prev + 1);
  };

  // Remove -- filter creates new array
  const removeTodo = (id: number) => {
    setTodos(prev => prev.filter(t => t.id !== id));
  };

  // Update -- map creates new array with replaced item
  const toggleTodo = (id: number) => {
    setTodos(prev =>
      prev.map(t => (t.id === id ? { ...t, done: !t.done } : t))
    );
  };

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <span
            style={{ textDecoration: todo.done ? "line-through" : "none" }}
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

### Lazy Initializer -- Expensive Computation

```typescript
interface AppSettings {
  theme: "light" | "dark";
  fontSize: number;
  locale: string;
}

function SettingsPanel() {
  // The function is only called on first render
  const [settings, setSettings] = useState<AppSettings>(() => {
    const saved = localStorage.getItem("settings");
    return saved ? JSON.parse(saved) : { theme: "light", fontSize: 14, locale: "en" };
  });

  // NEVER do this -- parseJSON runs on EVERY render:
  // const [settings, setSettings] = useState(JSON.parse(localStorage.getItem("settings")!));

  return <div>{settings.theme}</div>;
}
```

---

## useReducer Examples

### Form with Validation

```typescript
interface FormState {
  values: { username: string; password: string };
  errors: Record<string, string>;
  isSubmitting: boolean;
}

type FormAction =
  | { type: "SET_FIELD"; field: string; value: string }
  | { type: "SET_ERROR"; field: string; error: string }
  | { type: "CLEAR_ERRORS" }
  | { type: "SUBMIT_START" }
  | { type: "SUBMIT_SUCCESS" }
  | { type: "SUBMIT_FAILURE"; errors: Record<string, string> };

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case "SET_FIELD":
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        errors: { ...state.errors, [action.field]: "" },
      };
    case "SET_ERROR":
      return {
        ...state,
        errors: { ...state.errors, [action.field]: action.error },
      };
    case "CLEAR_ERRORS":
      return { ...state, errors: {} };
    case "SUBMIT_START":
      return { ...state, isSubmitting: true, errors: {} };
    case "SUBMIT_SUCCESS":
      return { ...state, isSubmitting: false };
    case "SUBMIT_FAILURE":
      return { ...state, isSubmitting: false, errors: action.errors };
  }
}

function LoginForm() {
  const [state, dispatch] = useReducer(formReducer, {
    values: { username: "", password: "" },
    errors: {},
    isSubmitting: false,
  });

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    dispatch({ type: "SUBMIT_START" });
    try {
      await loginAPI(state.values);
      dispatch({ type: "SUBMIT_SUCCESS" });
    } catch (err) {
      dispatch({ type: "SUBMIT_FAILURE", errors: { form: "Login failed" } });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={state.values.username}
        onChange={e => dispatch({ type: "SET_FIELD", field: "username", value: e.target.value })}
      />
      {state.errors.username && <span>{state.errors.username}</span>}
      <button disabled={state.isSubmitting}>
        {state.isSubmitting ? "Logging in..." : "Log In"}
      </button>
    </form>
  );
}
```

### Shopping Cart Reducer

```typescript
interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
}

type CartAction =
  | { type: "ADD_ITEM"; item: Omit<CartItem, "quantity"> }
  | { type: "REMOVE_ITEM"; id: string }
  | { type: "UPDATE_QUANTITY"; id: string; quantity: number }
  | { type: "CLEAR_CART" };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD_ITEM": {
      const existing = state.items.find(i => i.id === action.item.id);
      if (existing) {
        return {
          items: state.items.map(i =>
            i.id === action.item.id ? { ...i, quantity: i.quantity + 1 } : i
          ),
        };
      }
      return { items: [...state.items, { ...action.item, quantity: 1 }] };
    }
    case "REMOVE_ITEM":
      return { items: state.items.filter(i => i.id !== action.id) };
    case "UPDATE_QUANTITY":
      return {
        items: state.items.map(i =>
          i.id === action.id ? { ...i, quantity: action.quantity } : i
        ),
      };
    case "CLEAR_CART":
      return { items: [] };
  }
}

// Derived state -- computed during render, NEVER stored
function CartSummary({ items }: { items: CartItem[] }) {
  const total = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  const itemCount = items.reduce((sum, i) => sum + i.quantity, 0);

  return (
    <div>
      <p>{itemCount} items</p>
      <p>Total: ${total.toFixed(2)}</p>
    </div>
  );
}
```

---

## Lifting State Up

```typescript
// State lives in the closest common ancestor
function SearchPage() {
  const [query, setQuery] = useState<string>("");

  return (
    <div>
      <SearchBar query={query} onQueryChange={setQuery} />
      <SearchResults query={query} />
    </div>
  );
}

// Child receives state and updater via props
function SearchBar({
  query,
  onQueryChange,
}: {
  query: string;
  onQueryChange: (q: string) => void;
}) {
  return (
    <input
      value={query}
      onChange={e => onQueryChange(e.target.value)}
      placeholder="Search..."
    />
  );
}

function SearchResults({ query }: { query: string }) {
  const filtered = useMemo(
    () => allItems.filter(item => item.name.includes(query)),
    [query]
  );
  return (
    <ul>
      {filtered.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

---

## Derived State -- Compute, NEVER Store

```typescript
// CORRECT: Compute during render
function FilteredList({ items, filter }: { items: Item[]; filter: string }) {
  const filteredItems = useMemo(
    () => items.filter(i => i.category === filter),
    [items, filter]
  );
  const count = filteredItems.length; // Also derived -- no state needed

  return <p>{count} items match "{filter}"</p>;
}

// WRONG: Storing derived data in state
function FilteredListBad({ items, filter }: { items: Item[]; filter: string }) {
  const [filteredItems, setFilteredItems] = useState<Item[]>([]);

  // This useEffect is unnecessary and creates sync bugs
  useEffect(() => {
    setFilteredItems(items.filter(i => i.category === filter));
  }, [items, filter]);

  return <p>{filteredItems.length} items</p>;
}
```

---

## React 19: useActionState

```typescript
import { useActionState } from "react";

interface SubmitState {
  message: string;
  error: string | null;
}

async function submitForm(
  prevState: SubmitState,
  formData: FormData
): Promise<SubmitState> {
  const name = formData.get("name") as string;
  try {
    await saveToServer({ name });
    return { message: `Saved ${name}`, error: null };
  } catch {
    return { message: "", error: "Save failed" };
  }
}

function MyForm() {
  const [state, formAction, isPending] = useActionState(submitForm, {
    message: "",
    error: null,
  });

  return (
    <form action={formAction}>
      <input name="name" required />
      <button disabled={isPending}>
        {isPending ? "Saving..." : "Save"}
      </button>
      {state.error && <p className="error">{state.error}</p>}
      {state.message && <p className="success">{state.message}</p>}
    </form>
  );
}
```

---

## React 19: useOptimistic

```typescript
import { useOptimistic, startTransition } from "react";

interface Message {
  id: string;
  text: string;
  sending?: boolean;
}

function Chat({ messages }: { messages: Message[] }) {
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (state: Message[], newMessage: Message) => [...state, { ...newMessage, sending: true }]
  );

  const sendMessage = async (text: string) => {
    startTransition(async () => {
      addOptimisticMessage({ id: crypto.randomUUID(), text });
      await postMessageToServer(text);
      // On success, parent re-renders with new messages prop
      // On failure, optimistic message automatically disappears
    });
  };

  return (
    <div>
      {optimisticMessages.map(msg => (
        <div key={msg.id} style={{ opacity: msg.sending ? 0.5 : 1 }}>
          {msg.text}
        </div>
      ))}
    </div>
  );
}
```
