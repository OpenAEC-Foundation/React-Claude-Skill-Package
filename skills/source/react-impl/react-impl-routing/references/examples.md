# react-impl-routing -- Examples

## Full Application Router Setup

```tsx
// src/router.tsx
import { createBrowserRouter } from "react-router-dom";
import { RootLayout } from "./layouts/RootLayout";
import { RootError } from "./errors/RootError";
import { HomePage } from "./pages/HomePage";

export const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    errorElement: <RootError />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: "projects",
        lazy: () => import("./routes/projects"),
      },
      {
        path: "projects/:projectId",
        lazy: () => import("./routes/project-detail"),
      },
      {
        path: "projects/new",
        lazy: () => import("./routes/project-new"),
      },
      {
        path: "admin",
        lazy: () => import("./routes/admin"),
      },
      {
        path: "login",
        lazy: () => import("./routes/login"),
      },
    ],
  },
]);

// src/main.tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { RouterProvider } from "react-router-dom";
import { router } from "./router";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>
);
```

---

## Lazy Route Module

ALWAYS export named properties that map to `RouteObject` fields:

```tsx
// src/routes/projects.tsx
import { useLoaderData, type LoaderFunctionArgs } from "react-router-dom";

interface Project {
  id: string;
  name: string;
  status: "active" | "archived";
}

export async function loader({ request }: LoaderFunctionArgs): Promise<Project[]> {
  const url = new URL(request.url);
  const status = url.searchParams.get("status") ?? "active";
  const response = await fetch(`/api/projects?status=${status}`);
  if (!response.ok) {
    throw new Response("Failed to load projects", { status: response.status });
  }
  return response.json();
}

export function Component(): React.ReactElement {
  const projects = useLoaderData() as Project[];

  return (
    <div>
      <h1>Projects</h1>
      <ul>
        {projects.map((project) => (
          <li key={project.id}>
            <Link to={project.id}>{project.name}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}

Component.displayName = "ProjectList";
```

---

## CRUD Route with Loader + Action

```tsx
// src/routes/project-detail.tsx
import {
  useLoaderData,
  useActionData,
  Form,
  redirect,
  type LoaderFunctionArgs,
  type ActionFunctionArgs,
} from "react-router-dom";

interface Project {
  id: string;
  name: string;
  description: string;
}

interface ActionResult {
  errors?: { name?: string; description?: string };
}

export async function loader({ params }: LoaderFunctionArgs): Promise<Project> {
  const response = await fetch(`/api/projects/${params.projectId}`);
  if (!response.ok) {
    throw new Response("Project not found", { status: 404 });
  }
  return response.json();
}

export async function action({
  params,
  request,
}: ActionFunctionArgs): Promise<ActionResult | Response> {
  const formData = await request.formData();
  const intent = formData.get("intent");

  if (intent === "delete") {
    await fetch(`/api/projects/${params.projectId}`, { method: "DELETE" });
    return redirect("/projects");
  }

  const name = formData.get("name") as string;
  const description = formData.get("description") as string;
  const errors: ActionResult["errors"] = {};

  if (!name || name.length < 2) errors.name = "Name must be at least 2 characters";
  if (Object.keys(errors).length > 0) return { errors };

  await fetch(`/api/projects/${params.projectId}`, {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ name, description }),
  });

  return redirect("/projects");
}

export function Component(): React.ReactElement {
  const project = useLoaderData() as Project;
  const actionData = useActionData() as ActionResult | undefined;

  return (
    <div>
      <h1>Edit: {project.name}</h1>
      <Form method="post">
        <div>
          <label htmlFor="name">Name</label>
          <input id="name" name="name" defaultValue={project.name} />
          {actionData?.errors?.name && (
            <p className="error">{actionData.errors.name}</p>
          )}
        </div>
        <div>
          <label htmlFor="description">Description</label>
          <textarea
            id="description"
            name="description"
            defaultValue={project.description}
          />
        </div>
        <button type="submit">Save</button>
      </Form>
      <Form method="post">
        <input type="hidden" name="intent" value="delete" />
        <button type="submit" className="danger">Delete</button>
      </Form>
    </div>
  );
}

Component.displayName = "ProjectDetail";
```

---

## Protected Route with Auth Redirect

```tsx
// src/routes/admin.tsx
import {
  redirect,
  useLoaderData,
  type LoaderFunctionArgs,
} from "react-router-dom";

interface AdminData {
  users: Array<{ id: string; email: string; role: string }>;
  stats: { totalUsers: number; activeToday: number };
}

export async function loader({ request }: LoaderFunctionArgs): Promise<AdminData | Response> {
  const token = getAuthToken();
  if (!token) {
    const url = new URL(request.url);
    return redirect(`/login?returnTo=${url.pathname}`);
  }

  const response = await fetch("/api/admin/dashboard", {
    headers: { Authorization: `Bearer ${token}` },
  });

  if (response.status === 403) {
    throw new Response("Forbidden", { status: 403 });
  }

  if (!response.ok) {
    throw new Response("Failed to load admin data", { status: response.status });
  }

  return response.json();
}

export function Component(): React.ReactElement {
  const { users, stats } = useLoaderData() as AdminData;

  return (
    <div>
      <h1>Admin Dashboard</h1>
      <div className="stats">
        <p>Total users: {stats.totalUsers}</p>
        <p>Active today: {stats.activeToday}</p>
      </div>
      <table>
        <thead>
          <tr><th>Email</th><th>Role</th></tr>
        </thead>
        <tbody>
          {users.map((user) => (
            <tr key={user.id}>
              <td>{user.email}</td>
              <td>{user.role}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

Component.displayName = "AdminDashboard";
```

---

## Deferred Loading with Suspense

```tsx
// src/routes/dashboard.tsx
import { Suspense } from "react";
import {
  defer,
  Await,
  useLoaderData,
  type LoaderFunctionArgs,
} from "react-router-dom";

interface User {
  id: string;
  name: string;
}

interface Activity {
  id: string;
  message: string;
  timestamp: string;
}

interface Notification {
  id: string;
  title: string;
  read: boolean;
}

interface DashboardData {
  user: User;
  recentActivity: Promise<Activity[]>;
  notifications: Promise<Notification[]>;
}

export async function loader({ request }: LoaderFunctionArgs) {
  const user = await fetchUser();  // critical -- await immediately

  return defer({
    user,
    recentActivity: fetchActivity(user.id),      // deferred
    notifications: fetchNotifications(user.id),   // deferred
  });
}

export function Component(): React.ReactElement {
  const { user, recentActivity, notifications } = useLoaderData() as DashboardData;

  return (
    <div>
      <h1>Welcome, {user.name}</h1>

      <section>
        <h2>Recent Activity</h2>
        <Suspense fallback={<p>Loading activity...</p>}>
          <Await
            resolve={recentActivity}
            errorElement={<p>Failed to load activity</p>}
          >
            {(activities: Activity[]) => (
              <ul>
                {activities.map((a) => (
                  <li key={a.id}>{a.message}</li>
                ))}
              </ul>
            )}
          </Await>
        </Suspense>
      </section>

      <section>
        <h2>Notifications</h2>
        <Suspense fallback={<p>Loading notifications...</p>}>
          <Await
            resolve={notifications}
            errorElement={<p>Failed to load notifications</p>}
          >
            {(items: Notification[]) => (
              <ul>
                {items.map((n) => (
                  <li key={n.id} className={n.read ? "read" : "unread"}>
                    {n.title}
                  </li>
                ))}
              </ul>
            )}
          </Await>
        </Suspense>
      </section>
    </div>
  );
}

Component.displayName = "Dashboard";
```

---

## Search Parameters with Pagination

```tsx
import { useSearchParams, Link } from "react-router-dom";

interface PaginationProps {
  totalPages: number;
}

function Pagination({ totalPages }: PaginationProps): React.ReactElement {
  const [searchParams] = useSearchParams();
  const currentPage = Number(searchParams.get("page") ?? "1");

  const createPageUrl = (page: number): string => {
    const params = new URLSearchParams(searchParams);
    params.set("page", String(page));
    return `?${params.toString()}`;
  };

  return (
    <nav aria-label="Pagination">
      {currentPage > 1 && (
        <Link to={createPageUrl(currentPage - 1)}>Previous</Link>
      )}
      {Array.from({ length: totalPages }, (_, i) => i + 1).map((page) => (
        <Link
          key={page}
          to={createPageUrl(page)}
          className={page === currentPage ? "active" : ""}
          aria-current={page === currentPage ? "page" : undefined}
        >
          {page}
        </Link>
      ))}
      {currentPage < totalPages && (
        <Link to={createPageUrl(currentPage + 1)}>Next</Link>
      )}
    </nav>
  );
}
```

---

## Nested Layout with Breadcrumbs

```tsx
import { Outlet, useMatches, Link } from "react-router-dom";

interface RouteHandle {
  breadcrumb: string;
}

function BreadcrumbLayout(): React.ReactElement {
  const matches = useMatches();

  const breadcrumbs = matches
    .filter((match) => (match.handle as RouteHandle)?.breadcrumb)
    .map((match) => ({
      path: match.pathname,
      label: (match.handle as RouteHandle).breadcrumb,
    }));

  return (
    <div>
      <nav aria-label="Breadcrumb">
        <ol>
          {breadcrumbs.map((crumb, index) => (
            <li key={crumb.path}>
              {index < breadcrumbs.length - 1 ? (
                <Link to={crumb.path}>{crumb.label}</Link>
              ) : (
                <span aria-current="page">{crumb.label}</span>
              )}
            </li>
          ))}
        </ol>
      </nav>
      <Outlet />
    </div>
  );
}

// Route config with handle for breadcrumbs:
// {
//   path: "projects",
//   element: <BreadcrumbLayout />,
//   handle: { breadcrumb: "Projects" },
//   children: [
//     { index: true, element: <ProjectList />, handle: { breadcrumb: "All" } },
//     { path: ":id", element: <ProjectDetail />, handle: { breadcrumb: "Detail" } },
//   ],
// }
```
