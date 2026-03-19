# Advanced Hooks — Patterns and Examples

## Pattern: Accessible Form Fields with useId

```tsx
import { useId } from 'react';

interface FieldProps {
  label: string;
  type?: string;
  error?: string;
}

function FormField({ label, type = 'text', error }: FieldProps): JSX.Element {
  const id = useId();
  const inputId = id + '-input';
  const errorId = id + '-error';
  const hintId = id + '-hint';

  return (
    <div>
      <label htmlFor={inputId}>{label}</label>
      <input
        id={inputId}
        type={type}
        aria-describedby={error ? errorId : hintId}
        aria-invalid={!!error}
      />
      {error && <p id={errorId} role="alert">{error}</p>}
    </div>
  );
}
```

---

## Pattern: Search with Deferred Results

Keeps the text input responsive while deferring expensive filtering.

```tsx
import { useState, useDeferredValue, useMemo } from 'react';

interface Product {
  id: string;
  name: string;
  category: string;
}

function ProductSearch({ products }: { products: Product[] }): JSX.Element {
  const [query, setQuery] = useState<string>('');
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  const filteredProducts = useMemo(
    () => products.filter(p =>
      p.name.toLowerCase().includes(deferredQuery.toLowerCase())
    ),
    [products, deferredQuery]
  );

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search products..."
      />
      <div style={{ opacity: isStale ? 0.6 : 1, transition: 'opacity 0.2s' }}>
        {filteredProducts.map(p => <ProductCard key={p.id} product={p} />)}
      </div>
    </div>
  );
}
```

---

## Pattern: Tab Navigation with useTransition

Prevents UI jank when switching between tabs with expensive content.

```tsx
import { useState, useTransition, Suspense } from 'react';

type Tab = 'posts' | 'comments' | 'photos';

function TabContainer(): JSX.Element {
  const [tab, setTab] = useState<Tab>('posts');
  const [isPending, startTransition] = useTransition();

  function handleTabChange(nextTab: Tab): void {
    startTransition(() => {
      setTab(nextTab);
    });
  }

  return (
    <div>
      <nav>
        {(['posts', 'comments', 'photos'] as Tab[]).map(t => (
          <button
            key={t}
            onClick={() => handleTabChange(t)}
            style={{ fontWeight: tab === t ? 'bold' : 'normal' }}
          >
            {t}
          </button>
        ))}
      </nav>
      {isPending && <div className="spinner" />}
      <Suspense fallback={<div>Loading tab...</div>}>
        <TabContent tab={tab} />
      </Suspense>
    </div>
  );
}
```

---

## Pattern: External Store Subscription

Subscribe to browser APIs or third-party state without tearing.

```tsx
import { useSyncExternalStore, useCallback } from 'react';

// Store: window dimensions
function subscribe(callback: () => void): () => void {
  window.addEventListener('resize', callback);
  return () => window.removeEventListener('resize', callback);
}

function getSnapshot(): { width: number; height: number } {
  // ALWAYS return a cached/memoized object to avoid infinite re-renders
  return windowSize;
}

let windowSize = { width: window.innerWidth, height: window.innerHeight };
window.addEventListener('resize', () => {
  windowSize = { width: window.innerWidth, height: window.innerHeight };
});

function getServerSnapshot(): { width: number; height: number } {
  return { width: 1024, height: 768 }; // sensible default for SSR
}

function useWindowSize(): { width: number; height: number } {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}

function ResponsiveLayout(): JSX.Element {
  const { width } = useWindowSize();
  return width > 768 ? <DesktopLayout /> : <MobileLayout />;
}
```

---

## Pattern: Generic External Store Hook

```tsx
import { useSyncExternalStore } from 'react';

interface Store<T> {
  getState(): T;
  subscribe(listener: () => void): () => void;
}

function useStore<T>(store: Store<T>): T {
  return useSyncExternalStore(
    store.subscribe,
    store.getState,
    store.getState  // server snapshot
  );
}

function useStoreSelector<T, S>(store: Store<T>, selector: (state: T) => S): S {
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(store.getState())
  );
}
```

---

## Pattern: Custom Hook with useDebugValue

```tsx
import { useState, useEffect, useDebugValue } from 'react';

interface FetchState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: true,
    error: null,
  });

  useDebugValue(state.loading ? 'Loading' : state.error ? 'Error' : 'Ready');

  useEffect(() => {
    let ignore = false;
    setState({ data: null, loading: true, error: null });

    fetch(url)
      .then(res => res.json())
      .then((data: T) => {
        if (!ignore) setState({ data, loading: false, error: null });
      })
      .catch((error: Error) => {
        if (!ignore) setState({ data: null, loading: false, error });
      });

    return () => { ignore = true; };
  }, [url]);

  return state;
}
```

---

## React 19 Patterns

### Pattern: use() with Promises and Suspense

```tsx
import { use, Suspense } from 'react';

interface Article {
  id: string;
  title: string;
  content: string;
}

// Promise created in Server Component and passed as prop
function ArticleContent({ articlePromise }: { articlePromise: Promise<Article> }): JSX.Element {
  const article = use(articlePromise);
  return (
    <article>
      <h1>{article.title}</h1>
      <p>{article.content}</p>
    </article>
  );
}

function ArticlePage({ articlePromise }: { articlePromise: Promise<Article> }): JSX.Element {
  return (
    <Suspense fallback={<div>Loading article...</div>}>
      <ArticleContent articlePromise={articlePromise} />
    </Suspense>
  );
}
```

### Pattern: use() with Conditional Context

```tsx
import { use, createContext } from 'react';

const ThemeContext = createContext<{ dark: boolean }>({ dark: false });
const AuthContext = createContext<{ isAdmin: boolean }>({ isAdmin: false });

function AdminPanel({ show }: { show: boolean }): JSX.Element | null {
  if (!show) return null;

  // Conditional context reading -- ONLY possible with use(), not useContext
  const { isAdmin } = use(AuthContext);
  if (!isAdmin) return null;

  const theme = use(ThemeContext);
  return <div style={{ background: theme.dark ? '#333' : '#fff' }}>Admin Panel</div>;
}
```

---

### Pattern: Form Action with useActionState

```tsx
import { useActionState } from 'react';

interface FormState {
  message: string | null;
  errors: Record<string, string>;
}

async function createUser(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const email = formData.get('email') as string;
  const name = formData.get('name') as string;

  const errors: Record<string, string> = {};
  if (!email) errors.email = 'Email is required';
  if (!name) errors.name = 'Name is required';
  if (Object.keys(errors).length > 0) return { message: null, errors };

  await fetch('/api/users', {
    method: 'POST',
    body: JSON.stringify({ email, name }),
  });

  return { message: 'User created successfully', errors: {} };
}

function UserForm(): JSX.Element {
  const [state, dispatch, isPending] = useActionState(createUser, {
    message: null,
    errors: {},
  });

  return (
    <form action={dispatch}>
      <div>
        <input name="name" placeholder="Name" disabled={isPending} />
        {state.errors.name && <span className="error">{state.errors.name}</span>}
      </div>
      <div>
        <input name="email" type="email" placeholder="Email" disabled={isPending} />
        {state.errors.email && <span className="error">{state.errors.email}</span>}
      </div>
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create User'}
      </button>
      {state.message && <p className="success">{state.message}</p>}
    </form>
  );
}
```

---

### Pattern: Optimistic Todo List

```tsx
import { useOptimistic, startTransition } from 'react';

interface Todo {
  id: string;
  text: string;
  pending?: boolean;
}

function TodoList({ todos, onAdd }: {
  todos: Todo[];
  onAdd: (text: string) => Promise<void>;
}): JSX.Element {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (currentTodos: Todo[], newTodo: Todo) => [...currentTodos, newTodo]
  );

  function handleAdd(formData: FormData): void {
    const text = formData.get('text') as string;
    const newTodo: Todo = { id: crypto.randomUUID(), text, pending: true };

    startTransition(async () => {
      addOptimisticTodo(newTodo);
      await onAdd(text);
    });
  }

  return (
    <div>
      <form action={handleAdd}>
        <input name="text" placeholder="New todo..." />
        <button type="submit">Add</button>
      </form>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} style={{ opacity: todo.pending ? 0.5 : 1 }}>
            {todo.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

### Pattern: Form Status in Child Component

```tsx
import { useFormStatus } from 'react-dom';
import { useActionState } from 'react';

function SubmitButton({ label }: { label: string }): JSX.Element {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Processing...' : label}
    </button>
  );
}

function FormProgress(): JSX.Element {
  const { pending, data } = useFormStatus();
  if (!pending) return <></>;
  const name = data?.get('name') as string | null;
  return <p>Submitting {name ? `for ${name}` : ''}...</p>;
}

function RegistrationForm(): JSX.Element {
  const [state, dispatch] = useActionState(registerAction, { error: null });

  return (
    <form action={dispatch}>
      <input name="name" required />
      <input name="email" type="email" required />
      <FormProgress />
      <SubmitButton label="Register" />
      {state.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

---

### Pattern: Combining useActionState + useOptimistic

```tsx
import { useActionState, useOptimistic, startTransition } from 'react';

interface Message {
  id: string;
  text: string;
  sending?: boolean;
}

async function sendMessageAction(
  prevState: { messages: Message[] },
  formData: FormData
): Promise<{ messages: Message[] }> {
  const text = formData.get('text') as string;
  const saved = await saveMessage(text);
  return { messages: [...prevState.messages, saved] };
}

function Chat(): JSX.Element {
  const [state, dispatch, isPending] = useActionState(sendMessageAction, {
    messages: [],
  });

  const [optimisticMessages, addOptimistic] = useOptimistic(
    state.messages,
    (current: Message[], newMsg: Message) => [...current, newMsg]
  );

  function handleSubmit(formData: FormData): void {
    const text = formData.get('text') as string;
    addOptimistic({ id: crypto.randomUUID(), text, sending: true });
    dispatch(formData);
  }

  return (
    <div>
      <ul>
        {optimisticMessages.map(msg => (
          <li key={msg.id} style={{ opacity: msg.sending ? 0.5 : 1 }}>
            {msg.text}
          </li>
        ))}
      </ul>
      <form action={handleSubmit}>
        <input name="text" disabled={isPending} />
        <button type="submit" disabled={isPending}>Send</button>
      </form>
    </div>
  );
}
```
