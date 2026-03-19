# Server Component Anti-Patterns

> Reference file for `react-impl-server-components` skill.
> Common mistakes when working with React Server Components, with explanations and corrections.

---

## Anti-Pattern 1: Importing Server Components Inside Client Components

### WRONG

```tsx
// Dashboard.tsx — Client Component
"use client";
import { UserStats } from './UserStats'; // UserStats is a Server Component

export function Dashboard() {
  const [tab, setTab] = useState('stats');
  return (
    <div>
      <button onClick={() => setTab('stats')}>Stats</button>
      {tab === 'stats' && <UserStats />} {/* BROKEN: UserStats becomes a Client Component */}
    </div>
  );
}
```

### WHY IT FAILS

When a `"use client"` file imports another module, the bundler treats that module as client code. The Server Component loses all server capabilities — `async/await`, database access, filesystem access — and WILL error at runtime or silently become a client-only component.

### CORRECT

```tsx
// Dashboard.tsx — Client Component
"use client";
import { useState, type ReactNode } from 'react';

export function Dashboard({ statsSlot }: { statsSlot: ReactNode }) {
  const [tab, setTab] = useState('stats');
  return (
    <div>
      <button onClick={() => setTab('stats')}>Stats</button>
      {tab === 'stats' && statsSlot}
    </div>
  );
}

// page.tsx — Server Component
import { Dashboard } from './Dashboard';
import { UserStats } from './UserStats'; // Server Component, imported from server context

export default async function Page() {
  return <Dashboard statsSlot={<UserStats />} />;
}
```

**Rule**: ALWAYS pass Server Component output as `children` or named `ReactNode` props from a Server Component parent. NEVER import Server Components directly in Client Component files.

---

## Anti-Pattern 2: Using "use server" to Mark Server Components

### WRONG

```tsx
// UserProfile.tsx
"use server"; // WRONG — this marks the file's exports as Server Functions

export default async function UserProfile() {
  const user = await db.users.getCurrent();
  return <div>{user.name}</div>;
}
```

### WHY IT FAILS

`"use server"` makes every exported function a **Server Function** (callable from the client via RPC). It does NOT mark the file as a Server Component. The component will NOT render as expected — the framework treats it as a callable function, not a renderable component.

### CORRECT

```tsx
// UserProfile.tsx — Server Component (NO directive needed)
export default async function UserProfile() {
  const user = await db.users.getCurrent();
  return <div>{user.name}</div>;
}
```

**Rule**: Server Components are the DEFAULT — they need NO directive. `"use server"` is EXCLUSIVELY for Server Functions.

---

## Anti-Pattern 3: Passing Non-Serializable Props Across the Boundary

### WRONG

```tsx
// Server Component
import { DataTable } from './DataTable'; // Client Component

export default async function Page() {
  const formatDate = (d: Date) => d.toLocaleDateString();

  return (
    <DataTable
      data={rows}
      formatDate={formatDate} // FAILS: regular functions are not serializable
      dbConnection={db}       // FAILS: class instance
      renderRow={(row) => <tr><td>{row.name}</td></tr>} // FAILS: function
    />
  );
}
```

### WHY IT FAILS

Only serializable values can cross the server-to-client boundary. Regular functions, class instances, Symbols, and DOM nodes cannot be serialized. The framework throws a serialization error at build time or runtime.

### CORRECT

```tsx
// Server Component
import { DataTable } from './DataTable';

export default async function Page() {
  const rows = await db.query('SELECT * FROM items');

  return (
    <DataTable
      data={rows}             // plain array of objects — serializable
      dateFormat="en-US"       // pass configuration, not functions
      columns={['name', 'date', 'status']} // serializable descriptor
    />
  );
}
```

```tsx
// DataTable.tsx — Client Component
"use client";

export function DataTable({ data, dateFormat, columns }: Props) {
  // Format logic lives in the Client Component
  const formatDate = (d: string) => new Date(d).toLocaleDateString(dateFormat);

  return (
    <table>
      <tbody>
        {data.map((row, i) => (
          <tr key={i}>
            {columns.map(col => (
              <td key={col}>{col === 'date' ? formatDate(row[col]) : row[col]}</td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Rule**: ALWAYS move formatting and rendering logic into the Client Component. Pass only serializable data and configuration strings/numbers/booleans across the boundary.

---

## Anti-Pattern 4: Using Hooks in Server Components

### WRONG

```tsx
// Server Component — NO directive
import { useState, useEffect } from 'react';

export default function Settings() {
  const [theme, setTheme] = useState('light'); // FAILS: hooks require client
  useEffect(() => {
    document.body.className = theme;
  }, [theme]);

  return <div>Settings page</div>;
}
```

### WHY IT FAILS

Server Components run on the server where there is no persistent component instance, no re-rendering, and no DOM. `useState`, `useEffect`, `useReducer`, `useRef`, and other stateful hooks CANNOT work in this environment.

### CORRECT

Split into a Server Component for data and a Client Component for interactivity:

```tsx
// page.tsx — Server Component
import { ThemeSettings } from './ThemeSettings';
import { getUserPreferences } from '@/lib/db';

export default async function SettingsPage() {
  const prefs = await getUserPreferences();
  return <ThemeSettings initialTheme={prefs.theme} />;
}
```

```tsx
// ThemeSettings.tsx — Client Component
"use client";
import { useState, useEffect } from 'react';

export function ThemeSettings({ initialTheme }: { initialTheme: string }) {
  const [theme, setTheme] = useState(initialTheme);
  useEffect(() => {
    document.body.className = theme;
  }, [theme]);

  return (
    <select value={theme} onChange={e => setTheme(e.target.value)}>
      <option value="light">Light</option>
      <option value="dark">Dark</option>
    </select>
  );
}
```

**Rule**: If a component needs ANY hook (`useState`, `useEffect`, `useReducer`, `useRef`, `useContext`), it MUST be a Client Component with `"use client"`.

---

## Anti-Pattern 5: Using Server Components Without a Framework

### WRONG

```tsx
// In a plain Vite + React app (NO Next.js or RSC framework)
// src/components/ServerPage.tsx

export default async function ServerPage() {
  const data = await fetch('https://api.example.com/data');
  const json = await data.json();
  return <div>{json.title}</div>; // FAILS: plain React does not support async components
}
```

### WHY IT FAILS

React Server Components require a framework that provides:
1. A server runtime to execute Server Components
2. A bundler that understands `"use client"` and `"use server"` directives
3. A streaming protocol to send RSC payload to the client
4. Client-side hydration that stitches Server and Client Components together

Plain React (Vite, CRA, Parcel) provides NONE of these.

### CORRECT

Use Next.js App Router (13.4+) or another RSC-compatible framework:

```
npx create-next-app@latest my-app --app
```

Then async Server Components work in the `app/` directory by default.

**Rule**: NEVER attempt to use Server Components in a plain React project. ALWAYS use Next.js App Router or another framework that implements the RSC protocol.

---

## Anti-Pattern 6: Data Fetching in Client Components When Server Components Suffice

### WRONG

```tsx
// UserList.tsx — Client Component (unnecessarily)
"use client";
import { useState, useEffect } from 'react';

export function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(data => { setUsers(data); setLoading(false); });
  }, []);

  if (loading) return <p>Loading...</p>;
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### WHY IT IS SUBOPTIMAL

This pattern:
1. Ships the component code to the client (bundle size)
2. Requires an API route as a middleman
3. Creates a waterfall: HTML loads -> JS loads -> fetch fires -> data renders
4. Shows a loading spinner while the server already has the data

### CORRECT

```tsx
// UserList.tsx — Server Component
import { db } from '@/lib/db';

export default async function UserList() {
  const users = await db.users.findMany();
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

This pattern:
1. Zero client JavaScript for this component
2. No API route needed — direct database access
3. No waterfall — data is fetched and rendered on the server in one step
4. HTML arrives with data already rendered

**Rule**: ALWAYS use Server Components for data display that requires no interactivity. Reserve Client Components for components that NEED hooks, event handlers, or browser APIs.

---

## Anti-Pattern 7: Exposing Secrets Through Server Functions

### WRONG

```tsx
// actions.ts
"use server";

export async function getData() {
  // API key is safe on server, but...
  const res = await fetch('https://api.example.com/data', {
    headers: { Authorization: `Bearer ${process.env.API_KEY}` },
  });
  const data = await res.json();

  return {
    items: data.items,
    apiKey: process.env.API_KEY, // EXPOSED: returned to client
    dbUrl: process.env.DATABASE_URL, // EXPOSED: returned to client
  };
}
```

### WHY IT FAILS

Server Functions execute on the server, but their RETURN VALUES are sent to the client. Any secret included in the return value is exposed to the browser.

### CORRECT

```tsx
// actions.ts
"use server";

export async function getData() {
  const res = await fetch('https://api.example.com/data', {
    headers: { Authorization: `Bearer ${process.env.API_KEY}` },
  });
  const data = await res.json();

  return {
    items: data.items, // ONLY return what the client needs
  };
}
```

**Rule**: NEVER include environment variables, API keys, database URLs, or other secrets in Server Function return values. ALWAYS return only the data the client needs to render.

---

## Anti-Pattern 8: Missing Suspense Around Async Server Components

### WRONG

```tsx
// page.tsx — Server Component
export default function Page() {
  return (
    <main>
      <h1>Dashboard</h1>
      <SlowDataComponent />  {/* No Suspense — entire page blocks */}
      <FastContent />         {/* This also waits, even though it is fast */}
    </main>
  );
}

async function SlowDataComponent() {
  const data = await slowApiCall(); // 3 seconds
  return <div>{data.summary}</div>;
}
```

### WHY IT IS SUBOPTIMAL

Without Suspense, the entire page waits for ALL async Server Components to resolve before ANY content is sent to the browser. The user sees nothing for 3 seconds.

### CORRECT

```tsx
// page.tsx — Server Component
import { Suspense } from 'react';

export default function Page() {
  return (
    <main>
      <h1>Dashboard</h1>
      <FastContent /> {/* Renders immediately */}
      <Suspense fallback={<p>Loading data...</p>}>
        <SlowDataComponent /> {/* Streams in when ready */}
      </Suspense>
    </main>
  );
}
```

**Rule**: ALWAYS wrap async Server Components that perform slow data fetching in `<Suspense>` boundaries. This enables streaming — fast content renders immediately while slow content shows a fallback.

---

## Anti-Pattern Summary Table

| Anti-Pattern | Symptom | Fix |
|-------------|---------|-----|
| Import Server in Client | Server Component loses async capabilities | Pass as `children` or `ReactNode` prop |
| `"use server"` on component | Component treated as RPC function | Remove directive (server is default) |
| Non-serializable props | Serialization error at boundary | Pass data/config, move logic to client |
| Hooks in Server Component | Runtime error (hooks undefined) | Split into Server + Client Components |
| RSC without framework | Async components fail to render | Use Next.js App Router or equivalent |
| Client fetch when server suffices | Unnecessary waterfall and bundle | Use async Server Component instead |
| Secrets in return values | API keys exposed to browser | Return only client-needed data |
| Missing Suspense | Entire page blocks on slow data | Wrap async components in Suspense |
