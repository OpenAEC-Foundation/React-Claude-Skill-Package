# react-impl-routing -- Anti-Patterns

## AP-01: Using Legacy BrowserRouter for New Projects

**NEVER** use `<BrowserRouter>` with `<Routes>` and `<Route>` in new projects.

```tsx
// WRONG -- legacy pattern, no loader/action support
import { BrowserRouter, Routes, Route } from "react-router-dom";

function App(): React.ReactElement {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </BrowserRouter>
  );
}
```

```tsx
// CORRECT -- data router with full feature support
import { createBrowserRouter, RouterProvider } from "react-router-dom";

const router = createBrowserRouter([
  { path: "/", element: <Home /> },
  { path: "/about", element: <About /> },
]);

function App(): React.ReactElement {
  return <RouterProvider router={router} />;
}
```

**Why**: The legacy API cannot use `loader`, `action`, `lazy`, `defer`, `useFetcher`, or `useNavigation`. These are the core features that make React Router v6.4+ powerful.

---

## AP-02: Defining Router Inside a Component

**NEVER** create the router object inside a React component.

```tsx
// WRONG -- router recreated every render, destroys all state
function App(): React.ReactElement {
  const router = createBrowserRouter([
    { path: "/", element: <Home /> },
  ]);
  return <RouterProvider router={router} />;
}
```

```tsx
// CORRECT -- router defined at module scope
const router = createBrowserRouter([
  { path: "/", element: <Home /> },
]);

function App(): React.ReactElement {
  return <RouterProvider router={router} />;
}
```

**Why**: `createBrowserRouter` creates a stateful router instance. Recreating it on every render resets all navigation state, loader caches, and pending navigations.

---

## AP-03: Calling navigate() During Render

**NEVER** call `navigate()` in the component body or during render.

```tsx
// WRONG -- causes infinite re-render loop
function Dashboard(): React.ReactElement {
  const navigate = useNavigate();
  const { user } = useAuth();

  if (!user) {
    navigate("/login"); // called during render!
  }

  return <div>Dashboard</div>;
}
```

```tsx
// CORRECT -- use Navigate component for render-time redirects
function Dashboard(): React.ReactElement {
  const { user } = useAuth();

  if (!user) {
    return <Navigate to="/login" replace />;
  }

  return <div>Dashboard</div>;
}

// ALSO CORRECT -- use useEffect for side-effect navigation
function Dashboard(): React.ReactElement {
  const navigate = useNavigate();
  const { user } = useAuth();

  useEffect(() => {
    if (!user) navigate("/login", { replace: true });
  }, [user, navigate]);

  return <div>Dashboard</div>;
}

// BEST -- use loader-based redirect (no component render needed)
function dashboardLoader(): null | Response {
  if (!checkAuth()) return redirect("/login");
  return null;
}
```

**Why**: `navigate()` triggers a state update. Calling it during render causes React to re-render, which calls navigate again, creating an infinite loop.

---

## AP-04: Using useEffect for Data Fetching Instead of Loaders

**NEVER** fetch route data in `useEffect` when a loader is available.

```tsx
// WRONG -- waterfall: render component, then fetch, then render again
function ProjectDetail(): React.ReactElement {
  const { id } = useParams<{ id: string }>();
  const [project, setProject] = useState<Project | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/projects/${id}`)
      .then((r) => r.json())
      .then(setProject)
      .finally(() => setLoading(false));
  }, [id]);

  if (loading) return <Spinner />;
  if (!project) return <NotFound />;
  return <div>{project.name}</div>;
}
```

```tsx
// CORRECT -- loader fetches before render, no loading state needed
async function loader({ params }: LoaderFunctionArgs): Promise<Project> {
  const response = await fetch(`/api/projects/${params.id}`);
  if (!response.ok) throw new Response("Not found", { status: 404 });
  return response.json();
}

function ProjectDetail(): React.ReactElement {
  const project = useLoaderData() as Project;
  return <div>{project.name}</div>;
}
```

**Why**: Loaders run in parallel with route transitions, eliminating render-fetch waterfalls. The component receives data already loaded, removing the need for loading states and null checks.

---

## AP-05: Using Native form Instead of Router Form

**NEVER** use `<form>` when you want the router to handle submission.

```tsx
// WRONG -- bypasses router, triggers full page navigation
function CreateProject(): React.ReactElement {
  return (
    <form method="post" action="/projects/new">
      <input name="name" />
      <button type="submit">Create</button>
    </form>
  );
}
```

```tsx
// CORRECT -- router intercepts, calls route action, stays client-side
import { Form } from "react-router-dom";

function CreateProject(): React.ReactElement {
  return (
    <Form method="post">
      <input name="name" />
      <button type="submit">Create</button>
    </Form>
  );
}
```

**Why**: Native `<form>` causes a full-page POST request. React Router's `<Form>` intercepts the submission, serializes form data into a `Request`, and passes it to the route's `action` function -- all client-side.

---

## AP-06: Returning undefined from Loaders or Actions

**NEVER** forget to return a value from a loader or action.

```tsx
// WRONG -- returns undefined, causes runtime error
async function loader({ params }: LoaderFunctionArgs) {
  const response = await fetch(`/api/items/${params.id}`);
  if (response.ok) {
    return response.json();
  }
  // falls through to undefined when not ok!
}
```

```tsx
// CORRECT -- ALWAYS return or throw
async function loader({ params }: LoaderFunctionArgs): Promise<Item> {
  const response = await fetch(`/api/items/${params.id}`);
  if (!response.ok) {
    throw new Response("Not found", { status: 404 });
  }
  return response.json();
}
```

**Why**: React Router expects loaders and actions to return a value or throw. Returning `undefined` causes "Cannot read properties of undefined" errors when `useLoaderData()` or `useActionData()` attempts to access the result.

---

## AP-07: Not Handling Errors with errorElement

**NEVER** leave routes without error boundaries.

```tsx
// WRONG -- error in loader crashes entire app
const router = createBrowserRouter([
  {
    path: "/",
    element: <Root />,
    children: [
      { path: "projects/:id", element: <Project />, loader: projectLoader },
    ],
  },
]);
```

```tsx
// CORRECT -- root errorElement catches unhandled errors,
// child errorElement provides granular recovery
const router = createBrowserRouter([
  {
    path: "/",
    element: <Root />,
    errorElement: <RootError />,  // catch-all
    children: [
      {
        path: "projects/:id",
        element: <Project />,
        loader: projectLoader,
        errorElement: <ProjectError />,  // route-specific
      },
    ],
  },
]);
```

**Why**: Without `errorElement`, an error in a loader, action, or component render bubbles up and crashes the entire application. ALWAYS place an `errorElement` on the root route at minimum.

---

## AP-08: Using Lazy with Default Export

**NEVER** use `default` export in lazy route modules.

```tsx
// WRONG -- lazy() does not use default export
// routes/admin.tsx
export default function Admin() { return <div>Admin</div>; }

// router config
{ path: "admin", lazy: () => import("./routes/admin") }
// Result: nothing renders because there is no `Component` export
```

```tsx
// CORRECT -- use named exports matching RouteObject properties
// routes/admin.tsx
export function Component() { return <div>Admin</div>; }
export async function loader() { return fetchAdminData(); }

// router config
{ path: "admin", lazy: () => import("./routes/admin") }
```

**Why**: `lazy()` spreads the returned module's named exports onto the route object. It looks for `Component`, `loader`, `action`, `ErrorBoundary`, etc. A `default` export is ignored entirely.

---

## AP-09: Accessing Component State in Loaders

**NEVER** try to access React state, context, or hooks in loaders/actions.

```tsx
// WRONG -- loaders run outside React, cannot use hooks or context
async function loader(): Promise<Data> {
  const { token } = useAuth(); // ERROR: hooks only work in components
  return fetch("/api/data", { headers: { Authorization: token } });
}
```

```tsx
// CORRECT -- use non-React state management in loaders
async function loader({ request }: LoaderFunctionArgs): Promise<Data> {
  const token = getTokenFromCookie(); // plain function, no React
  if (!token) return redirect("/login");
  const response = await fetch("/api/data", {
    headers: { Authorization: `Bearer ${token}` },
  });
  return response.json();
}
```

**Why**: Loaders and actions execute outside the React component tree. They have no access to React hooks, context, or component state. Use plain JavaScript functions, cookies, localStorage, or module-level singletons for data that loaders need.

---

## AP-10: Mutating searchParams Directly

**NEVER** mutate the `URLSearchParams` object from `useSearchParams` without using the setter.

```tsx
// WRONG -- mutating the object does not trigger navigation
function FilterBar(): React.ReactElement {
  const [searchParams] = useSearchParams();

  const handleFilter = (value: string): void => {
    searchParams.set("filter", value); // mutates but does NOT update URL
  };

  return <button onClick={() => handleFilter("active")}>Active</button>;
}
```

```tsx
// CORRECT -- use the setter function
function FilterBar(): React.ReactElement {
  const [searchParams, setSearchParams] = useSearchParams();

  const handleFilter = (value: string): void => {
    setSearchParams((prev) => {
      prev.set("filter", value);
      return prev;
    });
  };

  return <button onClick={() => handleFilter("active")}>Active</button>;
}
```

**Why**: `URLSearchParams` is a mutable object, but React Router only updates the URL when `setSearchParams` is called. Directly mutating the object silently does nothing visible.
