# Server Component Examples

> Reference file for `react-impl-server-components` skill.
> All examples use TypeScript/TSX and target React 19 with Next.js App Router.

---

## Example 1: Full-Stack CRUD with Server Components and Server Actions

### Database Layer (Server-Only)

```tsx
// lib/db.ts — Server-only module
import { prisma } from './prisma';

export interface Note {
  id: string;
  title: string;
  content: string;
  createdAt: Date;
  updatedAt: Date;
}

export async function getNotes(): Promise<Note[]> {
  return prisma.note.findMany({ orderBy: { updatedAt: 'desc' } });
}

export async function getNote(id: string): Promise<Note | null> {
  return prisma.note.findUnique({ where: { id } });
}

export async function createNote(title: string, content: string): Promise<Note> {
  return prisma.note.create({ data: { title, content } });
}

export async function deleteNote(id: string): Promise<void> {
  await prisma.note.delete({ where: { id } });
}
```

### Server Actions File

```tsx
// app/notes/actions.ts
"use server";

import { revalidatePath } from 'next/cache';
import { createNote, deleteNote } from '@/lib/db';

interface ActionState {
  error?: string;
  success?: boolean;
}

export async function createNoteAction(
  prevState: ActionState,
  formData: FormData
): Promise<ActionState> {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  if (!title?.trim()) return { error: 'Title is required' };
  if (!content?.trim()) return { error: 'Content is required' };

  try {
    await createNote(title.trim(), content.trim());
    revalidatePath('/notes');
    return { success: true };
  } catch {
    return { error: 'Failed to create note' };
  }
}

export async function deleteNoteAction(noteId: string): Promise<ActionState> {
  try {
    await deleteNote(noteId);
    revalidatePath('/notes');
    return { success: true };
  } catch {
    return { error: 'Failed to delete note' };
  }
}
```

### Server Component (Page)

```tsx
// app/notes/page.tsx — Server Component
import { Suspense } from 'react';
import { getNotes } from '@/lib/db';
import { CreateNoteForm } from './CreateNoteForm';
import { NoteCard } from './NoteCard';

export default async function NotesPage() {
  const notes = await getNotes();

  return (
    <main>
      <h1>Notes</h1>
      <CreateNoteForm />
      <Suspense fallback={<p>Loading notes...</p>}>
        <section>
          {notes.map(note => (
            <NoteCard key={note.id} note={note} />
          ))}
          {notes.length === 0 && <p>No notes yet.</p>}
        </section>
      </Suspense>
    </main>
  );
}
```

### Client Component (Form with useActionState)

```tsx
// app/notes/CreateNoteForm.tsx
"use client";

import { useActionState } from 'react';
import { createNoteAction } from './actions';

export function CreateNoteForm() {
  const [state, submitAction, isPending] = useActionState(createNoteAction, {});

  return (
    <form action={submitAction}>
      <div>
        <label htmlFor="title">Title</label>
        <input id="title" name="title" required disabled={isPending} />
      </div>
      <div>
        <label htmlFor="content">Content</label>
        <textarea id="content" name="content" required disabled={isPending} />
      </div>
      {state.error && <p role="alert" className="error">{state.error}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Note'}
      </button>
    </form>
  );
}
```

### Client Component (Delete with useTransition)

```tsx
// app/notes/NoteCard.tsx
"use client";

import { useTransition } from 'react';
import { deleteNoteAction } from './actions';
import type { Note } from '@/lib/db';

export function NoteCard({ note }: { note: Note }) {
  const [isPending, startTransition] = useTransition();

  function handleDelete() {
    startTransition(async () => {
      await deleteNoteAction(note.id);
    });
  }

  return (
    <article style={{ opacity: isPending ? 0.5 : 1 }}>
      <h2>{note.title}</h2>
      <p>{note.content}</p>
      <button onClick={handleDelete} disabled={isPending}>
        {isPending ? 'Deleting...' : 'Delete'}
      </button>
    </article>
  );
}
```

---

## Example 2: Parallel Data Fetching with Suspense

```tsx
// app/dashboard/page.tsx — Server Component
import { Suspense } from 'react';
import { UserProfile } from './UserProfile';
import { RecentActivity } from './RecentActivity';
import { Statistics } from './Statistics';

export default function DashboardPage() {
  return (
    <main>
      <h1>Dashboard</h1>
      <div className="grid">
        {/* Each Suspense boundary streams independently */}
        <Suspense fallback={<div className="skeleton">Loading profile...</div>}>
          <UserProfile />
        </Suspense>
        <Suspense fallback={<div className="skeleton">Loading activity...</div>}>
          <RecentActivity />
        </Suspense>
        <Suspense fallback={<div className="skeleton">Loading stats...</div>}>
          <Statistics />
        </Suspense>
      </div>
    </main>
  );
}
```

```tsx
// app/dashboard/UserProfile.tsx — Server Component
import { getCurrentUser } from '@/lib/auth';

export async function UserProfile() {
  const user = await getCurrentUser(); // fetches on server
  return (
    <section>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <p>Member since {user.createdAt.toLocaleDateString()}</p>
    </section>
  );
}
```

```tsx
// app/dashboard/Statistics.tsx — Server Component
import { getStats } from '@/lib/analytics';

export async function Statistics() {
  const stats = await getStats(); // independent fetch, streams when ready
  return (
    <section>
      <h2>This Month</h2>
      <dl>
        <dt>Page Views</dt><dd>{stats.pageViews.toLocaleString()}</dd>
        <dt>Users</dt><dd>{stats.activeUsers.toLocaleString()}</dd>
        <dt>Revenue</dt><dd>${stats.revenue.toFixed(2)}</dd>
      </dl>
    </section>
  );
}
```

---

## Example 3: Server Component with Promise Streaming to Client

```tsx
// app/products/[id]/page.tsx — Server Component
import { Suspense } from 'react';
import { getProduct, getReviews } from '@/lib/db';
import { AddToCartButton } from './AddToCartButton';
import { ReviewList } from './ReviewList';

export default async function ProductPage({ params }: { params: { id: string } }) {
  // Await critical data (blocks render)
  const product = await getProduct(params.id);
  if (!product) return <p>Product not found</p>;

  // Start non-critical data as promise (streams to client)
  const reviewsPromise = getReviews(product.id);

  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p className="price">${product.price.toFixed(2)}</p>
      <AddToCartButton productId={product.id} />

      <Suspense fallback={<p>Loading reviews...</p>}>
        <ReviewList reviewsPromise={reviewsPromise} />
      </Suspense>
    </main>
  );
}
```

```tsx
// app/products/[id]/ReviewList.tsx — Client Component
"use client";

import { use } from 'react';

interface Review {
  id: string;
  author: string;
  rating: number;
  text: string;
}

export function ReviewList({ reviewsPromise }: { reviewsPromise: Promise<Review[]> }) {
  const reviews = use(reviewsPromise);

  if (reviews.length === 0) return <p>No reviews yet.</p>;

  return (
    <section>
      <h2>Reviews ({reviews.length})</h2>
      {reviews.map(review => (
        <article key={review.id}>
          <strong>{review.author}</strong>
          <span>{'★'.repeat(review.rating)}{'☆'.repeat(5 - review.rating)}</span>
          <p>{review.text}</p>
        </article>
      ))}
    </section>
  );
}
```

---

## Example 4: Optimistic UI with Server Actions

```tsx
// app/todos/actions.ts
"use server";

import { revalidatePath } from 'next/cache';
import { db } from '@/lib/db';

export async function addTodo(text: string): Promise<{ id: string; text: string }> {
  const todo = await db.todo.create({ data: { text, completed: false } });
  revalidatePath('/todos');
  return { id: todo.id, text: todo.text };
}

export async function toggleTodo(id: string): Promise<void> {
  const todo = await db.todo.findUnique({ where: { id } });
  if (todo) {
    await db.todo.update({ where: { id }, data: { completed: !todo.completed } });
  }
  revalidatePath('/todos');
}
```

```tsx
// app/todos/TodoList.tsx — Client Component with optimistic updates
"use client";

import { useOptimistic, useTransition } from 'react';
import { addTodo, toggleTodo } from './actions';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  pending?: boolean;
}

export function TodoList({ initialTodos }: { initialTodos: Todo[] }) {
  const [isPending, startTransition] = useTransition();
  const [optimisticTodos, setOptimisticTodos] = useOptimistic(
    initialTodos,
    (currentTodos: Todo[], action: { type: 'add'; text: string } | { type: 'toggle'; id: string }) => {
      if (action.type === 'add') {
        return [...currentTodos, { id: crypto.randomUUID(), text: action.text, completed: false, pending: true }];
      }
      if (action.type === 'toggle') {
        return currentTodos.map(t =>
          t.id === action.id ? { ...t, completed: !t.completed, pending: true } : t
        );
      }
      return currentTodos;
    }
  );

  function handleAdd(formData: FormData) {
    const text = formData.get('text') as string;
    if (!text.trim()) return;

    startTransition(async () => {
      setOptimisticTodos({ type: 'add', text });
      await addTodo(text);
    });
  }

  function handleToggle(id: string) {
    startTransition(async () => {
      setOptimisticTodos({ type: 'toggle', id });
      await toggleTodo(id);
    });
  }

  return (
    <div>
      <form action={handleAdd}>
        <input name="text" placeholder="New todo..." required />
        <button type="submit">Add</button>
      </form>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} style={{ opacity: todo.pending ? 0.6 : 1 }}>
            <label>
              <input
                type="checkbox"
                checked={todo.completed}
                onChange={() => handleToggle(todo.id)}
              />
              <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
                {todo.text}
              </span>
            </label>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## Example 5: Layout with Mixed Server/Client Components

```tsx
// app/layout.tsx — Server Component (Root Layout)
import { Navigation } from './Navigation';
import { Footer } from './Footer';
import { ThemeProvider } from './ThemeProvider';
import { getCurrentUser } from '@/lib/auth';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const user = await getCurrentUser();

  return (
    <html lang="en">
      <body>
        <ThemeProvider>
          <Navigation user={user} />
          {children}
          <Footer />
        </ThemeProvider>
      </body>
    </html>
  );
}
```

```tsx
// app/ThemeProvider.tsx — Client Component (context requires client)
"use client";

import { createContext, useState, type ReactNode } from 'react';

export const ThemeContext = createContext<{ theme: string; toggle: () => void }>({
  theme: 'light',
  toggle: () => {},
});

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState('light');
  const toggle = () => setTheme(t => (t === 'light' ? 'dark' : 'light'));

  return (
    <ThemeContext value={{ theme, toggle }}>
      <div data-theme={theme}>{children}</div>
    </ThemeContext>
  );
}
```

```tsx
// app/Navigation.tsx — Client Component (needs onClick for mobile menu)
"use client";

import { useState } from 'react';

interface User {
  name: string;
  avatarUrl: string;
}

export function Navigation({ user }: { user: User | null }) {
  const [menuOpen, setMenuOpen] = useState(false);

  return (
    <nav>
      <a href="/">Home</a>
      <button onClick={() => setMenuOpen(!menuOpen)} aria-label="Toggle menu">
        Menu
      </button>
      {menuOpen && (
        <ul>
          <li><a href="/dashboard">Dashboard</a></li>
          <li><a href="/notes">Notes</a></li>
          {user && <li>{user.name}</li>}
        </ul>
      )}
    </nav>
  );
}
```

```tsx
// app/Footer.tsx — Server Component (no interactivity needed)
export function Footer() {
  const year = new Date().getFullYear();
  return (
    <footer>
      <p>&copy; {year} My App. All rights reserved.</p>
    </footer>
  );
}
```
