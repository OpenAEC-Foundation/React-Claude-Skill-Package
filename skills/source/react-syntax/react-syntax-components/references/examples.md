# Component Typing Examples

> Reference file for `react-syntax-components`. Complete TypeScript examples for component patterns.

---

## Basic Function Component

```tsx
interface GreetingProps {
  readonly name: string;
  readonly greeting?: string;
}

function Greeting({ name, greeting = 'Hello' }: GreetingProps): React.ReactElement {
  return <h1>{greeting}, {name}!</h1>;
}
```

---

## Component with Event Handlers

```tsx
interface ButtonProps {
  readonly label: string;
  readonly variant?: 'primary' | 'secondary' | 'danger';
  readonly disabled?: boolean;
  readonly onClick: (event: React.MouseEvent<HTMLButtonElement>) => void;
}

function Button({
  label,
  variant = 'primary',
  disabled = false,
  onClick,
}: ButtonProps): React.ReactElement {
  return (
    <button
      className={`btn btn-${variant}`}
      disabled={disabled}
      onClick={onClick}
    >
      {label}
    </button>
  );
}
```

---

## Component with Children

```tsx
interface LayoutProps {
  readonly sidebar: React.ReactNode;
  readonly children: React.ReactNode;
}

function Layout({ sidebar, children }: LayoutProps): React.ReactElement {
  return (
    <div className="layout">
      <aside className="sidebar">{sidebar}</aside>
      <main className="content">{children}</main>
    </div>
  );
}

// Usage
<Layout sidebar={<NavMenu />}>
  <h1>Page Title</h1>
  <p>Page content here.</p>
</Layout>
```

---

## Generic Data Table Component

```tsx
interface Column<T> {
  readonly key: keyof T & string;
  readonly header: string;
  readonly render?: (value: T[keyof T], row: T) => React.ReactNode;
}

interface DataTableProps<T> {
  readonly data: readonly T[];
  readonly columns: readonly Column<T>[];
  readonly keyExtractor: (row: T) => string;
  readonly onRowClick?: (row: T) => void;
}

function DataTable<T>({
  data,
  columns,
  keyExtractor,
  onRowClick,
}: DataTableProps<T>): React.ReactElement {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={col.key}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((row) => (
          <tr key={keyExtractor(row)} onClick={() => onRowClick?.(row)}>
            {columns.map((col) => (
              <td key={col.key}>
                {col.render ? col.render(row[col.key], row) : String(row[col.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Usage with type inference
interface User {
  id: string;
  name: string;
  email: string;
  active: boolean;
}

const columns: Column<User>[] = [
  { key: 'name', header: 'Name' },
  { key: 'email', header: 'Email' },
  { key: 'active', header: 'Status', render: (val) => (val ? 'Active' : 'Inactive') },
];

<DataTable<User>
  data={users}
  columns={columns}
  keyExtractor={(u) => u.id}
  onRowClick={(user) => navigate(`/users/${user.id}`)}
/>
```

---

## Discriminated Union Props

```tsx
// Use discriminated unions when props depend on a variant
type AlertProps =
  | { readonly variant: 'success'; readonly message: string }
  | { readonly variant: 'error'; readonly message: string; readonly retry: () => void }
  | { readonly variant: 'loading' };

function Alert(props: AlertProps): React.ReactElement {
  switch (props.variant) {
    case 'success':
      return <div className="alert-success">{props.message}</div>;
    case 'error':
      return (
        <div className="alert-error">
          {props.message}
          <button onClick={props.retry}>Retry</button>
        </div>
      );
    case 'loading':
      return <div className="alert-loading">Loading...</div>;
  }
}

// TypeScript enforces correct props per variant
<Alert variant="error" message="Failed" retry={() => refetch()} />
// <Alert variant="error" message="Failed" />  // ERROR: retry is required
```

---

## Component with Ref (React 19)

```tsx
interface TextAreaProps {
  readonly placeholder?: string;
  readonly ref?: React.Ref<HTMLTextAreaElement>;
  readonly onChange?: (value: string) => void;
}

function TextArea({ placeholder, ref, onChange }: TextAreaProps): React.ReactElement {
  return (
    <textarea
      ref={ref}
      placeholder={placeholder}
      onChange={(e) => onChange?.(e.target.value)}
    />
  );
}

// Parent
function Form(): React.ReactElement {
  const textAreaRef = React.useRef<HTMLTextAreaElement>(null);

  const handleSubmit = () => {
    const value = textAreaRef.current?.value ?? '';
    console.log('Submitted:', value);
  };

  return (
    <form onSubmit={(e) => { e.preventDefault(); handleSubmit(); }}>
      <TextArea ref={textAreaRef} placeholder="Enter text..." />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Component with Ref (React 18 -- forwardRef)

```tsx
interface TextAreaProps {
  readonly placeholder?: string;
  readonly onChange?: (value: string) => void;
}

const TextArea = React.forwardRef<HTMLTextAreaElement, TextAreaProps>(
  function TextArea({ placeholder, onChange }, ref): React.ReactElement {
    return (
      <textarea
        ref={ref}
        placeholder={placeholder}
        onChange={(e) => onChange?.(e.target.value)}
      />
    );
  }
);
```

---

## Memo with Generic Component

```tsx
interface SelectOptionProps<T> {
  readonly value: T;
  readonly label: string;
  readonly isSelected: boolean;
  readonly onSelect: (value: T) => void;
}

// React.memo with generics requires a type assertion
const SelectOption = React.memo(function SelectOption<T>({
  value,
  label,
  isSelected,
  onSelect,
}: SelectOptionProps<T>): React.ReactElement {
  return (
    <li
      className={isSelected ? 'selected' : ''}
      onClick={() => onSelect(value)}
    >
      {label}
    </li>
  );
}) as <T>(props: SelectOptionProps<T>) => React.ReactElement;
```

---

## Lazy-Loaded Route Components

```tsx
// ALWAYS at module top level
const Dashboard = React.lazy(() => import('./pages/Dashboard'));
const Settings = React.lazy(() => import('./pages/Settings'));
const Profile = React.lazy(() => import('./pages/Profile'));

function AppRoutes(): React.ReactElement {
  return (
    <React.Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </React.Suspense>
  );
}

function PageSkeleton(): React.ReactElement {
  return (
    <div className="skeleton">
      <div className="skeleton-header" />
      <div className="skeleton-body" />
    </div>
  );
}
```

---

## Portal: Tooltip Component

```tsx
import { createPortal } from 'react-dom';

interface TooltipProps {
  readonly text: string;
  readonly targetRect: DOMRect | null;
}

function Tooltip({ text, targetRect }: TooltipProps): React.ReactElement | null {
  if (!targetRect) return null;

  const style: React.CSSProperties = {
    position: 'fixed',
    top: targetRect.bottom + 8,
    left: targetRect.left + targetRect.width / 2,
    transform: 'translateX(-50%)',
  };

  return createPortal(
    <div className="tooltip" style={style}>{text}</div>,
    document.body
  );
}

// Usage with hover state
function IconButton({ icon, tooltip }: { icon: string; tooltip: string }): React.ReactElement {
  const [targetRect, setTargetRect] = React.useState<DOMRect | null>(null);
  const buttonRef = React.useRef<HTMLButtonElement>(null);

  return (
    <>
      <button
        ref={buttonRef}
        onPointerEnter={() => setTargetRect(buttonRef.current?.getBoundingClientRect() ?? null)}
        onPointerLeave={() => setTargetRect(null)}
      >
        {icon}
      </button>
      <Tooltip text={tooltip} targetRect={targetRect} />
    </>
  );
}
```
