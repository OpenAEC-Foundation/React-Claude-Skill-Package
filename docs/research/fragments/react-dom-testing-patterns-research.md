# React DOM APIs, Testing, Patterns & Error Handling — Research Fragment

**Part 3 of React Vooronderzoek**
**Date**: 2025-03-19
**Sources**: All claims sourced from official documentation (react.dev, testing-library.com)

---

## Table of Contents

- [§1 React DOM APIs](#1-react-dom-apis)
- [§2 React DOM Components & Form Handling](#2-react-dom-components--form-handling)
- [§3 Error Boundaries](#3-error-boundaries)
- [§4 JSX Patterns & Rendering](#4-jsx-patterns--rendering)
- [§5 Performance APIs](#5-performance-apis)
- [§6 Testing with React Testing Library](#6-testing-with-react-testing-library)
- [§7 Common Anti-Patterns & Mistakes](#7-common-anti-patterns--mistakes)

---

## §1 React DOM APIs

**Source**: https://react.dev/reference/react-dom

### §1.1 Core APIs (React 18 + 19)

#### createPortal

Renders children into a different DOM node, outside the parent component's DOM hierarchy.

```typescript
import { createPortal } from 'react-dom';

function createPortal(
  children: React.ReactNode,
  domNode: Element,
  key?: string | number
): React.ReactPortal
```

**Key behaviors**:
- Physical DOM placement changes, but React tree hierarchy is preserved
- Context access works through the React tree (not DOM tree)
- Events bubble according to the **React tree**, NOT the DOM tree — clicking inside a portal wrapped in a `<div onClick>` WILL trigger that handler

**Source**: https://react.dev/reference/react-dom/createPortal

**Use cases**:
- Modals/dialogs that must escape `overflow: hidden` containers
- Tooltips and dropdowns
- Rendering React into non-React server markup
- Managing content inside third-party widget DOM nodes

```typescript
function Modal({ children, onClose }: { children: React.ReactNode; onClose: () => void }) {
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body
  );
}
```

**Caveats**:
- ALWAYS stop event propagation inside portals to prevent unexpected handler firing on ancestors
- ALWAYS ensure the target DOM node exists before rendering
- ALWAYS follow WAI-ARIA Modal Authoring Practices for accessibility

#### flushSync

Forces React to flush state updates and update the DOM synchronously.

```typescript
import { flushSync } from 'react-dom';

function flushSync<R>(callback: () => R): R
```

**Source**: https://react.dev/reference/react-dom/flushSync

**Use sparingly** — breaks React's batching optimization. Only use when you need the DOM to be updated before the next line of code runs (e.g., measuring DOM after a state update).

### §1.2 Client Entry Points (React 18 + 19)

#### createRoot

```typescript
import { createRoot } from 'react-dom/client';

interface RootOptions {
  onCaughtError?: (error: Error, errorInfo: { componentStack: string }) => void;
  onUncaughtError?: (error: Error, errorInfo: { componentStack: string }) => void;
  onRecoverableError?: (error: Error, errorInfo: { componentStack: string }) => void;
  identifierPrefix?: string;
}

function createRoot(domNode: Element, options?: RootOptions): Root;

interface Root {
  render(reactNode: React.ReactNode): void;
  unmount(): void;
}
```

**Source**: https://react.dev/reference/react-dom/client/createRoot

**Usage**:
```typescript
const root = createRoot(document.getElementById('root')!, {
  onCaughtError: (error, errorInfo) => {
    console.error('Caught:', error, errorInfo.componentStack);
  },
  onUncaughtError: (error, errorInfo) => {
    console.error('Uncaught:', error, errorInfo.componentStack);
  },
});
root.render(<App />);
```

**Caveats**:
- First call clears all existing HTML inside the root
- Subsequent calls update the DOM — React reuses unchanged parts
- Rendering is asynchronous; use `flushSync()` for synchronous updates
- NEVER use `createRoot()` for server-rendered HTML — use `hydrateRoot()` instead

#### hydrateRoot

For server-rendered HTML. Attaches React to existing server-rendered markup.

```typescript
import { hydrateRoot } from 'react-dom/client';

function hydrateRoot(
  domNode: Element,
  reactNode: React.ReactNode,
  options?: RootOptions
): Root;
```

**Source**: https://react.dev/reference/react-dom/client/hydrateRoot

### §1.3 Resource Preloading APIs (React 19 ONLY)

These APIs are new in React 19 and enable resource hints for better loading performance. React-based frameworks typically handle these automatically.

| API | Purpose | Source |
|-----|---------|--------|
| `prefetchDNS(href)` | Prefetch DNS for a domain you expect to connect to | https://react.dev/reference/react-dom/prefetchDNS |
| `preconnect(href)` | Establish early connection to a server | https://react.dev/reference/react-dom/preconnect |
| `preload(href, options)` | Fetch stylesheet, font, image, or script you expect to use | https://react.dev/reference/react-dom/preload |
| `preloadModule(href)` | Fetch an ESM module you expect to use | https://react.dev/reference/react-dom/preloadModule |
| `preinit(href, options)` | Fetch AND evaluate a script, or fetch AND insert a stylesheet | https://react.dev/reference/react-dom/preinit |
| `preinitModule(href)` | Fetch AND evaluate an ESM module | https://react.dev/reference/react-dom/preinitModule |

```typescript
import { prefetchDNS, preconnect, preload, preinit } from 'react-dom';

function MyComponent() {
  prefetchDNS('https://api.example.com');
  preconnect('https://cdn.example.com');
  preload('https://cdn.example.com/styles.css', { as: 'style' });
  preinit('https://cdn.example.com/analytics.js', { as: 'script' });
  // ...
}
```

### §1.4 Removed APIs in React 19

| Removed API | Replacement |
|-------------|-------------|
| `render()` | `createRoot().render()` |
| `hydrate()` | `hydrateRoot()` |
| `unmountComponentAtNode()` | `root.unmount()` |
| `findDOMNode()` | Use refs |
| `renderToNodeStream()` | `react-dom/server` APIs |

**Source**: https://react.dev/reference/react-dom

---

## §2 React DOM Components & Form Handling

**Source**: https://react.dev/reference/react-dom/components

### §2.1 Common Props

All React DOM components support React-specific props:
- `ref` — access the underlying DOM node
- `className` — CSS class (NOT `class`)
- `style` — inline styles as an object with camelCase properties
- `dangerouslySetInnerHTML` — set raw HTML (use with extreme caution)
- `key` — unique identifier for list items (NOT passed as a prop to the component)
- `data-*` and `aria-*` — passed through as-is (with dashes, not camelCase)

**Naming convention**: All HTML attributes use camelCase in JSX except `aria-*` and `data-*`.

### §2.2 Form Elements: Controlled vs Uncontrolled

#### Controlled Components

React controls the value. State is the single source of truth.

```typescript
function ControlledForm() {
  const [name, setName] = useState<string>('');
  const [age, setAge] = useState<number>(0);
  const [bio, setBio] = useState<string>('');
  const [role, setRole] = useState<string>('developer');

  return (
    <form>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input type="number" value={age} onChange={(e) => setAge(Number(e.target.value))} />
      <textarea value={bio} onChange={(e) => setBio(e.target.value)} />
      <select value={role} onChange={(e) => setRole(e.target.value)}>
        <option value="developer">Developer</option>
        <option value="designer">Designer</option>
      </select>
    </form>
  );
}
```

#### Uncontrolled Components

The DOM manages the value. Use `defaultValue` for initial values.

```typescript
function UncontrolledForm() {
  return (
    <form>
      <input defaultValue="initial text" />
      <textarea defaultValue="initial bio" />
      <select defaultValue="developer">
        <option value="developer">Developer</option>
        <option value="designer">Designer</option>
      </select>
    </form>
  );
}
```

**Rule**: NEVER mix `value` and `defaultValue` on the same element. Choose controlled OR uncontrolled.

### §2.3 React 19: Form Actions

**Source**: https://react.dev/reference/react-dom/components/form

React 19 adds the `action` prop to `<form>`, accepting a function that receives `FormData`.

```typescript
function SearchForm() {
  async function handleSearch(formData: FormData) {
    const query = formData.get('query') as string;
    const results = await searchAPI(query);
    // handle results
  }

  return (
    <form action={handleSearch}>
      <input name="query" />
      <button type="submit">Search</button>
    </form>
  );
}
```

**Key behaviors**:
- Form submission is handled inside a React Transition
- HTTP method is ALWAYS POST when using a function (regardless of `method` prop)
- Uncontrolled form fields automatically reset after successful submission
- Forms work without JavaScript when using Server Actions (progressive enhancement)

#### useFormStatus (React 19)

```typescript
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}
```

**Caveat**: `useFormStatus` MUST be called from a component rendered inside the `<form>`. It does NOT work if called in the same component that renders the `<form>`.

#### useActionState (React 19)

```typescript
import { useActionState } from 'react';

const [state, formAction, isPending] = useActionState(
  async (prevState: string | null, formData: FormData) => {
    try {
      await submitData(formData);
      return null;
    } catch (err) {
      return (err as Error).message;
    }
  },
  null // initial state
);
```

#### useOptimistic (React 19)

```typescript
import { useOptimistic } from 'react';

interface Message {
  text: string;
  sending?: boolean;
}

function Chat({ messages }: { messages: Message[] }) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (state: Message[], newMessage: string) => [
      ...state,
      { text: newMessage, sending: true },
    ]
  );
  // ...
}
```

#### Multiple Submit Actions

```typescript
function Editor() {
  function publish(formData: FormData) { /* publish logic */ }
  function saveDraft(formData: FormData) { /* save logic */ }

  return (
    <form action={publish}>
      <textarea name="content" />
      <button type="submit">Publish</button>
      <button formAction={saveDraft}>Save Draft</button>
    </form>
  );
}
```

### §2.4 Resource & Metadata Components (React 19)

React 19 supports rendering these elements into `<head>` and can suspend while loading:
- `<link>` — external resources (stylesheets)
- `<meta>` — metadata tags
- `<script>` — external scripts
- `<style>` — inline styles
- `<title>` — document title

---

## §3 Error Boundaries

**Source**: https://react.dev/reference/react/Component

Error Boundaries are class components that catch JavaScript errors during rendering in their child component tree. There is NO function component equivalent — you MUST use a class component (or the `react-error-boundary` package).

### §3.1 API: getDerivedStateFromError

```typescript
static getDerivedStateFromError(error: Error): Partial<State>
```

- Called during the **render phase** (no side effects allowed)
- MUST be a pure function
- Returns a state update object to trigger fallback UI
- The `error` parameter can be any thrown value (not always an `Error` instance)

### §3.2 API: componentDidCatch

```typescript
componentDidCatch(error: Error, info: { componentStack: string }): void
```

- Called during the **commit phase** (side effects allowed)
- Use for logging errors to analytics/reporting services
- `info.componentStack` contains the component stack trace
- ALWAYS pair with `getDerivedStateFromError` for a complete Error Boundary

### §3.3 Complete TypeScript Implementation

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): Partial<State> {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo): void {
    console.error('Error caught by boundary:', error);
    console.error('Component stack:', info.componentStack);
    // Send to error reporting service
  }

  render(): ReactNode {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

**Usage**:
```typescript
<ErrorBoundary fallback={<p>Something went wrong.</p>}>
  <UserProfile userId={userId} />
</ErrorBoundary>
```

### §3.4 What Error Boundaries DO NOT Catch

| Not Caught | Use Instead |
|------------|-------------|
| Event handler errors | `try/catch` in the handler |
| Async code (setTimeout, promises) | `try/catch` or `.catch()` |
| Server-side rendering errors | Server-side error handling |
| Errors in the Error Boundary itself | A parent Error Boundary |

**Exception**: Errors thrown inside `startTransition` callbacks ARE caught by Error Boundaries.

### §3.5 Development vs Production Behavior

- **Development**: Errors bubble up to `window` (triggers `window.onerror`)
- **Production**: Errors do NOT bubble — only ancestor Error Boundaries receive them

**Source**: https://react.dev/reference/react/Component

---

## §4 JSX Patterns & Rendering

### §4.1 JSX Rules

**Source**: https://react.dev/learn/writing-markup-with-jsx

Three rules of JSX:

1. **Return a single root element** — use `<div>` or `<>...</>` (Fragment)
2. **Close ALL tags** — including self-closing: `<img />`, `<br />`, `<input />`
3. **camelCase for attributes** — `className`, `strokeWidth`, `onClick`

**Exceptions**: `aria-*` and `data-*` keep their dashes.

**WHY a single root**: JSX compiles to JavaScript objects. A function cannot return two objects without wrapping them.

### §4.2 Conditional Rendering

**Source**: https://react.dev/learn/conditional-rendering

| Pattern | When to Use | Example |
|---------|-------------|---------|
| `if`/`else` + early return | Completely different branches | `if (isPacked) return <Packed />;` |
| Ternary `? :` | Inline, two options | `{isPacked ? <Done /> : <Pending />}` |
| Logical AND `&&` | Show or nothing | `{isPacked && <CheckIcon />}` |
| Variable assignment | Complex multi-step logic | `let content = ...; return <div>{content}</div>;` |
| `return null` | Hide component entirely | `if (hidden) return null;` |

**CRITICAL PITFALL — the `0` trap with `&&`**:

```typescript
// WRONG: Renders "0" on screen when messageCount is 0
{messageCount && <p>New messages</p>}

// CORRECT: Always use a boolean expression on the left side of &&
{messageCount > 0 && <p>New messages</p>}
```

**WHY**: React renders `false`, `null`, and `undefined` as nothing, but renders the number `0` as text.

### §4.3 Rendering Lists

**Source**: https://react.dev/learn/rendering-lists

```typescript
interface Person {
  id: number;
  name: string;
  profession: string;
}

function PeopleList({ people }: { people: Person[] }) {
  return (
    <ul>
      {people.map((person) => (
        <li key={person.id}>
          {person.name} — {person.profession}
        </li>
      ))}
    </ul>
  );
}
```

**Key Rules**:
1. Keys MUST be unique among siblings
2. Keys MUST NOT change between renders
3. NEVER generate keys during rendering (`Math.random()`, `crypto.randomUUID()` inline)
4. JSX elements directly inside `map()` ALWAYS need keys

**Where to get keys**:
- Database IDs (preferred)
- Pre-generated stable IDs
- Content-based hashes (if items are truly immutable)

**ANTI-PATTERN — index as key**:

```typescript
// WRONG: Causes bugs on reorder, insert, or delete
{items.map((item, index) => <Item key={index} {...item} />)}

// CORRECT: Use a stable unique identifier
{items.map((item) => <Item key={item.id} {...item} />)}
```

**WHY index-as-key is wrong**: When items reorder, indices shift. React associates the wrong state with the wrong component. User input in list items gets mixed up.

**Fragment with key** — use named import, not shorthand `<>`:

```typescript
import { Fragment } from 'react';

{people.map((person) => (
  <Fragment key={person.id}>
    <h2>{person.name}</h2>
    <p>{person.bio}</p>
  </Fragment>
))}
```

### §4.4 Props Patterns

**Source**: https://react.dev/learn/passing-props-to-a-component

```typescript
// TypeScript interface for props
interface AvatarProps {
  person: { name: string; imageId: string };
  size?: number;      // optional with default
  children?: React.ReactNode;
}

// Destructuring with defaults
function Avatar({ person, size = 100, children }: AvatarProps) {
  return (
    <div>
      <img src={getImageUrl(person)} width={size} height={size} alt={person.name} />
      {children}
    </div>
  );
}

// Spreading all props (use sparingly)
function Profile(props: AvatarProps) {
  return <Avatar {...props} />;
}
```

**Rules**:
- Props are immutable snapshots — NEVER mutate props
- Default values apply when prop is `undefined` (NOT when `null` or `0`)
- Use `children` prop for composition/layout patterns
- Use spread syntax sparingly — it makes prop flow harder to trace

---

## §5 Performance APIs

### §5.1 React.memo

**Source**: https://react.dev/reference/react/memo

```typescript
const MemoizedComponent = memo<Props>(
  function MyComponent(props: Props) { /* ... */ },
  arePropsEqual?: (prevProps: Props, nextProps: Props) => boolean
);
```

**When to use**: Component re-renders frequently with the same props AND rendering is noticeably slow.

**When NOT to use**: Most components. Profile first with React DevTools before adding memo.

**How comparison works**: `Object.is()` shallow comparison for each prop individually.

**CRITICAL**: `memo` does NOT prevent re-renders from:
- Internal state changes (`useState`, `useReducer`)
- Context value changes (`useContext`)

**Strategies to make memo effective**:

```typescript
// Strategy 1: Pass primitives instead of objects
<Profile name={name} age={age} />  // GOOD: primitives are stable
<Profile person={{ name, age }} /> // BAD: new object every render

// Strategy 2: useMemo for object props
const person = useMemo(() => ({ name, age }), [name, age]);
<Profile person={person} />

// Strategy 3: useCallback for function props
const handleClick = useCallback(() => { /* ... */ }, [deps]);
<MemoizedChild onClick={handleClick} />

// Strategy 4: Derive minimal data
const hasGroups = person.groups !== null;
<CallToAction hasGroups={hasGroups} /> // Boolean instead of object
```

**ANTI-PATTERN — custom arePropsEqual that skips function comparison**:

```typescript
// WRONG: Ignoring function props causes stale closures
function arePropsEqual(prev: Props, next: Props) {
  return prev.data === next.data;
  // Missing: prev.onClick === next.onClick
}

// WRONG: Deep equality via JSON.stringify — terrible performance
function arePropsEqual(prev: Props, next: Props) {
  return JSON.stringify(prev) === JSON.stringify(next);
}
```

### §5.2 React 19: React Compiler eliminates manual memoization

The React Compiler (React 19) automatically applies memoization equivalent to `memo`, `useMemo`, and `useCallback`. When using React Compiler, manual memoization wrappers become unnecessary.

**Source**: https://react.dev/reference/react/memo

### §5.3 useMemo

**Source**: https://react.dev/reference/react/useMemo

```typescript
const cachedValue = useMemo<T>(
  calculateValue: () => T,
  dependencies: DependencyList
): T
```

**When to use**:
- Expensive calculations (>1ms on target hardware)
- Values passed to `memo`-wrapped children
- Values used as dependencies for other hooks

**When NOT to use**:
- Most calculations (fast by default)
- As a semantic guarantee (cache may be discarded)

**Dependency array rules**:
- Include ALL reactive values used inside the calculation
- Use `Object.is()` comparison
- Omitting the array = recalculate every render (defeats the purpose)

**Common mistake — arrow function without return**:

```typescript
// WRONG: Returns undefined (block body needs explicit return)
const options = useMemo(() => {
  matchMode: 'whole-word', text
}, [text]);

// CORRECT: Wrap object literal in parentheses
const options = useMemo(() => ({
  matchMode: 'whole-word', text
}), [text]);
```

**Caveats**:
- Strict Mode calls the calculation TWICE in development (to detect impurities)
- React may discard cached values (on suspend, file edit in dev)
- NEVER rely on `useMemo` as a semantic guarantee — use `useRef` for that

### §5.4 useCallback

**Source**: https://react.dev/reference/react/useCallback

```typescript
const cachedFn = useCallback<T extends (...args: any[]) => any>(
  fn: T,
  dependencies: DependencyList
): T
```

**Relationship to useMemo**: `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

**When to use**:
- Passing functions to `memo`-wrapped components
- Functions used as dependencies of other hooks
- Custom hooks returning stable function references

**Best practice — move functions inside Effects instead of memoizing**:

```typescript
// GOOD but verbose: useCallback + effect dependency
const createOptions = useCallback(() => ({
  serverUrl: 'https://localhost:1234',
  roomId,
}), [roomId]);

useEffect(() => {
  const connection = createConnection(createOptions());
  connection.connect();
  return () => connection.disconnect();
}, [createOptions]);

// BETTER: Move function inside Effect, no useCallback needed
useEffect(() => {
  const options = { serverUrl: 'https://localhost:1234', roomId };
  const connection = createConnection(options);
  connection.connect();
  return () => connection.disconnect();
}, [roomId]);
```

**Reducing dependencies with updater functions**:

```typescript
// WRONG: todos in dependency array, changes every update
const handleAddTodo = useCallback((text: string) => {
  setTodos([...todos, { id: nextId++, text }]);
}, [todos]);

// CORRECT: Use updater function, no todos dependency
const handleAddTodo = useCallback((text: string) => {
  setTodos((prev) => [...prev, { id: nextId++, text }]);
}, []);
```

### §5.5 Profiler

**Source**: https://react.dev/reference/react/Profiler

```typescript
<Profiler id="App" onRender={onRender}>
  <App />
</Profiler>

function onRender(
  id: string,           // which Profiler tree committed
  phase: 'mount' | 'update' | 'nested-update',
  actualDuration: number,  // ms spent rendering (memoization benefit visible here)
  baseDuration: number,    // ms to render without memoization (worst case)
  startTime: number,       // timestamp when rendering began
  commitTime: number       // timestamp when React committed
): void {
  // Log or aggregate performance data
}
```

**Caveats**:
- Disabled in production builds by default (requires special profiling build)
- Adds CPU and memory overhead
- Can be nested and used in multiple locations
- `actualDuration` vs `baseDuration` comparison shows memoization effectiveness

---

## §6 Testing with React Testing Library

### §6.1 Philosophy

**Source**: https://testing-library.com/docs/react-testing-library/intro

> "The more your tests resemble the way your software is used, the more confidence they can give you."

**Core principles**:
- Test from the user's perspective, not implementation details
- Query elements the way users would (by label, text, role)
- Use `data-testid` only as a last resort
- NEVER test internal state or props directly

**What it tests**: User interactions, accessibility, visible behavior.
**What it does NOT test**: Component internals, state management, implementation details.

### §6.2 Installation

```bash
npm install --save-dev @testing-library/react @testing-library/dom @testing-library/jest-dom
# For TypeScript
npm install --save-dev @types/react @types/react-dom
```

### §6.3 Core API

**Source**: https://testing-library.com/docs/react-testing-library/api

#### render

```typescript
import { render, RenderResult } from '@testing-library/react';

interface RenderOptions {
  container?: Element;
  baseElement?: Element;
  hydrate?: boolean;
  wrapper?: React.ComponentType;
  queries?: Queries;
  reactStrictMode?: boolean;
}

const result: RenderResult = render(<Component />, options?);
```

**Return value properties**:
- All query methods (`getByText`, `queryByRole`, `findByLabelText`, etc.)
- `container` — the rendering DOM node
- `debug()` — logs formatted DOM output
- `rerender(ui)` — update component with new props
- `unmount()` — remove the component
- `asFragment()` — returns a DocumentFragment (useful for snapshot tests)

#### Query Types

| Query Type | No Match | 1 Match | >1 Match | Async? |
|------------|----------|---------|----------|--------|
| `getBy...` | throw | return | throw | No |
| `queryBy...` | `null` | return | throw | No |
| `findBy...` | throw | return | throw | Yes |
| `getAllBy...` | throw | array | array | No |
| `queryAllBy...` | `[]` | array | array | No |
| `findAllBy...` | throw | array | array | Yes |

**Priority of queries** (prefer top):
1. `getByRole` — accessible role (best for accessibility)
2. `getByLabelText` — form field by label
3. `getByPlaceholderText` — input placeholder
4. `getByText` — visible text
5. `getByDisplayValue` — current form value
6. `getByAltText` — image alt text
7. `getByTitle` — title attribute
8. `getByTestId` — `data-testid` (last resort)

#### screen

```typescript
import { render, screen } from '@testing-library/react';

render(<App />);
const heading = screen.getByRole('heading', { name: /welcome/i });
const submitButton = screen.getByRole('button', { name: /submit/i });
const emailInput = screen.getByLabelText(/email/i);
```

#### cleanup

Automatically called after each test in most frameworks (Jest, Vitest). Unmounts all rendered trees.

#### act

Wrapper around React's `act()` — ALWAYS wrap state updates and effects. Most RTL utilities already handle this internally.

#### renderHook

```typescript
import { renderHook } from '@testing-library/react';

const { result, rerender, unmount } = renderHook(
  (props: { initialCount: number }) => useCounter(props.initialCount),
  { initialProps: { initialCount: 0 } }
);

expect(result.current.count).toBe(0);
```

### §6.4 Testing Patterns with Vitest

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// Basic render + query
describe('Greeting', () => {
  it('renders the greeting message', () => {
    render(<Greeting name="World" />);
    expect(screen.getByText('Hello, World!')).toBeInTheDocument();
  });
});

// User interaction with userEvent (preferred over fireEvent)
describe('Counter', () => {
  it('increments on click', async () => {
    const user = userEvent.setup();
    render(<Counter />);

    await user.click(screen.getByRole('button', { name: /increment/i }));
    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });
});

// Async testing with waitFor
describe('UserList', () => {
  it('loads and displays users', async () => {
    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
    });
  });
});

// Testing form submission
describe('LoginForm', () => {
  it('submits with entered values', async () => {
    const handleSubmit = vi.fn();
    const user = userEvent.setup();
    render(<LoginForm onSubmit={handleSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(handleSubmit).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password123',
    });
  });
});

// Testing with context providers
describe('ThemedButton', () => {
  it('uses theme from context', () => {
    render(<ThemedButton />, {
      wrapper: ({ children }) => (
        <ThemeProvider value="dark">{children}</ThemeProvider>
      ),
    });
    expect(screen.getByRole('button')).toHaveClass('dark-theme');
  });
});

// Testing hooks
describe('useCounter', () => {
  it('increments counter', () => {
    const { result } = renderHook(() => useCounter(0));
    act(() => {
      result.current.increment();
    });
    expect(result.current.count).toBe(1);
  });
});
```

### §6.5 fireEvent vs userEvent

| Feature | `fireEvent` | `userEvent` |
|---------|-------------|-------------|
| Speed | Synchronous, fast | Async, more realistic |
| Events | Single event | Full event sequence (focus, keydown, input, keyup) |
| Validation | Skips browser behavior | Simulates real browser behavior |
| Recommended | Low-level tests | All user interaction tests |

ALWAYS prefer `userEvent` over `fireEvent` for simulating user interactions.

---

## §7 Common Anti-Patterns & Mistakes

### §7.1 Unnecessary Effects

**Source**: https://react.dev/learn/you-might-not-need-an-effect

This is the single most common category of React mistakes.

#### Anti-Pattern 1: Transforming Data in Effects

```typescript
// WRONG: Extra render cycle for derived data
const [firstName, setFirstName] = useState('Taylor');
const [lastName, setLastName] = useState('Swift');
const [fullName, setFullName] = useState('');

useEffect(() => {
  setFullName(firstName + ' ' + lastName);
}, [firstName, lastName]);

// CORRECT: Calculate during render
const fullName = firstName + ' ' + lastName;
```

**WHY this is wrong**: Creates an unnecessary render cycle. React renders with stale `fullName`, then the Effect runs, then React re-renders with the updated value. Doubled work for zero benefit.

#### Anti-Pattern 2: Caching Expensive Calculations in Effects

```typescript
// WRONG: Effect for computed values
const [visibleTodos, setVisibleTodos] = useState<Todo[]>([]);
useEffect(() => {
  setVisibleTodos(getFilteredTodos(todos, filter));
}, [todos, filter]);

// CORRECT: useMemo
const visibleTodos = useMemo(() => getFilteredTodos(todos, filter), [todos, filter]);
```

#### Anti-Pattern 3: Resetting State on Prop Change

```typescript
// WRONG: Effect to reset state
function ProfilePage({ userId }: { userId: string }) {
  const [comment, setComment] = useState('');
  useEffect(() => {
    setComment('');
  }, [userId]);
}

// CORRECT: Use key prop to force remount
function ProfilePage({ userId }: { userId: string }) {
  return <Profile userId={userId} key={userId} />;
}

function Profile({ userId }: { userId: string }) {
  const [comment, setComment] = useState(''); // Automatically reset on key change
}
```

**WHY this is wrong**: Renders stale state first (old comment visible for one frame), then re-renders. The `key` prop forces React to treat it as a new component instance — clean and instant.

#### Anti-Pattern 4: Event Logic in Effects

```typescript
// WRONG: Side effect of display, not user action
useEffect(() => {
  if (product.isInCart) {
    showNotification(`Added ${product.name} to cart!`);
  }
}, [product]);

// CORRECT: Put in the event handler
function handleBuyClick() {
  addToCart(product);
  showNotification(`Added ${product.name} to cart!`);
}
```

**WHY this is wrong**: The notification appears on every render where the product is in the cart (including page reload), not just when the user clicks "Buy".

#### Anti-Pattern 5: Effect Chains

```typescript
// WRONG: Cascading Effects
useEffect(() => {
  if (card?.gold) setGoldCardCount((c) => c + 1);
}, [card]);

useEffect(() => {
  if (goldCardCount > 3) { setRound((r) => r + 1); setGoldCardCount(0); }
}, [goldCardCount]);

useEffect(() => {
  if (round > 5) alert('Game over!');
}, [round]);

// CORRECT: Calculate all in event handler
function handlePlaceCard(nextCard: Card) {
  setCard(nextCard);
  if (nextCard.gold) {
    if (goldCardCount < 3) {
      setGoldCardCount(goldCardCount + 1);
    } else {
      setGoldCardCount(0);
      setRound(round + 1);
      if (round === 5) alert('Good game!');
    }
  }
}
```

**WHY this is wrong**: Each Effect triggers a state update that causes a re-render, which triggers the next Effect. Multiple unnecessary render cycles and brittle, hard-to-debug code.

#### Anti-Pattern 6: Notifying Parents via Effects

```typescript
// WRONG: Decoupled from the action
function Toggle({ onChange }: { onChange: (isOn: boolean) => void }) {
  const [isOn, setIsOn] = useState(false);
  useEffect(() => {
    onChange(isOn);
  }, [isOn, onChange]);
}

// CORRECT: Notify in the event handler
function Toggle({ onChange }: { onChange: (isOn: boolean) => void }) {
  const [isOn, setIsOn] = useState(false);
  function handleClick() {
    const nextIsOn = !isOn;
    setIsOn(nextIsOn);
    onChange(nextIsOn); // Both in same event
  }
}
```

#### Anti-Pattern 7: Data Fetching Race Conditions

```typescript
// WRONG: No cleanup, stale responses overwrite fresh ones
useEffect(() => {
  fetchResults(query).then((json) => setResults(json));
}, [query]);

// CORRECT: Cleanup flag to ignore stale responses
useEffect(() => {
  let ignore = false;
  fetchResults(query).then((json) => {
    if (!ignore) setResults(json);
  });
  return () => { ignore = true; };
}, [query]);
```

**WHY this is wrong**: Fast typing sends multiple requests. Responses arrive out of order. The last response to arrive (which may be for an earlier query) overwrites the correct one.

#### Anti-Pattern 8: External Store Subscriptions

```typescript
// WRONG: Manual subscription in Effect
useEffect(() => {
  const handler = () => setIsOnline(navigator.onLine);
  window.addEventListener('online', handler);
  window.addEventListener('offline', handler);
  return () => {
    window.removeEventListener('online', handler);
    window.removeEventListener('offline', handler);
  };
}, []);

// CORRECT: useSyncExternalStore
const isOnline = useSyncExternalStore(
  (callback) => {
    window.addEventListener('online', callback);
    window.addEventListener('offline', callback);
    return () => {
      window.removeEventListener('online', callback);
      window.removeEventListener('offline', callback);
    };
  },
  () => navigator.onLine,  // client snapshot
  () => true               // server snapshot
);
```

### §7.2 Purity Violations

**Source**: https://react.dev/learn/keeping-components-pure

#### Anti-Pattern: Mutating External Variables During Render

```typescript
// WRONG: Mutating variable outside component
let guest = 0;
function Cup() {
  guest = guest + 1; // Side effect during render!
  return <h2>Tea cup for guest #{guest}</h2>;
}

// CORRECT: Pass as prop
function Cup({ guest }: { guest: number }) {
  return <h2>Tea cup for guest #{guest}</h2>;
}
```

**WHY this is wrong**: Calling `Cup` multiple times produces different results. Strict Mode calls components twice to detect this. In production, this leads to unpredictable behavior.

#### Anti-Pattern: Mutating Props or Received Arrays

```typescript
// WRONG: Mutating a prop array
function StoryTray({ stories }: { stories: Story[] }) {
  stories.push({ id: 'create', label: 'Create Story' }); // Mutation!
  return <ul>{stories.map((s) => <li key={s.id}>{s.label}</li>)}</ul>;
}

// CORRECT: Copy first
function StoryTray({ stories }: { stories: Story[] }) {
  const items = [...stories, { id: 'create', label: 'Create Story' }];
  return <ul>{items.map((s) => <li key={s.id}>{s.label}</li>)}</ul>;
}
```

#### Anti-Pattern: DOM Manipulation During Render

```typescript
// WRONG: Direct DOM manipulation in render
function Clock({ time }: { time: Date }) {
  document.getElementById('time')!.className = time.getHours() > 6 ? 'day' : 'night';
  return <h1 id="time">{time.toLocaleTimeString()}</h1>;
}

// CORRECT: Calculate and return in JSX
function Clock({ time }: { time: Date }) {
  const className = time.getHours() > 6 ? 'day' : 'night';
  return <h1 className={className}>{time.toLocaleTimeString()}</h1>;
}
```

### §7.3 Decision Tree: Where Does This Code Belong?

| Scenario | Belongs In |
|----------|-----------|
| Derived/computed data | Render body (calculated during render) |
| Expensive computed data | `useMemo` |
| Response to user action | Event handler |
| Reset state when prop changes | `key` prop |
| Sync with external system | `useEffect` |
| Subscribe to external store | `useSyncExternalStore` |
| One-time app initialization | Module-level code or guarded Effect |
| Notify parent of state change | Event handler (alongside `setState`) |

**The rule**: If it can be calculated during rendering, calculate it during rendering. If it is caused by a user action, put it in an event handler. Use Effects ONLY for synchronizing with external systems.

---

## §8 React 18 vs 19 Differences Summary

| Feature | React 18 | React 19 |
|---------|----------|----------|
| Resource preloading APIs | Not available | `prefetchDNS`, `preconnect`, `preload`, `preloadModule`, `preinit`, `preinitModule` |
| Form actions | Not available | `<form action={fn}>`, `useFormStatus`, `useActionState`, `useOptimistic` |
| React Compiler | Not available | Auto-memoization (replaces manual `memo`, `useMemo`, `useCallback`) |
| `use()` hook | Not available | Read resources (promises, context) in render |
| Metadata components | Manual `<Helmet>` or similar | Native `<title>`, `<meta>`, `<link>` in components |
| `ref` as prop | Requires `forwardRef` | Direct `ref` prop on function components |
| `render()` / `hydrate()` | Deprecated | Removed (use `createRoot` / `hydrateRoot`) |
| `findDOMNode()` | Deprecated | Removed |
| Error reporting | Basic | `onCaughtError`, `onUncaughtError`, `onRecoverableError` on root |

---

## Sources

All content in this document is sourced from official documentation:

| Topic | URL | Verified |
|-------|-----|----------|
| React DOM APIs | https://react.dev/reference/react-dom | 2025-03-19 |
| createPortal | https://react.dev/reference/react-dom/createPortal | 2025-03-19 |
| createRoot | https://react.dev/reference/react-dom/client/createRoot | 2025-03-19 |
| DOM Components | https://react.dev/reference/react-dom/components | 2025-03-19 |
| Form component | https://react.dev/reference/react-dom/components/form | 2025-03-19 |
| Error Boundaries | https://react.dev/reference/react/Component | 2025-03-19 |
| JSX | https://react.dev/learn/writing-markup-with-jsx | 2025-03-19 |
| Conditional Rendering | https://react.dev/learn/conditional-rendering | 2025-03-19 |
| Rendering Lists | https://react.dev/learn/rendering-lists | 2025-03-19 |
| Props | https://react.dev/learn/passing-props-to-a-component | 2025-03-19 |
| React.memo | https://react.dev/reference/react/memo | 2025-03-19 |
| useMemo | https://react.dev/reference/react/useMemo | 2025-03-19 |
| useCallback | https://react.dev/reference/react/useCallback | 2025-03-19 |
| Profiler | https://react.dev/reference/react/Profiler | 2025-03-19 |
| React Testing Library intro | https://testing-library.com/docs/react-testing-library/intro | 2025-03-19 |
| React Testing Library API | https://testing-library.com/docs/react-testing-library/api | 2025-03-19 |
| You Might Not Need an Effect | https://react.dev/learn/you-might-not-need-an-effect | 2025-03-19 |
| Keeping Components Pure | https://react.dev/learn/keeping-components-pure | 2025-03-19 |
