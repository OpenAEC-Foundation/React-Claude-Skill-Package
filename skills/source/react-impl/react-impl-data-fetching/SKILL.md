---
name: react-impl-data-fetching
description: "Guides React data fetching patterns including TanStack Query for server state management, useQuery and useMutation patterns, Suspense for loading states, React 19 use() hook for promises, error boundaries for fetch errors, caching strategies, optimistic updates, and pagination patterns. Activates when fetching data from APIs, managing server state, implementing loading/error states, or choosing a data fetching strategy."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-impl-data-fetching

## Quick Reference

### Data Fetching Strategy Decision Tree

```
Need to fetch data from an API?
├── Server state (remote data, shared, async)?
│   ├── YES → Use TanStack Query (RECOMMENDED)
│   │   ├── Read data → useQuery / useSuspenseQuery
│   │   ├── Write data → useMutation + invalidateQueries
│   │   └── Paginated → useInfiniteQuery
│   └── Using React Router? → Loader functions (route-level)
├── React 19 with Suspense architecture?
│   └── use() hook for reading cached promises
└── Simple one-off fetch (rare)?
    └── useEffect with cleanup (LAST RESORT — see anti-patterns)
```

### TanStack Query Setup

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // 5 minutes before data is considered stale
      gcTime: 1000 * 60 * 30,    // 30 minutes before inactive data is garbage collected
      retry: 3,                   // Retry failed requests 3 times
      refetchOnWindowFocus: true, // Refetch when user returns to tab
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router />
    </QueryClientProvider>
  );
}
```

### Critical Warnings

**NEVER** fetch data in `useEffect` without a cleanup flag -- race conditions cause stale responses to overwrite fresh ones. Use TanStack Query instead.

**NEVER** manage server state with `useState` + `useEffect` -- you lose caching, deduplication, background refresh, and error retry for free.

**NEVER** create a new promise inside a component's render body when using React 19 `use()` -- the promise is recreated every render, causing infinite loops. ALWAYS cache the promise outside render.

**NEVER** call `queryClient.invalidateQueries()` without awaiting mutation completion -- invalidation before the server processes the mutation returns stale data.

**ALWAYS** wrap your app in `QueryClientProvider` with a `QueryClient` instance created OUTSIDE the component -- creating it inside causes a new client every render, destroying all cache.

**ALWAYS** use array-based `queryKey` values -- TanStack Query uses structural sharing for cache matching. Include all variables the query depends on.

---

## useQuery — Reading Server Data

```typescript
import { useQuery } from '@tanstack/react-query';

interface User {
  id: number;
  name: string;
  email: string;
}

function UserProfile({ userId }: { userId: number }) {
  const { data, isLoading, isError, error, isFetching } = useQuery<User>({
    queryKey: ['user', userId],           // Cache key (MUST include all variables)
    queryFn: () => fetchUser(userId),     // Fetch function (MUST return a promise)
    enabled: userId > 0,                  // Only fetch when condition is true
    staleTime: 1000 * 60 * 5,            // Data fresh for 5 minutes
    gcTime: 1000 * 60 * 30,              // Keep in cache 30 minutes after unmount
    select: (data) => data.name,          // Transform response (only re-renders on change)
    placeholderData: { id: 0, name: 'Loading...', email: '' },
  });

  if (isLoading) return <Skeleton />;
  if (isError) return <ErrorMessage error={error} />;
  return <div>{data.name}</div>;
}
```

### Key useQuery Options

| Option | Type | Purpose |
|--------|------|---------|
| `queryKey` | `unknown[]` | Unique cache key -- include ALL dependent variables |
| `queryFn` | `() => Promise<T>` | Function that fetches data |
| `enabled` | `boolean` | Disable query until condition is met |
| `staleTime` | `number` | Milliseconds before data is considered stale |
| `gcTime` | `number` | Milliseconds before inactive cache is garbage collected |
| `select` | `(data: T) => U` | Transform or select from cached data |
| `placeholderData` | `T \| (prev) => T` | Show while loading (no Suspense trigger) |
| `retry` | `number \| boolean` | Number of retry attempts on failure |
| `refetchInterval` | `number` | Poll interval in milliseconds |

---

## useMutation — Writing Server Data

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreateUserForm() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newUser: { name: string; email: string }) =>
      fetch('/api/users', {
        method: 'POST',
        body: JSON.stringify(newUser),
        headers: { 'Content-Type': 'application/json' },
      }).then((res) => res.json()),

    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
    onError: (error) => {
      console.error('Failed to create user:', error);
    },
    onSettled: () => {
      // Runs after success OR error -- use for cleanup
    },
  });

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    mutation.mutate({
      name: formData.get('name') as string,
      email: formData.get('email') as string,
    });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" required />
      <input name="email" type="email" required />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create User'}
      </button>
      {mutation.isError && <p>Error: {mutation.error.message}</p>}
    </form>
  );
}
```

---

## Suspense Integration

### useSuspenseQuery (TanStack Query v5+)

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

function UserList() {
  const { data } = useSuspenseQuery<User[]>({
    queryKey: ['users'],
    queryFn: fetchUsers,
  });

  // data is ALWAYS defined -- loading/error handled by Suspense/ErrorBoundary
  return (
    <ul>
      {data.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}

// Parent handles loading and error states declaratively
function UsersPage() {
  return (
    <ErrorBoundary fallback={<p>Failed to load users.</p>}>
      <Suspense fallback={<Skeleton />}>
        <UserList />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**ALWAYS** wrap `useSuspenseQuery` components in both `<Suspense>` and `<ErrorBoundary>` -- `useSuspenseQuery` throws promises (for Suspense) and errors (for ErrorBoundary).

### React 19 use() Hook

```typescript
import { use, Suspense, cache } from 'react';

// CORRECT: Cache the promise OUTSIDE render
const fetchUser = cache(async (id: number): Promise<User> => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

function UserProfile({ userId }: { userId: number }) {
  const user = use(fetchUser(userId));  // Suspends until resolved
  return <div>{user.name}</div>;
}

function App() {
  return (
    <Suspense fallback={<Skeleton />}>
      <UserProfile userId={1} />
    </Suspense>
  );
}
```

**React 18**: `use()` is NOT available. Use `useSuspenseQuery` from TanStack Query for Suspense-based data fetching.

**React 19**: `use()` can read promises and context. ALWAYS ensure the promise is cached (via `cache()`, `useMemo`, or module scope) to prevent re-creation on every render.

---

## Caching Strategy

### staleTime vs gcTime

| Setting | Controls | Default | Recommendation |
|---------|----------|---------|----------------|
| `staleTime` | How long data is "fresh" (no refetch) | `0` (always stale) | Set per query based on data volatility |
| `gcTime` | How long inactive cache is kept in memory | `5 min` | ALWAYS >= staleTime |

### Query Invalidation

```typescript
const queryClient = useQueryClient();

// Invalidate a specific query
queryClient.invalidateQueries({ queryKey: ['user', userId] });

// Invalidate all queries starting with 'users'
queryClient.invalidateQueries({ queryKey: ['users'] });

// Invalidate everything
queryClient.invalidateQueries();
```

### Prefetching

```typescript
// Prefetch on hover for instant navigation
function UserLink({ userId }: { userId: number }) {
  const queryClient = useQueryClient();

  function handleMouseEnter() {
    queryClient.prefetchQuery({
      queryKey: ['user', userId],
      queryFn: () => fetchUser(userId),
      staleTime: 1000 * 60 * 5,
    });
  }

  return (
    <Link to={`/users/${userId}`} onMouseEnter={handleMouseEnter}>
      View Profile
    </Link>
  );
}
```

---

## Optimistic Updates

```typescript
const queryClient = useQueryClient();

const updateUser = useMutation({
  mutationFn: (updatedUser: User) =>
    fetch(`/api/users/${updatedUser.id}`, {
      method: 'PUT',
      body: JSON.stringify(updatedUser),
      headers: { 'Content-Type': 'application/json' },
    }).then((res) => res.json()),

  onMutate: async (newUser) => {
    // Cancel outgoing refetches to avoid overwriting optimistic update
    await queryClient.cancelQueries({ queryKey: ['user', newUser.id] });

    // Snapshot previous value for rollback
    const previousUser = queryClient.getQueryData<User>(['user', newUser.id]);

    // Optimistically update cache
    queryClient.setQueryData(['user', newUser.id], newUser);

    return { previousUser };
  },

  onError: (_err, newUser, context) => {
    // Rollback on error
    if (context?.previousUser) {
      queryClient.setQueryData(['user', newUser.id], context.previousUser);
    }
  },

  onSettled: (_data, _error, variables) => {
    // ALWAYS refetch after mutation to ensure server truth
    queryClient.invalidateQueries({ queryKey: ['user', variables.id] });
  },
});
```

---

## Pagination with useInfiniteQuery

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

interface PaginatedResponse {
  items: User[];
  nextCursor: string | null;
}

function UserListPaginated() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery<PaginatedResponse>({
    queryKey: ['users'],
    queryFn: ({ pageParam }) =>
      fetch(`/api/users?cursor=${pageParam}`).then((res) => res.json()),
    initialPageParam: '',
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  return (
    <div>
      {data?.pages.map((page, i) => (
        <div key={i}>
          {page.items.map((user) => (
            <UserCard key={user.id} user={user} />
          ))}
        </div>
      ))}
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading...' : hasNextPage ? 'Load More' : 'No more users'}
      </button>
    </div>
  );
}
```

---

## Error Handling for Data Fetching

### Query-Level Error Handling

```typescript
const { data, error, isError } = useQuery({
  queryKey: ['user', userId],
  queryFn: fetchUser,
  retry: 2,                          // Retry twice before surfacing error
  retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
});

if (isError) {
  return <p>Error: {error.message}</p>;
}
```

### Declarative Error Boundaries

```typescript
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

function DataSection() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallbackRender={({ resetErrorBoundary }) => (
            <div>
              <p>Something went wrong.</p>
              <button onClick={resetErrorBoundary}>Try again</button>
            </div>
          )}
        >
          <Suspense fallback={<Skeleton />}>
            <UserList />
          </Suspense>
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete data fetching patterns with TanStack Query and Suspense
- [references/patterns.md](references/patterns.md) -- Caching strategies, error handling, and advanced query patterns
- [references/anti-patterns.md](references/anti-patterns.md) -- Data fetching mistakes and why they fail

### Official Sources

- https://tanstack.com/query/latest/docs/framework/react/overview
- https://tanstack.com/query/latest/docs/framework/react/reference/useQuery
- https://tanstack.com/query/latest/docs/framework/react/reference/useMutation
- https://tanstack.com/query/latest/docs/framework/react/reference/useInfiniteQuery
- https://react.dev/reference/react/use
- https://react.dev/reference/react/Suspense
- https://react.dev/learn/you-might-not-need-an-effect
