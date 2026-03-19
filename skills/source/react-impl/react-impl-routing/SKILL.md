---
name: react-impl-routing
description: "Guides React Router v6+ patterns including createBrowserRouter, route configuration, nested routes with Outlet, data loaders and actions, lazy routes for code splitting, protected routes, URL parameters, search parameters, and navigation. Activates when implementing client-side routing, setting up route configuration, handling navigation, or building protected routes."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with React Router 6.4+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-impl-routing

## Quick Reference

### Router Setup (React Router 6.4+)

| Concept | API | Purpose |
|---------|-----|---------|
| Router creation | `createBrowserRouter()` | Define route tree as object array |
| Router rendering | `<RouterProvider router={router} />` | Mount the data router in React |
| Nested rendering | `<Outlet />` | Render child route element inside parent layout |
| Navigation link | `<Link to="/path">` | Client-side navigation without reload |
| Active link | `<NavLink className={({isActive}) => ...}>` | Link with active state styling |
| Programmatic nav | `useNavigate()` | Navigate from event handlers or effects |
| URL params | `useParams()` | Read dynamic route segments |
| Query strings | `useSearchParams()` | Read and update URL search parameters |
| Loader data | `useLoaderData()` | Access data returned by route loader |
| Action data | `useActionData()` | Access data returned by route action |
| Error info | `useRouteError()` | Access error thrown in loader/action/render |

### Critical Warnings

**NEVER** use the legacy `<BrowserRouter>` + `<Routes>` pattern for new projects -- ALWAYS use `createBrowserRouter` with `<RouterProvider>`. The data router API enables loaders, actions, and lazy routes that the legacy API cannot support.

**NEVER** call `navigate()` during render -- ALWAYS call it inside `useEffect`, event handlers, or loader/action functions. Calling during render causes infinite re-render loops.

**NEVER** define route objects inside a component -- ALWAYS define routes at module scope or in a separate file. Defining inside a component recreates the router on every render, destroying all state.

**NEVER** use `loader` or `action` as async arrow functions that capture component scope -- loaders and actions run outside React component lifecycle. They receive `{params, request}` as arguments.

**ALWAYS** return or throw a `Response` or value from loaders and actions -- returning `undefined` causes runtime errors.

**ALWAYS** use `<Form>` from `react-router-dom` instead of `<form>` when you want route actions to handle submission -- native `<form>` bypasses the router entirely.

---

## Route Configuration

### createBrowserRouter Pattern

ALWAYS define routes as a configuration object array:

```tsx
import {
  createBrowserRouter,
  RouterProvider,
} from "react-router-dom";

const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    errorElement: <RootError />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: "projects",
        element: <ProjectsLayout />,
        children: [
          { index: true, element: <ProjectList /> },
          {
            path: ":projectId",
            element: <ProjectDetail />,
            loader: projectLoader,
            action: projectAction,
            errorElement: <ProjectError />,
          },
        ],
      },
      {
        path: "settings",
        lazy: () => import("./routes/settings"),
      },
    ],
  },
]);

function App(): React.ReactElement {
  return <RouterProvider router={router} />;
}
```

### Route Properties

| Property | Type | Purpose |
|----------|------|---------|
| `path` | `string` | URL segment to match |
| `element` | `ReactElement` | Component to render when matched |
| `errorElement` | `ReactElement` | Fallback UI when loader/action/render throws |
| `loader` | `LoaderFunction` | Fetch data before rendering |
| `action` | `ActionFunction` | Handle form submissions / mutations |
| `lazy` | `() => Promise<RouteObject>` | Code-split route module |
| `children` | `RouteObject[]` | Nested child routes |
| `index` | `boolean` | Default child route (renders in parent Outlet) |

---

## Nested Routes and Layouts

Use `<Outlet />` in parent routes to render matched child routes:

```tsx
import { Outlet, NavLink } from "react-router-dom";

function RootLayout(): React.ReactElement {
  return (
    <div>
      <nav>
        <NavLink
          to="/"
          className={({ isActive }) => (isActive ? "active" : "")}
        >
          Home
        </NavLink>
        <NavLink
          to="/projects"
          className={({ isActive }) => (isActive ? "active" : "")}
        >
          Projects
        </NavLink>
      </nav>
      <main>
        <Outlet />
      </main>
    </div>
  );
}
```

### Layout Route (pathless)

A route without a `path` serves as a layout wrapper without adding a URL segment:

```tsx
{
  element: <AuthenticatedLayout />,
  children: [
    { path: "dashboard", element: <Dashboard /> },
    { path: "profile", element: <Profile /> },
  ],
}
```

### Index Route

An index route is the default child that renders when the parent path matches exactly:

```tsx
{
  path: "projects",
  element: <ProjectsLayout />,
  children: [
    { index: true, element: <ProjectList /> },   // matches /projects
    { path: ":id", element: <ProjectDetail /> },  // matches /projects/123
  ],
}
```

---

## Navigation

### Link and NavLink

```tsx
import { Link, NavLink } from "react-router-dom";

<Link to="/projects">Projects</Link>
<Link to="../settings">Settings</Link>  {/* relative to current route */}

<NavLink
  to="/projects"
  className={({ isActive, isPending }) =>
    isPending ? "pending" : isActive ? "active" : ""
  }
>
  Projects
</NavLink>
```

### useNavigate

```tsx
const navigate = useNavigate();

// In event handler or effect -- NEVER during render
navigate(`/projects/${id}`);
navigate("/login", { replace: true });         // replace history entry
navigate(-1);                                   // go back
navigate("/dash", { state: { from: "/" } });    // pass state
```

---

## URL Parameters and Search Parameters

### useParams

```tsx
import { useParams } from "react-router-dom";

// Route: { path: "projects/:projectId/tasks/:taskId?" }

function TaskView(): React.ReactElement {
  const { projectId, taskId } = useParams<{
    projectId: string;
    taskId?: string;  // optional segment marked with ?
  }>();

  // ALWAYS check params exist -- useParams returns string | undefined
  if (!projectId) throw new Error("projectId is required");

  return <div>Project: {projectId}, Task: {taskId ?? "none"}</div>;
}
```

### useSearchParams

```tsx
function ProjectList(): React.ReactElement {
  const [searchParams, setSearchParams] = useSearchParams();

  const filter = searchParams.get("filter") ?? "all";
  const page = Number(searchParams.get("page") ?? "1");

  const updateFilter = (newFilter: string): void => {
    setSearchParams((prev) => {
      prev.set("filter", newFilter);
      prev.set("page", "1");
      return prev;
    });
  };

  return (
    <div>
      <button onClick={() => updateFilter("active")}>Active</button>
      <button onClick={() => updateFilter("all")}>All</button>
      <p>Filter: {filter}, Page: {page}</p>
    </div>
  );
}
```

---

## Data Loading

### Route Loader

Loaders receive `{params, request}` and run before the route renders:

```tsx
import { useLoaderData, type LoaderFunctionArgs } from "react-router-dom";

interface Project { id: string; name: string }

async function projectLoader({ params, request }: LoaderFunctionArgs): Promise<Project> {
  const response = await fetch(`/api/projects/${params.projectId}`);
  if (!response.ok) throw new Response("Project not found", { status: 404 });
  return response.json();
}

function ProjectDetail(): React.ReactElement {
  const project = useLoaderData() as Project;
  return <h1>{project.name}</h1>;
}
// Register: { path: ":projectId", element: <ProjectDetail />, loader: projectLoader }
```

### Deferred Data with defer + Await

Use `defer()` to return a mix of awaited (critical) and deferred (non-critical) data. Render deferred promises with `<Suspense>` + `<Await>`:

```tsx
export async function loader(): Promise<ReturnType<typeof defer>> {
  const user = await fetchUser();             // critical -- await immediately
  return defer({
    user,
    recommendations: fetchRecommendations(),  // deferred -- NOT awaited
  });
}

export function Component(): React.ReactElement {
  const { user, recommendations } = useLoaderData() as {
    user: User;
    recommendations: Promise<Recommendation[]>;
  };
  return (
    <div>
      <h1>Welcome {user.name}</h1>
      <Suspense fallback={<p>Loading...</p>}>
        <Await resolve={recommendations}>
          {(data: Recommendation[]) => <RecommendationList items={data} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

See [references/examples.md](references/examples.md) for a full deferred loading example.

---

## Route Actions

### Form and Action Pattern

ALWAYS use `<Form>` from `react-router-dom` (not native `<form>`) to trigger route actions:

```tsx
import { Form, useActionData, redirect, type ActionFunctionArgs } from "react-router-dom";

interface ActionErrors { name?: string }

async function createAction({ request }: ActionFunctionArgs): Promise<ActionErrors | Response> {
  const formData = await request.formData();
  const name = formData.get("name") as string;
  if (!name || name.length < 3) return { name: "Name must be at least 3 characters" };
  const project = await createProject({ name });
  return redirect(`/projects/${project.id}`);
}

function NewProject(): React.ReactElement {
  const errors = useActionData() as ActionErrors | undefined;
  return (
    <Form method="post">
      <input name="name" type="text" />
      {errors?.name && <span className="error">{errors.name}</span>}
      <button type="submit">Create</button>
    </Form>
  );
}
// Register: { path: "new", element: <NewProject />, action: createAction }
```

---

## Lazy Routes

ALWAYS use `lazy()` for route-level code splitting on routes not needed at initial load:

```tsx
const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: "admin",
        lazy: () => import("./routes/admin"),
      },
    ],
  },
]);

// ./routes/admin.tsx -- MUST export named properties matching RouteObject
export async function loader(): Promise<AdminData> {
  return fetchAdminData();
}

export function Component(): React.ReactElement {
  const data = useLoaderData() as AdminData;
  return <AdminPanel data={data} />;
}

// Optional: export errorElement, action, etc.
```

The `lazy()` function MUST return an object with route properties (`Component`, `loader`, `action`, `errorElement`). It NEVER returns a `default` export -- use named exports matching the route property names.

---

## Protected Routes

### Loader-Based Protection (Preferred)

```tsx
import { redirect, type LoaderFunctionArgs } from "react-router-dom";

function protectedLoader({ request }: LoaderFunctionArgs): null | Response {
  const isAuthenticated = checkAuth();
  if (!isAuthenticated) {
    const url = new URL(request.url);
    return redirect(`/login?returnTo=${url.pathname}`);
  }
  return null;
}

// Apply to route:
// { path: "dashboard", element: <Dashboard />, loader: protectedLoader }
```

### Wrapper Component Pattern

```tsx
import { Navigate, Outlet, useLocation } from "react-router-dom";

function RequireAuth(): React.ReactElement {
  const { user } = useAuth();
  const location = useLocation();

  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;
}

// Use as layout route:
// {
//   element: <RequireAuth />,
//   children: [
//     { path: "dashboard", element: <Dashboard /> },
//     { path: "settings", element: <Settings /> },
//   ],
// }
```

---

## Error Handling

### errorElement and useRouteError

```tsx
import { useRouteError, isRouteErrorResponse } from "react-router-dom";

function RouteError(): React.ReactElement {
  const error = useRouteError();
  if (isRouteErrorResponse(error)) {
    return <div><h1>{error.status}</h1><p>{error.statusText}</p></div>;
  }
  return <div><h1>Error</h1><p>{error instanceof Error ? error.message : "Unknown"}</p></div>;
}
// ALWAYS place errorElement on the root route as a catch-all.
// Place specific errorElement on child routes for granular error UIs.
```

---

## Decision Trees

### Which Router Pattern?

```
New project?
├── YES → createBrowserRouter + RouterProvider (ALWAYS)
└── NO (legacy codebase with BrowserRouter)
    ├── Can migrate? → YES → migrate to createBrowserRouter
    └── Cannot migrate yet → keep BrowserRouter, but do NOT add loaders/actions
```

### Where to Put Auth Check?

```
Need to redirect before any rendering?
├── YES → Use loader-based protection (redirect in loader)
└── NO (need access to React context like auth provider)
    └── Use wrapper component pattern (<RequireAuth> with <Outlet>)
```

### How to Load Data?

```
Data needed before route renders?
├── YES → Use route loader
│   ├── All data critical? → await everything in loader
│   └── Some data non-critical? → use defer() + <Await>
└── NO (data loaded after user interaction)
    └── Use useEffect or event handler in component
```

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete routing patterns with TypeScript
- [references/api-table.md](references/api-table.md) -- React Router hooks and components reference
- [references/anti-patterns.md](references/anti-patterns.md) -- Common routing mistakes and fixes

### Official Sources

- https://reactrouter.com/en/main/start/overview
- https://reactrouter.com/en/main/routers/create-browser-router
- https://reactrouter.com/en/main/route/route
- https://reactrouter.com/en/main/route/loader
- https://reactrouter.com/en/main/route/action
- https://reactrouter.com/en/main/route/lazy
