# Event Handling Anti-Patterns

Common mistakes in React event handling with corrections.

---

## AP-001: Using `any` for Event Types

### Wrong

```tsx
const handleClick = (e: any) => {
  console.log(e.currentTarget.value); // No type safety
};
```

### Correct

```tsx
const handleClick = (e: React.MouseEvent<HTMLButtonElement>): void => {
  console.log(e.currentTarget.disabled); // Full autocomplete and type checking
};
```

**Why**: Using `any` defeats TypeScript's purpose. You lose autocomplete, miss typos, and allow impossible property access (e.g., `.value` on a button).

---

## AP-002: Using onKeyPress (Deprecated)

### Wrong

```tsx
<input onKeyPress={(e) => {
  if (e.key === "Escape") { /* NEVER fires -- Escape is not a character key */ }
}} />
```

### Correct

```tsx
<input onKeyDown={(e: React.KeyboardEvent<HTMLInputElement>) => {
  if (e.key === "Escape") { clearInput(); }
}} />
```

**Why**: `onKeyPress` is deprecated, does not fire for non-character keys (Escape, Arrow, Tab, etc.), and will be removed in a future browser version.

---

## AP-003: Using keyCode Instead of key

### Wrong

```tsx
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>): void => {
  if (e.keyCode === 13) { submit(); } // magic number, deprecated
};
```

### Correct

```tsx
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>): void => {
  if (e.key === "Enter") { submit(); } // readable, standard
};
```

**Why**: `keyCode` is deprecated, requires memorizing numeric codes, and varies across keyboard layouts. `e.key` returns a human-readable string.

---

## AP-004: Forgetting preventDefault on Form Submit

### Wrong

```tsx
const handleSubmit = (e: React.FormEvent<HTMLFormElement>): void => {
  // Missing e.preventDefault() -- causes full page reload
  submitData();
};
```

### Correct

```tsx
const handleSubmit = (e: React.FormEvent<HTMLFormElement>): void => {
  e.preventDefault();
  submitData();
};
```

**Why**: Without `preventDefault()`, the browser performs a full form submission (GET/POST request + page reload), destroying React state.

---

## AP-005: Missing Element Generic Parameter

### Wrong

```tsx
const handleChange = (e: React.ChangeEvent): void => {
  console.log(e.currentTarget.value); // Error: Property 'value' does not exist on type 'HTMLElement'
};
```

### Correct

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
  console.log(e.currentTarget.value); // OK: HTMLInputElement has .value
};
```

**Why**: Without the generic parameter, `currentTarget` is typed as `HTMLElement`, which does not have `.value`, `.checked`, `.files`, or other element-specific properties.

---

## AP-006: Reading currentTarget in Async Code

### Wrong

```tsx
const handleClick = async (e: React.MouseEvent<HTMLButtonElement>): Promise<void> => {
  await someAsyncOperation();
  console.log(e.currentTarget.id); // currentTarget is null after handler returns
};
```

### Correct

```tsx
const handleClick = async (e: React.MouseEvent<HTMLButtonElement>): Promise<void> => {
  const buttonId = e.currentTarget.id; // Extract value synchronously
  await someAsyncOperation();
  console.log(buttonId); // Use the extracted value
};
```

**Why**: React sets `currentTarget` to `null` after the synchronous event handler completes. Any access after an `await` or in a `setTimeout` callback reads `null`.

---

## AP-007: Overusing stopPropagation

### Wrong

```tsx
const handleClick = (e: React.MouseEvent<HTMLButtonElement>): void => {
  e.stopPropagation(); // "Just in case" -- breaks everything above
  doSomething();
};
```

### Correct

```tsx
// Only use stopPropagation when a parent handler would cause incorrect behavior
const handleInnerClick = (e: React.MouseEvent<HTMLButtonElement>): void => {
  e.stopPropagation(); // Prevents parent's onClick from closing the modal
  selectItem();
};
```

**Why**: `stopPropagation()` prevents ALL parent handlers from receiving the event. This breaks analytics event listeners, modal backdrop close handlers, dropdown close-on-outside-click patterns, and any other parent-level event logic. ONLY use it when you have identified a specific parent handler that must not fire.

---

## AP-008: Using .bind() in Function Components

### Wrong

```tsx
const Component = ({ items }: Props): JSX.Element => (
  <ul>
    {items.map((item) => (
      <li key={item.id}>
        <button onClick={handleDelete.bind(null, item.id)}>Delete</button>
      </li>
    ))}
  </ul>
);
```

### Correct

```tsx
const Component = ({ items }: Props): JSX.Element => (
  <ul>
    {items.map((item) => (
      <li key={item.id}>
        <button onClick={() => handleDelete(item.id)}>Delete</button>
      </li>
    ))}
  </ul>
);
```

**Why**: `.bind()` creates a new function every render (same as arrow functions) but is harder to read and loses TypeScript type inference for the bound arguments. In function components, arrow functions are the standard pattern.

---

## AP-009: Calling the Handler Instead of Passing It

### Wrong

```tsx
<button onClick={handleClick()}>Click</button>
// handleClick() is CALLED during render, not on click
```

### Correct

```tsx
<button onClick={handleClick}>Click</button>
// or with arguments:
<button onClick={() => handleClick(id)}>Click</button>
```

**Why**: `onClick={handleClick()}` executes `handleClick` immediately during render and passes its return value (usually `undefined`) as the event handler. This causes the handler to fire on every render, not on click.

---

## AP-010: Wrong Event Type for onChange on Input

### Wrong

```tsx
const handleChange = (e: React.InputEvent<HTMLInputElement>): void => {
  // React.InputEvent does not exist in React's type definitions
};
```

### Correct

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
  setValue(e.currentTarget.value);
};
```

**Why**: React does not expose `InputEvent` as a synthetic event type. The `onChange` handler in React uses `React.ChangeEvent`, which fires on every keystroke (unlike the native `change` event that fires on blur).

---

## AP-011: Using e.target Instead of e.currentTarget for Value Access

### Wrong

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
  setValue((e.target as HTMLInputElement).value); // requires type assertion
};
```

### Correct

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
  setValue(e.currentTarget.value); // already typed as HTMLInputElement
};
```

**Why**: `e.target` is typed as `EventTarget` (could be any child element). `e.currentTarget` is typed with the generic parameter you specified, giving correct types without assertions. Use `e.target` only when you specifically need the element that triggered the event (which may be a child).

---

## AP-012: Forgetting preventDefault on DragOver

### Wrong

```tsx
<div
  onDragOver={(e: React.DragEvent<HTMLDivElement>) => {
    setIsDragOver(true);
    // Missing e.preventDefault() -- drop will NOT work
  }}
  onDrop={(e: React.DragEvent<HTMLDivElement>) => {
    handleFiles(e.dataTransfer.files); // onDrop never fires
  }}
/>
```

### Correct

```tsx
<div
  onDragOver={(e: React.DragEvent<HTMLDivElement>) => {
    e.preventDefault(); // REQUIRED to make the element a valid drop target
    setIsDragOver(true);
  }}
  onDrop={(e: React.DragEvent<HTMLDivElement>) => {
    e.preventDefault();
    handleFiles(e.dataTransfer.files);
  }}
/>
```

**Why**: The browser default behavior for `dragover` is to reject the drop. You MUST call `preventDefault()` on `dragover` to signal that the element accepts drops. Without it, the `onDrop` handler will never fire.
