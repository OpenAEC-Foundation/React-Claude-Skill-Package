# JSX Anti-Patterns

Common JSX mistakes with explanations and correct alternatives.

---

## AP-1: The `0 &&` Trap

**Severity**: HIGH — renders visible "0" text on screen

```tsx
// WRONG: When messageCount is 0, renders "0" as text
function Inbox({ messageCount }: { messageCount: number }): JSX.Element {
  return (
    <div>
      {messageCount && <span>{messageCount} new messages</span>}
    </div>
  );
}

// CORRECT: Explicitly convert to boolean
function Inbox({ messageCount }: { messageCount: number }): JSX.Element {
  return (
    <div>
      {messageCount > 0 && <span>{messageCount} new messages</span>}
    </div>
  );
}

// ALSO CORRECT: Use ternary
{messageCount ? <span>{messageCount} new messages</span> : null}

// ALSO CORRECT: Double negation
{!!messageCount && <span>{messageCount} new messages</span>}
```

**WHY**: JavaScript's `&&` returns the first falsy value. `0` is falsy but React renders it as text. `false`, `null`, and `undefined` render as nothing. The same trap applies to `""` (empty string) — it renders as an empty text node.

---

## AP-2: Array Index as Key for Dynamic Lists

**Severity**: HIGH — causes state corruption on reorder, insert, or delete

```tsx
interface Todo {
  id: string;
  text: string;
}

// WRONG: Index key causes input state to mix up on reorder
function TodoList({ todos }: { todos: Todo[] }): JSX.Element {
  return (
    <ul>
      {todos.map((todo, index) => (
        <li key={index}>
          <input defaultValue={todo.text} />
        </li>
      ))}
    </ul>
  );
}

// CORRECT: Stable unique key preserves component identity
function TodoList({ todos }: { todos: Todo[] }): JSX.Element {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input defaultValue={todo.text} />
        </li>
      ))}
    </ul>
  );
}
```

**WHY**: When you delete item at index 2, the item that was at index 3 now has key "2". React thinks this is the same component and reuses its DOM and state. The user sees the wrong input values in the wrong rows.

**ONLY acceptable for index keys**: Static, read-only lists that NEVER reorder, insert, or delete items.

---

## AP-3: Generating Keys During Render

**Severity**: HIGH — destroys all component state every render

```tsx
// WRONG: New key every render = new component instance every render
{items.map((item) => (
  <Item key={Math.random()} data={item} />
))}

// WRONG: Same problem with crypto
{items.map((item) => (
  <Item key={crypto.randomUUID()} data={item} />
))}

// CORRECT: Use a stable identifier from the data
{items.map((item) => (
  <Item key={item.id} data={item} />
))}
```

**WHY**: React uses keys to match components between renders. A new key means React destroys the old component and mounts a new one. This causes: lost input focus, lost form state, lost scroll position, unnecessary DOM operations, and broken animations.

---

## AP-4: Lowercase Custom Component Names

**Severity**: MEDIUM — component renders as unknown HTML element

```tsx
// WRONG: Lowercase treated as HTML element
function myCard({ title }: { title: string }): JSX.Element {
  return <div>{title}</div>;
}
// <myCard title="Test" /> renders as <mycard title="Test"></mycard> in the DOM

// CORRECT: PascalCase for custom components
function MyCard({ title }: { title: string }): JSX.Element {
  return <div>{title}</div>;
}
// <MyCard title="Test" /> correctly invokes the component
```

**WHY**: JSX compilation uses the case of the tag name to decide whether to create an HTML element (lowercase) or call a component function (PascalCase).

---

## AP-5: Using `class` Instead of `className`

**Severity**: LOW — React warns and auto-corrects, but code is technically wrong

```tsx
// WRONG: class is a reserved JavaScript keyword
<div class="container">Content</div>

// CORRECT: Use className in JSX
<div className="container">Content</div>

// WRONG: for is a reserved JavaScript keyword
<label for="email">Email</label>

// CORRECT: Use htmlFor in JSX
<label htmlFor="email">Email</label>
```

---

## AP-6: Statements Inside JSX Expressions

**Severity**: HIGH — syntax error, code does not compile

```tsx
// WRONG: if is a statement, not an expression
<div>
  {if (isLoggedIn) { return <Dashboard />; }}
</div>

// CORRECT: Use ternary (expression)
<div>
  {isLoggedIn ? <Dashboard /> : <LoginPrompt />}
</div>

// WRONG: for loop is a statement
<ul>
  {for (let i = 0; i < items.length; i++) { <li>{items[i]}</li> }}
</ul>

// CORRECT: Use map (expression)
<ul>
  {items.map((item) => <li key={item.id}>{item.name}</li>)}
</ul>
```

**WHY**: JSX curly braces accept JavaScript **expressions** (code that produces a value). Statements (`if`, `for`, `while`, `switch`, declarations) do NOT produce values and cause syntax errors.

---

## AP-7: Missing Key on Mapped Elements

**Severity**: MEDIUM — React warns, performance degrades, subtle state bugs possible

```tsx
// WRONG: No key prop
<ul>
  {items.map((item) => (
    <li>{item.name}</li>
  ))}
</ul>

// CORRECT: Key on the outermost returned element
<ul>
  {items.map((item) => (
    <li key={item.id}>{item.name}</li>
  ))}
</ul>

// WRONG: Key on an inner element instead of the outermost
{items.map((item) => (
  <div>
    <span key={item.id}>{item.name}</span>
  </div>
))}

// CORRECT: Key on the outermost element
{items.map((item) => (
  <div key={item.id}>
    <span>{item.name}</span>
  </div>
))}
```

---

## AP-8: Using Short Fragment When Key Is Needed

**Severity**: MEDIUM — syntax error or missing key warning

```tsx
// WRONG: Short fragment syntax cannot take key
{items.map((item) => (
  <key={item.id}>  {/* Syntax error */}
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </>
))}

// CORRECT: Use named Fragment import for keys
import { Fragment } from 'react';

{items.map((item) => (
  <Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </Fragment>
))}
```

---

## AP-9: Spread Props Without Type Safety

**Severity**: MEDIUM — passes unexpected or dangerous props to DOM elements

```tsx
// WRONG: Spreads all props including custom ones onto a DOM element
function Card(props: Record<string, unknown>): JSX.Element {
  return <div {...props}>{props.children}</div>;
  // If props contains { onCustomEvent: fn }, React warns about unknown DOM prop
}

// CORRECT: Destructure known props, spread the rest with proper typing
interface CardProps extends React.HTMLAttributes<HTMLDivElement> {
  title: string;
  variant?: 'default' | 'outlined';
}

function Card({ title, variant, children, ...divProps }: CardProps): JSX.Element {
  return (
    <div {...divProps} className={`card card--${variant ?? 'default'}`}>
      <h3>{title}</h3>
      {children}
    </div>
  );
}
```

**WHY**: Spreading unfiltered props onto DOM elements passes unrecognized attributes to the DOM, causing React warnings and potentially invalid HTML.

---

## AP-10: Mutating Props in JSX

**Severity**: HIGH — violates React's data flow, causes unpredictable behavior

```tsx
// WRONG: Mutating a prop array before rendering
function ItemList({ items }: { items: Item[] }): JSX.Element {
  items.sort((a, b) => a.name.localeCompare(b.name)); // Mutates parent's array!
  return (
    <ul>
      {items.map((item) => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}

// CORRECT: Create a sorted copy
function ItemList({ items }: { items: Item[] }): JSX.Element {
  const sorted = [...items].sort((a, b) => a.name.localeCompare(b.name));
  return (
    <ul>
      {sorted.map((item) => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}
```

**WHY**: Props are owned by the parent component. Mutating them changes the parent's data without the parent knowing, breaking unidirectional data flow and causing bugs that are extremely hard to trace.

---

## Decision Tree: Is My JSX Pattern Safe?

| Check | If Yes | If No |
|-------|--------|-------|
| Left side of `&&` can be `0` or `""` ? | Convert to boolean (`> 0`, `!!`, `Boolean()`) | Safe to use `&&` |
| List items can reorder/insert/delete? | NEVER use index as key | Index key is acceptable |
| Need key on a Fragment? | Use `<Fragment key={}>` | Use short syntax `<>` |
| Passing props to a DOM element? | Destructure and spread typed rest | Spread is safe for components |
| Sorting/filtering an array prop? | Create a copy first (`[...arr]`) | Direct use is fine for read-only |
| Custom component name? | MUST be PascalCase | Lowercase for HTML elements only |
