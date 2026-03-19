# Server/Client Boundary Patterns

> Reference file for `react-impl-server-components` skill.
> Patterns for composing Server and Client Components across the RSC boundary.

---

## Pattern 1: Server Parent, Client Child (Most Common)

Server Components ALWAYS render Client Components by importing them. Data flows down as serializable props.

```tsx
// Server Component
import { InteractiveWidget } from './InteractiveWidget'; // Client Component

export default async function Page() {
  const data = await fetchData();
  return (
    <main>
      <h1>{data.title}</h1>
      <InteractiveWidget items={data.items} /> {/* serializable props only */}
    </main>
  );
}
```

**Rule**: Props passed to Client Components MUST be serializable. NEVER pass functions (except Server Functions), class instances, or DOM nodes.

---

## Pattern 2: Client Component Receiving Server Content as Children

When a Client Component needs to wrap server-rendered content, pass it as `children`. The Server Component pre-renders the children, and the Client Component receives the rendered output.

```tsx
// Server Component
import { Tabs } from './Tabs';         // Client Component
import { ServerContent } from './ServerContent'; // Server Component

export default async function Page() {
  return (
    <Tabs labels={['Overview', 'Details']}>
      <ServerContent />  {/* Pre-rendered on server, passed as JSX */}
    </Tabs>
  );
}
```

```tsx
// Tabs.tsx — Client Component
"use client";
import { useState, type ReactNode } from 'react';

export function Tabs({ labels, children }: { labels: string[]; children: ReactNode }) {
  const [active, setActive] = useState(0);
  return (
    <div>
      <div role="tablist">
        {labels.map((label, i) => (
          <button key={label} role="tab" onClick={() => setActive(i)}>{label}</button>
        ))}
      </div>
      <div role="tabpanel">{children}</div>
    </div>
  );
}
```

**Rule**: The `children` prop carries pre-rendered JSX — it crosses the boundary as serialized React elements, NOT as component references.

---

## Pattern 3: Slot Pattern (Multiple Server Component Regions in Client)

Pass multiple server-rendered regions as named props to a Client Component.

```tsx
// Server Component
import { SplitLayout } from './SplitLayout'; // Client Component
import { Sidebar } from './Sidebar';           // Server Component
import { MainContent } from './MainContent';   // Server Component

export default async function Page() {
  return (
    <SplitLayout
      sidebar={<Sidebar />}        {/* Server-rendered slot */}
      main={<MainContent />}       {/* Server-rendered slot */}
    />
  );
}
```

```tsx
// SplitLayout.tsx — Client Component
"use client";
import { useState, type ReactNode } from 'react';

export function SplitLayout({ sidebar, main }: { sidebar: ReactNode; main: ReactNode }) {
  const [sidebarOpen, setSidebarOpen] = useState(true);
  return (
    <div className="layout">
      {sidebarOpen && <aside>{sidebar}</aside>}
      <button onClick={() => setSidebarOpen(!sidebarOpen)}>Toggle</button>
      <main>{main}</main>
    </div>
  );
}
```

**Rule**: ALWAYS use `ReactNode` type for slot props. These carry pre-rendered JSX across the boundary.

---

## Pattern 4: Context Provider at the Boundary

Context providers require `useState` or `useReducer`, making them Client Components. Place them high in the tree and pass server-rendered children through them.

```tsx
// app/layout.tsx — Server Component
import { AuthProvider } from './AuthProvider';
import { getCurrentUser } from '@/lib/auth';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const user = await getCurrentUser(); // server-side auth check

  return (
    <html lang="en">
      <body>
        <AuthProvider initialUser={user}>
          {children} {/* Server Components render inside Client provider */}
        </AuthProvider>
      </body>
    </html>
  );
}
```

```tsx
// AuthProvider.tsx — Client Component
"use client";
import { createContext, useState, type ReactNode } from 'react';

interface User { id: string; name: string; email: string; }

export const AuthContext = createContext<{ user: User | null }>({ user: null });

export function AuthProvider({ initialUser, children }: { initialUser: User | null; children: ReactNode }) {
  const [user] = useState(initialUser);
  return <AuthContext value={{ user }}>{children}</AuthContext>;
}
```

**Rule**: Fetch data in the Server Component parent, pass it as `initialUser` prop to the Client provider. NEVER fetch in the Client provider itself.

---

## Pattern 5: Server Function Passed to Client Component

Server Functions (marked with `"use server"`) are the ONLY functions that can cross the server-to-client boundary.

```tsx
// Server Component
import { LikeButton } from './LikeButton';

export default async function Post({ postId }: { postId: string }) {
  async function handleLike() {
    'use server';
    await db.posts.incrementLikes(postId);
  }

  return (
    <article>
      <p>Post content...</p>
      <LikeButton onLike={handleLike} /> {/* Server Function reference */}
    </article>
  );
}
```

```tsx
// LikeButton.tsx — Client Component
"use client";
import { useTransition } from 'react';

export function LikeButton({ onLike }: { onLike: () => Promise<void> }) {
  const [isPending, startTransition] = useTransition();

  return (
    <button
      onClick={() => startTransition(() => onLike())}
      disabled={isPending}
    >
      {isPending ? 'Liking...' : 'Like'}
    </button>
  );
}
```

**Rule**: The framework serializes the Server Function as a reference (`$$typeof: Symbol.for("react.server.reference")`). When the client calls it, the framework sends a request to the server to execute it.

---

## Pattern 6: Conditional Rendering at the Boundary

Server Components can conditionally render Client Components based on server-side data.

```tsx
// Server Component
import { AdminPanel } from './AdminPanel';   // Client Component
import { UserDashboard } from './UserDashboard'; // Client Component
import { getCurrentUser } from '@/lib/auth';

export default async function Page() {
  const user = await getCurrentUser();

  if (user.role === 'admin') {
    return <AdminPanel permissions={user.permissions} />;
  }

  return <UserDashboard userId={user.id} />;
}
```

**Rule**: The conditional logic runs on the server. Only the selected Client Component ships to the client — the other is NEVER sent.

---

## Pattern 7: Streaming with Nested Suspense Boundaries

Use nested Suspense boundaries to stream different parts of the page independently.

```tsx
// Server Component
import { Suspense } from 'react';
import { Header } from './Header';
import { Recommendations } from './Recommendations';
import { Comments } from './Comments';

export default async function ArticlePage({ id }: { id: string }) {
  const article = await getArticle(id); // blocks — critical content

  return (
    <main>
      <Header title={article.title} />
      <article>{article.content}</article>

      {/* These stream independently as they resolve */}
      <Suspense fallback={<div>Loading recommendations...</div>}>
        <Recommendations articleId={id} />
      </Suspense>

      <Suspense fallback={<div>Loading comments...</div>}>
        <Comments articleId={id} />
      </Suspense>
    </main>
  );
}
```

```tsx
// Recommendations.tsx — Server Component (async)
export async function Recommendations({ articleId }: { articleId: string }) {
  const items = await getRecommendations(articleId); // slow API call
  return (
    <section>
      <h2>Recommended</h2>
      <ul>{items.map(item => <li key={item.id}>{item.title}</li>)}</ul>
    </section>
  );
}
```

**Rule**: Each Suspense boundary creates an independent streaming slot. The shell (everything outside Suspense) renders first, then each boundary fills in as its data resolves.

---

## Pattern 8: Error Boundaries with Server Components

Wrap async Server Components in Error Boundaries to handle fetch failures gracefully.

```tsx
// Server Component
import { Suspense } from 'react';
import { ErrorBoundary } from './ErrorBoundary'; // Client Component

export default function Page() {
  return (
    <ErrorBoundary fallback={<p>Something went wrong.</p>}>
      <Suspense fallback={<p>Loading...</p>}>
        <AsyncServerComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

```tsx
// ErrorBoundary.tsx — Client Component (class component required)
"use client";
import { Component, type ReactNode } from 'react';

interface Props {
  fallback: ReactNode;
  children: ReactNode;
}

interface State {
  hasError: boolean;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}
```

**Rule**: Error Boundaries MUST be Client Components (they use `getDerivedStateFromError` lifecycle method). ALWAYS place them above async Server Components that might fail.

---

## Pattern Summary Table

| Pattern | Use When | Key Rule |
|---------|----------|----------|
| Server Parent, Client Child | Fetching data for interactive UI | Props MUST be serializable |
| Children Prop | Client wrapper around server content | Pass `ReactNode`, not component references |
| Slot Pattern | Multiple server regions in client layout | Use named `ReactNode` props |
| Context at Boundary | Shared state with server-fetched initial data | Fetch in server, pass as initial prop |
| Server Function | Client needs to trigger server mutation | ONLY `"use server"` functions cross boundary |
| Conditional Rendering | Different UI based on server data | Only selected component ships to client |
| Nested Suspense | Independent streaming sections | Each boundary streams independently |
| Error Boundary | Graceful failure handling | Error Boundary MUST be Client Component |
