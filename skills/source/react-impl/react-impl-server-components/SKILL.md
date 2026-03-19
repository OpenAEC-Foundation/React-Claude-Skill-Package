---
name: react-impl-server-components
description: "Guides React Server Components including Server vs Client Component rules, 'use server' and 'use client' directives, Server Actions for mutations, serialization rules across the boundary, component composition across server/client, and framework integration with Next.js. Activates when building with Server Components, deciding server vs client boundaries, implementing Server Actions, or migrating to the Server Component model."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x (partial) or 19.x with a supporting framework (Next.js 13+)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-impl-server-components

## Quick Reference

### Framework Requirement

**NEVER** attempt to use Server Components without a supporting framework. React Server Components require a bundler and server integration — standalone React (`create-react-app`, plain Vite) does NOT support them. Use Next.js App Router (13.4+), or another RSC-compatible framework.

### Server vs Client Component Rules

| Capability | Server Component | Client Component |
|-----------|-----------------|-----------------|
| Directive needed | NONE (default) | `"use client"` at top of file |
| `async/await` in render | YES | NO |
| Direct DB/filesystem access | YES | NO |
| `useState` / `useReducer` | NO | YES |
| `useEffect` / `useLayoutEffect` | NO | YES |
| Event handlers (`onClick`, `onChange`) | NO | YES |
| Browser APIs (`localStorage`, `window`) | NO | YES |
| `useContext` / `use(Context)` | NO | YES |
| Import Client Components | YES | YES |
| Import Server Components | YES | NO |
| Receive Server Component as `children` | N/A | YES |

### Directive Rules

- **Server Components**: NO directive needed — they are the DEFAULT in RSC frameworks
- **Client Components**: MUST have `"use client"` as the FIRST line of the file
- **Server Functions**: Use `"use server"` — either inline in a Server Component function body, or at the top of a dedicated file
- `"use server"` marks **Server Functions**, NOT Server Components — NEVER use it to "mark" a component as server-side

### Serialization Rules (Server-to-Client Boundary)

| Data Type | Can Cross Boundary? | Notes |
|-----------|:-------------------:|-------|
| `string`, `number`, `boolean` | YES | Primitive values serialize directly |
| `null`, `undefined` | YES | |
| Plain objects / arrays | YES | Values must themselves be serializable |
| `Date` | YES | Serialized as ISO string |
| `Promise<T>` | YES | Consumed via `use()` on client (React 19) |
| JSX elements | YES | Pre-rendered output, not source code |
| Server Functions | YES | Sent as serializable references, not code |
| Regular functions | NO | NEVER pass callbacks across the boundary |
| Class instances | NO | Not serializable |
| DOM nodes | NO | Server has no DOM |
| `Symbol` | NO | Not serializable |
| Database connections | NO | Server-only resources |
| Secrets / API keys | NO | NEVER expose to client |

### Critical Warnings

**NEVER** import a Server Component inside a Client Component file — the bundler treats everything imported from a `"use client"` file as client code. ALWAYS pass Server Component output as `children` or other JSX props instead.

**NEVER** pass regular functions as props from Server to Client Components — only Server Functions (marked with `"use server"`) can cross the boundary. All other functions are not serializable.

**NEVER** use `"use server"` at the top of a component file thinking it makes the component a Server Component — Server Components need NO directive. `"use server"` is exclusively for Server Functions.

**ALWAYS** wrap async Server Components in `<Suspense>` boundaries when they perform data fetching — without Suspense, the entire page blocks until the data resolves.

**ALWAYS** use a supporting framework (Next.js App Router 13.4+) — React alone does NOT provide the server infrastructure for RSC.

---

## Decision Tree: Server or Client Component?

```
Does this component need useState, useEffect, useReducer, or other hooks?
├─ YES → Client Component ("use client")
│
Does this component need event handlers (onClick, onChange, onSubmit)?
├─ YES → Client Component ("use client")
│
Does this component need browser APIs (localStorage, window, navigator)?
├─ YES → Client Component ("use client")
│
Does this component fetch data from a database or filesystem?
├─ YES → Server Component (default, no directive)
│
Does this component use heavy libraries only needed for rendering (markdown, syntax highlighting)?
├─ YES → Server Component (zero client bundle cost)
│
Is this a layout, page, or data-fetching wrapper?
├─ YES → Server Component (default)
│
Is this a leaf component with no interactivity?
├─ YES → Server Component (default)
│
Unsure?
└─ Start as Server Component. Add "use client" ONLY when you need client features.
```

---

## Server Components: Core Patterns

### Async Data Fetching (Server Component)

```tsx
// app/notes/[id]/page.tsx — Server Component (no directive)
import { Suspense } from 'react';
import { NoteViewer } from './NoteViewer';
import { CommentList } from './CommentList';
import { db } from '@/lib/db';

interface PageProps {
  params: { id: string };
}

export default async function NotePage({ params }: PageProps) {
  const note = await db.notes.findUnique({ where: { id: params.id } });
  if (!note) return <p>Note not found</p>;

  // Start promise on server, stream to client
  const commentsPromise = db.comments.findMany({ where: { noteId: note.id } });

  return (
    <article>
      <h1>{note.title}</h1>
      <NoteViewer content={note.content} />
      <Suspense fallback={<p>Loading comments...</p>}>
        <CommentList commentsPromise={commentsPromise} />
      </Suspense>
    </article>
  );
}
```

### Server-to-Client Data Streaming (React 19)

```tsx
// CommentList.tsx — Client Component consuming server promise
"use client";
import { use } from 'react';

interface Comment {
  id: string;
  text: string;
  author: string;
}

export function CommentList({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise); // suspends until resolved
  return (
    <ul>
      {comments.map(c => (
        <li key={c.id}>{c.author}: {c.text}</li>
      ))}
    </ul>
  );
}
```

### Bundle Size Optimization

```tsx
// Server Component — these libraries NEVER ship to the client
import { marked } from 'marked';           // 35.9K gzipped
import sanitizeHtml from 'sanitize-html';  // 63.3K gzipped

export default async function MarkdownPage({ slug }: { slug: string }) {
  const content = await fs.readFile(`content/${slug}.md`, 'utf8');
  return <div dangerouslySetInnerHTML={{ __html: sanitizeHtml(marked(content)) }} />;
}
```

---

## Server Functions (Server Actions)

### Terminology

- **Server Function**: ANY `async` function marked with `"use server"`
- **Server Action**: A Server Function used as a form `action` or called from an Action context

### Creating Server Functions

#### Method 1: Inline in Server Component

```tsx
// Server Component
import { SubmitButton } from './SubmitButton';

export default function NewNote() {
  async function createNote(formData: FormData) {
    'use server';
    const title = formData.get('title') as string;
    await db.notes.create({ data: { title } });
  }

  return (
    <form action={createNote}>
      <input name="title" required />
      <SubmitButton />
    </form>
  );
}
```

#### Method 2: Dedicated File (importable by Client Components)

```tsx
// actions.ts
"use server";

export async function createNote(formData: FormData): Promise<{ error?: string }> {
  const title = formData.get('title') as string;
  if (!title) return { error: 'Title is required' };
  await db.notes.create({ data: { title } });
  return {};
}

export async function deleteNote(noteId: string): Promise<void> {
  await db.notes.delete({ where: { id: noteId } });
}
```

### Form Integration with Progressive Enhancement

```tsx
// ClientForm.tsx
"use client";
import { useActionState } from 'react';
import { createNote } from './actions';

export function CreateNoteForm() {
  const [state, submitAction, isPending] = useActionState(createNote, { error: undefined });

  return (
    <form action={submitAction}>
      <input name="title" disabled={isPending} required />
      {state.error && <p className="error">{state.error}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Note'}
      </button>
    </form>
  );
}
```

Forms with Server Actions work **without JavaScript** — the browser submits the form as a standard POST request, and the server processes it. This is progressive enhancement.

---

## Composition: Server + Client Components

### Pattern: Server Component Wrapping Client Component

```tsx
// Server Component (layout or page)
import { Sidebar } from './Sidebar';       // Client Component
import { db } from '@/lib/db';

export default async function Dashboard() {
  const user = await db.users.getCurrent();
  const navItems = await db.nav.getForUser(user.id);

  return (
    <div className="dashboard">
      <Sidebar items={navItems} userName={user.name} />
      <main>{/* more server-rendered content */}</main>
    </div>
  );
}
```

### Pattern: Passing Server Content as Children to Client Component

```tsx
// Server Component
import { Expandable } from './Expandable'; // Client Component
import { db } from '@/lib/db';

export default async function NoteList() {
  const notes = await db.notes.findMany();
  return (
    <div>
      {notes.map(note => (
        <Expandable key={note.id} title={note.title}>
          <p>{note.content}</p> {/* Server-rendered, passed as children */}
        </Expandable>
      ))}
    </div>
  );
}
```

```tsx
// Expandable.tsx — Client Component
"use client";
import { useState, type ReactNode } from 'react';

export function Expandable({ title, children }: { title: string; children: ReactNode }) {
  const [expanded, setExpanded] = useState(false);
  return (
    <div>
      <button onClick={() => setExpanded(!expanded)}>{title}</button>
      {expanded && children}
    </div>
  );
}
```

The browser NEVER receives Server Component source code — only the pre-rendered HTML/JSX output and Client Component code for hydration.

---

## React 18 vs React 19 Compatibility

| Feature | React 18 | React 19 |
|---------|----------|----------|
| Server Components | Canary only (experimental) | Stable |
| `"use client"` directive | Canary only | Stable |
| `"use server"` directive | Canary only | Stable |
| `use()` for promises | NOT available | Stable |
| `useActionState` | NOT available (`useFormState` in canary) | Stable |
| Server Function return types | Limited | Full serializable support |
| Streaming with Suspense | Available via `renderToPipeableStream` | Enhanced with RSC integration |

**ALWAYS** target React 19 for Server Component projects. React 18 support was experimental (canary) and is NOT recommended for production RSC usage.

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete Server Component patterns with working code
- [references/patterns.md](references/patterns.md) -- Server/Client boundary composition patterns
- [references/anti-patterns.md](references/anti-patterns.md) -- Common RSC mistakes and how to avoid them

### Official Sources

- https://react.dev/reference/rsc/server-components
- https://react.dev/reference/rsc/server-functions
- https://react.dev/reference/react/use
- https://react.dev/reference/react/useActionState
- https://react.dev/blog/2024/12/05/react-19
