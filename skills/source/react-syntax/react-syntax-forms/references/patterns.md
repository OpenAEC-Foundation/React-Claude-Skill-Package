# react-syntax-forms — Patterns

Form architecture and composition patterns for React 18 and 19.

---

## Pattern: Reusable Form Field Component

Extract form fields into reusable components with consistent error display and accessibility.

```tsx
interface FieldProps {
  label: string;
  name: string;
  type?: string;
  value: string;
  error?: string;
  onChange: (e: React.ChangeEvent<HTMLInputElement>) => void;
  required?: boolean;
}

function FormField({ label, name, type = 'text', value, error, onChange, required }: FieldProps) {
  const errorId = `${name}-error`;
  return (
    <div>
      <label htmlFor={name}>{label}</label>
      <input
        id={name}
        name={name}
        type={type}
        value={value}
        onChange={onChange}
        required={required}
        aria-invalid={!!error}
        aria-describedby={error ? errorId : undefined}
      />
      {error && <span id={errorId} role="alert">{error}</span>}
    </div>
  );
}
```

---

## Pattern: Form State with useReducer

For forms with many fields and complex validation, `useReducer` provides cleaner state management than multiple `useState` calls.

```tsx
import { useReducer, type FormEvent } from 'react';

interface FormState {
  values: { email: string; password: string; confirmPassword: string };
  errors: Record<string, string>;
  isSubmitting: boolean;
}

type FormAction =
  | { type: 'SET_FIELD'; field: string; value: string }
  | { type: 'SET_ERRORS'; errors: Record<string, string> }
  | { type: 'SUBMIT_START' }
  | { type: 'SUBMIT_END' }
  | { type: 'RESET' };

const initialState: FormState = {
  values: { email: '', password: '', confirmPassword: '' },
  errors: {},
  isSubmitting: false,
};

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'SET_FIELD':
      return { ...state, values: { ...state.values, [action.field]: action.value } };
    case 'SET_ERRORS':
      return { ...state, errors: action.errors };
    case 'SUBMIT_START':
      return { ...state, isSubmitting: true, errors: {} };
    case 'SUBMIT_END':
      return { ...state, isSubmitting: false };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}

function RegistrationForm() {
  const [state, dispatch] = useReducer(formReducer, initialState);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    const errors = validate(state.values);
    if (Object.keys(errors).length > 0) {
      dispatch({ type: 'SET_ERRORS', errors });
      return;
    }
    dispatch({ type: 'SUBMIT_START' });
    await submitRegistration(state.values);
    dispatch({ type: 'SUBMIT_END' });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={state.values.email}
        onChange={(e) => dispatch({ type: 'SET_FIELD', field: 'email', value: e.target.value })}
        disabled={state.isSubmitting}
      />
      {state.errors.email && <span role="alert">{state.errors.email}</span>}
      <button type="submit" disabled={state.isSubmitting}>Register</button>
    </form>
  );
}
```

---

## Pattern: Controlled Form Reset

### React 18: Manual Reset

```tsx
function ResettableForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');

  function handleReset() {
    setName('');
    setEmail('');
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
      <button type="submit">Submit</button>
      <button type="button" onClick={handleReset}>Reset</button>
    </form>
  );
}
```

### React 18: Key-Based Reset

Force full re-mount by changing the `key` prop. ALWAYS prefer this for complex forms.

```tsx
function FormContainer() {
  const [formKey, setFormKey] = useState(0);

  function handleSubmitSuccess() {
    setFormKey((k) => k + 1); // forces re-mount with fresh state
  }

  return <MyForm key={formKey} onSuccess={handleSubmitSuccess} />;
}
```

### React 19: Automatic Reset with Form Actions

> **React 19+**

Uncontrolled form fields automatically reset after a successful form action. No manual reset needed.

```tsx
function AutoResetForm() {
  async function handleAction(formData: FormData) {
    await submitToServer(formData);
    // Form fields reset automatically on success
  }

  return (
    <form action={handleAction}>
      <input name="name" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Pattern: Debounced Search Input

For search-as-you-type where you need to delay API calls.

```tsx
import { useState, useEffect, useRef } from 'react';

function DebouncedSearch({ onSearch }: { onSearch: (query: string) => void }) {
  const [query, setQuery] = useState('');
  const timerRef = useRef<ReturnType<typeof setTimeout>>();

  useEffect(() => {
    clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => {
      if (query.length > 0) onSearch(query);
    }, 300);
    return () => clearTimeout(timerRef.current);
  }, [query, onSearch]);

  return (
    <input
      type="search"
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

---

## Pattern: Dynamic Form Fields

Add and remove fields dynamically. ALWAYS use stable IDs for keys, NEVER array indices.

```tsx
import { useState } from 'react';

interface FieldEntry {
  id: string;
  value: string;
}

let nextId = 0;

function DynamicForm() {
  const [fields, setFields] = useState<FieldEntry[]>([
    { id: String(nextId++), value: '' },
  ]);

  function addField() {
    setFields((prev) => [...prev, { id: String(nextId++), value: '' }]);
  }

  function removeField(id: string) {
    setFields((prev) => prev.filter((f) => f.id !== id));
  }

  function updateField(id: string, value: string) {
    setFields((prev) => prev.map((f) => (f.id === id ? { ...f, value } : f)));
  }

  return (
    <form>
      {fields.map((field) => (
        <div key={field.id}>
          <input
            value={field.value}
            onChange={(e) => updateField(field.id, e.target.value)}
          />
          <button type="button" onClick={() => removeField(field.id)}>Remove</button>
        </div>
      ))}
      <button type="button" onClick={addField}>Add Field</button>
    </form>
  );
}
```

---

## Pattern: Composing useFormStatus with useActionState

> **React 19+**

Combine both hooks for a complete form solution with error state and pending UI.

```tsx
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';

// Child component — reads parent form pending state
function FormContent({ error }: { error: string | null }) {
  const { pending } = useFormStatus();

  return (
    <>
      <input name="email" type="email" required disabled={pending} />
      <input name="password" type="password" required disabled={pending} />
      {error && <p role="alert">{error}</p>}
      <button type="submit" disabled={pending}>
        {pending ? 'Logging in...' : 'Log In'}
      </button>
    </>
  );
}

// Parent component — manages form action and state
function LoginForm() {
  const [state, formAction] = useActionState(
    async (prev: { error: string | null }, formData: FormData) => {
      const email = formData.get('email') as string;
      const password = formData.get('password') as string;
      const result = await loginAPI(email, password);
      if (!result.ok) return { error: result.message };
      return { error: null };
    },
    { error: null }
  );

  return (
    <form action={formAction}>
      <FormContent error={state.error} />
    </form>
  );
}
```

---

## Pattern: Progressive Enhancement with Server Functions

> **React 19+**

Forms that work before JavaScript loads when using a framework with Server Functions.

```tsx
// actions.ts
"use server";

export async function createTodo(prevState: { error: string | null }, formData: FormData) {
  const title = formData.get('title') as string;
  if (!title.trim()) return { error: 'Title is required' };
  await db.todos.create({ title });
  return { error: null };
}
```

```tsx
// TodoForm.tsx
"use client";
import { useActionState } from 'react';
import { createTodo } from './actions';

function TodoForm() {
  const [state, formAction, isPending] = useActionState(createTodo, { error: null });

  return (
    <form action={formAction}>
      <input name="title" required disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Adding...' : 'Add Todo'}
      </button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

This form submits as a standard HTML form if JavaScript has not loaded yet. When JavaScript loads, React enhances it with pending state and client-side error display.

---

## Pattern: Accessible Form Structure

ALWAYS follow these accessibility rules for forms:

1. **ALWAYS** associate labels with inputs via `htmlFor`/`id`
2. **ALWAYS** use `aria-invalid` on inputs with validation errors
3. **ALWAYS** use `aria-describedby` to link error messages to inputs
4. **ALWAYS** use `role="alert"` on error messages for screen reader announcements
5. **ALWAYS** use `<fieldset>` and `<legend>` for grouped controls (radio groups, checkbox groups)
6. **NEVER** rely solely on color to indicate errors

```tsx
<form>
  <fieldset>
    <legend>Contact Information</legend>

    <div>
      <label htmlFor="fullName">Full Name</label>
      <input
        id="fullName"
        name="fullName"
        aria-required="true"
        aria-invalid={!!errors.fullName}
        aria-describedby={errors.fullName ? 'fullName-error' : 'fullName-hint'}
      />
      <span id="fullName-hint">Enter your first and last name</span>
      {errors.fullName && (
        <span id="fullName-error" role="alert">{errors.fullName}</span>
      )}
    </div>
  </fieldset>

  <button type="submit">Submit</button>
</form>
```
