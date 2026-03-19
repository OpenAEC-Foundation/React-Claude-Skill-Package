# react-syntax-forms — Examples

Complete form patterns with TypeScript. All examples verified against react.dev official documentation.

---

## Controlled Text Input with TypeScript

```tsx
import { useState, type ChangeEvent, type FormEvent } from 'react';

function TextInputForm() {
  const [name, setName] = useState<string>('');

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    setName(e.target.value);
  }

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    console.log('Submitted:', name);
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="name">Name</label>
      <input id="name" type="text" value={name} onChange={handleChange} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Controlled Number Input

```tsx
import { useState } from 'react';

function AgeInput() {
  const [age, setAge] = useState<number>(0);

  return (
    <input
      type="number"
      value={age}
      onChange={(e) => setAge(Number(e.target.value))}
      min={0}
      max={150}
    />
  );
}
```

**ALWAYS** use `Number(e.target.value)` to convert the string value. `e.target.value` is ALWAYS a string, even for number inputs.

---

## Controlled Checkbox

```tsx
import { useState } from 'react';

function CheckboxExample() {
  const [agreed, setAgreed] = useState<boolean>(false);

  return (
    <label>
      <input
        type="checkbox"
        checked={agreed}
        onChange={(e) => setAgreed(e.target.checked)}
      />
      I agree to the terms
    </label>
  );
}
```

**ALWAYS** use `checked` (not `value`) for checkboxes. Use `e.target.checked` (not `e.target.value`).

---

## Controlled Radio Group

```tsx
import { useState } from 'react';

type Size = 'small' | 'medium' | 'large';

function RadioGroupExample() {
  const [size, setSize] = useState<Size>('medium');

  return (
    <fieldset>
      <legend>Size</legend>
      {(['small', 'medium', 'large'] as const).map((option) => (
        <label key={option}>
          <input
            type="radio"
            name="size"
            value={option}
            checked={size === option}
            onChange={(e) => setSize(e.target.value as Size)}
          />
          {option}
        </label>
      ))}
    </fieldset>
  );
}
```

**ALWAYS** give all radios in a group the same `name` attribute. Use `checked` to control selection.

---

## Controlled Select

```tsx
import { useState } from 'react';

function SelectExample() {
  const [country, setCountry] = useState<string>('nl');

  return (
    <select value={country} onChange={(e) => setCountry(e.target.value)}>
      <option value="">-- Select --</option>
      <option value="nl">Netherlands</option>
      <option value="de">Germany</option>
      <option value="be">Belgium</option>
    </select>
  );
}
```

### Multiple Select

```tsx
import { useState, type ChangeEvent } from 'react';

function MultiSelectExample() {
  const [selected, setSelected] = useState<string[]>([]);

  function handleChange(e: ChangeEvent<HTMLSelectElement>) {
    const options = Array.from(e.target.selectedOptions, (opt) => opt.value);
    setSelected(options);
  }

  return (
    <select multiple value={selected} onChange={handleChange}>
      <option value="react">React</option>
      <option value="vue">Vue</option>
      <option value="angular">Angular</option>
    </select>
  );
}
```

---

## Controlled Textarea

```tsx
import { useState } from 'react';

function TextareaExample() {
  const [bio, setBio] = useState<string>('');

  return (
    <div>
      <textarea
        value={bio}
        onChange={(e) => setBio(e.target.value)}
        rows={4}
        maxLength={500}
      />
      <p>{bio.length}/500</p>
    </div>
  );
}
```

**NEVER** use `<textarea>content</textarea>` in React. ALWAYS use the `value` prop (controlled) or `defaultValue` (uncontrolled).

---

## Multi-Field Form with Single State Object

```tsx
import { useState, type ChangeEvent, type FormEvent } from 'react';

interface ContactForm {
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  message: string;
}

const initialState: ContactForm = {
  firstName: '', lastName: '', email: '', phone: '', message: '',
};

function ContactFormExample() {
  const [form, setForm] = useState<ContactForm>(initialState);

  function handleChange(e: ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) {
    const { name, value } = e.target;
    setForm((prev) => ({ ...prev, [name]: value }));
  }

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    console.log('Submitted:', form);
    setForm(initialState); // reset
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="firstName" value={form.firstName} onChange={handleChange} />
      <input name="lastName" value={form.lastName} onChange={handleChange} />
      <input name="email" type="email" value={form.email} onChange={handleChange} />
      <input name="phone" type="tel" value={form.phone} onChange={handleChange} />
      <textarea name="message" value={form.message} onChange={handleChange} />
      <button type="submit">Send</button>
    </form>
  );
}
```

---

## File Input with Multiple Files

```tsx
import { useRef, type FormEvent } from 'react';

function MultiFileUpload() {
  const fileRef = useRef<HTMLInputElement>(null);

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const files = fileRef.current?.files;
    if (!files || files.length === 0) return;

    const fileArray: File[] = Array.from(files);
    fileArray.forEach((file) => {
      console.log(`${file.name} — ${file.size} bytes — ${file.type}`);
    });

    // Upload via FormData
    const formData = new FormData();
    fileArray.forEach((file) => formData.append('files', file));
    // await fetch('/upload', { method: 'POST', body: formData });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input type="file" ref={fileRef} multiple accept="image/*,.pdf" />
      <button type="submit">Upload</button>
    </form>
  );
}
```

---

## Uncontrolled Form with useRef

```tsx
import { useRef, type FormEvent } from 'react';

function UncontrolledLoginForm() {
  const emailRef = useRef<HTMLInputElement>(null);
  const passwordRef = useRef<HTMLInputElement>(null);

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const email = emailRef.current?.value ?? '';
    const password = passwordRef.current?.value ?? '';
    console.log('Login:', { email, password });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={emailRef} type="email" defaultValue="" placeholder="Email" />
      <input ref={passwordRef} type="password" defaultValue="" placeholder="Password" />
      <button type="submit">Log In</button>
    </form>
  );
}
```

---

## React 19: Form Action with useActionState

> **React 19+**

```tsx
import { useActionState } from 'react';

interface SignupState {
  error: string | null;
  success: boolean;
}

async function signupAction(prev: SignupState, formData: FormData): Promise<SignupState> {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;

  if (password.length < 8) {
    return { error: 'Password must be at least 8 characters', success: false };
  }

  try {
    await fetch('/api/signup', {
      method: 'POST',
      body: JSON.stringify({ email, password }),
      headers: { 'Content-Type': 'application/json' },
    });
    return { error: null, success: true };
  } catch {
    return { error: 'Signup failed. Try again.', success: false };
  }
}

function SignupForm() {
  const [state, formAction, isPending] = useActionState(signupAction, {
    error: null,
    success: false,
  });

  return (
    <form action={formAction}>
      <input name="email" type="email" required disabled={isPending} />
      <input name="password" type="password" required disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Signing up...' : 'Sign Up'}
      </button>
      {state.error && <p role="alert">{state.error}</p>}
      {state.success && <p>Account created!</p>}
    </form>
  );
}
```

---

## React 19: useFormStatus in Child Component

> **React 19+**

```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton({ label }: { label: string }) {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Processing...' : label}
    </button>
  );
}

function FormWithStatus() {
  async function handleAction(formData: FormData) {
    await new Promise((resolve) => setTimeout(resolve, 2000));
    console.log('Submitted:', formData.get('name'));
  }

  return (
    <form action={handleAction}>
      <input name="name" required />
      <SubmitButton label="Save" />
    </form>
  );
}
```

---

## React 19: useOptimistic with Form

> **React 19+**

```tsx
import { useOptimistic, startTransition } from 'react';

interface Message {
  id: string;
  text: string;
  pending?: boolean;
}

function MessageList({
  messages,
  sendMessage,
}: {
  messages: Message[];
  sendMessage: (text: string) => Promise<void>;
}) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (current: Message[], newText: string): Message[] => [
      ...current,
      { id: 'optimistic-' + Date.now(), text: newText, pending: true },
    ]
  );

  async function handleSubmit(formData: FormData) {
    const text = formData.get('text') as string;
    addOptimistic(text);
    await sendMessage(text);
  }

  return (
    <div>
      <ul>
        {optimisticMessages.map((msg) => (
          <li key={msg.id} style={{ opacity: msg.pending ? 0.5 : 1 }}>
            {msg.text}
            {msg.pending && ' (sending...)'}
          </li>
        ))}
      </ul>
      <form action={handleSubmit}>
        <input name="text" required />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

---

## Validation with Error Display and Accessibility

```tsx
import { useState, type FormEvent } from 'react';

interface FormErrors {
  email?: string;
  password?: string;
}

function ValidatedLoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<FormErrors>({});

  function validate(): FormErrors {
    const newErrors: FormErrors = {};
    if (!email) newErrors.email = 'Email is required';
    else if (!email.includes('@')) newErrors.email = 'Invalid email format';
    if (!password) newErrors.password = 'Password is required';
    else if (password.length < 8) newErrors.password = 'Minimum 8 characters';
    return newErrors;
  }

  function handleSubmit(e: FormEvent) {
    e.preventDefault();
    const validationErrors = validate();
    setErrors(validationErrors);
    if (Object.keys(validationErrors).length === 0) {
      // submit form
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : undefined}
        />
        {errors.email && (
          <span id="email-error" role="alert">{errors.email}</span>
        )}
      </div>
      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          aria-invalid={!!errors.password}
          aria-describedby={errors.password ? 'password-error' : undefined}
        />
        {errors.password && (
          <span id="password-error" role="alert">{errors.password}</span>
        )}
      </div>
      <button type="submit">Log In</button>
    </form>
  );
}
```

---

## FormData API Usage (React 18 + 19)

```tsx
function FormDataExample() {
  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const fd = new FormData(e.currentTarget);

    // Get individual values
    const name = fd.get('name') as string;
    const email = fd.get('email') as string;

    // Get all values for multi-select or repeated fields
    const tags = fd.getAll('tags') as string[];

    // Convert to plain object
    const data = Object.fromEntries(fd.entries());

    console.log({ name, email, tags, data });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" />
      <input name="email" type="email" />
      <select name="tags" multiple>
        <option value="react">React</option>
        <option value="typescript">TypeScript</option>
      </select>
      <button type="submit">Submit</button>
    </form>
  );
}
```
