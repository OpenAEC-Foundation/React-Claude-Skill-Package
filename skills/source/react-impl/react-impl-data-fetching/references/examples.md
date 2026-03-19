# Data Fetching Examples

## Complete TanStack Query Setup

### Installation

```bash
npm install @tanstack/react-query
# Optional but recommended: DevTools
npm install @tanstack/react-query-devtools
```

### App-Level Provider with DevTools

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// ALWAYS create QueryClient OUTSIDE the component
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,
      gcTime: 1000 * 60 * 30,
      retry: 3,
      refetchOnWindowFocus: true,
    },
    mutations: {
      retry: 1,
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

---

## Basic CRUD with TanStack Query

### Fetch a Single Resource

```typescript
import { useQuery } from '@tanstack/react-query';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

async function fetchProduct(id: number): Promise<Product> {
  const res = await fetch(`/api/products/${id}`);
  if (!res.ok) throw new Error(`Failed to fetch product: ${res.statusText}`);
  return res.json();
}

function ProductDetail({ productId }: { productId: number }) {
  const {
    data: product,
    isLoading,
    isError,
    error,
  } = useQuery<Product>({
    queryKey: ['product', productId],
    queryFn: () => fetchProduct(productId),
    enabled: productId > 0,
  });

  if (isLoading) return <ProductSkeleton />;
  if (isError) return <p>Error: {error.message}</p>;

  return (
    <article>
      <h1>{product.name}</h1>
      <p>${product.price.toFixed(2)}</p>
      <span>{product.category}</span>
    </article>
  );
}
```

### Fetch a List with Filtering

```typescript
interface ProductFilters {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
}

function useProducts(filters: ProductFilters) {
  return useQuery<Product[]>({
    // ALWAYS include filter values in queryKey for correct caching
    queryKey: ['products', filters],
    queryFn: async () => {
      const params = new URLSearchParams();
      if (filters.category) params.set('category', filters.category);
      if (filters.minPrice) params.set('minPrice', String(filters.minPrice));
      if (filters.maxPrice) params.set('maxPrice', String(filters.maxPrice));

      const res = await fetch(`/api/products?${params}`);
      if (!res.ok) throw new Error('Failed to fetch products');
      return res.json();
    },
  });
}

function ProductList() {
  const [filters, setFilters] = useState<ProductFilters>({ category: 'all' });
  const { data: products, isLoading } = useProducts(filters);

  return (
    <div>
      <FilterBar value={filters} onChange={setFilters} />
      {isLoading ? (
        <Skeleton count={6} />
      ) : (
        <ul>
          {products?.map((p) => (
            <li key={p.id}>{p.name} - ${p.price}</li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### Create a Resource

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function useCreateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (newProduct: Omit<Product, 'id'>): Promise<Product> => {
      const res = await fetch('/api/products', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newProduct),
      });
      if (!res.ok) throw new Error('Failed to create product');
      return res.json();
    },
    onSuccess: () => {
      // Invalidate all product queries to refetch with new data
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}

function CreateProductForm() {
  const createProduct = useCreateProduct();

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    createProduct.mutate({
      name: formData.get('name') as string,
      price: Number(formData.get('price')),
      category: formData.get('category') as string,
    });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" placeholder="Product name" required />
      <input name="price" type="number" step="0.01" required />
      <input name="category" placeholder="Category" required />
      <button type="submit" disabled={createProduct.isPending}>
        {createProduct.isPending ? 'Creating...' : 'Create Product'}
      </button>
      {createProduct.isError && <p>Error: {createProduct.error.message}</p>}
      {createProduct.isSuccess && <p>Product created!</p>}
    </form>
  );
}
```

### Update a Resource with Optimistic Update

```typescript
function useUpdateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (product: Product): Promise<Product> => {
      const res = await fetch(`/api/products/${product.id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(product),
      });
      if (!res.ok) throw new Error('Failed to update product');
      return res.json();
    },

    onMutate: async (updatedProduct) => {
      await queryClient.cancelQueries({ queryKey: ['product', updatedProduct.id] });
      const previous = queryClient.getQueryData<Product>(['product', updatedProduct.id]);
      queryClient.setQueryData(['product', updatedProduct.id], updatedProduct);
      return { previous };
    },

    onError: (_err, updatedProduct, context) => {
      if (context?.previous) {
        queryClient.setQueryData(['product', updatedProduct.id], context.previous);
      }
    },

    onSettled: (_data, _error, variables) => {
      queryClient.invalidateQueries({ queryKey: ['product', variables.id] });
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

### Delete a Resource

```typescript
function useDeleteProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (productId: number): Promise<void> => {
      const res = await fetch(`/api/products/${productId}`, { method: 'DELETE' });
      if (!res.ok) throw new Error('Failed to delete product');
    },

    onSuccess: (_data, productId) => {
      // Remove from individual cache
      queryClient.removeQueries({ queryKey: ['product', productId] });
      // Refetch list
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

---

## Suspense-Based Data Fetching

### With TanStack Query (useSuspenseQuery)

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

function UserProfile({ userId }: { userId: number }) {
  // data is ALWAYS defined -- Suspense handles loading, ErrorBoundary handles errors
  const { data: user } = useSuspenseQuery<User>({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

function UserPosts({ userId }: { userId: number }) {
  const { data: posts } = useSuspenseQuery<Post[]>({
    queryKey: ['posts', userId],
    queryFn: () => fetchUserPosts(userId),
  });

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}

// Parallel Suspense: both queries suspend independently
function UserPage({ userId }: { userId: number }) {
  return (
    <ErrorBoundary fallback={<p>Failed to load user data.</p>}>
      <Suspense fallback={<ProfileSkeleton />}>
        <UserProfile userId={userId} />
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <UserPosts userId={userId} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### With React 19 use() Hook

```typescript
import { use, Suspense, cache } from 'react';

// Cache the fetch function so the same promise is returned for the same args
const fetchUser = cache(async (id: number): Promise<User> => {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error('Failed to fetch user');
  return res.json();
});

function UserCard({ userId }: { userId: number }) {
  const user = use(fetchUser(userId));
  return <div>{user.name} ({user.email})</div>;
}

function App() {
  return (
    <Suspense fallback={<p>Loading user...</p>}>
      <UserCard userId={1} />
    </Suspense>
  );
}
```

---

## Infinite Scroll / Load More

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';
import { useInView } from 'react-intersection-observer';
import { useEffect } from 'react';

interface Page {
  items: Product[];
  nextCursor: string | null;
  totalCount: number;
}

function InfiniteProductList() {
  const { ref, inView } = useInView();

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isError,
    error,
  } = useInfiniteQuery<Page>({
    queryKey: ['products', 'infinite'],
    queryFn: async ({ pageParam }): Promise<Page> => {
      const res = await fetch(`/api/products?cursor=${pageParam}&limit=20`);
      if (!res.ok) throw new Error('Failed to fetch products');
      return res.json();
    },
    initialPageParam: '',
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  // Auto-fetch next page when sentinel enters viewport
  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage]);

  if (isLoading) return <Skeleton count={10} />;
  if (isError) return <p>Error: {error.message}</p>;

  const allProducts = data.pages.flatMap((page) => page.items);

  return (
    <div>
      <ul>
        {allProducts.map((product) => (
          <li key={product.id}>{product.name}</li>
        ))}
      </ul>
      {/* Sentinel element for intersection observer */}
      <div ref={ref}>
        {isFetchingNextPage && <Spinner />}
      </div>
    </div>
  );
}
```

---

## Dependent Queries

```typescript
function UserDashboard({ userId }: { userId: number }) {
  // First query: fetch user
  const { data: user } = useQuery<User>({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // Second query: fetch user's projects (depends on user.teamId)
  const { data: projects } = useQuery<Project[]>({
    queryKey: ['projects', user?.teamId],
    queryFn: () => fetchTeamProjects(user!.teamId),
    // ONLY runs when user is loaded and teamId is available
    enabled: !!user?.teamId,
  });

  if (!user) return <Skeleton />;

  return (
    <div>
      <h1>{user.name}</h1>
      {projects ? (
        <ProjectList projects={projects} />
      ) : (
        <p>Loading projects...</p>
      )}
    </div>
  );
}
```

---

## Custom Query Hook Pattern

ALWAYS extract queries into custom hooks for reusability and testability.

```typescript
// hooks/useUser.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilters) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: number) => [...userKeys.details(), id] as const,
};

export function useUser(userId: number) {
  return useQuery<User>({
    queryKey: userKeys.detail(userId),
    queryFn: () => fetchUser(userId),
    enabled: userId > 0,
  });
}

export function useUsers(filters: UserFilters = {}) {
  return useQuery<User[]>({
    queryKey: userKeys.list(filters),
    queryFn: () => fetchUsers(filters),
  });
}

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updateUser,
    onSuccess: (data) => {
      queryClient.setQueryData(userKeys.detail(data.id), data);
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
    },
  });
}
```

---

## Polling and Real-Time Data

```typescript
function LiveDashboard() {
  const { data: metrics } = useQuery<DashboardMetrics>({
    queryKey: ['dashboard', 'metrics'],
    queryFn: fetchDashboardMetrics,
    refetchInterval: 5000,               // Poll every 5 seconds
    refetchIntervalInBackground: false,   // Stop polling when tab is hidden
  });

  return <MetricsDisplay data={metrics} />;
}
```

---

## Prefetching on Route Change

```typescript
import { useQueryClient } from '@tanstack/react-query';

function Navigation() {
  const queryClient = useQueryClient();

  function prefetchUserProfile(userId: number) {
    queryClient.prefetchQuery({
      queryKey: ['user', userId],
      queryFn: () => fetchUser(userId),
      staleTime: 1000 * 60 * 5,
    });
  }

  return (
    <nav>
      {users.map((user) => (
        <Link
          key={user.id}
          to={`/users/${user.id}`}
          onMouseEnter={() => prefetchUserProfile(user.id)}
        >
          {user.name}
        </Link>
      ))}
    </nav>
  );
}
```
