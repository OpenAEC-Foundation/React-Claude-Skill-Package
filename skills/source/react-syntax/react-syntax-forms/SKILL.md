---
name: react-syntax-forms
description: >
  Use when building forms, handling form submission, implementing form validation,
  or using React 19 form actions. Prevents the common mistake of mixing controlled
  and uncontrolled patterns or missing form accessibility. Covers controlled vs
  uncontrolled inputs, React 19 form actions, useFormStatus, useActionState,
  useOptimistic, file inputs, validation patterns.
  Keywords: form, controlled, uncontrolled, useFormStatus, useActionState, action, file upload, form validation, React Hook Form, Formik, Zod, form submit, input handling..
license: MIT
compatibility: "Designed for Claude Code. Requires React 18.x or 19.x with TypeScript."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# react-syntax-forms

## Quick Reference

### Form Approach Overview

| Approach | React Version | State Owner | Best For |
|----------|--------------|-------------|----------|
| Controlled inputs | 18 + 19 | React (`value` + `onChange`) | Real-time validation, conditional fields, formatted input |
| Uncontrolled inputs | 18 + 19 | DOM (`defaultValue` + `ref`) | File inputs, third-party integrations, simple forms |
| Form actions | **19+** | FormData (`<form action={fn}>`) | Async submissions, progressive enhancement, Server Functions |

### Input Type Quick Reference

| Input Type | Controlled Prop | Event Value Access | TypeScript Type |
|------------|----------------|-------------------|-----------------|
| `text` | `value` | `e.target.value` | `string` |
| `number` | `value` | `Number(e.target.value)` | `number` |
| `checkbox` | `checked` | `e.target.checked` | `boolean` |
| `radio` | `checked` | `e.target.value` | `string` |
| `select` | `value` | `e.target.value` | `string` |
| `textarea` | `value` | `e.target.value` | `string` |
| `file` | ALWAYS uncontrolled | `e.target.files` | `FileList \| null` |

### Critical Warnings

**NEVER** mix `value` and `defaultValue` on the same element. Choose controlled OR uncontrolled.

**NEVER** set `value` on a file input. File inputs are ALWAYS uncontrolled because their value is read-only for security reasons.

**NEVER** call `useFormStatus` in the same component that renders `<form>`. It MUST be called from a child component rendered inside the `<form>`.

**NEVER** use the deprecated `useFormState` from `react-dom`. ALWAYS use `useActionState` from `react` instead.

**ALWAYS** call `e.preventDefault()` in `onSubmit` handlers (React 18 pattern). Form actions (React 19) handle this automatically.

---

## Decision Tree: Which Form Approach?

```
Need real-time validation or formatted input?
  YES --> Controlled inputs (value + onChange)
  NO -->
    Using React 19 with async submission?
      YES --> Form actions (<form action={fn}>)
      NO -->
        Need file upload or third-party widget?
          YES --> Uncontrolled inputs (defaultValue + ref)
          NO -->
            Simple form with few fields?
              YES --> Uncontrolled inputs
              NO --> Controlled inputs
```

### When to Use Each

| Use Case | Approach | Why |
|----------|----------|-----|
| Search-as-you-type | Controlled | Need value on every keystroke |
| Credit card formatting | Controlled | Must transform input in real time |
| File upload | Uncontrolled | File inputs cannot be controlled |
| Server action form | Form actions (React 19) | Built-in pending state, progressive enhancement |
| Simple contact form | Uncontrolled or form actions | No per-keystroke logic needed |
| Multi-step wizard | Controlled | Must track and validate across steps |

---

## Controlled Inputs

React state is the single source of truth. Every keystroke triggers a re-render.

```tsx
function ControlledForm() {
  const [name, setName] = useState<string>('');
  const [age, setAge] = useState<number>(0);
  const [agreed, setAgreed] = useState<boolean>(false);
  const [role, setRole] = useState<string>('developer');

  return (
    <form onSubmit={(e) => { e.preventDefault(); /* submit */ }}>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input type="number" value={age} onChange={(e) => setAge(Number(e.target.value))} />
      <input type="checkbox" checked={agreed} onChange={(e) => setAgreed(e.target.checked)} />
      <select value={role} onChange={(e) => setRole(e.target.value)}>
        <option value="developer">Developer</option>
        <option value="designer">Designer</option>
      </select>
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Multi-Field State: Object vs Individual

**Individual `useState` calls** — ALWAYS use for forms with 1-4 unrelated fields:

```tsx
const [email, setEmail] = useState('');
const [password, setPassword] = useState('');
```

**Single state object** — ALWAYS use for forms with 5+ fields or related fields:

```tsx
interface FormData {
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  address: string;
}

const [form, setForm] = useState<FormData>({
  firstName: '', lastName: '', email: '', phone: '', address: ''
});

const updateField = (field: keyof FormData, value: string) => {
  setForm((prev) => ({ ...prev, [field]: value }));
};
```

---

## Uncontrolled Inputs

The DOM manages the value. Use `defaultValue` for initial values and refs for reading.

```tsx
function UncontrolledForm() {
  const nameRef = useRef<HTMLInputElement>(null);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const name = nameRef.current?.value;
    // submit name
  }

  return (
    <form onSubmit={handleSubmit}>
      <input defaultValue="initial text" ref={nameRef} />
      <textarea defaultValue="initial bio" />
      <select defaultValue="developer">
        <option value="developer">Developer</option>
        <option value="designer">Designer</option>
      </select>
      <button type="submit">Submit</button>
    </form>
  );
}
```

### File Inputs

File inputs are ALWAYS uncontrolled. NEVER attempt to set `value` on a file input.

```tsx
function FileUpload() {
  const fileRef = useRef<HTMLInputElement>(null);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const files: FileList | null = fileRef.current?.files ?? null;
    if (files && files.length > 0) {
      const file: File = files[0];
      // process file
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input type="file" ref={fileRef} accept=".pdf,.jpg,.png" />
      <input type="file" ref={fileRef} multiple /> {/* multiple files */}
      <button type="submit">Upload</button>
    </form>
  );
}
```

---

## Form Submission with FormData

Works in both React 18 and 19. Uses the browser's native `FormData` API.

```tsx
function FormDataExample() {
  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const name = formData.get('name') as string;
    const email = formData.get('email') as string;
    // submit { name, email }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" required />
      <input name="email" type="email" required />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## React 19 Form Actions

> **React 19+**

Pass an async function directly to `<form action>`. React handles submission inside a Transition.

```tsx
function SearchForm() {
  async function handleSearch(formData: FormData) {
    const query = formData.get('query') as string;
    await searchAPI(query);
  }

  return (
    <form action={handleSearch}>
      <input name="query" />
      <button type="submit">Search</button>
    </form>
  );
}
```

**Key behaviors:**
- HTTP method is ALWAYS POST when using a function action
- Uncontrolled fields automatically reset after successful submission
- Forms work without JavaScript when using Server Functions (progressive enhancement)

### Multiple Submit Actions

> **React 19+**

```tsx
function Editor() {
  async function publish(formData: FormData) { /* publish */ }
  async function saveDraft(formData: FormData) { /* save */ }

  return (
    <form action={publish}>
      <textarea name="content" />
      <button type="submit">Publish</button>
      <button formAction={saveDraft}>Save Draft</button>
    </form>
  );
}
```

---

## useFormStatus

> **React 19+**

Reads the parent form's submission state. Import from `react-dom`.

```tsx
import { useFormStatus } from 'react-dom';

// MUST be a separate component rendered INSIDE <form>
function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

function MyForm() {
  return (
    <form action={submitAction}>
      <input name="email" />
      <SubmitButton /> {/* Reads parent form state */}
    </form>
  );
}
```

| Property | Type | Description |
|----------|------|-------------|
| `pending` | `boolean` | `true` while parent form is submitting |
| `data` | `FormData \| null` | Form data being submitted; `null` when idle |
| `method` | `'get' \| 'post'` | HTTP method of parent form |
| `action` | `function \| null` | Reference to form's action function |

---

## useActionState

> **React 19+**

Manages action state, error handling, and pending state. Import from `react`.

```tsx
import { useActionState } from 'react';

interface FormState {
  error: string | null;
  success: boolean;
}

async function submitForm(prev: FormState, formData: FormData): Promise<FormState> {
  const email = formData.get('email') as string;
  if (!email.includes('@')) return { error: 'Invalid email', success: false };
  await saveEmail(email);
  return { error: null, success: true };
}

function EmailForm() {
  const [state, formAction, isPending] = useActionState(submitForm, {
    error: null, success: false,
  });

  return (
    <form action={formAction}>
      <input name="email" disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
      {state.error && <p className="error">{state.error}</p>}
      {state.success && <p className="success">Saved!</p>}
    </form>
  );
}
```

---

## useOptimistic

> **React 19+**

Provides instant UI feedback during async operations. Automatically reverts on failure.

```tsx
import { useOptimistic } from 'react';

interface Todo {
  id: string;
  text: string;
  pending?: boolean;
}

function TodoList({ todos, addTodo }: { todos: Todo[]; addTodo: (text: string) => Promise<void> }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (current: Todo[], newText: string) => [
      ...current,
      { id: 'temp', text: newText, pending: true },
    ]
  );

  async function handleSubmit(formData: FormData) {
    const text = formData.get('text') as string;
    addOptimistic(text);
    await addTodo(text);
  }

  return (
    <div>
      <ul>
        {optimisticTodos.map((t) => (
          <li key={t.id} style={{ opacity: t.pending ? 0.5 : 1 }}>{t.text}</li>
        ))}
      </ul>
      <form action={handleSubmit}>
        <input name="text" />
        <button type="submit">Add</button>
      </form>
    </div>
  );
}
```

---

## Validation Patterns

### HTML5 Built-in Validation

ALWAYS use native validation attributes as the first line of defense:

```tsx
<input type="email" required />
<input type="text" minLength={3} maxLength={50} required />
<input type="number" min={0} max={100} step={1} />
<input type="text" pattern="[A-Za-z]{3,}" title="At least 3 letters" />
```

### Custom Validation with Controlled Inputs

```tsx
function ValidatedForm() {
  const [email, setEmail] = useState('');
  const [errors, setErrors] = useState<Record<string, string>>({});

  function validate(): boolean {
    const newErrors: Record<string, string> = {};
    if (!email.includes('@')) newErrors.email = 'Invalid email address';
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  }

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (validate()) { /* submit */ }
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        aria-invalid={!!errors.email}
        aria-describedby={errors.email ? 'email-error' : undefined}
      />
      {errors.email && <span id="email-error" role="alert">{errors.email}</span>}
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Reference Links

- [references/examples.md](references/examples.md) -- Complete form patterns with TypeScript
- [references/patterns.md](references/patterns.md) -- Form architecture and composition patterns
- [references/anti-patterns.md](references/anti-patterns.md) -- Common form mistakes and fixes

### Official Sources

- https://react.dev/reference/react-dom/components/form
- https://react.dev/reference/react-dom/components/input
- https://react.dev/reference/react-dom/components/select
- https://react.dev/reference/react-dom/components/textarea
- https://react.dev/reference/react/useActionState
- https://react.dev/reference/react/useOptimistic
- https://react.dev/reference/react-dom/hooks/useFormStatus
