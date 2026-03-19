# Data Fetching Patterns — Caching and Error Handling

## Caching Strategies

### Understanding staleTime and gcTime

```
Timeline of a query lifecycle:

[Mount]──fresh──[staleTime]──stale──[Unmount]──inactive──[gcTime]──[Garbage Collected]
                                                   │
                                        Background refetch on:
                                        - Window focus
                                        - Network reconnect
                                        - refetchInterval
                                        - Manual invalidation
```

| Setting | What It Controls | Default | Effect |
|---------|-----------------|---------|--------|
| `staleTime: 0` | Data is immediately stale | Yes | Refetches on every mount, focus, reconnect |
| `staleTime: Infinity` | Data never goes stale | No | NEVER refetches automatically |
| `staleTime: 60_000` | Fresh for 1 minute | No | No refetch within 1 minute of last fetch |
| `gcTime: 300_000` | Cache lives 5 min after unmount | Yes | Inactive cache removed after 5 minutes |
| `gcTime: 0` | Cache removed immediately on unmount | No | No cache benefit between navigations |
| `gcTime: Infinity` | Cache lives forever | No | Memory grows unbounded -- use with caution |

### Setting Defaults per Query Type

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // 5 min default
      gcTime: 1000 * 60 * 30,    // 30 min default
    },
  },
});

// Override per query for different data volatility
const { data: config } = useQuery({
  queryKey: ['app-config'],
  queryFn: fetchAppConfig,
  staleTime: Infinity,  // Config rarely changes
});

const { data: notifications } = useQuery({
  queryKey: ['notifications'],
  queryFn: fetchNotifications,
  staleTime: 1000 * 30,  // 30 seconds -- notifications change frequently
});
```

---

## Query Key Strategies

### Hierarchical Keys

ALWAYS structure query keys hierarchically so invalidation works at the right granularity.

```typescript
// Key factory pattern (RECOMMENDED)
const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: TodoFilters) => [...todoKeys.lists(), filters] as const,
  details: () => [...todoKeys.all, 'detail'] as const,
  detail: (id: number) => [...todoKeys.details(), id] as const,
};

// Usage
useQuery({ queryKey: todoKeys.detail(5), ... });
useQuery({ queryKey: todoKeys.list({ status: 'done' }), ... });

// Invalidation at different levels
queryClient.invalidateQueries({ queryKey: todoKeys.all });           // ALL todo queries
queryClient.invalidateQueries({ queryKey: todoKeys.lists() });       // All todo lists
queryClient.invalidateQueries({ queryKey: todoKeys.detail(5) });     // Single todo
```

### Key Matching Rules

TanStack Query uses **prefix matching** for invalidation:

| queryKey | `invalidateQueries({ queryKey: ['todos'] })` matches? |
|----------|------------------------------------------------------|
| `['todos']` | Yes |
| `['todos', 'list']` | Yes |
| `['todos', 'detail', 5]` | Yes |
| `['users']` | No |

---

## Error Handling Patterns

### Pattern 1: Per-Query Error Handling

```typescript
function UserProfile({ userId }: { userId: number }) {
  const { data, isError, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    retry: (failureCount, error) => {
      // Do NOT retry 404s
      if (error instanceof Response && error.status === 404) return false;
      return failureCount < 3;
    },
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
  });

  if (isError) {
    return <InlineError message={error.message} />;
  }

  return <div>{data?.name}</div>;
}
```

### Pattern 2: Global Error Handler

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 3,
    },
  },
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Global error handler for all queries
      if (error instanceof Response && error.status === 401) {
        redirectToLogin();
      }
      console.error(`Query ${query.queryKey} failed:`, error);
    },
  }),
  mutationCache: new MutationCache({
    onError: (error) => {
      toast.error(`Operation failed: ${error.message}`);
    },
  }),
});
```

### Pattern 3: Error Boundaries with Query Reset

```typescript
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

function PageWithErrorRecovery() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallbackRender={({ error, resetErrorBoundary }) => (
            <div role="alert">
              <h2>Something went wrong</h2>
              <pre>{error.message}</pre>
              <button onClick={resetErrorBoundary}>Try again</button>
            </div>
          )}
        >
          <Suspense fallback={<PageSkeleton />}>
            <PageContent />
          </Suspense>
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

### Pattern 4: Typed Error Handling

```typescript
class ApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public code: string,
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

async function fetchWithErrorHandling<T>(url: string): Promise<T> {
  const res = await fetch(url);
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new ApiError(
      body.message || res.statusText,
      res.status,
      body.code || 'UNKNOWN',
    );
  }
  return res.json();
}

// Usage with typed error
const { error } = useQuery<User, ApiError>({
  queryKey: ['user', userId],
  queryFn: () => fetchWithErrorHandling<User>(`/api/users/${userId}`),
});

if (error) {
  switch (error.status) {
    case 404: return <NotFound />;
    case 403: return <Forbidden />;
    default: return <GenericError message={error.message} />;
  }
}
```

---

## Loading State Patterns

### Pattern 1: Skeleton Loading (Non-Suspense)

```typescript
function UserCard({ userId }: { userId: number }) {
  const { data, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  if (isLoading) return <UserCardSkeleton />;

  return (
    <div>
      <h2>{data!.name}</h2>
      <p>{data!.email}</p>
    </div>
  );
}
```

### Pattern 2: Previous Data While Loading (placeholderData)

```typescript
import { keepPreviousData } from '@tanstack/react-query';

function ProductList({ page }: { page: number }) {
  const { data, isPlaceholderData } = useQuery({
    queryKey: ['products', page],
    queryFn: () => fetchProducts(page),
    placeholderData: keepPreviousData,  // Show previous page while loading next
  });

  return (
    <div style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
      {data?.map((p) => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}
```

### Pattern 3: Background Refetch Indicator

```typescript
function DataList() {
  const { data, isFetching, isLoading } = useQuery({
    queryKey: ['items'],
    queryFn: fetchItems,
  });

  return (
    <div>
      {/* Show spinner only on first load */}
      {isLoading && <FullPageSpinner />}

      {/* Show subtle indicator for background refetch */}
      {isFetching && !isLoading && <RefetchIndicator />}

      {data?.map((item) => <ItemRow key={item.id} item={item} />)}
    </div>
  );
}
```

---

## Parallel and Sequential Queries

### Parallel Queries

```typescript
function Dashboard() {
  // These three queries fire simultaneously
  const usersQuery = useQuery({ queryKey: ['users'], queryFn: fetchUsers });
  const postsQuery = useQuery({ queryKey: ['posts'], queryFn: fetchPosts });
  const statsQuery = useQuery({ queryKey: ['stats'], queryFn: fetchStats });

  const isLoading = usersQuery.isLoading || postsQuery.isLoading || statsQuery.isLoading;
  if (isLoading) return <DashboardSkeleton />;

  return (
    <div>
      <UserWidget users={usersQuery.data!} />
      <PostWidget posts={postsQuery.data!} />
      <StatsWidget stats={statsQuery.data!} />
    </div>
  );
}
```

### Dynamic Parallel Queries with useQueries

```typescript
import { useQueries } from '@tanstack/react-query';

function UserAvatars({ userIds }: { userIds: number[] }) {
  const queries = useQueries({
    queries: userIds.map((id) => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
      staleTime: 1000 * 60 * 5,
    })),
  });

  const isLoading = queries.some((q) => q.isLoading);
  if (isLoading) return <AvatarSkeleton count={userIds.length} />;

  return (
    <div>
      {queries.map((q, i) => (
        <Avatar key={userIds[i]} user={q.data!} />
      ))}
    </div>
  );
}
```

---

## Window Focus and Network Refetching

### Default Behavior

TanStack Query automatically refetches stale queries when:
- The browser window regains focus (`refetchOnWindowFocus: true`)
- The network reconnects (`refetchOnReconnect: true`)
- A component using the query mounts (`refetchOnMount: true`)

### Customizing Refetch Behavior

```typescript
const { data } = useQuery({
  queryKey: ['dashboard'],
  queryFn: fetchDashboard,
  refetchOnWindowFocus: 'always',       // Refetch even if not stale
  refetchOnReconnect: true,             // Refetch on network restore
  refetchOnMount: false,                // Do NOT refetch when component mounts
  refetchInterval: 30_000,              // Poll every 30 seconds
  refetchIntervalInBackground: false,   // Stop polling when tab not visible
});
```

---

## Query Cancellation

```typescript
const { data } = useQuery({
  queryKey: ['search', searchTerm],
  queryFn: async ({ signal }) => {
    // Pass AbortSignal to fetch for automatic cancellation
    const res = await fetch(`/api/search?q=${searchTerm}`, { signal });
    if (!res.ok) throw new Error('Search failed');
    return res.json();
  },
});
```

TanStack Query automatically cancels in-flight queries when:
- The component unmounts
- The query key changes (new search triggers cancellation of previous)
- Manual cancellation via `queryClient.cancelQueries()`

ALWAYS pass the `signal` parameter from `queryFn` context to your fetch calls to enable automatic cancellation.
