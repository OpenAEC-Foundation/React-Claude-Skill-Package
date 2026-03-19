# Data Fetching Anti-Patterns

## Anti-Pattern 1: useEffect for Data Fetching Without Cleanup

```typescript
// WRONG: Race condition -- stale responses overwrite fresh ones
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState<Result[]>([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    setIsLoading(true);
    fetchResults(query).then((data) => {
      setResults(data);    // Stale response may arrive AFTER a newer one
      setIsLoading(false);
    });
  }, [query]);

  return <ResultList results={results} />;
}
```

**WHY this fails**: User types "react" quickly. Requests fire for "r", "re", "rea", "reac", "react". Responses arrive out of order. The response for "re" arrives last and overwrites the correct "react" results.

**Additional problems with useEffect fetching**:
- No request deduplication (5 components fetching the same data = 5 requests)
- No caching (navigating away and back refetches everything)
- No background refresh (data goes stale silently)
- No retry logic (one failure = permanent error state)
- No prefetching capability
- Requires manual loading/error state management

```typescript
// CORRECT: Use TanStack Query
function SearchResults({ query }: { query: string }) {
  const { data: results, isLoading } = useQuery({
    queryKey: ['search', query],
    queryFn: () => fetchResults(query),
    enabled: query.length > 0,
  });

  return <ResultList results={results ?? []} />;
}
```

---

## Anti-Pattern 2: Managing Server State with useState

```typescript
// WRONG: Manual server state management
function UserDashboard() {
  const [users, setUsers] = useState<User[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;
    setIsLoading(true);
    setError(null);

    fetch('/api/users')
      .then((res) => {
        if (!res.ok) throw new Error('Failed to fetch');
        return res.json();
      })
      .then((data) => {
        if (!cancelled) {
          setUsers(data);
          setIsLoading(false);
        }
      })
      .catch((err) => {
        if (!cancelled) {
          setError(err.message);
          setIsLoading(false);
        }
      });

    return () => { cancelled = true; };
  }, []);

  // ... 20+ lines just for loading/error/data handling
}
```

**WHY this fails**:
- 15+ lines of boilerplate for EVERY data fetch
- Data is stale after first load (no background refresh)
- No cache sharing between components (each component fetches independently)
- Manual cancellation logic is error-prone
- No retry mechanism
- Loading state resets on every refetch

```typescript
// CORRECT: TanStack Query handles ALL of this
function UserDashboard() {
  const { data: users, isLoading, isError, error } = useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const res = await fetch('/api/users');
      if (!res.ok) throw new Error('Failed to fetch');
      return res.json();
    },
  });

  if (isLoading) return <Skeleton />;
  if (isError) return <p>Error: {error.message}</p>;
  return <UserList users={users} />;
}
```

---

## Anti-Pattern 3: Creating Promises in Render (React 19 use())

```typescript
// WRONG: New promise created every render = infinite loop
function UserProfile({ userId }: { userId: number }) {
  // This creates a NEW promise on every render
  const userPromise = fetch(`/api/users/${userId}`).then((r) => r.json());
  const user = use(userPromise);  // Suspends, re-renders, creates new promise, suspends again...
  return <div>{user.name}</div>;
}
```

**WHY this fails**: `use()` suspends the component. When it resumes, the component re-renders. The re-render creates a new promise (different reference). React sees a new promise and suspends again. Infinite loop.

```typescript
// CORRECT: Cache the promise outside render
import { cache } from 'react';

const fetchUser = cache(async (id: number): Promise<User> => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

function UserProfile({ userId }: { userId: number }) {
  const user = use(fetchUser(userId));  // Same promise reference on re-render
  return <div>{user.name}</div>;
}
```

---

## Anti-Pattern 4: QueryClient Inside Component

```typescript
// WRONG: New QueryClient every render, cache destroyed
function App() {
  const queryClient = new QueryClient();  // Created on EVERY render

  return (
    <QueryClientProvider client={queryClient}>
      <Router />
    </QueryClientProvider>
  );
}
```

**WHY this fails**: Every render creates a new `QueryClient`, which means a new empty cache. All queries refetch. All cached data is lost. Performance is terrible.

```typescript
// CORRECT: Create QueryClient outside the component
const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router />
    </QueryClientProvider>
  );
}

// ALSO CORRECT: useState for lazy initialization (useful for SSR)
function App() {
  const [queryClient] = useState(() => new QueryClient());

  return (
    <QueryClientProvider client={queryClient}>
      <Router />
    </QueryClientProvider>
  );
}
```

---

## Anti-Pattern 5: Invalidating Before Mutation Completes

```typescript
// WRONG: Invalidation races with the mutation
const mutation = useMutation({
  mutationFn: updateUser,
  onMutate: () => {
    // Invalidating HERE runs BEFORE the server processes the update
    queryClient.invalidateQueries({ queryKey: ['users'] });
  },
});
```

**WHY this fails**: `onMutate` fires BEFORE the mutation request is sent. The refetch triggered by invalidation returns the OLD data because the server has not processed the update yet.

```typescript
// CORRECT: Invalidate in onSuccess or onSettled
const mutation = useMutation({
  mutationFn: updateUser,
  onSuccess: () => {
    // Server has processed the update -- safe to refetch
    queryClient.invalidateQueries({ queryKey: ['users'] });
  },
});
```

---

## Anti-Pattern 6: Missing Variables in queryKey

```typescript
// WRONG: queryKey does not include the filter
function ProductList({ category }: { category: string }) {
  const { data } = useQuery({
    queryKey: ['products'],  // Same key for ALL categories
    queryFn: () => fetchProducts(category),
  });
  // Switching category does NOT trigger a refetch -- cache returns wrong data
}
```

**WHY this fails**: TanStack Query uses the `queryKey` to determine cache identity. If the key does not change, the query does not refetch. Switching from "electronics" to "clothing" returns the cached "electronics" data.

```typescript
// CORRECT: Include ALL variables in queryKey
function ProductList({ category }: { category: string }) {
  const { data } = useQuery({
    queryKey: ['products', category],  // Different key per category
    queryFn: () => fetchProducts(category),
  });
}
```

---

## Anti-Pattern 7: Fetching in Event Handlers Without Mutation

```typescript
// WRONG: Manual fetch in event handler, bypassing cache
async function handleDelete(userId: number) {
  await fetch(`/api/users/${userId}`, { method: 'DELETE' });
  // Cache is now stale but TanStack Query does not know
  // UI still shows the deleted user
  window.location.reload();  // Desperate reload to fix stale UI
}
```

```typescript
// CORRECT: Use useMutation for all write operations
const deleteMutation = useMutation({
  mutationFn: (userId: number) =>
    fetch(`/api/users/${userId}`, { method: 'DELETE' }),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['users'] });
  },
});

function handleDelete(userId: number) {
  deleteMutation.mutate(userId);
}
```

---

## Anti-Pattern 8: Not Handling Loading and Error States

```typescript
// WRONG: Assumes data is always available
function UserProfile({ userId }: { userId: number }) {
  const { data } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // TypeError: Cannot read property 'name' of undefined
  return <h1>{data.name}</h1>;
}
```

```typescript
// CORRECT: Handle all states
function UserProfile({ userId }: { userId: number }) {
  const { data, isLoading, isError, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  if (isLoading) return <Skeleton />;
  if (isError) return <ErrorMessage error={error} />;

  return <h1>{data.name}</h1>;
}

// ALSO CORRECT: Use useSuspenseQuery (data is always defined)
function UserProfile({ userId }: { userId: number }) {
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // data is ALWAYS defined -- Suspense handles loading, ErrorBoundary handles errors
  return <h1>{data.name}</h1>;
}
```

---

## Anti-Pattern 9: Waterfall Requests

```typescript
// WRONG: Sequential fetching creates a waterfall
function Dashboard() {
  const { data: user } = useQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
  });

  // This does not start until user is loaded -- even though they are independent
  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    enabled: !!user,  // Unnecessary dependency
  });

  const { data: notifications } = useQuery({
    queryKey: ['notifications'],
    queryFn: fetchNotifications,
    enabled: !!posts,  // Unnecessary dependency
  });
}
```

**WHY this fails**: Each query waits for the previous one. Total time = fetch1 + fetch2 + fetch3. These queries are independent and should run in parallel.

```typescript
// CORRECT: Independent queries fire in parallel
function Dashboard() {
  const userQuery = useQuery({ queryKey: ['user'], queryFn: fetchUser });
  const postsQuery = useQuery({ queryKey: ['posts'], queryFn: fetchPosts });
  const notifQuery = useQuery({ queryKey: ['notifications'], queryFn: fetchNotifications });

  // Total time = max(fetch1, fetch2, fetch3) instead of sum
}

// Use `enabled` ONLY when a query truly depends on another query's result
```

---

## Anti-Pattern 10: Forgetting AbortSignal for Cancellation

```typescript
// WRONG: Request continues even after component unmount or key change
const { data } = useQuery({
  queryKey: ['search', query],
  queryFn: async () => {
    const res = await fetch(`/api/search?q=${query}`);
    return res.json();
  },
});
```

```typescript
// CORRECT: Pass signal for automatic cancellation
const { data } = useQuery({
  queryKey: ['search', query],
  queryFn: async ({ signal }) => {
    const res = await fetch(`/api/search?q=${query}`, { signal });
    if (!res.ok) throw new Error('Search failed');
    return res.json();
  },
});
```

**WHY this matters**: Without the signal, changing the search term fires a new request but the old one continues running. The old response may arrive after the new one and TanStack Query has no way to cancel it. With `signal`, the browser aborts the previous request automatically.

---

## Summary Table

| Anti-Pattern | Problem | Solution |
|-------------|---------|----------|
| useEffect for fetching | Race conditions, no cache, no dedup | TanStack Query `useQuery` |
| useState for server state | Stale data, no background refresh | TanStack Query `useQuery` |
| Promise created in render | Infinite loop with `use()` | `cache()` or module-scope promise |
| QueryClient inside component | Cache destroyed every render | Create outside component |
| Invalidate before mutation | Refetch returns old data | Invalidate in `onSuccess` |
| Missing variables in queryKey | Wrong cached data returned | Include ALL variables in key |
| Manual fetch for writes | Cache goes stale, UI inconsistent | `useMutation` + invalidation |
| No loading/error handling | Runtime errors on undefined data | Check `isLoading`/`isError` or use Suspense |
| Waterfall requests | Slow page loads | Parallel independent queries |
| Missing AbortSignal | Wasted requests, potential stale data | Pass `signal` to fetch |
