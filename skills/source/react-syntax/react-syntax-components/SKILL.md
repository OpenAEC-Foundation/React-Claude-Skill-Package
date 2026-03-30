---
name: react-syntax-components
description: >
  Use when creating components, typing props with TypeScript, forwarding refs, or
  implementing component composition patterns. Prevents the common mistake of
  using incorrect prop typing or missing forwardRef when exposing DOM elements.
  Covers function component typing, children patterns, React.memo, forwardRef,
  React.lazy, createPortal, compound components, render props, HOCs.
  Keywords: component, props, children, forwardRef, React.memo, Portal, lazy, create component, TypeScript props, children prop, wrap component, higher-order component..
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-syntax-components

## Quick Reference

### Function Component Typing

| Pattern | Syntax | When |
|---------|--------|------|
| Props interface + return | `function Comp(props: Props): React.ReactElement` | Default for all components |
| Destructured props | `function Comp({ name, age }: Props)` | When accessing props directly |
| Generic component | `function List<T>(props: ListProps<T>)` | Reusable data-driven components |
| Default props | `{ size = 'md' }: Props` | Optional props with defaults |

### Component API Quick Lookup

| API | Purpose | React 18 | React 19 |
|-----|---------|----------|----------|
| `React.memo` | Skip re-render when props unchanged | Yes | Yes (Compiler reduces need) |
| `forwardRef` | Pass ref through component | Required | DEPRECATED -- use ref as prop |
| `React.lazy` | Code-split with dynamic import | Yes | Yes |
| `createPortal` | Render outside DOM parent | Yes | Yes |
| `useImperativeHandle` | Expose custom ref handle | With forwardRef | With ref prop |

### Critical Warnings

**NEVER** use class components for new code -- ALWAYS use function components with hooks. Class components are legacy and cannot use hooks.

**NEVER** define components inside other components -- this destroys state on every render. ALWAYS define components at module scope.

**NEVER** call hooks conditionally inside components -- React relies on consistent hook call order. ALWAYS call hooks at the top level.

**ALWAYS** use TypeScript interfaces for props -- bare `any` or untyped props defeat type safety and make refactoring dangerous.

**ALWAYS** use `React.ReactNode` for children type -- it covers strings, numbers, elements, arrays, fragments, portals, `null`, and `undefined`.

---

## Decision Tree: Component Pattern Selection

```
Need to render children?
  +-- Fixed structure --> Standard props interface
  +-- Flexible layout --> children: React.ReactNode
  +-- Parent needs control over rendering --> Render props pattern

Need to share behavior across components?
  +-- Shared state logic --> Custom hook (ALWAYS prefer this)
  +-- Shared rendering wrapper --> Compound component pattern
  +-- Cross-cutting concern (rare) --> HOC pattern

Need performance optimization?
  +-- Expensive render, stable props --> React.memo
  +-- Code splitting by route --> React.lazy + Suspense

Need DOM access from parent?
  +-- React 18 --> forwardRef + useImperativeHandle
  +-- React 19 --> ref as prop + useImperativeHandle

Need to render outside DOM hierarchy?
  +-- Modals, tooltips, overlays --> createPortal
```

---

## Function Components with TypeScript

### Props Interface Pattern

```tsx
interface UserCardProps {
  readonly name: string;
  readonly email: string;
  readonly role?: 'admin' | 'user' | 'guest';  // optional with union
  readonly onSelect: (id: string) => void;
}

function UserCard({ name, email, role = 'user', onSelect }: UserCardProps): React.ReactElement {
  return (
    <div onClick={() => onSelect(name)}>
      <h2>{name}</h2>
      <p>{email}</p>
      <span>{role}</span>
    </div>
  );
}
```

**ALWAYS** mark props as `readonly` -- props must never be mutated.

### Generic Component Pattern

```tsx
interface ListProps<T> {
  readonly items: readonly T[];
  readonly renderItem: (item: T, index: number) => React.ReactNode;
  readonly keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>): React.ReactElement {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}

// Usage -- TypeScript infers T from items
<List items={users} renderItem={(user) => <span>{user.name}</span>} keyExtractor={(u) => u.id} />
```

---

## Children Patterns

### React.ReactNode (default)

```tsx
interface CardProps {
  readonly title: string;
  readonly children: React.ReactNode;
}

function Card({ title, children }: CardProps): React.ReactElement {
  return (
    <div className="card">
      <h3>{title}</h3>
      <div className="card-body">{children}</div>
    </div>
  );
}
```

### PropsWithChildren Utility

```tsx
import { type PropsWithChildren } from 'react';

interface PanelProps {
  readonly variant: 'primary' | 'secondary';
}

function Panel({ variant, children }: PropsWithChildren<PanelProps>): React.ReactElement {
  return <div className={`panel-${variant}`}>{children}</div>;
}
```

**Note**: `PropsWithChildren<P>` adds `children?: React.ReactNode` -- children becomes optional.

---

## React.memo -- Memoized Components

```tsx
interface ExpensiveListProps {
  readonly items: readonly string[];
  readonly onItemClick: (item: string) => void;
}

const ExpensiveList = React.memo(function ExpensiveList(
  { items, onItemClick }: ExpensiveListProps
): React.ReactElement {
  return (
    <ul>
      {items.map((item) => (
        <li key={item} onClick={() => onItemClick(item)}>{item}</li>
      ))}
    </ul>
  );
});
```

### Custom Comparison

```tsx
const UserRow = React.memo(
  function UserRow({ user, onSelect }: UserRowProps): React.ReactElement {
    return <tr onClick={() => onSelect(user.id)}><td>{user.name}</td></tr>;
  },
  (prevProps, nextProps) => prevProps.user.id === nextProps.user.id
);
```

**ALWAYS** ensure parent stabilizes callback props with `useCallback` -- otherwise `memo` is ineffective because a new function reference is created every render.

**React 19**: The React Compiler auto-memoizes, making manual `memo` largely unnecessary. Keep `memo` for React 18 compatibility.

---

## forwardRef and Ref Forwarding

### React 19 -- ref as Regular Prop (PREFERRED)

```tsx
interface InputProps {
  readonly label: string;
  readonly ref?: React.Ref<HTMLInputElement>;
}

function TextInput({ label, ref, ...props }: InputProps): React.ReactElement {
  return (
    <label>
      {label}
      <input ref={ref} {...props} />
    </label>
  );
}
```

### React 18 -- forwardRef Required

```tsx
interface InputProps {
  readonly label: string;
}

const TextInput = React.forwardRef<HTMLInputElement, InputProps>(
  function TextInput({ label, ...props }, ref): React.ReactElement {
    return (
      <label>
        {label}
        <input ref={ref} {...props} />
      </label>
    );
  }
);
```

### useImperativeHandle -- Custom Ref API

```tsx
interface TextInputHandle {
  focus: () => void;
  clear: () => void;
}

// React 19
function TextInput({ ref }: { ref?: React.Ref<TextInputHandle> }): React.ReactElement {
  const inputRef = React.useRef<HTMLInputElement>(null);

  React.useImperativeHandle(ref, () => ({
    focus() { inputRef.current?.focus(); },
    clear() { if (inputRef.current) inputRef.current.value = ''; },
  }), []);

  return <input ref={inputRef} />;
}

// Parent usage
const inputRef = React.useRef<TextInputHandle>(null);
<TextInput ref={inputRef} />
// inputRef.current?.focus();
```

### React 19 Ref Cleanup Function

```tsx
// React 19 -- ref callbacks can return a cleanup function
<div ref={(node) => {
  // Setup: node is attached
  node?.addEventListener('scroll', handleScroll);
  // Cleanup: returned function runs when ref detaches
  return () => node?.removeEventListener('scroll', handleScroll);
}} />
```

---

## React.lazy -- Code Splitting

```tsx
// ALWAYS declare lazy components at module top level
const AdminPanel = React.lazy(() => import('./AdminPanel'));
const UserDashboard = React.lazy(() => import('./UserDashboard'));

function App(): React.ReactElement {
  return (
    <React.Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/admin" element={<AdminPanel />} />
        <Route path="/dashboard" element={<UserDashboard />} />
      </Routes>
    </React.Suspense>
  );
}
```

**ALWAYS** wrap lazy components in `<Suspense>` -- without it, a lazy component throws an error.

**ALWAYS** declare `lazy()` at module scope -- declaring inside a component recreates it every render and destroys state.

---

## createPortal -- Rendering Outside the DOM Tree

```tsx
import { createPortal } from 'react-dom';

interface ModalProps {
  readonly isOpen: boolean;
  readonly onClose: () => void;
  readonly children: React.ReactNode;
}

function Modal({ isOpen, onClose, children }: ModalProps): React.ReactElement | null {
  if (!isOpen) return null;

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

**Event bubbling**: Events from portals bubble through the React tree (not the DOM tree). A click inside a portal still triggers `onClick` on React ancestors.

**ALWAYS** use portals for modals, tooltips, and overlays that must escape `overflow: hidden` or `z-index` stacking contexts.

---

## Composition Patterns

### Compound Components (Context-based)

See [references/patterns.md](references/patterns.md) for the full compound component pattern with Context.

### Render Props

See [references/patterns.md](references/patterns.md) for render props and function-as-children patterns.

### Higher-Order Components (HOC)

See [references/patterns.md](references/patterns.md) for HOC typing with generics. **ALWAYS** prefer custom hooks over HOCs for new code.

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete component typing examples
- [references/patterns.md](references/patterns.md) -- Compound components, render props, HOC patterns
- [references/anti-patterns.md](references/anti-patterns.md) -- Component mistakes and fixes

### Official Sources

- https://react.dev/reference/react/memo
- https://react.dev/reference/react/forwardRef
- https://react.dev/reference/react/lazy
- https://react.dev/reference/react-dom/createPortal
- https://react.dev/reference/react/useImperativeHandle
- https://react.dev/learn/passing-props-to-a-component
- https://react.dev/learn/typescript
