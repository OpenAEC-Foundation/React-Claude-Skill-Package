---
name: react-core-concurrent
description: "Guides React concurrent rendering features including Suspense for loading states, startTransition and useTransition for non-blocking updates, useDeferredValue for expensive computations, lazy loading with React.lazy, and streaming SSR. Activates when implementing loading states, optimizing perceived performance, handling slow renders, or using Suspense boundaries."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-core-concurrent

## Quick Reference

### Concurrent APIs at a Glance

| API | Purpose | Returns | React Version |
|-----|---------|---------|---------------|
| `<Suspense>` | Show fallback while children load | JSX boundary | 16.6+ (full in 18+) |
| `useTransition()` | Mark updates as non-blocking + track pending | `[isPending, startTransition]` | 18+ |
| `startTransition()` | Mark updates as non-blocking (no pending state) | `void` | 18+ |
| `useDeferredValue(value, initial?)` | Defer re-renders for a value | deferred value | 18+ (`initial` in 19) |
| `lazy(load)` | Code-split a component | lazy component | 16.6+ |
| `use(resource)` | Read promise or context (conditional OK) | resolved value | 19+ |

### API Signatures

```tsx
// Suspense
<Suspense fallback={<Spinner />}>{children}</Suspense>

// useTransition
const [isPending, startTransition] = useTransition();
startTransition(() => { setState(newValue); });

// Module-level startTransition
import { startTransition } from "react";
startTransition(() => { setState(newValue); });

// useDeferredValue
const deferredValue = useDeferredValue(value);
const deferredValue = useDeferredValue(value, initialValue); // React 19+

// lazy
const LazyComponent = lazy(() => import("./Component"));

// use (React 19+)
const value = use(promise);
const value = use(SomeContext);
```

---

## Critical Warnings

**NEVER** call `lazy()` inside a component body -- ALWAYS declare lazy components at module top level. Declaring inside a component resets state on every re-render.

**NEVER** use `startTransition` or `useTransition` for controlled text input updates -- transitions are interruptible and will cause input lag. Use `useDeferredValue` to defer downstream renders instead.

**NEVER** create new objects inside render and pass them to `useDeferredValue` -- ALWAYS pass primitives or objects created outside rendering. New object references on every render cause unnecessary background re-renders.

**NEVER** call `use()` inside a try-catch block -- it integrates with Suspense and Error Boundaries. Use `promise.catch()` or `<ErrorBoundary>` instead.

**NEVER** expect Suspense to detect data fetching in event handlers or useEffect -- ONLY Suspense-enabled data sources (frameworks, `lazy()`, `use()`) trigger Suspense boundaries.

**NEVER** wrap state updates after `await` without a new `startTransition` call -- updates scheduled after an `await` lose their transition context. ALWAYS wrap post-await state updates in a fresh `startTransition`.

---

## Decision Tree: Which Concurrent Feature?

```
Need to show loading UI while content loads?
  YES --> Use <Suspense> with a fallback
    Need to load a component lazily?
      YES --> Use React.lazy() + <Suspense>
    Need to read a promise in a component? (React 19)
      YES --> Use use(promise) inside <Suspense>
    Need to fetch data with loading state?
      YES --> Use Suspense-enabled framework (Next.js, Relay)

Need to keep UI responsive during a state update?
  YES --> Do you control the state setter?
    YES --> Do you need a pending indicator?
      YES --> Use useTransition() (gives isPending)
      NO  --> Use startTransition() (simpler)
    NO  --> Use useDeferredValue(value)

Need to defer an expensive re-render?
  YES --> Is the slow component receiving a prop?
    YES --> Use useDeferredValue(prop) + memo() on the slow component
    NO  --> Use useTransition() around the state update

Need to prevent Suspense fallback from re-appearing?
  YES --> Wrap navigation/update in startTransition()

Need code splitting / route-based splitting?
  YES --> Use React.lazy() + <Suspense>
```

---

## Patterns

### 1. Suspense with Nested Boundaries

ALWAYS use nested `<Suspense>` boundaries to create progressive loading experiences. The outermost boundary catches the first suspension; inner boundaries handle independent sections.

```tsx
<Suspense fallback={<PageSkeleton />}>
  <Header />
  <Suspense fallback={<SidebarSkeleton />}>
    <Sidebar />
  </Suspense>
  <Suspense fallback={<ContentSkeleton />}>
    <MainContent />
  </Suspense>
</Suspense>
```

**Loading sequence**: `PageSkeleton` shows until `Header` loads. Then `Header` appears, `SidebarSkeleton` and `ContentSkeleton` show independently until their children resolve.

### 2. Transitions for Navigation

ALWAYS use `startTransition` when navigating between views that suspend. This prevents hiding already-visible content behind a fallback.

```tsx
import { useState, useTransition } from "react";

function Router(): JSX.Element {
  const [page, setPage] = useState<string>("/home");
  const [isPending, startTransition] = useTransition();

  function navigate(url: string): void {
    startTransition(() => {
      setPage(url);
    });
  }

  return (
    <div style={{ opacity: isPending ? 0.7 : 1 }}>
      <Nav onNavigate={navigate} />
      <Suspense fallback={<PageLoader />}>
        <PageContent page={page} />
      </Suspense>
    </div>
  );
}
```

### 3. Deferred Value for Search

ALWAYS pair `useDeferredValue` with `memo()` on the receiving component. Without `memo()`, the child re-renders immediately with the old value anyway, defeating the purpose.

```tsx
import { useState, useDeferredValue, Suspense, memo } from "react";

function SearchPage(): JSX.Element {
  const [query, setQuery] = useState<string>("");
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Suspense fallback={<p>Loading...</p>}>
        <div style={{ opacity: isStale ? 0.5 : 1 }}>
          <SearchResults query={deferredQuery} />
        </div>
      </Suspense>
    </>
  );
}

const SearchResults = memo(function SearchResults({ query }: { query: string }) {
  // Expensive render or data fetch
  const results = use(fetchResults(query));
  return <ul>{results.map((r) => <li key={r.id}>{r.title}</li>)}</ul>;
});
```

### 4. Code Splitting with React.lazy

```tsx
import { lazy, Suspense } from "react";

// ALWAYS at module top level
const Settings = lazy(() => import("./Settings"));
const Dashboard = lazy(() => import("./Dashboard"));

function App({ page }: { page: string }): JSX.Element {
  return (
    <Suspense fallback={<Loading />}>
      {page === "settings" ? <Settings /> : <Dashboard />}
    </Suspense>
  );
}
```

### 5. Suspense Boundary Reset with Key

ALWAYS use the `key` prop to reset a Suspense boundary when navigating to different content of the same type (e.g., different user profiles).

```tsx
<Suspense fallback={<ProfileSkeleton />}>
  <ProfilePage key={userId} userId={userId} />
</Suspense>
```

---

## Anti-Patterns

See [references/anti-patterns.md](references/anti-patterns.md) for detailed examples.

**Summary of NEVER rules:**
- NEVER nest `startTransition` calls expecting independent tracking -- use separate `useTransition` hooks
- NEVER use `useDeferredValue` without `memo()` on the consuming component
- NEVER declare `lazy()` inside component bodies
- NEVER expect Suspense to catch event handler or useEffect fetches
- NEVER use transitions for urgent updates (text input, toggles)

---

## Version Notes: React 18 vs React 19

| Feature | React 18 | React 19 |
|---------|----------|----------|
| `<Suspense>` | Full concurrent support | Improved: no longer re-mounts effects on reveal |
| `useTransition` | Sync actions only | Supports async functions in `startTransition` |
| `startTransition` | Sync actions only | Supports async functions |
| `useDeferredValue` | `useDeferredValue(value)` | Adds optional `initialValue` parameter |
| `use()` | Not available | New: reads promises and context conditionally |
| Streaming SSR | `renderToPipeableStream` | Same API, improved performance |
| Suspense for data | Framework-dependent | `use(promise)` enables direct promise reading |

### React 19: use() Hook

```tsx
// React 19 ONLY -- read a promise directly
import { use, Suspense } from "react";

function UserProfile({ userPromise }: { userPromise: Promise<User> }): JSX.Element {
  const user = use(userPromise); // Suspends until resolved
  return <h1>{user.name}</h1>;
}

// use() works in conditionals (unlike other hooks)
function MaybeThemed({ showTheme }: { showTheme: boolean }): JSX.Element {
  if (showTheme) {
    const theme = use(ThemeContext); // Allowed in conditionals
    return <div className={theme}>Themed</div>;
  }
  return <div>Default</div>;
}
```

### React 19: useDeferredValue initialValue

```tsx
// React 19 ONLY -- provide initial value for first render
const deferredQuery = useDeferredValue(query, ""); // "" on first render
```

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete concurrent feature code examples
- [references/patterns.md](references/patterns.md) -- Suspense and transition patterns in depth
- [references/anti-patterns.md](references/anti-patterns.md) -- Common concurrent feature mistakes

### Official Sources

- https://react.dev/reference/react/Suspense
- https://react.dev/reference/react/useTransition
- https://react.dev/reference/react/useDeferredValue
- https://react.dev/reference/react/lazy
- https://react.dev/reference/react/use
- https://react.dev/reference/react/startTransition
- https://react.dev/reference/react-dom/server/renderToPipeableStream
