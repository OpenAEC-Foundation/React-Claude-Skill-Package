# react-core-concurrent: Suspense and Transition Patterns

## Pattern 1: Progressive Disclosure Loading

**When to use:** Complex pages with multiple independent data sections.

**Strategy:** Use nested Suspense boundaries so each section loads independently. The outermost boundary covers the page shell; inner boundaries cover individual widgets.

```tsx
<Suspense fallback={<AppShell />}>
  <Navigation />
  <main>
    <Suspense fallback={<HeroSkeleton />}>
      <HeroSection />
    </Suspense>
    <div className="grid">
      <Suspense fallback={<WidgetSkeleton />}>
        <StatsWidget />
      </Suspense>
      <Suspense fallback={<WidgetSkeleton />}>
        <ChartWidget />
      </Suspense>
      <Suspense fallback={<WidgetSkeleton />}>
        <RecentActivity />
      </Suspense>
    </div>
  </main>
</Suspense>
```

**Design rules:**
- ALWAYS coordinate Suspense boundary placement with the design team
- NEVER make boundaries finer than the intended loading experience
- Group components that should appear together inside the SAME boundary
- Use independent boundaries for sections that can load at different speeds

---

## Pattern 2: Transition-Guarded Navigation

**When to use:** Route transitions where you want to keep the current page visible instead of showing a loading spinner.

**Strategy:** Wrap route state updates in `startTransition`. The current view remains visible (optionally dimmed) while the next page loads.

```tsx
import { useState, useTransition, Suspense } from "react";

function Router(): JSX.Element {
  const [currentRoute, setRoute] = useState<string>("/home");
  const [isPending, startTransition] = useTransition();

  function navigate(path: string): void {
    startTransition(() => {
      setRoute(path);
    });
  }

  return (
    <>
      <header style={{ opacity: isPending ? 0.7 : 1 }}>
        <NavBar onNavigate={navigate} currentRoute={currentRoute} />
        {isPending && <ProgressBar />}
      </header>
      <Suspense fallback={<PageLoader />}>
        <RouteContent route={currentRoute} />
      </Suspense>
    </>
  );
}
```

**Key behavior:** When `startTransition` wraps the state update:
1. React does NOT immediately show the Suspense fallback
2. The current content stays visible (stale but complete)
3. `isPending` becomes `true` so you can show a progress indicator
4. Once the new content is ready, React swaps it in atomically

**ALWAYS** use this pattern when navigating between pages that use Suspense for data fetching. Without it, users see a jarring flash of the fallback skeleton.

---

## Pattern 3: Deferred Search with Stale Indication

**When to use:** Search inputs where typing must remain responsive but results are expensive to compute or fetch.

**Strategy:** Use `useDeferredValue` on the search query. The input updates immediately; results re-render in the background with the deferred value.

```tsx
import { useState, useDeferredValue, Suspense, memo } from "react";

function SearchPage(): JSX.Element {
  const [query, setQuery] = useState<string>("");
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <div>
      <input
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <Suspense fallback={<ResultsSkeleton />}>
        <div
          style={{
            opacity: isStale ? 0.5 : 1,
            transition: "opacity 0.2s",
          }}
        >
          <SearchResults query={deferredQuery} />
        </div>
      </Suspense>
    </div>
  );
}

// CRITICAL: must be memoized for useDeferredValue to skip re-renders
const SearchResults = memo(function SearchResults({
  query,
}: {
  query: string;
}): JSX.Element {
  // ...expensive render or data fetch
});
```

**Three-phase render cycle:**
1. **Immediate:** Input updates, `query` changes, `deferredQuery` keeps old value
2. **Background:** React re-renders with new `deferredQuery` (interruptible)
3. **Commit:** New results appear, `isStale` becomes `false`, opacity returns to 1

**ALWAYS** compare `query !== deferredQuery` to detect staleness and dim the UI accordingly.

---

## Pattern 4: Optimistic UI with Transitions

**When to use:** User actions that trigger server requests where you want immediate visual feedback.

**Strategy:** Update UI optimistically, wrap the server call in `startTransition`, and revert on error.

```tsx
import { useState, useTransition, useOptimistic } from "react";

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

function TodoList({ todos }: { todos: Todo[] }): JSX.Element {
  const [isPending, startTransition] = useTransition();
  const [optimisticTodos, setOptimisticTodos] = useOptimistic(todos);

  async function toggleTodo(id: string): Promise<void> {
    startTransition(async () => {
      // Show optimistic update immediately
      setOptimisticTodos((prev) =>
        prev.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t))
      );
      // Server request happens in the background
      await updateTodoOnServer(id);
    });
  }

  return (
    <ul>
      {optimisticTodos.map((todo) => (
        <li
          key={todo.id}
          onClick={() => toggleTodo(todo.id)}
          style={{ opacity: isPending ? 0.8 : 1 }}
        >
          {todo.completed ? "Done" : "Todo"}: {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

---

## Pattern 5: Code Splitting by Route

**When to use:** Single-page applications with multiple routes where you want to load route components on demand.

**Strategy:** Use `React.lazy` for each route component. Combine with `startTransition` for smooth transitions.

```tsx
import { lazy, Suspense, useState, useTransition } from "react";

// ALWAYS at module top level
const Home = lazy(() => import("./pages/Home"));
const About = lazy(() => import("./pages/About"));
const Settings = lazy(() => import("./pages/Settings"));

const routes: Record<string, React.LazyExoticComponent<React.ComponentType>> = {
  "/": Home,
  "/about": About,
  "/settings": Settings,
};

function App(): JSX.Element {
  const [path, setPath] = useState<string>("/");
  const [isPending, startTransition] = useTransition();

  const PageComponent = routes[path] ?? Home;

  return (
    <div>
      <nav>
        {Object.keys(routes).map((route) => (
          <a
            key={route}
            href={route}
            onClick={(e) => {
              e.preventDefault();
              startTransition(() => setPath(route));
            }}
          >
            {route}
          </a>
        ))}
      </nav>
      {isPending && <TopLoadingBar />}
      <Suspense fallback={<PageSkeleton />}>
        <PageComponent />
      </Suspense>
    </div>
  );
}
```

---

## Pattern 6: Server-to-Client Promise Streaming (React 19)

**When to use:** Server Components passing data promises to Client Components.

**Strategy:** Create the promise in the Server Component, pass it as a prop, and read it with `use()` inside a Suspense boundary on the client.

```tsx
// Server Component (e.g., Next.js app router)
export default async function Page({ params }: { params: { id: string } }) {
  // Create promise WITHOUT awaiting -- let client stream it
  const dataPromise = fetchProjectData(params.id);

  return (
    <Suspense fallback={<ProjectSkeleton />}>
      <ProjectDetails dataPromise={dataPromise} />
    </Suspense>
  );
}

// Client Component
"use client";
import { use } from "react";

function ProjectDetails({
  dataPromise,
}: {
  dataPromise: Promise<ProjectData>;
}): JSX.Element {
  const data = use(dataPromise); // Suspends until resolved

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.description}</p>
    </div>
  );
}
```

**Key advantage:** The server starts the data fetch immediately. The client receives a streaming promise -- no client-side waterfall.

---

## Pattern 7: Suspense with Error Recovery

**When to use:** Any data-fetching Suspense boundary where failures are possible.

**Strategy:** ALWAYS wrap data-fetching Suspense boundaries in an Error Boundary. Provide a retry mechanism.

```tsx
import { Suspense, useState } from "react";
import { ErrorBoundary } from "react-error-boundary";

function DataSection(): JSX.Element {
  const [retryKey, setRetryKey] = useState<number>(0);

  return (
    <ErrorBoundary
      key={retryKey}
      fallback={
        <div>
          <p>Failed to load data.</p>
          <button onClick={() => setRetryKey((k) => k + 1)}>Retry</button>
        </div>
      }
    >
      <Suspense fallback={<DataSkeleton />}>
        <DataDisplay />
      </Suspense>
    </ErrorBoundary>
  );
}
```

Changing the `key` on ErrorBoundary resets its state, causing the children to re-mount and re-fetch.

---

## Pattern 8: Selective Hydration Priority

**When to use:** Server-rendered pages with multiple interactive regions.

**Strategy:** Wrap each interactive region in its own Suspense boundary. React hydrates regions based on user interaction priority.

```tsx
// Server-rendered layout
function Page(): JSX.Element {
  return (
    <div>
      <StaticHeader />
      <Suspense fallback={<NavSkeleton />}>
        <InteractiveNav />
      </Suspense>
      <Suspense fallback={<ContentSkeleton />}>
        <InteractiveContent />
      </Suspense>
      <Suspense fallback={<SidebarSkeleton />}>
        <InteractiveSidebar />
      </Suspense>
    </div>
  );
}
```

**Hydration behavior:**
1. React streams HTML for all sections (user sees complete page fast)
2. React starts hydrating sections as JS bundles arrive
3. If user clicks on `InteractiveSidebar` before it hydrates, React prioritizes that region
4. Discrete events (clicks) are replayed after hydration completes

ALWAYS use separate Suspense boundaries for independent interactive regions in SSR to enable selective hydration.
