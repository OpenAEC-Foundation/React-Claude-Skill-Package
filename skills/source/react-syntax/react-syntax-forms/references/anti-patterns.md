# react-syntax-forms — Anti-Patterns

Common form mistakes and their fixes. Every anti-pattern includes WHY it is wrong and the correct alternative.

---

## Anti-Pattern 1: Mixing value and defaultValue

```tsx
// WRONG: React ignores defaultValue when value is present
<input value={name} defaultValue="fallback" onChange={handleChange} />

// CORRECT: Choose one approach
// Controlled:
<input value={name} onChange={handleChange} />
// Uncontrolled:
<input defaultValue="fallback" />
```

**WHY**: React treats an input as controlled when `value` is set. `defaultValue` is silently ignored. This creates confusion about which value the input actually displays.

---

## Anti-Pattern 2: Controlled Input Without onChange

```tsx
// WRONG: Input is frozen — user cannot type
<input value={name} />

// CORRECT: Always pair value with onChange
<input value={name} onChange={(e) => setName(e.target.value)} />

// Or use readOnly if intentional:
<input value={name} readOnly />
```

**WHY**: Setting `value` without `onChange` makes the input read-only. React logs a warning: "You provided a `value` prop to a form field without an `onChange` handler."

---

## Anti-Pattern 3: Setting value on File Inputs

```tsx
// WRONG: Throws an error — file inputs are read-only
<input type="file" value={fileName} onChange={handleChange} />

// CORRECT: File inputs are ALWAYS uncontrolled
<input type="file" ref={fileRef} onChange={(e) => handleFiles(e.target.files)} />
```

**WHY**: Browsers prohibit programmatic setting of file input values for security reasons. ALWAYS use refs or FormData to access file input values.

---

## Anti-Pattern 4: useFormStatus in the Form Component

```tsx
// WRONG: pending is ALWAYS false — reads status of PARENT form, not THIS form
function MyForm() {
  const { pending } = useFormStatus();

  return (
    <form action={submitAction}>
      <input name="email" />
      <button disabled={pending}>Submit</button>
    </form>
  );
}

// CORRECT: Extract button to a child component
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button type="submit" disabled={pending}>{pending ? 'Saving...' : 'Save'}</button>;
}

function MyForm() {
  return (
    <form action={submitAction}>
      <input name="email" />
      <SubmitButton />
    </form>
  );
}
```

**WHY**: `useFormStatus` reads the status of the nearest PARENT `<form>`. When called in the same component that renders the `<form>`, there is no parent form to read from.

---

## Anti-Pattern 5: Using Deprecated useFormState

```tsx
// WRONG: useFormState is deprecated in React 19
import { useFormState } from 'react-dom';
const [state, formAction] = useFormState(fn, initialState);

// CORRECT: Use useActionState from 'react'
import { useActionState } from 'react';
const [state, formAction, isPending] = useActionState(fn, initialState);
```

**WHY**: `useFormState` was renamed to `useActionState` in React 19 and moved from `react-dom` to `react`. The new API also returns `isPending` as the third element, eliminating the need for a separate `useFormStatus` call in many cases.

---

## Anti-Pattern 6: Missing preventDefault on Form Submit

```tsx
// WRONG: Page reloads on submit (default browser behavior)
function MyForm() {
  function handleSubmit() {
    console.log('submitted');
  }

  return <form onSubmit={handleSubmit}><button type="submit">Go</button></form>;
}

// CORRECT: Prevent default browser submission
function MyForm() {
  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    console.log('submitted');
  }

  return <form onSubmit={handleSubmit}><button type="submit">Go</button></form>;
}
```

**WHY**: Without `preventDefault()`, the browser performs a full page navigation. This resets all React state and triggers a page reload. Note: React 19 form actions (`<form action={fn}>`) handle this automatically.

---

## Anti-Pattern 7: Syncing Form State with useEffect

```tsx
// WRONG: Extra render cycle, stale UI for one frame
function ProfileForm({ userId }: { userId: string }) {
  const [name, setName] = useState('');

  useEffect(() => {
    fetchUser(userId).then((user) => setName(user.name));
  }, [userId]);

  return <input value={name} onChange={(e) => setName(e.target.value)} />;
}

// BETTER: Use key to force remount when userId changes
function ProfilePage({ userId }: { userId: string }) {
  return <ProfileForm key={userId} userId={userId} />;
}

function ProfileForm({ userId }: { userId: string }) {
  const [name, setName] = useState(''); // Fresh state on each userId
  // Fetch and set name...
  return <input value={name} onChange={(e) => setName(e.target.value)} />;
}
```

**WHY**: Using `useEffect` to reset form state on prop changes causes a flash of stale content. The `key` prop forces React to destroy and recreate the component with fresh state.

---

## Anti-Pattern 8: Index as Key for Dynamic Form Fields

```tsx
// WRONG: Input state gets mixed up when fields are reordered or removed
{fields.map((field, index) => (
  <input key={index} value={field.value} onChange={...} />
))}

// CORRECT: Use stable unique IDs
{fields.map((field) => (
  <input key={field.id} value={field.value} onChange={...} />
))}
```

**WHY**: When you remove field at index 1, all subsequent indices shift. React reuses DOM nodes by key — so field 2's DOM node now holds field 3's state. User input appears in the wrong fields.

---

## Anti-Pattern 9: Mutating State Directly in Form Handlers

```tsx
// WRONG: Mutating the state object directly
function handleChange(field: string, value: string) {
  form[field] = value; // Direct mutation — React does NOT detect this
  setForm(form);       // Same reference — no re-render
}

// CORRECT: Create a new object
function handleChange(field: string, value: string) {
  setForm((prev) => ({ ...prev, [field]: value }));
}
```

**WHY**: React uses reference equality (`Object.is`) to determine if state changed. Mutating the existing object and passing it to `setState` produces the same reference, so React skips the re-render.

---

## Anti-Pattern 10: Not Disabling Inputs During Submission

```tsx
// WRONG: User can submit multiple times or change values mid-submission
function MyForm() {
  const [state, formAction, isPending] = useActionState(submitFn, initialState);

  return (
    <form action={formAction}>
      <input name="email" />
      <button type="submit">Submit</button>
    </form>
  );
}

// CORRECT: Disable all interactive elements while pending
function MyForm() {
  const [state, formAction, isPending] = useActionState(submitFn, initialState);

  return (
    <form action={formAction}>
      <input name="email" disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

**WHY**: Without disabling, users can submit the form multiple times (causing duplicate requests) or change input values while the previous submission is still in progress.

---

## Anti-Pattern 11: Using textarea Children Instead of value

```tsx
// WRONG: React ignores children of <textarea>
<textarea>Initial content</textarea>

// CORRECT (controlled):
<textarea value={content} onChange={(e) => setContent(e.target.value)} />

// CORRECT (uncontrolled):
<textarea defaultValue="Initial content" />
```

**WHY**: In HTML, `<textarea>` uses children for its content. In React, ALWAYS use `value` (controlled) or `defaultValue` (uncontrolled). React ignores the children of `<textarea>`.

---

## Anti-Pattern 12: Reading Form Values from State After Uncontrolled Input

```tsx
// WRONG: State is never updated — input is uncontrolled
function MyForm() {
  const [name, setName] = useState('');

  return (
    <form onSubmit={() => console.log(name)}>
      <input defaultValue="" /> {/* No onChange — name stays '' */}
      <button type="submit">Submit</button>
    </form>
  );
}

// CORRECT Option A: Make it controlled
<input value={name} onChange={(e) => setName(e.target.value)} />

// CORRECT Option B: Use ref for uncontrolled
const nameRef = useRef<HTMLInputElement>(null);
<input defaultValue="" ref={nameRef} />
// Read: nameRef.current?.value

// CORRECT Option C: Use FormData
function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();
  const fd = new FormData(e.currentTarget);
  console.log(fd.get('name'));
}
```

**WHY**: Uncontrolled inputs do not update React state. The DOM holds the value, but your `useState` variable stays at its initial value. Use refs or FormData to read uncontrolled input values.

---

## Summary: Decision Rules

| Situation | ALWAYS | NEVER |
|-----------|--------|-------|
| Controlled input | Pair `value` with `onChange` | Set `value` without `onChange` |
| File input | Use uncontrolled with ref | Set `value` on file input |
| Form submission (React 18) | Call `e.preventDefault()` | Let the browser navigate |
| `useFormStatus` | Call from a child inside `<form>` | Call in the component rendering `<form>` |
| Dynamic field keys | Use stable unique IDs | Use array index as key |
| State updates | Create new objects/arrays | Mutate existing state |
| During submission | Disable inputs and buttons | Allow interaction while pending |
| `<textarea>` value | Use `value` or `defaultValue` prop | Use children content |
