# react-impl-routing -- API Reference

## Router Creation Functions

| Function | Signature | Purpose |
|----------|-----------|---------|
| `createBrowserRouter` | `(routes: RouteObject[], opts?: { basename?: string }) => Router` | Create router using History API (production standard) |
| `createHashRouter` | `(routes: RouteObject[], opts?: { basename?: string }) => Router` | Create router using hash URLs (legacy/static hosting) |
| `createMemoryRouter` | `(routes: RouteObject[], opts?: { initialEntries?: string[], initialIndex?: number }) => Router` | Create in-memory router (testing, non-browser) |

---

## Router Components

| Component | Props | Purpose |
|-----------|-------|---------|
| `<RouterProvider>` | `router: Router`, `fallbackElement?: ReactElement` | Mount a data router in React tree |
| `<Outlet>` | `context?: unknown` | Render matched child route element |
| `<Link>` | `to: string \| Partial<Path>`, `replace?: boolean`, `state?: any` | Client-side navigation link |
| `<NavLink>` | Same as Link + `className?: string \| (props: { isActive, isPending }) => string`, `style?: ...`, `end?: boolean` | Navigation link with active state |
| `<Navigate>` | `to: string`, `replace?: boolean`, `state?: any` | Declarative redirect (renders nothing) |
| `<Form>` | `method?: string`, `action?: string`, `encType?: string`, `replace?: boolean` | Form that triggers route action |
| `<ScrollRestoration>` | `getKey?: (location, matches) => string` | Restore scroll position on navigation |
| `<Await>` | `resolve: Promise<T>`, `errorElement?: ReactElement`, `children: (data: T) => ReactElement` | Render deferred data inside Suspense |

---

## Hooks -- Navigation

| Hook | Signature | Purpose |
|------|-----------|---------|
| `useNavigate` | `() => NavigateFunction` | Programmatic navigation |
| `useLocation` | `() => Location` | Current location object `{ pathname, search, hash, state, key }` |
| `useHref` | `(to: string) => string` | Resolve a relative path to absolute href |
| `useResolvedPath` | `(to: string) => Path` | Resolve relative path to `{ pathname, search, hash }` |
| `useNavigation` | `() => Navigation` | Navigation state: `{ state: "idle" \| "loading" \| "submitting", formData, location }` |
| `useRevalidator` | `() => { revalidate(), state }` | Manually trigger loader revalidation |

### NavigateFunction

```tsx
type NavigateFunction = {
  (to: string, options?: { replace?: boolean; state?: any }): void;
  (delta: number): void;  // navigate(-1) = back, navigate(1) = forward
};
```

---

## Hooks -- Route Data

| Hook | Signature | Purpose |
|------|-----------|---------|
| `useParams` | `<T extends Record<string, string \| undefined>>() => T` | Read URL dynamic segments |
| `useSearchParams` | `() => [URLSearchParams, SetURLSearchParams]` | Read/write URL query string |
| `useLoaderData` | `() => unknown` | Access current route loader return value |
| `useActionData` | `() => unknown` | Access current route action return value |
| `useRouteLoaderData` | `(routeId: string) => unknown` | Access loader data from a parent route by ID |
| `useMatches` | `() => UIMatch[]` | All currently matched routes with data and handle |
| `useRouteError` | `() => unknown` | Access error thrown in loader/action/render |
| `useFetcher` | `<T>() => Fetcher<T>` | Call loaders/actions without navigation |
| `useFetchers` | `() => Fetcher[]` | All active fetchers |

### SetURLSearchParams

```tsx
type SetURLSearchParams = (
  nextInit: URLSearchParams | ((prev: URLSearchParams) => URLSearchParams),
  navigateOpts?: { replace?: boolean; state?: any }
) => void;
```

---

## Hooks -- Form Handling

| Hook | Signature | Purpose |
|------|-----------|---------|
| `useSubmit` | `() => SubmitFunction` | Programmatically submit a form |
| `useFormAction` | `(action?: string) => string` | Resolve form action URL |

### SubmitFunction

```tsx
type SubmitFunction = (
  target: FormData | URLSearchParams | Record<string, string> | null,
  options?: { method?: string; action?: string; encType?: string; replace?: boolean }
) => void;
```

---

## RouteObject Properties

| Property | Type | Description |
|----------|------|-------------|
| `path` | `string` | URL path segment. Supports `:param` (dynamic) and `:param?` (optional) |
| `index` | `boolean` | Marks as index route (default child). Mutually exclusive with `children` |
| `element` | `ReactElement` | Component to render when route matches |
| `Component` | `React.ComponentType` | Alternative to element -- pass component reference |
| `errorElement` | `ReactElement` | UI to render when loader/action/render throws |
| `ErrorBoundary` | `React.ComponentType` | Alternative to errorElement -- pass component reference |
| `loader` | `(args: LoaderFunctionArgs) => Promise<T> \| T` | Data loading function |
| `action` | `(args: ActionFunctionArgs) => Promise<T> \| T` | Mutation handling function |
| `lazy` | `() => Promise<Partial<RouteObject>>` | Code-split route module |
| `children` | `RouteObject[]` | Nested child routes |
| `handle` | `unknown` | Custom data accessible via `useMatches()` |
| `shouldRevalidate` | `(args: ShouldRevalidateFunctionArgs) => boolean` | Control when loaders re-run |
| `id` | `string` | Unique route identifier (auto-generated if omitted) |

---

## LoaderFunctionArgs / ActionFunctionArgs

| Property | Type | Description |
|----------|------|-------------|
| `params` | `Record<string, string>` | URL dynamic segment values |
| `request` | `Request` | Standard Fetch API Request object with URL, method, headers |

### Accessing query parameters in a loader:

```tsx
async function loader({ request }: LoaderFunctionArgs): Promise<Data> {
  const url = new URL(request.url);
  const query = url.searchParams.get("q");
  // ...
}
```

### Accessing form data in an action:

```tsx
async function action({ request, params }: ActionFunctionArgs): Promise<Response> {
  const formData = await request.formData();
  const name = formData.get("name") as string;
  // ...
}
```

---

## Utility Functions

| Function | Signature | Purpose |
|----------|-----------|---------|
| `redirect` | `(url: string, init?: number \| ResponseInit) => Response` | Return redirect response from loader/action |
| `json` | `(data: T, init?: ResponseInit) => Response` | Return JSON response (deprecated in v7, use plain return) |
| `defer` | `(data: Record<string, unknown>) => DeferredData` | Return mix of awaited and deferred promises |
| `isRouteErrorResponse` | `(error: unknown) => error is ErrorResponse` | Type guard for Response-based errors |
| `generatePath` | `(path: string, params: Record<string, string>) => string` | Fill dynamic segments in path template |
| `matchPath` | `(pattern: string, pathname: string) => PathMatch \| null` | Test if pathname matches a pattern |
| `matchRoutes` | `(routes: RouteObject[], location: string) => RouteMatch[] \| null` | Match routes against a location |

---

## Fetcher API

`useFetcher()` returns a fetcher that can call loaders/actions without causing navigation:

| Property / Method | Type | Description |
|-------------------|------|-------------|
| `fetcher.load(href)` | `(href: string) => void` | Call a route loader |
| `fetcher.submit(data, opts)` | `SubmitFunction` | Call a route action |
| `fetcher.data` | `T \| undefined` | Last returned data |
| `fetcher.state` | `"idle" \| "loading" \| "submitting"` | Current fetcher state |
| `fetcher.formData` | `FormData \| undefined` | Form data being submitted |
| `fetcher.Form` | `React.ComponentType` | Form component bound to this fetcher |

```tsx
function InlineDelete({ projectId }: { projectId: string }): React.ReactElement {
  const fetcher = useFetcher();
  const isDeleting = fetcher.state !== "idle";

  return (
    <fetcher.Form method="post" action={`/projects/${projectId}`}>
      <input type="hidden" name="intent" value="delete" />
      <button type="submit" disabled={isDeleting}>
        {isDeleting ? "Deleting..." : "Delete"}
      </button>
    </fetcher.Form>
  );
}
```
