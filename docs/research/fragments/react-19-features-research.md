# React 19 Features & Server Components — Research Document

> **Part 2 of vooronderzoek for React Claude Skill Package**
> Date: 2026-03-19
> Sources: All claims sourced to react.dev official documentation

---

## Table of Contents

- [S1: React 19 Release Overview](#s1-react-19-release-overview)
- [S2: New Hooks](#s2-new-hooks)
- [S3: Server Components](#s3-server-components)
- [S4: Server Functions](#s4-server-functions)
- [S5: React Compiler](#s5-react-compiler)
- [S6: Static Prerendering APIs](#s6-static-prerendering-apis)
- [S7: Other React 19 Improvements](#s7-other-react-19-improvements)
- [S8: Breaking Changes & Deprecations](#s8-breaking-changes--deprecations)
- [S9: Migration Guidance (React 18 to 19)](#s9-migration-guidance-react-18-to-19)

---

## S1: React 19 Release Overview

**Source**: https://react.dev/blog/2024/12/05/react-19

React 19 was released as stable on **December 5, 2024** (RC announced April 25, 2024). It introduces a unified "Actions" system for data mutations, four new hooks, Server Components (stable), React Compiler, and significant DX improvements.

### Core Themes

1. **Actions** — first-class async data mutation primitives with automatic pending state, error handling, optimistic updates, and form reset
2. **Server Components** — stable; render on the server, zero bundle cost for server-only code
3. **React Compiler** — automatic memoization replacing manual `useMemo`/`useCallback`/`React.memo`
4. **Document metadata** — native `<title>`, `<meta>`, `<link>` hoisting from components
5. **Resource loading** — built-in stylesheet, script, and font preloading APIs
6. **ref as prop** — `forwardRef` no longer needed for function components
7. **Improved error reporting** — single consolidated error with diff for hydration mismatches

---

## S2: New Hooks

### S2.1: `use()` — Read Promises and Context Conditionally

**Source**: https://react.dev/reference/react/use

**React 19 ONLY** — not backported to React 18.

```ts
const value = use(resource);
// resource: Promise<T> | React.Context<T>
// returns: T
```

**Key difference from other hooks**: `use()` CAN be called inside conditionals, loops, and after early returns. This is unique among React APIs.

#### Reading Promises (Suspense integration)

```tsx
import { use, Suspense } from 'react';

function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise); // suspends until resolved
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}

function Page({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}
```

#### Reading Context Conditionally

```tsx
import { use } from 'react';
import ThemeContext from './ThemeContext';

function Heading({ children }: { children: React.ReactNode | null }) {
  if (children == null) return null;

  // Valid after early return — impossible with useContext
  const theme = use(ThemeContext);
  return <h1 style={{ color: theme.color }}>{children}</h1>;
}
```

#### Rules and Caveats

- MUST be called inside a Component or Hook function
- CANNOT be called inside try-catch blocks (use Error Boundary instead)
- When used with promises: resolved values MUST be serializable when crossing server-client boundary
- ALWAYS prefer creating promises in Server Components and passing to Client Components (stable across re-renders)
- Promises created during client render are recreated every render (unstable)

#### Error Handling

- **Option 1**: Wrap in `<ErrorBoundary>` — catches rejected promises
- **Option 2**: Use `Promise.catch()` to provide fallback value
- **NOT supported**: try-catch around `use()` calls

#### `await` vs `use()` in Server Components

| Pattern | Behavior |
|---------|----------|
| `await` in Server Component | Blocks rendering until resolved |
| `use()` in Client Component with promise from server | Non-blocking, shows Suspense fallback |

---

### S2.2: `useActionState()` — Manage Action State and Pending

**Source**: https://react.dev/reference/react/useActionState

**React 19 ONLY** — replaces the deprecated `ReactDOM.useFormState` from React 18 canary.

```ts
const [state, dispatchAction, isPending] = useActionState(
  reducerAction: (previousState: T, actionPayload: P) => T | Promise<T>,
  initialState: T,
  permalink?: string
);
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `reducerAction` | `(prevState: T, payload: P) => T \| Promise<T>` | Called when action triggers. Can be async. Can perform side effects (unlike `useReducer`). |
| `initialState` | `T` | Initial state value. Must be serializable when using Server Functions. |
| `permalink` | `string` (optional) | URL for progressive enhancement. If form submits before JS loads, browser navigates here. |

#### Return Values

| Index | Name | Type | Description |
|-------|------|------|-------------|
| 0 | `state` | `T` | Current state (initially `initialState`) |
| 1 | `dispatchAction` | `(payload: P) => void` | Triggers `reducerAction`. Stable identity. |
| 2 | `isPending` | `boolean` | `true` while action is in flight |

#### Usage with Forms

```tsx
import { useActionState } from 'react';

async function formAction(prevState: { error: string | null }, formData: FormData) {
  const name = formData.get('name') as string;
  const result = await submitForm(name);
  if (result.error) return { error: result.error };
  return { error: null };
}

function Form() {
  const [state, submitAction, isPending] = useActionState(formAction, { error: null });

  return (
    <form action={submitAction}>
      <input name="name" disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>
      {state.error && <span className="error">{state.error}</span>}
    </form>
  );
}
```

#### Usage Outside Forms (with startTransition)

```tsx
import { useActionState, startTransition } from 'react';

function Counter() {
  const [count, dispatch, isPending] = useActionState(
    async (prev: number, payload: { type: string }) => {
      if (payload.type === 'ADD') return await addToCart(prev);
      return prev;
    },
    0
  );

  function handleClick() {
    startTransition(() => {
      dispatch({ type: 'ADD' });
    });
  }

  return <button onClick={handleClick} disabled={isPending}>Count: {count}</button>;
}
```

#### Action Queuing

Multiple `dispatchAction` calls are queued and executed sequentially. Each receives the `previousState` from the prior call's return value. Errors cancel all queued actions.

#### Comparison: useActionState vs useReducer

| Feature | `useReducer` | `useActionState` |
|---------|-------------|-----------------|
| Side effects | Reducer MUST be pure | Supports async/side effects |
| Form integration | Manual | Built-in via `action` prop |
| Progressive enhancement | No | Yes (via `permalink`) |
| State updates | Immediate (synchronous) | Sequential/queued |
| Server Functions | No | Yes |
| Pending state | Manual | Built-in `isPending` |

---

### S2.3: `useOptimistic()` — Instant UI Feedback During Async Actions

**Source**: https://react.dev/reference/react/useOptimistic

**React 19 ONLY** — not backported.

```ts
const [optimisticState, setOptimistic] = useOptimistic<T, A>(
  value: T,
  reducer?: (currentState: T, action: A) => T
);
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `value` | `T` | Baseline state. Returned when no action is pending. |
| `reducer` | `(current: T, action: A) => T` (optional) | Pure function to compute optimistic state. |

#### Return Values

| Index | Name | Type | Description |
|-------|------|------|-------------|
| 0 | `optimisticState` | `T` | Current optimistic state. Equals `value` when no action pending. |
| 1 | `setOptimistic` | `(action: A) => void` | Call inside an Action or `startTransition`. |

#### How It Works

1. `setOptimistic(newValue)` called — React immediately re-renders with optimistic value
2. Async action runs in background
3. When action completes: optimistic state converges with real state in a single render
4. If action throws: state reverts to baseline `value` (no extra render needed)

#### Pattern: Simple Toggle

```tsx
import { useOptimistic, startTransition } from 'react';

function LikeButton({ isLiked, toggleLike }: { isLiked: boolean; toggleLike: (v: boolean) => Promise<boolean> }) {
  const [optimisticIsLiked, setOptimisticIsLiked] = useOptimistic(isLiked);

  function handleClick() {
    startTransition(async () => {
      setOptimisticIsLiked(!optimisticIsLiked);
      await toggleLike(!optimisticIsLiked);
    });
  }

  return <button onClick={handleClick}>{optimisticIsLiked ? 'Unlike' : 'Like'}</button>;
}
```

#### Pattern: List with Reducer

```tsx
const [optimisticTodos, addOptimisticTodo] = useOptimistic(
  todos,
  (currentTodos: Todo[], newTodo: Todo) => [
    ...currentTodos,
    { ...newTodo, pending: true }
  ]
);

function handleAdd(text: string) {
  const newTodo = { id: crypto.randomUUID(), text };
  startTransition(async () => {
    addOptimisticTodo(newTodo);
    await saveTodoAction(newTodo);
  });
}
```

#### Rules

- MUST be called at component top level (standard hook rules)
- `setOptimistic` MUST be called inside an Action or `startTransition`
- Reducer MUST be pure (no side effects)

---

### S2.4: `useFormStatus()` — Read Parent Form Submission State

**Source**: https://react.dev/reference/react-dom/hooks/useFormStatus

**Note**: Available since React 18 canary releases; stable in React 19.

```ts
const { pending, data, method, action } = useFormStatus();
// No parameters
```

#### Return Values

| Property | Type | Description |
|----------|------|-------------|
| `pending` | `boolean` | `true` if parent `<form>` is submitting |
| `data` | `FormData \| null` | Form data being submitted; `null` when idle |
| `method` | `'get' \| 'post'` | HTTP method of parent form (default: `'get'`) |
| `action` | `function \| null` | Reference to form's `action` prop function; `null` if URI or no parent form |

#### Critical Rule

The component calling `useFormStatus` MUST be rendered **inside** a `<form>`. It reads the **parent** form's status only.

```tsx
// WRONG — pending is ALWAYS false
function Form() {
  const { pending } = useFormStatus(); // same component as <form>
  return <form action={submit}><button disabled={pending}>Go</button></form>;
}

// CORRECT — separate child component
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button type="submit" disabled={pending}>{pending ? 'Submitting...' : 'Submit'}</button>;
}

function Form() {
  return (
    <form action={submit}>
      <input name="name" />
      <SubmitButton />
    </form>
  );
}
```

---

## S3: Server Components

**Source**: https://react.dev/reference/rsc/server-components

**React 19**: Server Components are **stable**. The component API will not break between minor versions. However, the underlying bundler/framework APIs do NOT follow semver and may change between React 19.x minors.

### Core Concepts

Server Components render **ahead of time**, in an environment separate from the client app. They can run:
- **At build time** on a CI server (static content)
- **Per request** on a web server (dynamic content)

Server Components are the **default** — no directive needed. Client Components require `"use client"`.

### Server vs Client Component Rules

| Capability | Server Component | Client Component |
|-----------|-----------------|-----------------|
| `async/await` in render | YES | NO |
| Direct DB/filesystem access | YES | NO |
| `useState` | NO | YES |
| `useEffect` | NO | YES |
| Event handlers (`onClick`) | NO | YES |
| Browser APIs (`localStorage`) | NO | YES |
| `useContext` / `use(Context)` | NO (read-only via `use`) | YES |
| Import Server Functions | YES (inline `'use server'`) | YES (import from `'use server'` file) |

### Directive Rules

- **Server Components**: NO directive needed (default)
- **Client Components**: MUST have `"use client"` at top of file
- `"use server"` is for **Server Functions only**, NOT for marking Server Components

### Async Server Components

```tsx
// Server Component — async/await allowed
async function Page({ id }: { id: string }) {
  const note = await db.notes.get(id);
  // Start promise on server, complete on client
  const commentsPromise = db.comments.get(note.id);

  return (
    <div>
      <h1>{note.title}</h1>
      <Suspense fallback={<p>Loading comments...</p>}>
        <Comments commentsPromise={commentsPromise} />
      </Suspense>
    </div>
  );
}
```

```tsx
// Client Component — consumes server promise
"use client";
import { use } from 'react';

function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise);
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}
```

### Serialization Rules (Server-to-Client Boundary)

**CAN cross the boundary as props**:
- Serializable primitives (string, number, boolean, null, undefined)
- Arrays and plain objects of serializable values
- Promises (consumable via `use()` on client)
- JSX (rendered output)
- Server Functions (references)

**CANNOT cross the boundary**:
- Functions (except Server Functions)
- Class instances
- Database connections
- Secrets/API keys
- Non-serializable objects (Symbols, etc.)

### Bundle Size Benefit

```tsx
// WITHOUT Server Components — libraries ship to client
import marked from 'marked';           // 35.9K gzipped
import sanitizeHtml from 'sanitize-html'; // 63.3K gzipped

// WITH Server Components — zero client bundle cost
async function Page({ page }: { page: string }) {
  const content = await fs.readFile(`${page}.md`, 'utf8');
  return <div>{sanitizeHtml(marked(content))}</div>;
}
```

### Composition Pattern

Server Components can render Client Components as children. Client Components can receive Server Component output as `children` prop:

```tsx
// Server Component
import Expandable from './Expandable'; // Client Component

async function Notes() {
  const notes = await db.notes.getAll();
  return (
    <div>
      {notes.map(note => (
        <Expandable key={note.id}>
          <p>{note.text}</p>  {/* Server-rendered, passed as children */}
        </Expandable>
      ))}
    </div>
  );
}
```

```tsx
// Client Component
"use client";
import { useState } from 'react';

export default function Expandable({ children }: { children: React.ReactNode }) {
  const [expanded, setExpanded] = useState(false);
  return (
    <div>
      <button onClick={() => setExpanded(!expanded)}>Toggle</button>
      {expanded && children}
    </div>
  );
}
```

### What the Browser Receives

The browser NEVER receives Server Component source code. It receives:
- Pre-rendered HTML/JSX output from Server Components
- Client Component code for hydration
- Serialized props crossing the boundary

---

## S4: Server Functions

**Source**: https://react.dev/reference/rsc/server-functions

**React 19**: Stable API. Underlying bundler APIs may change between minors.

### Terminology

- **Server Functions**: ALL functions marked with `"use server"`
- **Server Actions**: Server Functions specifically passed to `action` props or called from Actions

### Creating Server Functions

#### Method 1: Inline in Server Components

```tsx
// Server Component
import Button from './Button';

function EmptyNote() {
  async function createNoteAction() {
    'use server';
    await db.notes.create();
  }

  return <Button onClick={createNoteAction} />;
}
```

#### Method 2: Dedicated File (importable by Client Components)

```tsx
// actions.ts
"use server";

export async function createNote(): Promise<void> {
  await db.notes.create();
}

export async function updateName(name: string): Promise<{ error?: string }> {
  if (!name) return { error: 'Name is required' };
  await db.users.updateName(name);
  return {};
}
```

```tsx
// ClientForm.tsx
"use client";
import { createNote } from './actions';

function EmptyNote() {
  return <button onClick={() => createNote()}>Create</button>;
}
```

### How Server Functions Work (Framework Mechanics)

When a Server Function is passed to a Client Component, the framework:
1. Creates a serializable reference: `{ $$typeof: Symbol.for("react.server.reference"), $$id: 'functionName' }`
2. Sends this reference (not the function) to the client
3. When the client calls it, React sends a request to the server
4. The server executes the function and returns the result

### Form Actions with Server Functions

```tsx
"use client";
import { updateName } from './actions';

function UpdateName() {
  return (
    <form action={updateName}>
      <input type="text" name="name" />
      <button type="submit">Update</button>
    </form>
  );
}
```

React automatically resets the form on successful submission.

### With useActionState for Pending + Error State

```tsx
"use client";
import { useActionState } from 'react';
import { updateName } from './actions';

function UpdateName() {
  const [state, submitAction, isPending] = useActionState(updateName, { error: null });

  return (
    <form action={submitAction}>
      <input type="text" name="name" disabled={isPending} />
      {state.error && <span>Failed: {state.error}</span>}
      <button type="submit" disabled={isPending}>Save</button>
    </form>
  );
}
```

### Progressive Enhancement

```tsx
"use client";
import { useActionState } from 'react';
import { updateName } from './actions';

function UpdateName() {
  const [, submitAction] = useActionState(updateName, null, '/name/update');
  // permalink '/name/update' — form works before JS loads

  return (
    <form action={submitAction}>
      <input type="text" name="name" />
      <button type="submit">Save</button>
    </form>
  );
}
```

### Calling Server Functions Outside Forms (useTransition)

```tsx
"use client";
import { useState, useTransition } from 'react';
import { updateName } from './actions';

function UpdateName() {
  const [name, setName] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    startTransition(async () => {
      const result = await updateName(name);
      if (result.error) {
        setError(result.error);
      } else {
        setName('');
      }
    });
  };

  return (
    <form action={handleSubmit}>
      <input value={name} onChange={e => setName(e.target.value)} disabled={isPending} />
      {error && <span>{error}</span>}
      <button type="submit" disabled={isPending}>Save</button>
    </form>
  );
}
```

### Return Types

Server Functions can return any serializable value, including error objects:

```tsx
"use server";

export async function updateName(name: string): Promise<{ error?: string }> {
  if (!name) return { error: 'Name is required' };
  await db.users.updateName(name);
  return {};
}
```

---

## S5: React Compiler

**Sources**:
- https://react.dev/learn/react-compiler
- https://react.dev/learn/react-compiler/installation
- https://react.dev/reference/react-compiler/configuration
- https://react.dev/reference/react-compiler/directives

### What It Does

React Compiler automatically optimizes React applications by applying **memoization at build time**. It replaces manual usage of:
- `useMemo()`
- `useCallback()`
- `React.memo()`

The compiler analyzes component code and automatically caches values, function references, and JSX that don't need to change between renders.

### Compatibility

- **Best with**: React 19
- **Also supports**: React 17 and 18 (requires `react-compiler-runtime` package)
- Works with TypeScript and JSX/TSX

### Installation

```bash
npm install -D babel-plugin-react-compiler@latest
```

**CRITICAL**: The babel plugin MUST run **first** in the plugin pipeline.

### Build Tool Setup

#### Vite (via vite-plugin-react)

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: ['babel-plugin-react-compiler'],
      },
    }),
  ],
});
```

#### Vite (via vite-plugin-babel)

```ts
import babel from 'vite-plugin-babel';
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react(),
    babel({
      babelConfig: {
        plugins: ['babel-plugin-react-compiler'],
      },
    }),
  ],
});
```

#### Babel (generic)

```js
// babel.config.js
module.exports = {
  plugins: [
    'babel-plugin-react-compiler', // MUST be first
    // ... other plugins
  ],
};
```

#### React Router (Remix)

```ts
import { defineConfig } from "vite";
import babel from "vite-plugin-babel";
import { reactRouter } from "@react-router/dev/vite";

const ReactCompilerConfig = { /* ... */ };

export default defineConfig({
  plugins: [
    reactRouter(),
    babel({
      filter: /\.[jt]sx?$/,
      babelConfig: {
        presets: ["@babel/preset-typescript"],
        plugins: [
          ["babel-plugin-react-compiler", ReactCompilerConfig],
        ],
      },
    }),
  ],
});
```

#### Next.js

See https://nextjs.org/docs/app/api-reference/next-config-js/reactCompiler

#### Webpack

Community loader: https://github.com/SukkaW/react-compiler-webpack

### ESLint Plugin

```bash
npm install -D eslint-plugin-react-hooks@latest
```

The ESLint plugin identifies Rules of React violations and shows which components cannot be optimized.

### Configuration Options

```ts
interface ReactCompilerConfig {
  compilationMode?: 'annotation' | 'infer' | 'all';
  target?: '17' | '18' | '19';  // default: '19'
  panicThreshold?: 'none' | string;  // 'none' = skip errors instead of failing build
  logger?: {
    logEvent(filename: string, event: CompileEvent): void;
  };
  gating?: {
    source: string;
    importSpecifierName: string;
  };
}
```

| Option | Description |
|--------|-------------|
| `compilationMode` | `'annotation'`: only compile `"use memo"` functions. `'infer'`: auto-decide. `'all'`: compile everything. |
| `target` | React version. `'17'`/`'18'` require `react-compiler-runtime` package. |
| `panicThreshold` | `'none'`: skip components with errors instead of failing build (recommended for production). |
| `logger` | Custom logging for compilation events. |
| `gating` | Runtime feature flags for A/B testing or gradual rollouts. |

### Directives

#### `"use memo"` — Opt In

```tsx
function OptimizedComponent() {
  "use memo";
  return <div>This will be compiled</div>;
}
```

#### `"use no memo"` — Opt Out

```tsx
function ProblematicComponent() {
  "use no memo";
  // Temporarily skip compilation for this component
  return <div>Not compiled</div>;
}
```

Directives can be placed at **module level** (top of file, before imports) to apply to all functions in that file. Function-level directives override module-level ones.

### Compiled Output Example

```tsx
// Input
export default function MyApp() {
  return <div>Hello World</div>;
}

// Output (compiled)
import { c as _c } from "react/compiler-runtime";
export default function MyApp() {
  const $ = _c(1);
  let t0;
  if ($[0] === Symbol.for("react.memo_cache_sentinel")) {
    t0 = <div>Hello World</div>;
    $[0] = t0;
  } else {
    t0 = $[0];
  }
  return t0;
}
```

### Verification

In React DevTools, compiled components show a "Memo" badge with a sparkle emoji next to their name.

---

## S6: Static Prerendering APIs

**Sources**:
- https://react.dev/reference/react-dom/static/prerender
- https://react.dev/reference/react-dom/static/prerenderToNodeStream

**React 19 ONLY** — new in `react-dom/static`.

### `prerender()` — Web Streams

```ts
async function prerender(
  reactNode: React.ReactNode,
  options?: PrerenderOptions
): Promise<{ prelude: ReadableStream<Uint8Array>; postponed: object | null }>
```

### `prerenderToNodeStream()` — Node.js Streams

```ts
async function prerenderToNodeStream(
  reactNode: React.ReactNode,
  options?: PrerenderOptions
): Promise<{ prelude: NodeJS.ReadableStream; postponed: object | null }>
```

### Options

| Option | Type | Description |
|--------|------|-------------|
| `bootstrapScriptContent` | `string` | Inline script content in `<script>` tag |
| `bootstrapScripts` | `string[]` | Script URLs for `<script>` tags (include hydration script) |
| `bootstrapModules` | `string[]` | Module URLs for `<script type="module">` tags |
| `identifierPrefix` | `string` | Prefix for `useId` — must match `hydrateRoot` prefix |
| `namespaceURI` | `string` | Root namespace (default: HTML; `'http://www.w3.org/2000/svg'` for SVG) |
| `onError` | `(error: Error) => void` | Fires on server errors |
| `progressiveChunkSize` | `number` | Bytes per chunk |
| `signal` | `AbortSignal` | Abort prerendering to render rest on client |

**NOT available**: `nonce` — nonces must be unique per request, inappropriate for prerendering.

### Key Behavior

- **Waits for ALL data** before returning (all Suspense boundaries must resolve)
- Only works with Suspense-enabled data sources (`lazy()`, `use()`, framework integrations)
- Does NOT detect data fetched in Effects or event handlers
- Does NOT stream content as it loads (use `renderToReadableStream`/`renderToPipeableStream` for that)

### Usage: Web Streams (Edge/Deno)

```tsx
import { prerender } from 'react-dom/static';

async function handler(request: Request): Promise<Response> {
  const { prelude } = await prerender(<App />, {
    bootstrapScripts: ['/main.js'],
  });
  return new Response(prelude, {
    headers: { 'content-type': 'text/html' },
  });
}
```

### Usage: Node.js

```tsx
import { prerenderToNodeStream } from 'react-dom/static';

app.use('/', async (req, res) => {
  const { prelude } = await prerenderToNodeStream(<App />, {
    bootstrapScripts: ['/main.js'],
  });
  res.setHeader('Content-Type', 'text/html');
  prelude.pipe(res);
});
```

### Aborting with Timeout

```tsx
const controller = new AbortController();
setTimeout(() => controller.abort(), 10000);

const { prelude } = await prerender(<App />, {
  signal: controller.signal,
});
```

Aborted prerender returns partial HTML with Suspense fallbacks for unresolved boundaries. Can be resumed with `resume()` or `resumeAndPrerender()`.

### Comparison with Other APIs

| API | Module | Purpose | Streaming |
|-----|--------|---------|-----------|
| `prerender` | `react-dom/static` | SSG (Web Streams) | No — waits for all data |
| `prerenderToNodeStream` | `react-dom/static` | SSG (Node.js) | No — waits for all data |
| `renderToReadableStream` | `react-dom/server` | SSR (Web Streams) | Yes — streams as data loads |
| `renderToPipeableStream` | `react-dom/server` | SSR (Node.js) | Yes — streams as data loads |
| `renderToString` | `react-dom/server` | Legacy SSR | No — does not wait for Suspense |

---

## S7: Other React 19 Improvements

**Source**: https://react.dev/blog/2024/12/05/react-19

### S7.1: `ref` as a Regular Prop

**React 19 ONLY**.

```tsx
// Before (React 18) — forwardRef required
const MyInput = forwardRef(({ placeholder }: Props, ref: Ref<HTMLInputElement>) => (
  <input placeholder={placeholder} ref={ref} />
));

// After (React 19) — ref is a regular prop
function MyInput({ placeholder, ref }: { placeholder: string; ref?: Ref<HTMLInputElement> }) {
  return <input placeholder={placeholder} ref={ref} />;
}
```

- `forwardRef` will be deprecated and removed in a future version
- Codemod available for migration
- Exception: `ref` to class components is still NOT passed as a prop

### S7.2: Ref Cleanup Functions

**React 19 ONLY**.

```tsx
<input
  ref={(element) => {
    // Setup
    return () => {
      // Cleanup on unmount
    };
  }}
/>
```

- React no longer calls ref callbacks with `null` on unmount
- TypeScript now rejects refs returning anything other than a cleanup function
- Migration codemod: `no-implicit-ref-callback-return`

```diff
- <div ref={current => (instance = current)} />
+ <div ref={current => { instance = current }} />
```

### S7.3: `useDeferredValue` Initial Value

**React 19 ONLY** — new `initialValue` parameter.

```tsx
const value = useDeferredValue(deferredValue, ''); // '' on initial render
```

### S7.4: Document Metadata Support

**React 19 ONLY**.

```tsx
function BlogPost({ post }: { post: Post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <title>{post.title}</title>
      <meta name="author" content="Josh" />
      <link rel="author" href="https://twitter.com/joshcstory/" />
      <meta name="keywords" content={post.keywords} />
      <p>{post.content}</p>
    </article>
  );
}
```

Supported tags (`<title>`, `<meta>`, `<link>`) are automatically hoisted to `<head>`. Works with client-only, SSR, and Server Components.

### S7.5: Stylesheet Support with Precedence

**React 19 ONLY**.

```tsx
function Component() {
  return (
    <Suspense fallback="loading...">
      <link rel="stylesheet" href="styles.css" precedence="default" />
      <link rel="stylesheet" href="critical.css" precedence="high" />
      <article className="styled">Content</article>
    </Suspense>
  );
}
```

- SSR: Stylesheet placed in `<head>`, prevents paint until loaded
- CSR: Waits for stylesheet to load before committing render
- Deduplication: Same URL loaded only once across entire app
- Precedence ordering controls insertion order in `<head>`

### S7.6: Async Script Support

**React 19 ONLY**.

```tsx
function MyComponent() {
  return (
    <div>
      <script async={true} src="https://example.com/script.js" />
      Hello World
    </div>
  );
}
```

- Deduplication across components
- SSR: Included in `<head>`, prioritized after critical resources
- CSR: Loads once even if rendered by multiple components

### S7.7: Resource Preloading APIs

**React 19 ONLY**.

```tsx
import { prefetchDNS, preconnect, preload, preinit } from 'react-dom';

function MyComponent() {
  prefetchDNS('https://example.com');
  preconnect('https://example.com');
  preload('https://example.com/font.woff', { as: 'font' });
  preload('https://example.com/style.css', { as: 'style' });
  preinit('https://example.com/script.js', { as: 'script' });
}
```

### S7.8: Context as Provider

**React 19 ONLY**.

```tsx
// Before
<ThemeContext.Provider value="dark">{children}</ThemeContext.Provider>

// After
<ThemeContext value="dark">{children}</ThemeContext>
```

Codemod available. `<Context.Provider>` will be deprecated in a future version.

### S7.9: Hydration Error Improvements

**React 19 ONLY**.

Old: Multiple duplicate console errors with no context.
New: Single error with a visual diff showing exactly what mismatched:

```
Uncaught Error: Hydration failed because the server rendered HTML didn't match the client...
  <App>
    <span>
+    Client
-    Server
```

### S7.10: Better Error Reporting

**React 19 ONLY**.

Old: 3 duplicate errors logged per failure.
New: 1 error with full component stack.

New root options:

```tsx
const root = createRoot(element, {
  onCaughtError: (error, errorInfo) => {},     // Caught by Error Boundary
  onUncaughtError: (error, errorInfo) => {},   // NOT caught by Error Boundary
  onRecoverableError: (error) => {},           // Auto-recovered
});
```

### S7.11: Custom Elements Support

**React 19 ONLY**. Full Custom Elements Everywhere test suite passing.

- SSR: Primitive types render as attributes; non-primitives omitted
- CSR: Props matching CE properties assigned as properties; others as attributes

### S7.12: Third-Party Script/Extension Compatibility

**React 19 ONLY**. Unexpected tags in `<head>`/`<body>` (from browser extensions) are skipped during hydration. Stylesheets from third-party scripts preserved on re-render.

---

## S8: Breaking Changes & Deprecations

**Source**: https://react.dev/blog/2024/12/05/react-19

### Deprecations

| Feature | Status | Replacement | Migration |
|---------|--------|-------------|-----------|
| `forwardRef` | Will be deprecated | `ref` as regular prop | Codemod available |
| `<Context.Provider>` | Will be deprecated | `<Context value={...}>` | Codemod available |
| `ReactDOM.useFormState` | Deprecated in React 19 | `useActionState` (from `react`) | Rename + add `isPending` |
| Ref called with `null` on unmount | Changed | Ref cleanup functions | Use block syntax `{ }` |

### TypeScript Breaking Changes

- Ref callbacks returning values other than cleanup functions are now rejected
- Requires explicit block syntax to avoid implicit returns:

```diff
- <div ref={current => (instance = current)} />
+ <div ref={current => { instance = current }} />
```

### Upgrade Guide

Full upgrade guide: https://react.dev/blog/2024/04/25/react-19-upgrade-guide

---

## S9: Migration Guidance (React 18 to 19)

### Hooks Migration

| React 18 Pattern | React 19 Replacement |
|------------------|---------------------|
| `useState` + `useEffect` for data fetching | `use()` with Suspense |
| Manual `isPending` state | `useActionState` with built-in `isPending` |
| Manual optimistic updates with `useState` | `useOptimistic` |
| `useFormState` (canary) | `useActionState` (moved from `react-dom` to `react`) |
| Prop drilling form pending state | `useFormStatus` in child component |
| `useMemo` / `useCallback` / `React.memo` | React Compiler (automatic) |

### Component Patterns Migration

| React 18 Pattern | React 19 Replacement |
|------------------|---------------------|
| `forwardRef` wrapper | `ref` as regular prop |
| `<Context.Provider value={...}>` | `<Context value={...}>` |
| `react-helmet` for `<head>` tags | Native `<title>`, `<meta>`, `<link>` in components |
| Manual stylesheet loading | `<link>` with `precedence` attribute |
| Manual script deduplication | Native `<script async>` deduplication |
| Manual resource preloading | `prefetchDNS`, `preconnect`, `preload`, `preinit` |

### Server-Side Migration

| React 18 Pattern | React 19 Replacement |
|------------------|---------------------|
| `renderToString` | `prerender` / `prerenderToNodeStream` (SSG) |
| Client-side API calls | Server Components with direct DB access |
| API routes for mutations | Server Functions with `"use server"` |
| Client-side data fetching waterfalls | Server Component async data fetching |

### What Was Backported to React 18

- `useFormStatus` was available in React 18 canary releases
- `useFormState` was in React 18 canary (now deprecated in favor of `useActionState`)
- Server Components were available in React 18 canary (stable in 19)

### What is React 19 ONLY (Not Backported)

- `use()` API
- `useActionState` hook
- `useOptimistic` hook
- `ref` as regular prop (no `forwardRef`)
- Ref cleanup functions
- `useDeferredValue` initial value parameter
- Document metadata hoisting
- Stylesheet precedence
- Async script deduplication
- Resource preloading APIs (`prefetchDNS`, `preconnect`, `preload`, `preinit`)
- `<Context>` as provider (without `.Provider`)
- Improved hydration error diffs
- `onCaughtError` / `onUncaughtError` root options
- Full Custom Elements support
- React Compiler
- `prerender` / `prerenderToNodeStream` static APIs
