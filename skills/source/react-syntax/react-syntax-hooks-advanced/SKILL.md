---
name: react-syntax-hooks-advanced
description: >
  Use when working with advanced hooks, integrating external stores, or using
  React 19 form and promise hooks. Prevents the common mistake of using useEffect
  for subscriptions instead of useSyncExternalStore. Covers useId, useTransition,
  useDeferredValue, useSyncExternalStore, useInsertionEffect, useDebugValue,
  React 19 use(), useActionState, useOptimistic, useFormStatus.
  Keywords: useId, useSyncExternalStore, use(), useActionState, useOptimistic, custom hook, external store, form status, optimistic update, advanced React patterns..
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-syntax-hooks-advanced

## Quick Reference

### Advanced Hooks at a Glance

| Hook | Version | Purpose | Returns |
|------|---------|---------|---------|
| `useId` | 18+ | SSR-safe unique IDs for accessibility | `string` |
| `useTransition` | 18+ | Mark state updates as non-blocking | `[isPending, startTransition]` |
| `useDeferredValue` | 18+ | Defer expensive re-renders | `T` (deferred value) |
| `useSyncExternalStore` | 18+ | Subscribe to external stores | `T` (store snapshot) |
| `useInsertionEffect` | 18+ | Inject styles before DOM mutations | `void` |
| `useDebugValue` | 18+ | Label custom hooks in DevTools | `void` |
| `use` | **19+** | Read promises (Suspense) and context conditionally | `T` |
| `useActionState` | **19+** | Manage async action state with pending flag | `[state, dispatch, isPending]` |
| `useOptimistic` | **19+** | Optimistic UI during async actions | `[optimisticState, setOptimistic]` |
| `useFormStatus` | **19+** | Read parent form submission state | `{pending, data, method, action}` |

### Critical Warnings

**NEVER** use `useId` to generate keys for list items -- keys MUST come from your data. `useId` generates IDs for accessibility attributes only.

**NEVER** use `useInsertionEffect` in application code -- it exists ONLY for CSS-in-JS library authors. Use `useEffect` or `useLayoutEffect` instead.

**NEVER** call `useFormStatus` in the same component that renders the `<form>` -- it MUST be called from a child component rendered inside the form.

**NEVER** call `use()` inside a `try`/`catch` block -- use an Error Boundary for error handling with promises.

**NEVER** wrap controlled text input updates in `startTransition` -- transitions are interruptible and will cause the input to feel broken.

---

## Decision Tree: Which Advanced Hook?

```
Need a unique ID for aria/htmlFor?
  YES --> useId

Need to keep UI responsive during expensive state update?
  YES --> Do you have the state setter?
           YES --> useTransition (wrap setState in startTransition)
           NO  --> useDeferredValue (defer the prop/value)

Need to subscribe to non-React state (Redux, browser API, etc.)?
  YES --> useSyncExternalStore

Need to inject <style> tags before DOM mutations?
  YES --> Are you building a CSS-in-JS library?
           YES --> useInsertionEffect
           NO  --> useEffect or useLayoutEffect

Need to read a Promise with Suspense? (React 19)
  YES --> use(promise)

Need to read context conditionally? (React 19)
  YES --> use(context)

Need form action with pending state and error handling? (React 19)
  YES --> useActionState

Need instant UI feedback before async action completes? (React 19)
  YES --> useOptimistic

Need form submission status in a child component? (React 19)
  YES --> useFormStatus
```

---

## React 18 Advanced Hooks

### useId

Generates a unique string ID that is stable across server and client renders.

```tsx
import { useId } from 'react';

function EmailField(): JSX.Element {
  const id = useId();
  return (
    <div>
      <label htmlFor={id + '-email'}>Email</label>
      <input id={id + '-email'} type="email" aria-describedby={id + '-hint'} />
      <p id={id + '-hint'}>We will never share your email.</p>
    </div>
  );
}
```

ALWAYS use `useId` as a prefix when generating multiple related IDs. ALWAYS pass `identifierPrefix` to the root when running multiple React apps on the same page.

---

### useTransition

Marks state updates as non-urgent so they do not block user interaction.

```tsx
import { useState, useTransition } from 'react';

function TabContainer(): JSX.Element {
  const [tab, setTab] = useState<string>('home');
  const [isPending, startTransition] = useTransition();

  function selectTab(nextTab: string): void {
    startTransition(() => {
      setTab(nextTab);
    });
  }

  return (
    <div>
      <button onClick={() => selectTab('about')}>About</button>
      {isPending && <span>Loading...</span>}
      <TabPanel tab={tab} />
    </div>
  );
}
```

`startTransition` executes its callback immediately -- the state update inside is what becomes interruptible. ALWAYS wrap `startTransition` inside `setTimeout`, NEVER the other way around. State updates after `await` inside an async `startTransition` require a nested `startTransition` call.

---

### useDeferredValue

Returns a deferred copy of a value that lags behind during urgent updates.

```tsx
import { useDeferredValue, useMemo } from 'react';

function SearchResults({ query }: { query: string }): JSX.Element {
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  const results = useMemo(() => filterResults(deferredQuery), [deferredQuery]);

  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <ResultsList items={results} />
    </div>
  );
}
```

> **React 19+** adds an optional `initialValue` parameter:
> ```tsx
> const deferredQuery = useDeferredValue(query, ''); // '' on first render
> ```

ALWAYS use primitives or objects created outside the render function. Passing a new object every render defeats the purpose of deferral.

---

### useSyncExternalStore

Subscribes a component to an external (non-React) data source with tear-free reads.

```tsx
import { useSyncExternalStore } from 'react';

function useOnlineStatus(): boolean {
  return useSyncExternalStore(
    subscribe,
    () => navigator.onLine,          // client snapshot
    () => true                        // server snapshot (SSR)
  );
}

function subscribe(callback: () => void): () => void {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}
```

ALWAYS declare the `subscribe` function outside the component or wrap it in `useCallback` -- a new function reference causes resubscription every render. `getSnapshot` MUST return immutable data; returning a new object every call causes an infinite re-render loop.

---

### useInsertionEffect

Fires before React makes any DOM changes. Reserved for CSS-in-JS libraries.

```tsx
import { useInsertionEffect } from 'react';

// ONLY for CSS-in-JS library authors
function useCSS(rule: string): string {
  useInsertionEffect(() => {
    const style = document.createElement('style');
    style.textContent = rule;
    document.head.appendChild(style);
    return () => { document.head.removeChild(style); };
  });
  return rule;
}
```

Cannot call `setState` inside. Refs are NOT attached yet. Does NOT run during SSR.

---

### useDebugValue

Adds a label to custom hooks visible in React DevTools.

```tsx
import { useDebugValue } from 'react';

function useOnlineStatus(): boolean {
  const isOnline = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
  useDebugValue(isOnline ? 'Online' : 'Offline');
  return isOnline;
}
```

ALWAYS pass a formatter function for expensive formatting to avoid cost when DevTools is closed:

```tsx
useDebugValue(date, (d: Date) => d.toISOString());
```

---

## React 19 Hooks

### use()

> **React 19+**

Reads a Promise (triggering Suspense) or a Context value. Unlike all other hooks, `use()` CAN be called inside conditionals, loops, and after early returns.

```tsx
import { use, Suspense } from 'react';

function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }): JSX.Element {
  const comments = use(commentsPromise);
  return <ul>{comments.map(c => <li key={c.id}>{c.text}</li>)}</ul>;
}

// Conditional context reading -- impossible with useContext
function Heading({ children }: { children: React.ReactNode | null }): JSX.Element | null {
  if (children == null) return null;
  const theme = use(ThemeContext);
  return <h1 style={{ color: theme.color }}>{children}</h1>;
}
```

ALWAYS create promises in Server Components and pass them to Client Components -- promises created during client render are recreated every render. ALWAYS wrap the consuming component in `<Suspense>` and `<ErrorBoundary>`.

---

### useActionState

> **React 19+**

Manages state from form actions with built-in pending tracking. Replaces the deprecated `useFormState`.

```tsx
import { useActionState } from 'react';

async function submitForm(
  prevState: { error: string | null },
  formData: FormData
): Promise<{ error: string | null }> {
  const name = formData.get('name') as string;
  if (!name) return { error: 'Name is required' };
  await saveToDatabase(name);
  return { error: null };
}

function ContactForm(): JSX.Element {
  const [state, dispatch, isPending] = useActionState(submitForm, { error: null });

  return (
    <form action={dispatch}>
      <input name="name" disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
      {state.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

Multiple dispatches are queued sequentially -- each receives the previous action's return value. Errors cancel all queued actions and trigger the nearest Error Boundary.

---

### useOptimistic

> **React 19+**

Shows an optimistic state while an async action is in progress. Automatically reverts when the action completes.

```tsx
import { useOptimistic, startTransition } from 'react';

function LikeButton({ liked, onToggle }: {
  liked: boolean;
  onToggle: (next: boolean) => Promise<void>;
}): JSX.Element {
  const [optimisticLiked, setOptimisticLiked] = useOptimistic(liked);

  function handleClick(): void {
    startTransition(async () => {
      setOptimisticLiked(!optimisticLiked);
      await onToggle(!optimisticLiked);
    });
  }

  return <button onClick={handleClick}>{optimisticLiked ? 'Unlike' : 'Like'}</button>;
}
```

`setOptimistic` MUST be called inside a `startTransition` or an Action prop. The reducer (if provided) MUST be pure -- no side effects.

---

### useFormStatus

> **React 19+**

Reads the submission status of the nearest parent `<form>`. Imported from `react-dom`.

```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton(): JSX.Element {
  const { pending, data } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

function MyForm(): JSX.Element {
  return (
    <form action={handleSubmit}>
      <input name="email" type="email" />
      <SubmitButton />  {/* MUST be a child of <form> */}
    </form>
  );
}
```

Returns `{ pending, data, method, action }`. `pending` is ALWAYS `false` when called outside a `<form>` or in the same component that renders the `<form>`.

---

## Reference Links

- [references/examples.md](references/examples.md) -- Advanced hook patterns and compositions
- [references/api-table.md](references/api-table.md) -- Complete hook signatures with TypeScript types
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do with advanced hooks

### Official Sources

- https://react.dev/reference/react/useId
- https://react.dev/reference/react/useTransition
- https://react.dev/reference/react/useDeferredValue
- https://react.dev/reference/react/useSyncExternalStore
- https://react.dev/reference/react/useInsertionEffect
- https://react.dev/reference/react/useDebugValue
- https://react.dev/reference/react/use
- https://react.dev/reference/react/useActionState
- https://react.dev/reference/react/useOptimistic
- https://react.dev/reference/react-dom/hooks/useFormStatus
