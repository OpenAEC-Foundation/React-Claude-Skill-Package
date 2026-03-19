# react-core-architecture -- Core Type Reference

## React Element

### createElement

```tsx
function createElement(
  type: string | React.ComponentType<P>,
  props: P | null,
  ...children: React.ReactNode[]
): React.ReactElement;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | `string \| ComponentType` | HTML tag name or component function/class |
| `props` | `object \| null` | Properties; `key` and `ref` are extracted, not in `element.props` |
| `...children` | `ReactNode[]` | Child elements, strings, numbers, null, or arrays |

**Returns**: Immutable `ReactElement` object:

```tsx
interface ReactElement<P = any, T = string | React.ComponentType<P>> {
  type: T;
  props: P;
  key: string | null;
  ref: React.Ref<any> | null;
}
```

---

## createRoot (react-dom/client)

### Signature

```tsx
function createRoot(
  domNode: Element | DocumentFragment,
  options?: CreateRootOptions
): Root;
```

### CreateRootOptions

| Option | Type | Description |
|--------|------|-------------|
| `onCaughtError` | `(error: Error, errorInfo: ErrorInfo) => void` | Called when Error Boundary catches an error |
| `onUncaughtError` | `(error: Error, errorInfo: ErrorInfo) => void` | Called when an error is not caught by any Error Boundary |
| `onRecoverableError` | `(error: Error, errorInfo: ErrorInfo) => void` | Called when React recovers automatically from an error |
| `identifierPrefix` | `string` | Prefix for `useId()` generated IDs; avoids conflicts with multiple roots |

### Root Object

| Method | Signature | Description |
|--------|-----------|-------------|
| `render` | `(reactNode: ReactNode) => void` | Display React content in the root DOM node |
| `unmount` | `() => void` | Destroy the rendered tree and detach React |

---

## hydrateRoot (react-dom/client)

### Signature

```tsx
function hydrateRoot(
  domNode: Element | Document,
  reactNode: ReactNode,
  options?: HydrateRootOptions
): Root;
```

ALWAYS use `hydrateRoot` instead of `createRoot` for server-rendered HTML. Using `createRoot` discards all server-rendered markup.

---

## ReactNode Types

| Type | Description | Example |
|------|-------------|---------|
| `ReactElement` | JSX element or createElement result | `<div />` |
| `string` | Text content | `"Hello"` |
| `number` | Rendered as text | `42` |
| `boolean` | Renders nothing | `true`, `false` |
| `null` | Renders nothing | `null` |
| `undefined` | Renders nothing | `undefined` |
| `ReactFragment` | Multiple children without wrapper | `<>...</>` |
| `ReactPortal` | Renders into a different DOM subtree | `createPortal(child, container)` |
| `Iterable<ReactNode>` | Arrays and iterables of nodes | `[<li />, <li />]` |

---

## Component Type Signatures

### Function Component

```tsx
type FC<P = {}> = (props: P) => ReactNode;

// Preferred: explicit function declaration with typed props
interface GreetingProps {
  name: string;
  age?: number;
}

function Greeting({ name, age }: GreetingProps): React.ReactElement {
  return <h1>Hello, {name}{age ? ` (${age})` : ''}</h1>;
}
```

### Memo Component

```tsx
const MemoizedComponent = React.memo<Props>(function MyComponent(props: Props) {
  return <div>{props.value}</div>;
});
```

### Lazy Component

```tsx
const LazyComponent = React.lazy<React.ComponentType<Props>>(
  () => import('./HeavyComponent')
);
```

---

## Fiber Node Structure (Internal Reference)

These fields are **internal implementation details** and MUST NOT be accessed directly. Listed for architectural understanding only.

| Field | Type | Purpose |
|-------|------|---------|
| `tag` | `number` | Fiber type (FunctionComponent=0, HostRoot=3, HostComponent=5, etc.) |
| `type` | `Function \| string \| null` | Component function or DOM tag name |
| `stateNode` | `DOM node \| null` | Actual DOM element (host components) or null (function components) |
| `child` | `Fiber \| null` | First child fiber |
| `sibling` | `Fiber \| null` | Next sibling fiber |
| `return` | `Fiber \| null` | Parent fiber |
| `memoizedState` | `Hook \| null` | Head of the hooks linked list |
| `memoizedProps` | `object` | Props from the last completed render |
| `pendingProps` | `object` | Props for the current in-progress render |
| `alternate` | `Fiber \| null` | The corresponding fiber in the other tree (current <-> work-in-progress) |
| `lanes` | `number` | Priority lane bits for scheduling |
| `flags` | `number` | Side-effect flags (Placement, Update, Deletion, etc.) |

### Double Buffering

React maintains two fiber trees:

| Tree | Role |
|------|------|
| **Current tree** | Represents what is currently on screen |
| **Work-in-progress tree** | Being built during the render phase |

After commit, the work-in-progress tree becomes the current tree. The `alternate` pointer connects corresponding fibers between the two trees.

---

## Reconciliation Heuristics

| Rule | Behavior |
|------|----------|
| Different `type` | Tear down old subtree, build new one from scratch |
| Same `type` (host element) | Keep DOM node, update only changed attributes |
| Same `type` (component) | Keep instance, update props, re-render |
| `key` changed | Unmount old instance, mount new one (even if type matches) |
| `key` stable | Reorder without unmounting (lists) |
| No `key` on list items | Fall back to index-based matching (fragile, causes bugs) |

### Priority Lanes (React 18+)

| Lane Category | Examples | Priority |
|--------------|----------|----------|
| Sync | `flushSync()` callbacks | Highest |
| Input | Discrete events (click, keydown) | High |
| Default | `setState` in event handlers | Normal |
| Transition | `startTransition()` updates | Low |
| Idle | `requestIdleCallback`-style work | Lowest |

Higher-priority lanes interrupt lower-priority rendering work, allowing React to stay responsive during expensive updates.

---

## StrictMode API

```tsx
import { StrictMode } from 'react';

// Wrap entire app (recommended)
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);

// Wrap specific subtree
function App(): React.ReactElement {
  return (
    <div>
      <Header />
      <StrictMode>
        <main>
          <NewFeature />
        </main>
      </StrictMode>
    </div>
  );
}
```

| Props | Type | Description |
|-------|------|-------------|
| `children` | `ReactNode` | The subtree to wrap with strict checks |

No other props. No way to opt out components inside a StrictMode boundary.
