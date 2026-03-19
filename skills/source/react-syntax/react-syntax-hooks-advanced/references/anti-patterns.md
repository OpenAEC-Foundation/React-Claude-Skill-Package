# Advanced Hooks — Anti-Patterns

## useId Anti-Patterns

### NEVER use useId for list keys

```tsx
// WRONG — useId is for accessibility, not list rendering
function TodoList({ items }: { items: string[] }): JSX.Element {
  return (
    <ul>
      {items.map(item => {
        const id = useId(); // BREAKS: hooks in loops
        return <li key={id}>{item}</li>;
      })}
    </ul>
  );
}

// CORRECT — keys come from data
function TodoList({ items }: { items: { id: string; text: string }[] }): JSX.Element {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.text}</li>)}
    </ul>
  );
}
```

**Why**: `useId` cannot be called inside loops (Rules of Hooks). Keys MUST be derived from your data, not generated per render.

### NEVER use useId for cache keys or data identifiers

```tsx
// WRONG — useId output changes between server/client in unpredictable ways
const cacheKey = useId(); // Not designed for this purpose
localStorage.setItem(cacheKey, data);

// CORRECT — use crypto.randomUUID() or data-derived keys
const cacheKey = `user-${userId}-preferences`;
```

---

## useTransition Anti-Patterns

### NEVER wrap controlled input updates in startTransition

```tsx
// WRONG — input becomes unresponsive
function Search(): JSX.Element {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  function handleChange(e: React.ChangeEvent<HTMLInputElement>): void {
    startTransition(() => {
      setQuery(e.target.value); // Transition makes typing laggy
    });
  }

  return <input value={query} onChange={handleChange} />;
}

// CORRECT — update input immediately, defer expensive work
function Search(): JSX.Element {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState<Item[]>([]);

  function handleChange(e: React.ChangeEvent<HTMLInputElement>): void {
    setQuery(e.target.value); // Immediate update
    startTransition(() => {
      setResults(filterItems(e.target.value)); // Deferred work
    });
  }

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <ResultList items={results} />}
    </div>
  );
}
```

**Why**: Transitions are interruptible. Wrapping a controlled input update in a transition causes the input to revert characters while typing.

### NEVER nest startTransition inside setTimeout

```tsx
// WRONG — loses transition context
startTransition(() => {
  setTimeout(() => {
    setState(newValue); // NOT in a transition anymore
  }, 1000);
});

// CORRECT — wrap startTransition inside setTimeout
setTimeout(() => {
  startTransition(() => {
    setState(newValue); // Properly marked as transition
  });
}, 1000);
```

**Why**: JavaScript async boundaries (setTimeout, Promise.then) exit the `startTransition` scope.

### NEVER forget nested startTransition after await

```tsx
// WRONG — updates after await lose transition context
startTransition(async () => {
  const data = await fetchData();
  setState(data); // NOT in transition after await in React 18
});

// CORRECT — re-wrap after await
startTransition(async () => {
  const data = await fetchData();
  startTransition(() => {
    setState(data); // Properly marked
  });
});
```

---

## useDeferredValue Anti-Patterns

### NEVER pass new objects created during render

```tsx
// WRONG — new object every render defeats deferral
function SearchResults({ query }: { query: string }): JSX.Element {
  const deferredOptions = useDeferredValue({ query, limit: 10 }); // New object every render
  return <Results options={deferredOptions} />;
}

// CORRECT — defer a primitive or stable reference
function SearchResults({ query }: { query: string }): JSX.Element {
  const deferredQuery = useDeferredValue(query); // Primitive — stable comparison
  const options = useMemo(() => ({ query: deferredQuery, limit: 10 }), [deferredQuery]);
  return <Results options={options} />;
}
```

**Why**: `useDeferredValue` compares with `Object.is`. A new object every render is always "different", so deferral never kicks in.

---

## useSyncExternalStore Anti-Patterns

### NEVER create subscribe function inline

```tsx
// WRONG — new function every render causes resubscription loop
function OnlineStatus(): JSX.Element {
  const isOnline = useSyncExternalStore(
    (callback) => {                              // New function reference each render
      window.addEventListener('online', callback);
      window.addEventListener('offline', callback);
      return () => {
        window.removeEventListener('online', callback);
        window.removeEventListener('offline', callback);
      };
    },
    () => navigator.onLine
  );
  return <span>{isOnline ? 'Online' : 'Offline'}</span>;
}

// CORRECT — stable subscribe function declared outside component
function subscribe(callback: () => void): () => void {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function OnlineStatus(): JSX.Element {
  const isOnline = useSyncExternalStore(subscribe, () => navigator.onLine, () => true);
  return <span>{isOnline ? 'Online' : 'Offline'}</span>;
}
```

**Why**: A different `subscribe` reference causes React to unsubscribe and resubscribe every render, wasting resources and potentially causing missed updates.

### NEVER return new objects from getSnapshot

```tsx
// WRONG — infinite re-render loop
function useWindowSize() {
  return useSyncExternalStore(
    subscribe,
    () => ({ width: window.innerWidth, height: window.innerHeight }), // New object every call
  );
}

// CORRECT — return cached object, update only on change
let cachedSize = { width: 0, height: 0 };
function getSnapshot(): { width: number; height: number } {
  const next = { width: window.innerWidth, height: window.innerHeight };
  if (next.width !== cachedSize.width || next.height !== cachedSize.height) {
    cachedSize = next;
  }
  return cachedSize;
}
```

**Why**: `getSnapshot` is called frequently. If it returns a new object every time, `Object.is` comparison always returns `false`, triggering infinite re-renders.

---

## useInsertionEffect Anti-Patterns

### NEVER use useInsertionEffect in application code

```tsx
// WRONG — useInsertionEffect is for CSS-in-JS libraries only
function MyComponent(): JSX.Element {
  useInsertionEffect(() => {
    document.title = 'Hello'; // Use useEffect for this
  });
  return <div />;
}

// CORRECT — use useEffect for side effects
function MyComponent(): JSX.Element {
  useEffect(() => {
    document.title = 'Hello';
  });
  return <div />;
}
```

**Why**: `useInsertionEffect` cannot update state, refs are not attached, and it runs before DOM mutations. It exists solely for injecting `<style>` tags in CSS-in-JS libraries.

---

## use() Anti-Patterns (React 19)

### NEVER call use() inside try/catch

```tsx
// WRONG — causes "Suspense Exception" error
function UserProfile({ userPromise }: { userPromise: Promise<User> }): JSX.Element {
  try {
    const user = use(userPromise); // Throws internal Suspense exception
    return <div>{user.name}</div>;
  } catch (e) {
    return <div>Error</div>;
  }
}

// CORRECT — use ErrorBoundary for error handling
function UserProfile({ userPromise }: { userPromise: Promise<User> }): JSX.Element {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

// Wrap with ErrorBoundary and Suspense
<ErrorBoundary fallback={<div>Error loading user</div>}>
  <Suspense fallback={<div>Loading...</div>}>
    <UserProfile userPromise={fetchUser(id)} />
  </Suspense>
</ErrorBoundary>
```

**Why**: `use()` throws a special Suspense exception internally. Catching it with try/catch breaks React's Suspense mechanism.

### NEVER create promises during client render

```tsx
// WRONG — new promise every render causes repeated suspense
function Comments({ postId }: { postId: string }): JSX.Element {
  const comments = use(fetchComments(postId)); // New promise every render!
  return <ul>{comments.map(c => <li key={c.id}>{c.text}</li>)}</ul>;
}

// CORRECT — create promise in Server Component or cache it
// Server Component creates the promise once
async function Page({ postId }: { postId: string }) {
  const commentsPromise = fetchComments(postId); // Created once on server
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}
```

---

## useFormStatus Anti-Patterns (React 19)

### NEVER call useFormStatus in the form component itself

```tsx
// WRONG — pending is ALWAYS false
function LoginForm(): JSX.Element {
  const { pending } = useFormStatus(); // Same component as <form>
  return (
    <form action={loginAction}>
      <input name="email" />
      <button disabled={pending}>Log in</button>
    </form>
  );
}

// CORRECT — extract button to a child component
function SubmitButton(): JSX.Element {
  const { pending } = useFormStatus(); // Inside <form> as a child
  return <button type="submit" disabled={pending}>{pending ? 'Logging in...' : 'Log in'}</button>;
}

function LoginForm(): JSX.Element {
  return (
    <form action={loginAction}>
      <input name="email" />
      <SubmitButton />
    </form>
  );
}
```

**Why**: `useFormStatus` reads the status of the **parent** `<form>`. When called in the same component that renders the form, there is no parent form.

---

## useOptimistic Anti-Patterns (React 19)

### NEVER call setOptimistic outside a Transition

```tsx
// WRONG — optimistic value flashes and immediately reverts
function LikeButton({ liked }: { liked: boolean }): JSX.Element {
  const [optimisticLiked, setOptimisticLiked] = useOptimistic(liked);

  function handleClick(): void {
    setOptimisticLiked(!liked); // Not inside startTransition — reverts immediately
    toggleLike(!liked);
  }

  return <button onClick={handleClick}>{optimisticLiked ? 'Liked' : 'Like'}</button>;
}

// CORRECT — wrap in startTransition
function LikeButton({ liked, onToggle }: {
  liked: boolean;
  onToggle: (v: boolean) => Promise<void>;
}): JSX.Element {
  const [optimisticLiked, setOptimisticLiked] = useOptimistic(liked);

  function handleClick(): void {
    startTransition(async () => {
      setOptimisticLiked(!liked);
      await onToggle(!liked);
    });
  }

  return <button onClick={handleClick}>{optimisticLiked ? 'Liked' : 'Like'}</button>;
}
```

**Why**: The optimistic state only persists while an Action/Transition is in progress. Without a transition, the optimistic value is applied and immediately replaced by the real value.

---

## useActionState Anti-Patterns (React 19)

### NEVER confuse useActionState with useReducer

```tsx
// WRONG — treating action like a pure reducer
const [state, dispatch] = useActionState(
  (prev: number, action: { type: string }) => {
    // Side effects are allowed here, unlike useReducer
    // But returning wrong type causes issues
    if (action.type === 'INCREMENT') return prev + 1;
    return prev; // Must return State, not void
  },
  0
);

// Using dispatch outside a form without startTransition
dispatch({ type: 'INCREMENT' }); // May not work as expected

// CORRECT — use with form action or wrap in startTransition
<form action={() => dispatch({ type: 'INCREMENT' })}>
  <button type="submit">+1</button>
</form>

// OR
startTransition(() => {
  dispatch({ type: 'INCREMENT' });
});
```

**Why**: `useActionState` dispatches MUST be called from within a `startTransition` or passed as a form `action`. Direct calls outside these contexts do not trigger the pending state correctly.
