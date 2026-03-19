# react-core-concurrent: Code Examples

## 1. Basic Suspense with Data Fetching

```tsx
import { Suspense } from "react";

interface Album {
  id: string;
  title: string;
  year: number;
}

// Assumes a Suspense-enabled data source (framework or use() in React 19)
function AlbumList({ artistId }: { artistId: string }): JSX.Element {
  const albums = useSuspenseData<Album[]>(`/api/artists/${artistId}/albums`);

  return (
    <ul>
      {albums.map((album) => (
        <li key={album.id}>
          {album.title} ({album.year})
        </li>
      ))}
    </ul>
  );
}

function ArtistPage({ artistId }: { artistId: string }): JSX.Element {
  return (
    <div>
      <h1>Albums</h1>
      <Suspense fallback={<p>Loading albums...</p>}>
        <AlbumList artistId={artistId} />
      </Suspense>
    </div>
  );
}
```

---

## 2. Progressive Loading with Nested Suspense

```tsx
import { Suspense } from "react";

function DashboardPage(): JSX.Element {
  return (
    <Suspense fallback={<FullPageSpinner />}>
      <DashboardHeader />
      <div className="dashboard-grid">
        <Suspense fallback={<CardSkeleton title="Revenue" />}>
          <RevenueCard />
        </Suspense>
        <Suspense fallback={<CardSkeleton title="Users" />}>
          <UsersCard />
        </Suspense>
        <Suspense fallback={<CardSkeleton title="Activity" />}>
          <ActivityFeed />
        </Suspense>
      </div>
    </Suspense>
  );
}

function FullPageSpinner(): JSX.Element {
  return <div className="spinner" aria-label="Loading dashboard" />;
}

function CardSkeleton({ title }: { title: string }): JSX.Element {
  return (
    <div className="card skeleton">
      <h3>{title}</h3>
      <div className="skeleton-bar" />
    </div>
  );
}
```

**Loading sequence:**
1. `FullPageSpinner` shows until `DashboardHeader` resolves
2. Header appears; each card shows its own skeleton independently
3. Cards appear as their data arrives (in any order)

---

## 3. useTransition for Tab Navigation

```tsx
import { useState, useTransition, Suspense } from "react";

type TabId = "posts" | "comments" | "photos";

function TabContainer(): JSX.Element {
  const [tab, setTab] = useState<TabId>("posts");
  const [isPending, startTransition] = useTransition();

  function selectTab(nextTab: TabId): void {
    startTransition(() => {
      setTab(nextTab);
    });
  }

  return (
    <div>
      <nav>
        {(["posts", "comments", "photos"] as const).map((t) => (
          <button
            key={t}
            onClick={() => selectTab(t)}
            className={tab === t ? "active" : ""}
            disabled={isPending && tab !== t}
          >
            {t}
          </button>
        ))}
      </nav>
      <div style={{ opacity: isPending ? 0.6 : 1, transition: "opacity 0.2s" }}>
        <Suspense fallback={<TabSkeleton />}>
          <TabContent tab={tab} />
        </Suspense>
      </div>
    </div>
  );
}

function TabContent({ tab }: { tab: TabId }): JSX.Element {
  switch (tab) {
    case "posts":
      return <PostsTab />;
    case "comments":
      return <CommentsTab />;
    case "photos":
      return <PhotosTab />;
  }
}
```

**Why this works:** `startTransition` tells React the tab update is non-urgent. React keeps showing the current tab (dimmed via `isPending`) instead of immediately showing the Suspense fallback.

---

## 4. useTransition with Async Actions (React 19)

```tsx
import { useState, useTransition } from "react";

interface FormData {
  name: string;
  email: string;
}

function ProfileForm(): JSX.Element {
  const [isPending, startTransition] = useTransition();
  const [error, setError] = useState<string | null>(null);
  const [profile, setProfile] = useState<FormData>({ name: "", email: "" });

  async function handleSubmit(formData: FormData): Promise<void> {
    setError(null);
    startTransition(async () => {
      const result = await saveProfile(formData);
      // IMPORTANT: wrap post-await state updates in startTransition
      startTransition(() => {
        if (result.error) {
          setError(result.error);
        } else {
          setProfile(result.data);
        }
      });
    });
  }

  return (
    <form onSubmit={(e) => { e.preventDefault(); handleSubmit(profile); }}>
      <input
        value={profile.name}
        onChange={(e) => setProfile((p) => ({ ...p, name: e.target.value }))}
      />
      <button type="submit" disabled={isPending}>
        {isPending ? "Saving..." : "Save"}
      </button>
      {error && <p className="error">{error}</p>}
    </form>
  );
}
```

---

## 5. useDeferredValue for Expensive List Filtering

```tsx
import { useState, useDeferredValue, memo } from "react";

interface Product {
  id: string;
  name: string;
  category: string;
}

function ProductSearch({ products }: { products: Product[] }): JSX.Element {
  const [filter, setFilter] = useState<string>("");
  const deferredFilter = useDeferredValue(filter);
  const isStale = filter !== deferredFilter;

  return (
    <div>
      <input
        type="search"
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filter products..."
      />
      <div style={{ opacity: isStale ? 0.5 : 1, transition: "opacity 0.15s" }}>
        <ProductList products={products} filter={deferredFilter} />
      </div>
    </div>
  );
}

// CRITICAL: memo() is required for useDeferredValue to provide any benefit
const ProductList = memo(function ProductList({
  products,
  filter,
}: {
  products: Product[];
  filter: string;
}): JSX.Element {
  const filtered = products.filter((p) =>
    p.name.toLowerCase().includes(filter.toLowerCase())
  );

  return (
    <ul>
      {filtered.map((p) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
});
```

**How it works:**
1. User types in the input -- `filter` updates immediately, input stays responsive
2. `deferredFilter` lags behind, so `ProductList` re-renders with the old value first
3. React schedules a background re-render with the new `deferredFilter`
4. If the user types again before the background render completes, React abandons it and starts fresh

---

## 6. useDeferredValue with initialValue (React 19)

```tsx
import { useDeferredValue, Suspense } from "react";

function SearchPage(): JSX.Element {
  const [query, setQuery] = useState<string>("");
  // On first render, deferredQuery is "" (the initial value)
  // On subsequent renders, it defers to the previous query value
  const deferredQuery = useDeferredValue(query, "");

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Suspense fallback={<p>Loading results...</p>}>
        {deferredQuery && <SearchResults query={deferredQuery} />}
      </Suspense>
    </>
  );
}
```

---

## 7. Code Splitting with React.lazy

```tsx
import { lazy, Suspense, useState, useTransition } from "react";

// ALWAYS declare at module top level
const AdminPanel = lazy(() => import("./AdminPanel"));
const UserDashboard = lazy(() => import("./UserDashboard"));
const Analytics = lazy(() => import("./Analytics"));

type Route = "dashboard" | "admin" | "analytics";

function App(): JSX.Element {
  const [route, setRoute] = useState<Route>("dashboard");
  const [isPending, startTransition] = useTransition();

  function navigate(to: Route): void {
    startTransition(() => {
      setRoute(to);
    });
  }

  return (
    <div>
      <nav>
        <button onClick={() => navigate("dashboard")}>Dashboard</button>
        <button onClick={() => navigate("admin")}>Admin</button>
        <button onClick={() => navigate("analytics")}>Analytics</button>
      </nav>
      {isPending && <div className="loading-bar" />}
      <Suspense fallback={<PageSkeleton />}>
        {route === "dashboard" && <UserDashboard />}
        {route === "admin" && <AdminPanel />}
        {route === "analytics" && <Analytics />}
      </Suspense>
    </div>
  );
}
```

---

## 8. use() Hook for Promise Reading (React 19)

```tsx
import { use, Suspense } from "react";

interface User {
  id: string;
  name: string;
  email: string;
}

// Parent creates the promise and passes it down
function UserPage({ userId }: { userId: string }): JSX.Element {
  // IMPORTANT: create promise outside the suspending component
  const userPromise = fetchUser(userId);

  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

// Child reads the promise with use()
function UserProfile({ userPromise }: { userPromise: Promise<User> }): JSX.Element {
  const user = use(userPromise); // Suspends until resolved

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

---

## 9. use() for Conditional Context (React 19)

```tsx
import { use, createContext } from "react";

interface Theme {
  primary: string;
  background: string;
}

const ThemeContext = createContext<Theme>({
  primary: "#007bff",
  background: "#ffffff",
});

// use() can be called conditionally -- unlike useContext()
function ThemedButton({
  themed,
  children,
}: {
  themed: boolean;
  children: React.ReactNode;
}): JSX.Element {
  if (themed) {
    const theme = use(ThemeContext);
    return (
      <button style={{ backgroundColor: theme.primary, color: "#fff" }}>
        {children}
      </button>
    );
  }
  return <button>{children}</button>;
}
```

---

## 10. Streaming SSR with Suspense

```tsx
// server.ts (Node.js)
import { renderToPipeableStream } from "react-dom/server";
import App from "./App";

function handleRequest(req: Request, res: Response): void {
  const { pipe } = renderToPipeableStream(<App />, {
    bootstrapScripts: ["/client.js"],
    onShellReady() {
      // Shell (content outside Suspense boundaries) is ready
      res.statusCode = 200;
      res.setHeader("Content-Type", "text/html");
      pipe(res);
    },
    onShellError(error: unknown) {
      // Critical error -- shell could not render
      res.statusCode = 500;
      res.send("Server error");
    },
    onError(error: unknown) {
      // Non-critical -- logged but stream continues
      console.error(error);
    },
  });
}
```

**How streaming SSR works with Suspense:**
1. React renders the shell (everything outside `<Suspense>` boundaries) immediately
2. Suspense fallbacks are included as placeholder HTML
3. When suspended content resolves, React streams additional HTML chunks
4. Client-side hydration replaces fallbacks with real content
5. Selective hydration prioritizes interactive regions the user engages with

---

## 11. Error Handling with Suspense and Error Boundaries

```tsx
import { Suspense } from "react";
import { ErrorBoundary } from "react-error-boundary";

function DataSection(): JSX.Element {
  return (
    <ErrorBoundary
      fallback={<p>Something went wrong loading data.</p>}
      onError={(error) => logError(error)}
    >
      <Suspense fallback={<DataSkeleton />}>
        <DataDisplay />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**ALWAYS** wrap Suspense boundaries in Error Boundaries when fetching data. A rejected promise without an Error Boundary crashes the entire component tree.

---

## 12. Suspense Key Reset for Profile Navigation

```tsx
import { Suspense } from "react";

function ProfilePage({ userId }: { userId: string }): JSX.Element {
  return (
    // key forces Suspense boundary to reset when userId changes
    <Suspense key={userId} fallback={<ProfileSkeleton />}>
      <ProfileContent userId={userId} />
      <Suspense fallback={<PostsSkeleton />}>
        <UserPosts userId={userId} />
      </Suspense>
    </Suspense>
  );
}
```

Without `key`, React would try to show stale content from the previous user while loading. With `key`, it shows the fallback immediately for a clean transition.
