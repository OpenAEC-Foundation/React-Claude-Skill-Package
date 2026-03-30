---
name: react-syntax-jsx
description: >
  Use when writing JSX/TSX, rendering lists, implementing conditional rendering,
  or typing components with TypeScript generics. Prevents the common mistake of
  using unstable keys (like array index) or missing key props on list items.
  Covers JSX expressions, conditional rendering, list rendering with keys,
  fragments, TypeScript generics, spread attributes.
  Keywords: JSX, TSX, conditional rendering, key, Fragment, map, expressions, conditional render, show if true, loop over array, render list, key prop, Fragment..
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-syntax-jsx

## Quick Reference

### JSX Compilation

JSX is syntactic sugar for `React.createElement()` calls. With the **new JSX transform** (React 17+, enabled by default in React 18/19), you do NOT need to import React for JSX to work:

```tsx
// What you write:
<Button color="blue">Click me</Button>

// What the compiler produces (new transform):
import { jsx as _jsx } from 'react/jsx-runtime';
_jsx(Button, { color: 'blue', children: 'Click me' });
```

**NEVER** import React solely for JSX in React 18/19 projects — the new JSX transform handles it automatically.

### Three Rules of JSX

1. **Return a single root element** — use a wrapper `<div>` or Fragment `<>...</>`
2. **Close ALL tags** — including self-closing: `<img />`, `<br />`, `<input />`
3. **camelCase for attributes** — `className`, `strokeWidth`, `onClick`, `htmlFor`

**Exceptions**: `aria-*` and `data-*` attributes keep their dashes (e.g., `aria-label`, `data-testid`).

### Attribute Name Mapping

| HTML | JSX | Why |
|------|-----|-----|
| `class` | `className` | `class` is a reserved word in JavaScript |
| `for` | `htmlFor` | `for` is a reserved word in JavaScript |
| `tabindex` | `tabIndex` | camelCase convention |
| `readonly` | `readOnly` | camelCase convention |
| `maxlength` | `maxLength` | camelCase convention |
| `aria-label` | `aria-label` | Exception: kept as-is |
| `data-id` | `data-id` | Exception: kept as-is |

### Critical Warnings

**NEVER** use `0 && <Component />` — React renders the number `0` as visible text. ALWAYS convert the left side to a boolean:

```tsx
// WRONG: Renders "0" on screen when count is 0
{messageCount && <Badge />}

// CORRECT: Boolean expression prevents rendering "0"
{messageCount > 0 && <Badge />}
```

**NEVER** use array index as key for lists that can reorder, insert, or delete items — this causes state corruption. ALWAYS use stable unique identifiers:

```tsx
// WRONG: Index keys break on reorder/insert/delete
{items.map((item, index) => <Item key={index} {...item} />)}

// CORRECT: Stable unique ID preserves component state
{items.map((item) => <Item key={item.id} {...item} />)}
```

**NEVER** generate keys during render — `Math.random()` or `crypto.randomUUID()` inline creates new keys every render, destroying all component state.

**NEVER** use lowercase names for custom components — React treats lowercase tags as HTML elements. ALWAYS use PascalCase for component names.

---

## Expressions in JSX

Use curly braces `{}` to embed JavaScript **expressions** inside JSX:

```tsx
const name: string = 'Alice';
const imgUrl: string = getAvatarUrl(user);

return (
  <div>
    <h1>Hello, {name}</h1>
    <img src={imgUrl} alt={`Avatar of ${name}`} />
    <p>Result: {2 + 2}</p>
    <p style={{ color: 'red', fontSize: 16 }}>Styled text</p>
  </div>
);
```

**NEVER** use statements inside `{}` — `if`, `for`, `switch`, `let`/`const` declarations are NOT expressions. Use ternaries, `&&`, or extract logic before the return.

### String Literals vs Expressions

```tsx
// String literal — use quotes
<input type="text" placeholder="Enter name" />

// Dynamic value — use braces
<input type="text" placeholder={dynamicPlaceholder} />

// NEVER mix quotes and braces on the same attribute
<input placeholder="{'wrong'}" />  // Renders the literal string "{'wrong'}"
```

---

## Conditional Rendering

| Pattern | When to Use |
|---------|-------------|
| `if`/`else` + early return | Completely different output branches |
| Ternary `? :` | Inline choice between two elements |
| `&&` short-circuit | Show something or nothing |
| Variable assignment | Complex multi-step logic |
| `return null` | Hide component entirely |

```tsx
// Early return
function Greeting({ isLoggedIn }: { isLoggedIn: boolean }): JSX.Element {
  if (!isLoggedIn) {
    return <LoginPrompt />;
  }
  return <Dashboard />;
}

// Ternary
{isPacked ? <span>Packed</span> : <span>Pending</span>}

// Safe && (ALWAYS use boolean left side)
{items.length > 0 && <ItemList items={items} />}

// Variable assignment for complex logic
let content: JSX.Element;
if (isLoading) {
  content = <Spinner />;
} else if (error) {
  content = <ErrorMessage error={error} />;
} else {
  content = <DataTable data={data} />;
}
return <div>{content}</div>;
```

---

## Lists and Keys

ALWAYS provide a `key` prop to the outermost element returned inside `map()`:

```tsx
interface Task {
  id: string;
  title: string;
  completed: boolean;
}

function TaskList({ tasks }: { tasks: Task[] }): JSX.Element {
  return (
    <ul>
      {tasks.map((task) => (
        <li key={task.id}>
          {task.title} {task.completed && '(done)'}
        </li>
      ))}
    </ul>
  );
}
```

### Key Rules

1. Keys MUST be **unique among siblings** (not globally)
2. Keys MUST NOT **change between renders**
3. Keys are NOT passed as a prop to the component — use a different prop name if needed
4. ALWAYS prefer database IDs or pre-generated stable IDs
5. Index as key is ONLY acceptable for **static lists that never reorder**

---

## Fragments

Use Fragments to group elements without adding extra DOM nodes:

```tsx
// Short syntax (cannot take props)
<>
  <Header />
  <Main />
  <Footer />
</>

// Named Fragment (required when using key)
import { Fragment } from 'react';

{sections.map((section) => (
  <Fragment key={section.id}>
    <h2>{section.title}</h2>
    <p>{section.content}</p>
  </Fragment>
))}
```

**ALWAYS** use `<Fragment key={...}>` (named import) when you need keys on fragments — the short syntax `<>` does NOT support the `key` prop.

---

## TypeScript Component Typing

### Function Component Signatures

```tsx
// Preferred: Direct annotation on props parameter
interface ButtonProps {
  label: string;
  variant?: 'primary' | 'secondary';
  onClick: () => void;
  children?: React.ReactNode;
}

function Button({ label, variant = 'primary', onClick, children }: ButtonProps): JSX.Element {
  return (
    <button className={variant} onClick={onClick}>
      {label}
      {children}
    </button>
  );
}
```

**NEVER** use `React.FC<P>` in new code — it previously included an implicit `children` prop (fixed in React 18 types) and hinders generic components. Direct annotation is clearer and more flexible.

### PropsWithChildren

```tsx
import type { PropsWithChildren } from 'react';

interface CardProps {
  title: string;
}

function Card({ title, children }: PropsWithChildren<CardProps>): JSX.Element {
  return (
    <div className="card">
      <h2>{title}</h2>
      {children}
    </div>
  );
}
```

### Generic Components

```tsx
interface SelectProps<T> {
  items: T[];
  selected: T;
  getLabel: (item: T) => string;
  onChange: (item: T) => void;
}

function Select<T>({ items, selected, getLabel, onChange }: SelectProps<T>): JSX.Element {
  return (
    <ul>
      {items.map((item, i) => (
        <li key={i} onClick={() => onChange(item)}>
          {getLabel(item)} {item === selected && '(selected)'}
        </li>
      ))}
    </ul>
  );
}

// Usage with type inference
<Select
  items={users}
  selected={currentUser}
  getLabel={(u) => u.name}
  onChange={setCurrentUser}
/>
```

---

## JSX Spread Attributes

### Forwarding All Props

```tsx
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
}

function LabeledInput({ label, ...inputProps }: InputProps): JSX.Element {
  return (
    <label>
      {label}
      <input {...inputProps} />
    </label>
  );
}

// Usage: all standard input attributes pass through
<LabeledInput label="Email" type="email" required placeholder="you@example.com" />
```

### Override Order

Props spread FIRST can be overridden by explicit props that follow:

```tsx
// className from defaults is overridden by the explicit className
<input {...defaults} className="custom" />
```

**ALWAYS** spread generic props first, then place specific overrides after — this ensures explicit props take precedence.

---

## Boolean Attributes

```tsx
// These are equivalent:
<input disabled />
<input disabled={true} />

// To NOT disable:
<input disabled={false} />

// NEVER omit the value when you want false — omitting means true
```

---

## Reference Links

- [references/examples.md](references/examples.md) — Complete JSX patterns and code examples
- [references/anti-patterns.md](references/anti-patterns.md) — Common JSX mistakes with explanations

### Official Sources

- https://react.dev/learn/writing-markup-with-jsx
- https://react.dev/learn/javascript-in-jsx-with-curly-braces
- https://react.dev/learn/conditional-rendering
- https://react.dev/learn/rendering-lists
- https://react.dev/learn/passing-props-to-a-component
