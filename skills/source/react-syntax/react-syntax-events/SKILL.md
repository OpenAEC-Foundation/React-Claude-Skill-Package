---
name: react-syntax-events
description: "Guides React event handling including synthetic event system, TypeScript event type annotations, event handler patterns, form events, keyboard and mouse events, event delegation model, stopPropagation and preventDefault, and custom event patterns. Activates when handling user interactions, typing event handlers, working with form inputs, or implementing keyboard/mouse interactions."
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-syntax-events

## Quick Reference

### Synthetic Event System

React wraps all native browser events in `SyntheticEvent` objects. These wrappers normalize cross-browser differences and provide a consistent API.

| Concept | Detail |
|---------|--------|
| Wrapper | Every handler receives a `SyntheticEvent`, NOT a native `Event` |
| Delegation | React attaches ONE listener to the root container, NOT to individual DOM nodes |
| Pooling | React 17+ does NOT pool events -- accessing `e` in async callbacks is safe |
| Native access | Use `e.nativeEvent` to access the underlying browser event |

### TypeScript Event Types

| React Type | Native Equivalent | Common Elements |
|------------|-------------------|-----------------|
| `React.MouseEvent<T>` | `MouseEvent` | `HTMLButtonElement`, `HTMLDivElement` |
| `React.ChangeEvent<T>` | `Event` | `HTMLInputElement`, `HTMLSelectElement`, `HTMLTextAreaElement` |
| `React.FormEvent<T>` | `Event` | `HTMLFormElement` |
| `React.KeyboardEvent<T>` | `KeyboardEvent` | `HTMLInputElement`, `HTMLDivElement` |
| `React.FocusEvent<T>` | `FocusEvent` | `HTMLInputElement`, `HTMLButtonElement` |
| `React.DragEvent<T>` | `DragEvent` | `HTMLDivElement`, `HTMLImageElement` |
| `React.TouchEvent<T>` | `TouchEvent` | `HTMLDivElement`, `HTMLButtonElement` |
| `React.ClipboardEvent<T>` | `ClipboardEvent` | `HTMLInputElement`, `HTMLDivElement` |
| `React.WheelEvent<T>` | `WheelEvent` | `HTMLDivElement`, `HTMLElement` |
| `React.PointerEvent<T>` | `PointerEvent` | `HTMLDivElement`, `HTMLButtonElement` |

The generic parameter `T` specifies the element the handler is attached to. ALWAYS provide it for correct `e.currentTarget` typing.

### Critical Warnings

**NEVER** use `onKeyPress` -- it is deprecated. ALWAYS use `onKeyDown` or `onKeyUp` instead.

**NEVER** type event handlers as `(e: any) => void` -- ALWAYS use the specific React event type with the correct element generic.

**NEVER** call `e.stopPropagation()` as a default habit -- it breaks parent listeners, analytics tools, and modal close handlers. ONLY use it when you have a specific reason.

**ALWAYS** call `e.preventDefault()` in `onSubmit` handlers to prevent full page reload.

**ALWAYS** provide the element generic parameter (e.g., `<HTMLInputElement>`) -- without it, `e.currentTarget` is typed as `HTMLElement` and you lose access to element-specific properties like `.value`.

**NEVER** read `e.currentTarget` inside `async`/`await` or `setTimeout` -- `currentTarget` is set to `null` after the event handler returns. Extract the value synchronously first.

---

## Event Handler Typing Patterns

### Inline Handler

```tsx
<button onClick={(e: React.MouseEvent<HTMLButtonElement>) => {
  console.log(e.currentTarget.disabled);
}}>
  Click
</button>
```

### Extracted Handler

```tsx
const handleClick = (e: React.MouseEvent<HTMLButtonElement>): void => {
  console.log(e.currentTarget.disabled);
};

<button onClick={handleClick}>Click</button>
```

### Handler as Props

```tsx
interface ButtonProps {
  onClick: (e: React.MouseEvent<HTMLButtonElement>) => void;
  onFocus?: (e: React.FocusEvent<HTMLButtonElement>) => void;
}

const Button = ({ onClick, onFocus }: ButtonProps): JSX.Element => (
  <button onClick={onClick} onFocus={onFocus}>Click</button>
);
```

### React.EventHandler Type

React provides shorthand types for handler props:

```tsx
interface Props {
  onClick: React.MouseEventHandler<HTMLButtonElement>;
  onChange: React.ChangeEventHandler<HTMLInputElement>;
  onSubmit: React.FormEventHandler<HTMLFormElement>;
  onKeyDown: React.KeyboardEventHandler<HTMLInputElement>;
}
```

These are equivalent to `(e: React.MouseEvent<HTMLButtonElement>) => void`, etc.

---

## Passing Data to Handlers

### Arrow Function (Preferred)

```tsx
const handleDelete = (id: string, e: React.MouseEvent<HTMLButtonElement>): void => {
  e.stopPropagation();
  deleteItem(id);
};

{items.map((item) => (
  <button key={item.id} onClick={(e) => handleDelete(item.id, e)}>
    Delete {item.name}
  </button>
))}
```

### Currying Pattern

```tsx
const handleAction = (id: string) => (e: React.MouseEvent<HTMLButtonElement>): void => {
  e.preventDefault();
  performAction(id);
};

<button onClick={handleAction(item.id)}>Act</button>
```

**NEVER** use `.bind(this, arg)` in function components -- it creates a new function reference every render just like arrow functions, but with worse readability.

---

## stopPropagation vs preventDefault

| Method | Purpose | Use When |
|--------|---------|----------|
| `e.preventDefault()` | Prevents the browser default action | Form submit, link navigation, drag default behavior |
| `e.stopPropagation()` | Stops the event from reaching parent handlers | Nested clickable elements, preventing parent dismissals |

### Decision Tree

1. Is the browser doing something unwanted (page reload, navigation)? --> `preventDefault()`
2. Is a parent handler firing when it should not? --> `stopPropagation()`
3. Both? --> Call both explicitly

```tsx
const handleLinkClick = (e: React.MouseEvent<HTMLAnchorElement>): void => {
  e.preventDefault();  // stop navigation
  e.stopPropagation(); // stop parent onClick
  openModal();
};
```

---

## Capture Phase

React supports capture-phase listeners by appending `Capture` to any event prop:

```tsx
<div onClickCapture={(e: React.MouseEvent<HTMLDivElement>) => {
  // Fires BEFORE child onClick handlers
  console.log("Captured click on:", e.target);
}}>
  <button onClick={() => console.log("Button clicked")}>
    Click me
  </button>
</div>
```

Event flow order: **Capture (root to target) --> Target --> Bubble (target to root)**

ALWAYS use capture-phase handlers when you need to intercept events before children process them (e.g., global escape key handling, focus trapping).

---

## Form Events

### onChange

Fires on every keystroke for text inputs (unlike native `change` which fires on blur):

```tsx
const [value, setValue] = useState<string>("");

const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
  setValue(e.currentTarget.value);
};

<input type="text" value={value} onChange={handleChange} />
```

### onSubmit

ALWAYS call `e.preventDefault()` to prevent page reload:

```tsx
const handleSubmit = (e: React.FormEvent<HTMLFormElement>): void => {
  e.preventDefault();
  const formData = new FormData(e.currentTarget);
  submitForm(Object.fromEntries(formData));
};

<form onSubmit={handleSubmit}>
  <input name="email" type="email" required />
  <button type="submit">Submit</button>
</form>
```

---

## Mouse Events

| Prop | Fires When | Bubbles |
|------|-----------|---------|
| `onClick` | Click (mousedown + mouseup) | Yes |
| `onDoubleClick` | Double click | Yes |
| `onMouseDown` / `onMouseUp` | Press / release | Yes |
| `onMouseEnter` | Pointer enters element | **No** |
| `onMouseLeave` | Pointer leaves element | **No** |
| `onMouseOver` / `onMouseOut` | Pointer enters/leaves (includes children) | Yes |
| `onContextMenu` | Right-click | Yes |

`onMouseEnter` / `onMouseLeave` do NOT bubble -- they fire only for the exact element, not its children. Use these for hover states. Use `onMouseOver` / `onMouseOut` when you need bubbling behavior.

---

## Keyboard Events

| Prop | Fires When |
|------|-----------|
| `onKeyDown` | Key pressed down (repeats while held) |
| `onKeyUp` | Key released |

**NEVER** use `onKeyPress` -- it is deprecated and does not fire for non-character keys (Escape, Arrow keys, etc.).

### Key Detection

ALWAYS use `e.key` (string value) instead of `e.keyCode` (deprecated numeric code):

```tsx
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>): void => {
  if (e.key === "Enter") {
    submitSearch();
  }
  if (e.key === "Escape") {
    clearInput();
  }
};
```

### Modifier Keys

```tsx
const handleShortcut = (e: React.KeyboardEvent<HTMLDivElement>): void => {
  if (e.key === "s" && (e.metaKey || e.ctrlKey)) {
    e.preventDefault(); // prevent browser save dialog
    saveDocument();
  }
};
```

| Property | Key |
|----------|-----|
| `e.ctrlKey` | Ctrl (Windows/Linux) |
| `e.metaKey` | Cmd (macOS) / Win (Windows) |
| `e.shiftKey` | Shift |
| `e.altKey` | Alt / Option |

---

## Focus Events

| Prop | Fires When | Bubbles |
|------|-----------|---------|
| `onFocus` | Element gains focus | Yes (in React) |
| `onBlur` | Element loses focus | Yes (in React) |

React's `onFocus` and `onBlur` bubble, unlike the native `focus`/`blur` events. This matches the native `focusin`/`focusout` behavior.

### relatedTarget

```tsx
const handleBlur = (e: React.FocusEvent<HTMLInputElement>): void => {
  if (e.relatedTarget === null) {
    // Focus left the document entirely
    validateField();
  }
};
```

`e.relatedTarget` is the element that received (onBlur) or lost (onFocus) focus. It is `null` when focus moves outside the document.

---

## React 18 vs React 19 Differences

| Feature | React 18 | React 19 |
|---------|----------|----------|
| Event delegation | Root container | Root container (unchanged) |
| Event pooling | Removed (since 17) | Removed |
| `ref` as prop | Use `forwardRef` for function components | `ref` is a regular prop -- no `forwardRef` needed |
| Form actions | Not available | `<form action={fn}>` with `useActionState` |

In React 19, forms can use the `action` prop directly. For new projects on React 19, prefer `action` + `useActionState` over manual `onSubmit` + `preventDefault` for form submissions with server mutations.

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete event handling examples with TypeScript
- [references/api-table.md](references/api-table.md) -- All synthetic event types with properties and TypeScript signatures
- [references/anti-patterns.md](references/anti-patterns.md) -- Common event handling mistakes and corrections

### Official Sources

- https://react.dev/learn/responding-to-events
- https://react.dev/reference/react-dom/components/common#event-handler
- https://react.dev/learn/typescript#typing-dom-events
- https://react.dev/reference/react-dom/components/input
- https://react.dev/reference/react-dom/components/form
