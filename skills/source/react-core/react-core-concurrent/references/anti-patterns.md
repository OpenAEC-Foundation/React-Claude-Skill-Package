# react-core-concurrent: Anti-Patterns

## Anti-Pattern 1: Declaring lazy() Inside Components

**Problem:** Declaring `lazy()` inside a component function causes the lazy component to be re-created on every render, resetting all state.

```tsx
// BAD -- lazy() inside component body
function App(): JSX.Element {
  const Settings = lazy(() => import("./Settings")); // Re-created every render!
  return (
    <Suspense fallback={<Loading />}>
      <Settings />
    </Suspense>
  );
}

// GOOD -- lazy() at module top level
const Settings = lazy(() => import("./Settings")); // Created once

function App(): JSX.Element {
  return (
    <Suspense fallback={<Loading />}>
      <Settings />
    </Suspense>
  );
}
```

**Rule:** ALWAYS declare `lazy()` at module top level. React caches the resolved component per lazy reference -- a new reference means a new cache entry and a new load.

---

## Anti-Pattern 2: useDeferredValue Without memo()

**Problem:** `useDeferredValue` defers the value, but the child component still re-renders with the old value on the first pass. Without `memo()`, the child re-renders twice (old value + new value) with no performance benefit.

```tsx
// BAD -- ProductList re-renders on every keystroke despite deferral
function SearchPage(): JSX.Element {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ProductList filter={deferredQuery} /> {/* Re-renders anyway! */}
    </>
  );
}

function ProductList({ filter }: { filter: string }): JSX.Element {
  // Expensive filtering runs on EVERY render
  return <ul>{/* ... */}</ul>;
}

// GOOD -- memo() prevents re-render when deferredQuery hasn't changed
const ProductList = memo(function ProductList({ filter }: { filter: string }): JSX.Element {
  return <ul>{/* ... */}</ul>;
});
```

**Rule:** ALWAYS wrap the component receiving a deferred value in `memo()`. Without it, `useDeferredValue` provides zero performance benefit.

---

## Anti-Pattern 3: Using Transitions for Text Input

**Problem:** Transitions are interruptible -- React deprioritizes them. For a controlled text input, this means the input value lags behind keystrokes, creating a broken typing experience.

```tsx
// BAD -- input becomes unresponsive
function SearchForm(): JSX.Element {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  return (
    <input
      value={query}
      onChange={(e) => {
        startTransition(() => {
          setQuery(e.target.value); // Input update is deprioritized!
        });
      }}
    />
  );
}

// GOOD -- update input immediately, defer expensive downstream work
function SearchForm(): JSX.Element {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Suspense fallback={<Loading />}>
        <SearchResults query={deferredQuery} />
      </Suspense>
    </>
  );
}
```

**Rule:** NEVER use `startTransition` for controlled input state updates. Use `useDeferredValue` to defer the downstream render instead.

---

## Anti-Pattern 4: Expecting Suspense to Catch Effect/Event Handler Fetches

**Problem:** Suspense only catches suspensions from Suspense-enabled data sources (`lazy()`, `use()`, framework integrations). Data fetches initiated in `useEffect` or event handlers do NOT trigger Suspense.

```tsx
// BAD -- useEffect fetch does NOT trigger Suspense
function UserProfile({ userId }: { userId: string }): JSX.Element {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    fetchUser(userId).then(setUser); // Suspense boundary will NEVER show fallback
  }, [userId]);

  if (!user) return <Loading />; // Manual loading state needed

  return <h1>{user.name}</h1>;
}

// GOOD (React 19) -- use() integrates with Suspense
function UserProfile({ userPromise }: { userPromise: Promise<User> }): JSX.Element {
  const user = use(userPromise); // Suspends properly
  return <h1>{user.name}</h1>;
}
```

**Rule:** NEVER expect Suspense to detect data fetching in `useEffect` or event handlers. Use `use()` (React 19), `React.lazy()`, or a Suspense-enabled framework.

---

## Anti-Pattern 5: Missing Error Boundaries Around Data-Fetching Suspense

**Problem:** If a promise rejects inside a Suspense boundary without an Error Boundary, the error propagates up and can crash the entire application.

```tsx
// BAD -- no Error Boundary, rejected promise crashes the app
function App(): JSX.Element {
  return (
    <Suspense fallback={<Loading />}>
      <DataComponent /> {/* If this rejects, entire app crashes */}
    </Suspense>
  );
}

// GOOD -- Error Boundary catches rejected promises
function App(): JSX.Element {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<Loading />}>
        <DataComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**Rule:** ALWAYS wrap data-fetching Suspense boundaries in an Error Boundary. This prevents a single failed request from taking down the entire component tree.

---

## Anti-Pattern 6: Forgetting startTransition After await

**Problem:** In React 19 async transitions, state updates after an `await` lose their transition context. They become urgent updates and can cause Suspense fallbacks to re-appear.

```tsx
// BAD -- post-await setState is NOT inside a transition
function SaveButton(): JSX.Element {
  const [isPending, startTransition] = useTransition();
  const [status, setStatus] = useState<string>("idle");

  function handleSave(): void {
    startTransition(async () => {
      setStatus("saving"); // Inside transition
      const result = await saveData();
      setStatus("done"); // NOT inside transition! Becomes urgent update
    });
  }

  return <button onClick={handleSave}>{status}</button>;
}

// GOOD -- wrap post-await updates in a new startTransition
function SaveButton(): JSX.Element {
  const [isPending, startTransition] = useTransition();
  const [status, setStatus] = useState<string>("idle");

  function handleSave(): void {
    startTransition(async () => {
      setStatus("saving");
      const result = await saveData();
      startTransition(() => {
        setStatus("done"); // Properly inside a transition
      });
    });
  }

  return <button onClick={handleSave}>{status}</button>;
}
```

**Rule:** ALWAYS wrap state updates after `await` in a fresh `startTransition` call. The transition context is lost at every `await` boundary.

---

## Anti-Pattern 7: Creating New Objects for useDeferredValue

**Problem:** Passing a new object reference on every render to `useDeferredValue` triggers unnecessary background re-renders because React sees a "new" value every time.

```tsx
// BAD -- new object on every render
function FilteredList({ items }: { items: Item[] }): JSX.Element {
  const [query, setQuery] = useState("");
  // New object created every render -- useDeferredValue always schedules background work
  const deferredFilter = useDeferredValue({ query, items });

  return <ExpensiveList filter={deferredFilter} />;
}

// GOOD -- pass primitive value
function FilteredList({ items }: { items: Item[] }): JSX.Element {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query); // Primitive, stable reference

  return <ExpensiveList query={deferredQuery} items={items} />;
}
```

**Rule:** ALWAYS pass primitive values or objects created outside rendering to `useDeferredValue`. New object references on every render defeat the deferred comparison.

---

## Anti-Pattern 8: Over-Granular Suspense Boundaries

**Problem:** Wrapping every single component in its own Suspense boundary creates a "popcorn" loading effect where dozens of skeletons appear and resolve independently, creating a jarring user experience.

```tsx
// BAD -- every component has its own boundary
function Dashboard(): JSX.Element {
  return (
    <div>
      <Suspense fallback={<Skeleton />}>
        <UserName />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <UserAvatar />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <UserEmail />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <UserBio />
      </Suspense>
    </div>
  );
}

// GOOD -- group related content in a single boundary
function Dashboard(): JSX.Element {
  return (
    <div>
      <Suspense fallback={<UserCardSkeleton />}>
        <UserName />
        <UserAvatar />
        <UserEmail />
        <UserBio />
      </Suspense>
    </div>
  );
}
```

**Rule:** NEVER make Suspense boundaries finer than the intended loading experience. Group components that should appear together inside the SAME boundary.

---

## Anti-Pattern 9: Using use() in Try-Catch Blocks

**Problem:** `use()` integrates with Suspense and Error Boundaries by throwing. Wrapping it in try-catch intercepts these throws and breaks the Suspense/Error Boundary mechanism.

```tsx
// BAD -- try-catch intercepts Suspense throws
function DataDisplay({ dataPromise }: { dataPromise: Promise<Data> }): JSX.Element {
  try {
    const data = use(dataPromise); // Throws for Suspense -- caught by try-catch!
    return <div>{data.value}</div>;
  } catch (e) {
    return <div>Error</div>; // Catches Suspense throws too!
  }
}

// GOOD -- let Suspense and Error Boundaries handle it
function DataDisplay({ dataPromise }: { dataPromise: Promise<Data> }): JSX.Element {
  const data = use(dataPromise); // Throws handled by Suspense + ErrorBoundary
  return <div>{data.value}</div>;
}

// Usage
<ErrorBoundary fallback={<ErrorMessage />}>
  <Suspense fallback={<Loading />}>
    <DataDisplay dataPromise={fetchData()} />
  </Suspense>
</ErrorBoundary>
```

**Rule:** NEVER call `use()` inside a try-catch block. Use Error Boundaries for error handling and Suspense for loading states.

---

## Anti-Pattern 10: Nesting startTransition Expecting Independent Tracking

**Problem:** Nesting `startTransition` calls from the same `useTransition` hook does not create independent transitions. The inner call shares the outer `isPending` state.

```tsx
// BAD -- nested transitions share isPending
function MultiAction(): JSX.Element {
  const [isPending, startTransition] = useTransition();

  function handleClick(): void {
    startTransition(() => {
      setA(newA);
      startTransition(() => {
        setB(newB); // Same isPending as outer transition
      });
    });
  }

  return <div>{isPending ? "Loading..." : "Ready"}</div>;
}

// GOOD -- separate useTransition hooks for independent tracking
function MultiAction(): JSX.Element {
  const [isPendingA, startTransitionA] = useTransition();
  const [isPendingB, startTransitionB] = useTransition();

  function handleClick(): void {
    startTransitionA(() => setA(newA));
    startTransitionB(() => setB(newB));
  }

  return (
    <div>
      {isPendingA && <span>Loading A...</span>}
      {isPendingB && <span>Loading B...</span>}
    </div>
  );
}
```

**Rule:** NEVER nest `startTransition` calls expecting independent pending state tracking. Use separate `useTransition` hooks for independently tracked transitions.
